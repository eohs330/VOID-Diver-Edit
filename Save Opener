#!/usr/bin/env python3
# -*- coding: utf-8 -*-

"""
VOID DIVER Demo save <-> JSON editor v1.1

지원
- player.money           = f6.f5
- player.otherworld_coin = f6.f10
- storage                = top-level repeated f9
- inventory              = top-level repeated f10
- equipment              = top-level repeated f8 (조회용, 원본 보존)

설계 원칙
- .proto 스키마가 없으므로 의미가 확정되지 않은 필드는 원본 protobuf 바이트를 보존합니다.
- JSON을 수정하지 않고 다시 SAV로 만들면 원본과 byte-for-byte 동일하도록 설계했습니다.
- 기존 storage/inventory 레코드는 원본 레코드를 기반으로 slot/item_id/quantity만 패치합니다.
- JSON의 _... 필드는 내부 보존용입니다. 가능하면 삭제하지 마세요.

사용
    python void_diver_save_json_editor.py export PlayerContext_0.sav
    python void_diver_save_json_editor.py import PlayerContext_0.json

자동 모드
    python void_diver_save_json_editor.py PlayerContext_0.sav
    python void_diver_save_json_editor.py PlayerContext_0.json

검사
    python void_diver_save_json_editor.py check PlayerContext_0.sav

인수 없이 실행하면 메뉴가 나옵니다.
"""

from __future__ import annotations

import argparse
import base64
import hashlib
import json
import sys
from pathlib import Path
from typing import Any


FORMAT_NAME = "VOID_DIVER_DEMO_SAVE_JSON_V1_1"


# ===========================================================================
# Protobuf wire-format
# ===========================================================================

def read_varint(data: bytes, offset: int) -> tuple[int, int]:
    value = 0
    shift = 0
    start = offset

    while offset < len(data):
        b = data[offset]
        offset += 1
        value |= (b & 0x7F) << shift

        if not (b & 0x80):
            return value, offset

        shift += 7
        if shift > 70:
            break

    raise ValueError(f"Invalid varint at offset 0x{start:X}")


def encode_varint(value: int) -> bytes:
    if not isinstance(value, int):
        raise TypeError(f"varint must be int, got {type(value).__name__}")
    if value < 0:
        raise ValueError("negative varint is not supported")

    out = bytearray()

    while True:
        b = value & 0x7F
        value >>= 7

        if value:
            out.append(b | 0x80)
        else:
            out.append(b)
            return bytes(out)


def encode_key(field_number: int, wire_type: int) -> bytes:
    return encode_varint((field_number << 3) | wire_type)


def parse_fields(data: bytes) -> list[dict[str, Any]]:
    fields: list[dict[str, Any]] = []
    offset = 0

    while offset < len(data):
        start = offset
        key, offset = read_varint(data, offset)
        number = key >> 3
        wire = key & 7
        key_bytes = data[start:offset]

        if number == 0:
            raise ValueError(f"invalid field number at 0x{start:X}")

        if wire == 0:
            value_start = offset
            value, offset = read_varint(data, offset)
            fields.append({
                "number": number,
                "wire": wire,
                "key": key_bytes,
                "raw": data[value_start:offset],
                "value": value,
            })

        elif wire == 1:
            value_start = offset
            offset += 8
            if offset > len(data):
                raise ValueError("64-bit field overflow")
            fields.append({
                "number": number,
                "wire": wire,
                "key": key_bytes,
                "raw": data[value_start:offset],
                "value": data[value_start:offset],
            })

        elif wire == 2:
            length_start = offset
            length, payload_start = read_varint(data, offset)
            payload_end = payload_start + length

            if payload_end > len(data):
                raise ValueError("length-delimited field overflow")

            offset = payload_end
            fields.append({
                "number": number,
                "wire": wire,
                "key": key_bytes,
                "raw": data[length_start:offset],
                "payload": data[payload_start:payload_end],
            })

        elif wire == 5:
            value_start = offset
            offset += 4
            if offset > len(data):
                raise ValueError("32-bit field overflow")
            fields.append({
                "number": number,
                "wire": wire,
                "key": key_bytes,
                "raw": data[value_start:offset],
                "value": data[value_start:offset],
            })

        else:
            raise ValueError(
                f"unsupported protobuf wire type {wire} at 0x{start:X}"
            )

    return fields


