---
date: "2026-09-02T17:13:40+07:00"
draft: true
title: "Add history to firefox by command"
categories:
  - Code
tags:
  - firefox
---

## Firefox settings

- Go to `about:preferences#search` and disable `Show search suggestions`
- Go to `about:preferences#privacy` and enable `Never remember history`

## Add history by command

- Firstly, close Firefox
- Source this function and run `add_history <url> <title>`

```sh
add_history () {
        usage () {
                echo "Usage: $0 <url> <title>"
                exit 1
        }
        if [[ $# -ne 2 ]]
        then
                usage
        fi
        URL="$1"
        TITLE="$2"
        PROFILE_DIR="$HOME/Library/Application Support/Firefox/Profiles"
        PROFILE=$(ls "$PROFILE_DIR" | grep 'default-release' | head -1)
        if [[ -z "$PROFILE" ]]
        then
                echo "Error: no Firefox default-release profile found." >&2
                exit 1
        fi
        DB="$PROFILE_DIR/$PROFILE/places.sqlite"
        sqlite3 "$DB" <<SQL || {
INSERT INTO moz_places (url, title, visit_count, last_visit_date)
VALUES ('$URL', '$TITLE', 1, strftime('%s', 'now') * 1000000);

INSERT INTO moz_historyvisits (place_id, visit_date, visit_type)
VALUES (last_insert_rowid(), strftime('%s', 'now') * 1000000, 1);
SQL
                echo "sqlite3 failed" >&2
                exit 1
        }
        echo "Added: $TITLE ($URL)"
}

```
