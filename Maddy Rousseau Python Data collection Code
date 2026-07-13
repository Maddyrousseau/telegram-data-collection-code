"""Collect public Telegram channel posts for research.

Required environment variables:
    TG_API_ID
    TG_API_HASH
    TG_CHANNEL_URL

Optional environment variables:
    TG_START_DATE   ISO date in YYYY-MM-DD format (default: 2025-01-01)
    TG_SESSION_NAME Local Telethon session name (default: telegram_research_session)
    TG_OUTPUT_NAME  Generic output prefix (default: telegram_posts)

Do not commit API credentials, session files, or collected data to GitHub.
"""

import csv
import json
import os
import re
import time
from datetime import datetime, timezone
from pathlib import Path

from telethon.errors import FloodWaitError, UsernameInvalidError, UsernameNotOccupiedError
from telethon.sync import TelegramClient
from telethon.utils import get_display_name, get_extension

URL_REGEX = re.compile(r"https?://\S+")


def required_env(name: str) -> str:
    """Return a required environment variable or stop with a clear message."""
    value = os.getenv(name)
    if not value:
        raise SystemExit(f"Missing required environment variable: {name}")
    return value


def parse_start_date(value: str) -> datetime:
    """Convert a YYYY-MM-DD string to a timezone-aware UTC datetime."""
    try:
        parsed = datetime.strptime(value, "%Y-%m-%d")
    except ValueError as exc:
        raise SystemExit("TG_START_DATE must use YYYY-MM-DD format.") from exc
    return parsed.replace(tzinfo=timezone.utc)


API_ID = int(required_env("TG_API_ID"))
API_HASH = required_env("TG_API_HASH")
CHANNEL_URL = required_env("TG_CHANNEL_URL")
SESSION_NAME = os.getenv("TG_SESSION_NAME", "telegram_research_session")
OUTPUT_NAME = os.getenv("TG_OUTPUT_NAME", "telegram_posts")
START_DATE_UTC = parse_start_date(os.getenv("TG_START_DATE", "2025-01-01"))

TODAY = datetime.now().strftime("%Y-%m-%d")
BASE_NAME = f"{OUTPUT_NAME}_{TODAY}"
JSONL_PATH = f"{BASE_NAME}.jsonl"
CSV_PATH = f"{BASE_NAME}.csv"
MEDIA_DIR = f"{OUTPUT_NAME}_media_{TODAY}"


def safe_ext_from_message(message) -> str:
    """Return a media file extension when one is available."""
    document = getattr(message, "document", None)
    if document:
        name = get_display_name(document)
        if name and "." in name:
            return os.path.splitext(name)[1]

    media = getattr(message, "media", None)
    if media:
        return get_extension(media) or ""
    return ""


def to_row(message, media_path=None) -> dict:
    """Convert a Telegram message to a research record.

    Sender identifiers are intentionally excluded from public-facing output.
    """
    text = (message.message or "").strip()
    return {
        "message_id": message.id,
        "date": message.date.isoformat() if message.date else None,
        "text": text,
        "views": getattr(message, "views", None),
        "forwards": getattr(message, "forwards", None),
        "reply_to_message_id": getattr(
            getattr(message, "reply_to", None), "reply_to_msg_id", None
        ),
        "urls": URL_REGEX.findall(text) if text else [],
        "media_path": media_path,
    }


def fetch_messages_since(client, entity, start_dt_utc):
    """Yield messages from newest to oldest until the start date is reached."""
    iterator = client.iter_messages(entity, limit=None)

    while True:
        try:
            message = next(iterator)
            if not getattr(message, "date", None):
                continue
            if message.date >= start_dt_utc:
                yield message
            else:
                break
        except FloodWaitError as exc:
            wait_seconds = getattr(exc, "seconds", 5)
            print(f"[info] Telegram rate limit: sleeping {wait_seconds} seconds.")
            time.sleep(wait_seconds + 1)
        except StopIteration:
            break


def main() -> None:
    Path(MEDIA_DIR).mkdir(parents=True, exist_ok=True)
    rows = []

    with TelegramClient(SESSION_NAME, API_ID, API_HASH) as client:
        try:
            entity = client.get_entity(CHANNEL_URL)
        except (UsernameInvalidError, UsernameNotOccupiedError, ValueError) as exc:
            raise SystemExit("Could not resolve the configured Telegram channel.") from exc

        media_count = 0

        for message in fetch_messages_since(client, entity, START_DATE_UTC):
            media_path = None

            if getattr(message, "media", None):
                extension = safe_ext_from_message(message)
                filename = f"media_{message.id}{extension}"
                target_path = os.path.join(MEDIA_DIR, filename)

                try:
                    media_path = client.download_media(message, file=target_path)
                    if media_path:
                        media_count += 1
                except Exception as exc:  # Continue collecting if one media file fails.
                    print(
                        f"[warn] Media download failed for message "
                        f"{message.id}: {exc}"
                    )

            rows.append(to_row(message, media_path=media_path))

    if rows:
        with open(JSONL_PATH, "w", encoding="utf-8") as jsonl_file:
            for row in rows:
                jsonl_file.write(json.dumps(row, ensure_ascii=False) + "\n")

        fieldnames = [
            "message_id",
            "date",
            "text",
            "views",
            "forwards",
            "reply_to_message_id",
            "urls",
            "media_path",
        ]

        with open(CSV_PATH, "w", newline="", encoding="utf-8") as csv_file:
            writer = csv.DictWriter(csv_file, fieldnames=fieldnames)
            writer.writeheader()

            for row in rows:
                output_row = dict(row)
                output_row["urls"] = json.dumps(
                    output_row.get("urls", []), ensure_ascii=False
                )
                writer.writerow(output_row)

    print(
        f"[done] Collected {len(rows)} messages on or after "
        f"{START_DATE_UTC.date()}. Media files: {media_count}"
    )
    print(f"[done] JSONL: {JSONL_PATH}")
    print(f"[done] CSV: {CSV_PATH}")
    print(f"[done] Media directory: {MEDIA_DIR}/")


if __name__ == "__main__":
    main()