def pack_length_delimited(field_number: int, payload: bytes) -> bytes:
    return (
        encode_key(field_number, 2)
        + encode_varint(len(payload))
        + payload
    )


def b64e(data: bytes) -> str:
    return base64.b64encode(data).decode("ascii")


def b64d(text: str) -> bytes:
    return base64.b64decode(text.encode("ascii"), validate=True)


# ===========================================================================
# Player f6
# ===========================================================================

def decode_player(payload: bytes) -> dict[str, int | None]:
    result = {
        "money": None,
        "otherworld_coin": None,
    }

    for field in parse_fields(payload):
        if field["number"] == 5 and field["wire"] == 0:
            result["money"] = field["value"]

        elif field["number"] == 10 and field["wire"] == 0:
            result["otherworld_coin"] = field["value"]

    return result


def patch_player(
    payload: bytes,
    *,
    money: int | None,
    otherworld_coin: int | None,
) -> bytes:
    rebuilt = bytearray()
    money_seen = False
    coin_seen = False

    for field in parse_fields(payload):
        if field["number"] == 5 and field["wire"] == 0:
            money_seen = True
            if money is None:
                rebuilt += field["key"] + field["raw"]
            else:
                rebuilt += field["key"] + encode_varint(money)

        elif field["number"] == 10 and field["wire"] == 0:
            coin_seen = True
            if otherworld_coin is None:
                rebuilt += field["key"] + field["raw"]
            else:
                rebuilt += field["key"] + encode_varint(otherworld_coin)

        else:
            rebuilt += field["key"] + field["raw"]

    if money is not None and not money_seen:
        rebuilt += encode_key(5, 0) + encode_varint(money)

    if otherworld_coin is not None and not coin_seen:
        rebuilt += encode_key(10, 0) + encode_varint(otherworld_coin)

    return bytes(rebuilt)


# ===========================================================================
# f9 / f10 item records
# ===========================================================================

def decode_item_record(
    payload: bytes,
    *,
    source_order: int,
) -> dict[str, Any]:
    stored_slot = 0
    slot_field_present = False
    record_type_f1 = None
    item_id = None
    quantity = None

    for field in parse_fields(payload):
        if field["number"] == 1 and field["wire"] == 0:
            record_type_f1 = field["value"]

        elif field["number"] == 2 and field["wire"] == 0:
            stored_slot = field["value"]
            slot_field_present = True

        elif field["number"] == 3 and field["wire"] == 2:
            for item_field in parse_fields(field["payload"]):
                if item_field["number"] == 1 and item_field["wire"] == 0:
                    item_id = item_field["value"]

                elif item_field["number"] == 2 and item_field["wire"] == 0:
                    quantity = item_field["value"]

    return {
        "slot": stored_slot + 1,
        "item_id": item_id,
        "quantity": quantity,
        "record_type_f1": record_type_f1,
        "_source_order": source_order,
        "_slot_field_present": slot_field_present,
        "_raw_record_b64": b64e(payload),
    }


def patch_item_message(
    payload: bytes,
    *,
    item_id: int | None,
    quantity: int | None,
) -> bytes:
    rebuilt = bytearray()
    item_id_seen = False
    quantity_seen = False

    for field in parse_fields(payload):
        if field["number"] == 1 and field["wire"] == 0:
            item_id_seen = True
            if item_id is None:
                rebuilt += field["key"] + field["raw"]
            else:
                rebuilt += field["key"] + encode_varint(item_id)

        elif field["number"] == 2 and field["wire"] == 0:
            quantity_seen = True
            if quantity is None:
                rebuilt += field["key"] + field["raw"]
            else:
                rebuilt += field["key"] + encode_varint(quantity)

        else:
            rebuilt += field["key"] + field["raw"]

    if item_id is not None and not item_id_seen:
        rebuilt += encode_key(1, 0) + encode_varint(item_id)

    if quantity is not None and not quantity_seen:
        rebuilt += encode_key(2, 0) + encode_varint(quantity)

    return bytes(rebuilt)


def patch_item_record(
    payload: bytes,
    *,
    slot: int,
    item_id: int | None,
    quantity: int | None,
    record_type_f1: int | None,
) -> bytes:
    if slot < 1:
        raise ValueError(f"slot must be >= 1, got {slot}")

    stored_slot = slot - 1
    rebuilt = bytearray()

    slot_seen = False
    record_type_seen = False
    item_message_seen = False

    for field in parse_fields(payload):
        if field["number"] == 1 and field["wire"] == 0:
            record_type_seen = True

            if record_type_f1 is None:
                rebuilt += field["key"] + field["raw"]
            else:
                rebuilt += field["key"] + encode_varint(record_type_f1)

        elif field["number"] == 2 and field["wire"] == 0:
            slot_seen = True
            rebuilt += field["key"] + encode_varint(stored_slot)

        elif field["number"] == 3 and field["wire"] == 2:
            item_message_seen = True
            new_item = patch_item_message(
                field["payload"],
                item_id=item_id,
                quantity=quantity,
            )
            rebuilt += field["key"] + encode_varint(len(new_item)) + new_item

        else:
            rebuilt += field["key"] + field["raw"]

    # slot 1은 protobuf default 0이라 원본에 f2가 없으면 계속 생략한다.
    if not slot_seen and stored_slot != 0:
        rebuilt += encode_key(2, 0) + encode_varint(stored_slot)

    # 기존 레코드에서 f1이 없었고 JSON도 null이면 그대로 없음.
    if record_type_f1 is not None and not record_type_seen:
        rebuilt += encode_key(1, 0) + encode_varint(record_type_f1)

    if not item_message_seen:
        if item_id is None:
            raise ValueError("cannot create item message with item_id=null")

        item_payload = bytearray()
        item_payload += encode_key(14, 2) + encode_varint(0)
        item_payload += encode_key(1, 0) + encode_varint(item_id)

        if quantity is not None:
            item_payload += encode_key(2, 0) + encode_varint(quantity)

        rebuilt += pack_length_delimited(3, bytes(item_payload))

    return bytes(rebuilt)


def build_new_storage_record(
    *,
    slot: int,
    item_id: int,
    quantity: int | None,
) -> bytes:
    out = bytearray()
    stored_slot = slot - 1

    if stored_slot != 0:
        out += encode_key(2, 0) + encode_varint(stored_slot)

    item_payload = bytearray()
    item_payload += encode_key(14, 2) + encode_varint(0)
    item_payload += encode_key(1, 0) + encode_varint(item_id)

    if quantity is not None:
        item_payload += encode_key(2, 0) + encode_varint(quantity)

    out += pack_length_delimited(3, bytes(item_payload))
    return bytes(out)


def build_new_inventory_record(
    *,
    slot: int,
    item_id: int,
    quantity: int | None,
    record_type_f1: int,
) -> bytes:
    out = bytearray()
    stored_slot = slot - 1

    out += encode_key(1, 0) + encode_varint(record_type_f1)

    if stored_slot != 0:
        out += encode_key(2, 0) + encode_varint(stored_slot)

    item_payload = bytearray()
    item_payload += encode_key(14, 2) + encode_varint(0)
    item_payload += encode_key(1, 0) + encode_varint(item_id)

    if quantity is not None:
        item_payload += encode_key(2, 0) + encode_varint(quantity)

    out += pack_length_delimited(3, bytes(item_payload))
    return bytes(out)


# ===========================================================================
# f8 equipment — inspection only
# ===========================================================================

def decode_equipment(
    payload: bytes,
    *,
    record_index: int,
) -> dict[str, Any]:
    varints: dict[str, list[int]] = {}

    for field in parse_fields(payload):
        if field["wire"] == 0:
            key = f"f{field['number']}"
            varints.setdefault(key, []).append(field["value"])

    instance_id = (
        varints.get("f2", [None])[0]
        if varints.get("f2")
        else None
    )
    base_item_id = (
        varints.get("f7", [None])[0]
        if varints.get("f7")
        else None
    )

    return {
        "record": record_index,
        "instance_id_f2": instance_id,
        "base_item_id_f7": base_item_id,
        "varints": varints,
        "editable": False,
        "_raw_record_b64": b64e(payload),
    }


# ===========================================================================
# SAV -> JSON
# ===========================================================================

def sav_to_json(
    sav_path: Path,
    json_path: Path | None = None,
) -> Path:
    data = sav_path.read_bytes()
    fields = parse_fields(data)

    if json_path is None:
        json_path = sav_path.with_suffix(".json")

    player: dict[str, int | None] = {
        "money": None,
        "otherworld_coin": None,
    }
    equipment: list[dict[str, Any]] = []
    storage: list[dict[str, Any]] = []
    inventory: list[dict[str, Any]] = []

    equipment_order = 0
    storage_order = 0
    inventory_order = 0

    for field in fields:
        if field["number"] == 6 and field["wire"] == 2:
            player = decode_player(field["payload"])

        elif field["number"] == 8 and field["wire"] == 2:
            equipment_order += 1
            equipment.append(
                decode_equipment(
                    field["payload"],
                    record_index=equipment_order,
                )
            )

        elif field["number"] == 9 and field["wire"] == 2:
            storage_order += 1
            storage.append(
                decode_item_record(
                    field["payload"],
                    source_order=storage_order,
                )
            )

        elif field["number"] == 10 and field["wire"] == 2:
            inventory_order += 1
            inventory.append(
                decode_item_record(
                    field["payload"],
                    source_order=inventory_order,
                )
            )

    # 사람은 슬롯 순으로 읽기 쉽도록 정렬.
    # 실제 SAV 재생성 순서는 _source_order로 복구한다.
    storage.sort(key=lambda x: (x["slot"], x["_source_order"]))
    inventory.sort(key=lambda x: (x["slot"], x["_source_order"]))

    doc = {
        "_meta": {
            "format": FORMAT_NAME,
            "source_file": sav_path.name,
            "source_size": len(data),
            "source_sha256": hashlib.sha256(data).hexdigest(),
            "editable": [
                "player.money",
                "player.otherworld_coin",
                "storage[].slot",
                "storage[].item_id",
                "storage[].quantity",
                "inventory[].slot",
                "inventory[].item_id",
                "inventory[].quantity",
            ],
            "notes": [
                "equipment is inspection-only in this version.",
                "quantity may be null when the original protobuf omitted that field.",
                "Keys beginning with '_' preserve unknown data and source ordering.",
                "Deleting a storage/inventory object removes that record from the rebuilt SAV.",
            ],
        },
        "player": player,
        "equipment": equipment,
        "storage": storage,
        "inventory": inventory,
        "_preserve": {
            "original_save_b64": b64e(data),
        },
    }

    json_path.write_text(
        json.dumps(doc, ensure_ascii=False, indent=2),
        encoding="utf-8",
    )
    return json_path


# ===========================================================================
# JSON -> SAV
# ===========================================================================

def validate_entry(
    entry: dict[str, Any],
    collection: str,
) -> None:
    if "slot" not in entry:
        raise ValueError(f"{collection} entry missing slot")

    if not isinstance(entry["slot"], int) or entry["slot"] < 1:
        raise ValueError(f"{collection}.slot must be integer >= 1")

    for key in ("item_id", "quantity"):
        value = entry.get(key)

        if value is not None:
            if not isinstance(value, int):
                raise ValueError(
                    f"{collection}.{key} must be int or null"
                )
            if value < 0:
                raise ValueError(
                    f"{collection}.{key} must be >= 0 or null"
                )

    record_type = entry.get("record_type_f1")
    if record_type is not None and not isinstance(record_type, int):
        raise ValueError(
            f"{collection}.record_type_f1 must be int or null"
        )


def order_entries(
    entries: list[dict[str, Any]],
) -> list[dict[str, Any]]:
    """
    Existing records retain original SAV order.
    New records without _source_order are appended in JSON order.
    """
    existing = []
    new = []

    for index, entry in enumerate(entries):
        order = entry.get("_source_order")

        if isinstance(order, int):
            existing.append((order, index, entry))
        else:
            new.append((index, entry))

    existing.sort(key=lambda x: (x[0], x[1]))

    return (
        [entry for _, _, entry in existing]
        + [entry for _, entry in new]
    )


def build_collection_records(
    entries: list[dict[str, Any]],
    *,
    collection: str,
) -> list[bytes]:
    records: list[bytes] = []
    seen_slots: set[int] = set()

    for entry in order_entries(entries):
        validate_entry(entry, collection)

        slot = entry["slot"]
        item_id = entry.get("item_id")
        quantity = entry.get("quantity")
        record_type_f1 = entry.get("record_type_f1")

        if slot in seen_slots:
            raise ValueError(
                f"duplicate {collection} slot in JSON: {slot}"
            )
        seen_slots.add(slot)

        raw_b64 = entry.get("_raw_record_b64")

        if raw_b64:
            raw_record = b64d(raw_b64)

            new_record = patch_item_record(
                raw_record,
                slot=slot,
                item_id=item_id,
                quantity=quantity,
                record_type_f1=record_type_f1,
            )

        elif collection == "storage":
            if item_id is None:
                raise ValueError(
                    "new storage record requires item_id"
                )

            new_record = build_new_storage_record(
                slot=slot,
                item_id=item_id,
                quantity=quantity,
            )

        else:
            if item_id is None:
                raise ValueError(
                    "new inventory record requires item_id"
                )

            if not isinstance(record_type_f1, int):
                raise ValueError(
                    "new inventory record requires record_type_f1. "
                    "Safest method: copy an existing inventory object, "
                    "then edit slot/item_id/quantity."
                )

            new_record = build_new_inventory_record(
                slot=slot,
                item_id=item_id,
                quantity=quantity,
                record_type_f1=record_type_f1,
            )

        parse_fields(new_record)
        records.append(new_record)

    return records


def json_to_sav(
    json_path: Path,
    sav_path: Path | None = None,
) -> Path:
    doc = json.loads(
        json_path.read_text(encoding="utf-8")
    )

    meta = doc.get("_meta", {})
    if meta.get("format") != FORMAT_NAME:
        raise ValueError(
            f"unsupported JSON format: {meta.get('format')!r}"
        )

    original_b64 = (
        doc.get("_preserve", {})
        .get("original_save_b64")
    )

    if not isinstance(original_b64, str) or not original_b64:
        raise ValueError(
            "_preserve.original_save_b64 is required"
        )

    original_data = b64d(original_b64)
    original_fields = parse_fields(original_data)

    player = doc.get("player", {})
    money = player.get("money")
    otherworld_coin = player.get("otherworld_coin")

    for name, value in (
        ("player.money", money),
        ("player.otherworld_coin", otherworld_coin),
    ):
        if value is not None:
            if not isinstance(value, int):
                raise ValueError(f"{name} must be int or null")
            if value < 0:
                raise ValueError(f"{name} must be >= 0 or null")

    storage_entries = doc.get("storage", [])
    inventory_entries = doc.get("inventory", [])

    if not isinstance(storage_entries, list):
        raise ValueError("storage must be an array")
    if not isinstance(inventory_entries, list):
        raise ValueError("inventory must be an array")

    storage_records = build_collection_records(
        storage_entries,
        collection="storage",
    )
    inventory_records = build_collection_records(
        inventory_entries,
        collection="inventory",
    )

    rebuilt = bytearray()

    player_written = False
    storage_written = False
    inventory_written = False

    for field in original_fields:
        if field["number"] == 6 and field["wire"] == 2:
            if not player_written:
                new_player = patch_player(
                    field["payload"],
                    money=money,
                    otherworld_coin=otherworld_coin,
                )
                rebuilt += pack_length_delimited(
                    6,
                    new_player,
                )
                player_written = True
            else:
                rebuilt += field["key"] + field["raw"]

        elif field["number"] == 9 and field["wire"] == 2:
            if not storage_written:
                for record in storage_records:
                    rebuilt += pack_length_delimited(
                        9,
                        record,
                    )
                storage_written = True
            # old f9 records are skipped

        elif field["number"] == 10 and field["wire"] == 2:
            if not inventory_written:
                for record in inventory_records:
                    rebuilt += pack_length_delimited(
                        10,
                        record,
                    )
                inventory_written = True
            # old f10 records are skipped

        else:
            # Includes f8 equipment: byte-for-byte preserved.
            rebuilt += field["key"] + field["raw"]

    if not storage_written and storage_records:
        for record in storage_records:
            rebuilt += pack_length_delimited(9, record)

    if not inventory_written and inventory_records:
        for record in inventory_records:
            rebuilt += pack_length_delimited(10, record)

    output_data = bytes(rebuilt)

    # full wire-format validation
    parse_fields(output_data)

    if sav_path is None:
        sav_path = json_path.with_name("PlayerContext_0.sav")

    sav_path.write_bytes(output_data)
    return sav_path


# ===========================================================================
# Round-trip test
# ===========================================================================

def roundtrip_check(
    sav_path: Path,
) -> tuple[bool, str]:
    temp_json = sav_path.with_name(
        sav_path.stem + ".__vd_test__.json"
    )
    temp_sav = sav_path.with_name(
        sav_path.stem + ".__vd_test__.sav"
    )

    try:
        sav_to_json(sav_path, temp_json)
        json_to_sav(temp_json, temp_sav)

        original = sav_path.read_bytes()
        rebuilt = temp_sav.read_bytes()

        if original == rebuilt:
            return True, "Byte-for-byte round trip OK"

        return (
            False,
            "Round trip parsed successfully but bytes differ. "
            f"original_sha256={hashlib.sha256(original).hexdigest()} "
            f"rebuilt_sha256={hashlib.sha256(rebuilt).hexdigest()}"
        )

    finally:
        if temp_json.exists():
            temp_json.unlink()

        if temp_sav.exists():
            temp_sav.unlink()


# ===========================================================================
# CLI
# ===========================================================================

def interactive_menu() -> int:
    print()
    print("VOID DIVER SAV <-> JSON Editor v1.1")
    print("=" * 40)
    print("1) SAV -> JSON")
    print("2) JSON -> SAV")
    print("3) SAV 무수정 왕복 검사")
    print("0) 종료")
    print()

    choice = input("선택: ").strip()

    if choice == "0":
        return 0

    if choice == "1":
        path = Path(
            input("SAV 파일 경로: ")
            .strip()
            .strip('"')
        )
        out = sav_to_json(path)
        print(f"\n완료: {out}")
        return 0

    if choice == "2":
        path = Path(
            input("JSON 파일 경로: ")
            .strip()
            .strip('"')
        )
        out = json_to_sav(path)
        print(f"\n완료: {out}")
        return 0

    if choice == "3":
        path = Path(
            input("SAV 파일 경로: ")
            .strip()
            .strip('"')
        )
        ok, message = roundtrip_check(path)
        print(f"\n{'성공' if ok else '주의'}: {message}")
        return 0 if ok else 2

    print("잘못된 선택입니다.")
    return 2


def main() -> int:
    # auto mode must be handled before argparse subcommand parsing,
    # because a plain filename is not a subcommand.
    if len(sys.argv) == 2:
        auto = Path(sys.argv[1])

        if auto.suffix.lower() == ".sav":
            try:
                print(sav_to_json(auto))
                return 0
            except Exception as exc:
                print(f"ERROR: {exc}", file=sys.stderr)
                return 1

        if auto.suffix.lower() == ".json":
            try:
                print(json_to_sav(auto))
                return 0
            except Exception as exc:
                print(f"ERROR: {exc}", file=sys.stderr)
                return 1

    parser = argparse.ArgumentParser(
        description="VOID DIVER Demo SAV <-> JSON editor"
    )
    sub = parser.add_subparsers(dest="command")

    export_p = sub.add_parser(
        "export",
        help="SAV -> JSON",
    )
    export_p.add_argument("input", type=Path)
    export_p.add_argument(
        "-o",
        "--output",
        type=Path,
    )

    import_p = sub.add_parser(
        "import",
        help="JSON -> SAV",
    )
    import_p.add_argument("input", type=Path)
    import_p.add_argument(
        "-o",
        "--output",
        type=Path,
    )

    check_p = sub.add_parser(
        "check",
        help="unedited round-trip validation",
    )
    check_p.add_argument("input", type=Path)

    args = parser.parse_args()

    try:
        if args.command == "export":
            print(
                sav_to_json(
                    args.input,
                    args.output,
                )
            )
            return 0

        if args.command == "import":
            print(
                json_to_sav(
                    args.input,
                    args.output,
                )
            )
            return 0

        if args.command == "check":
            ok, message = roundtrip_check(args.input)
            print(message)
            return 0 if ok else 2

        return interactive_menu()

    except Exception as exc:
        print(f"ERROR: {exc}", file=sys.stderr)
        return 1


if __name__ == "__main__":
    raise SystemExit(main())
