# qdredit - Qdrant memory manager for SillyTavern

## SYNTAX & USAGE GUIDE

Qdrant is designed to automatically store SillyTavern chat messages in chunked "points" which an embedding server processes into a vector array and payload, along with a timestamp field (in UNIX Epoch milliseconds format) and a UUID (universally unique identifier) such as `8be4df61-93ca-11d2-aa0d-00e098032b8c` in a named memory "collection". Within SillyTavern, you have very little ability to modify the collection or memories stored in it. `qdredit` lets you do pretty much whatever you want to a memory collection. Note: integer IDs are not used by default by the Qdrant SillyTavern extension, but they literally do not affect memory retrieval or stop Qdrant from matching the vector array just like with automatically stored entries with UUIDs. If you want to manually add entries to your collection, integer IDs are simpler to work with and do not harm anything.

---

## SETUP INSTRUCTIONS:

Set the four values listed just below inside the qdredit script according to these instructions for your specific uses. 

* **COLLECTION:** Qdrant's default is per character memory collections, so `mem_character1`, `mem_character2`, etc. You can set the collection name to edit here, or you can call `qdredit -c [collection name] <other options>` and it will override this default.
* **EMBEDDING URL:** You will need to set up an embedding server, either an online API or a local model. Put the url here. 
* **TIMEZONE:** Set your timezone here. Use the TZ Identifier that matches your timezone from here: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones Only used for converting UNIX Epoch to local time for printing by date, etc.
* **QDRANT URL:** This is the URL of Qdrant server you're pulling from, almost certainly running on your own PC. The default is standard. The server has to be running for `qdredit` to be able to read/modify the database.

```python
DEFAULT_COLLECTION = "mem_collection"
DEFAULT_EMBEDDING_URL = "http://localhost:8080/v1/embeddings"
TIMEZONE = "America/Chicago"
QD_URL = "http://localhost:6333"
```

---

## BASIC COMMANDS:

* `qdredit -count` Show total number of points (all types) in the collection.
* `qdredit -list` List integer ID ranges and UUID points in order by timestamp. Will correctly display gaps where points have been deleted in integer ranges.
* `qdredit -print <input>` Print formatted point(s) (ID, timestamp, text) with pretty card formatting.
* `qdredit -delete <input>` Delete point(s) after confirmation. Specify integer, or `uuid<index>`. Handles ranges for both. Use `qdredit -list` to get the indices for UUID points.
* `qdredit -consolidate` Checks for missing integer IDs or ranges and consolidates all integer IDs into one contiguous range starting from 0. Useful if you deleted some points from the middle or added points with a custom integer ID and want to get a single range from `0...N-1`. UUID points are untouched. Requires confirmation.
* `qdredit -stripdate <uuidX>` Remove leading `[YYYY-MM-DD]` line from UUID point(s). Useful if you want to get rid of the tag Qdrant prepends to every text payload so your AI GM/character doesn't get confused and to save tokens.
* `qdredit -add --text "..."` Add a new point (requires active embedding server). By default uses the earliest unused Integer ID, so if you have 100 integer IDs, delete point 0, and then add a point, it will be added as point 0 by default. `-add --text` accepts input from a file or from quoted text or from stdin as a backup. 
* `qdredit -fileprint <input>` Save full point data (including vectors) in a range you select to a file called 'qdrantpoints' in your current working directory. If you're outside `/home`, you might be denied permission. `-fileprint all` saves all points regardless of ID type. 
* `qdredit -fileadd` Upserts points from 'qdrantpoints' file directly to the database, using all fields as written in the file. If they were UUID points before they will be again. If they were integer points before, it will over/write whatever is present to add them back where they were before. Doesn't require an embedding server, because vector field data should be in the file. You must be in the directory where the file is located.

---

## OPTIONS:

* `-c, --collection NAME` Use a different collection (default: `mem_character_name`). Note that you can disable per character memories and tell Qdrant to use whatever you want as the collection name. Single collections are better for a Tabletop Roleplaying game with one player, one GM.
* `--color` Force color output even if not a TTY.
* `--no-color` Disable color output.

---

## ADDITIONAL OPTIONS FOR -add:

* `--text "..."` Required: the text content of the new point.
* `--id auto|INTEGER|UUID` Point ID (default: next integer).
* `--timestamp auto|'mm-dd-yyyy HH:MM:SS[.sss]'` Timestamp (default: local time).
* `--embedding-url URL` Embedding server endpoint (default: `http://127.0.0.1:8080/v1/embeddings`).

---

## INPUT FORMATS FOR -print, -delete, -fileprint:

* **Single integer ID:** `0`
* **Integer range:** `0-10000` (practical max: 10^12, hard: 2^63-1)
* **Date (local time):** `05-15-2026` (All points with that day's timestamp.)
* **UUID by index:** `uuid0`, `uuid101`, ... but only one at a time.
* **UUID range by index:** `uuid0-uuid1000` (ordered by timestamp and alphanumerically by UUID as a backup if timestamp matches.)

---

## EXAMPLES:

```bash
# Show total points
qdredit -count

# List all points (both integer and UUID)
qdredit -list

# Print a single integer point
qdredit -print 40
  
# Print a range of integer points
qdredit -print 100-200

# Print all points from a specific date
qdredit -print 05-15-2026

# Print a UUID point by its index (index available from -list)
qdredit -print uuid0

# Print a UUID point by its actual UUID:
qdredit -print 6ea8ec4d-3141-4e12-89f1-5d5e02a7bf51

# Print a range of UUID points by index
qdredit -print uuid700-uuid1500

# Delete an integer point
qdredit -delete 123

# Delete a range of integer points
qdredit -delete 100-200

# Delete a UUID point by index
qdredit -delete uuid2

# Delete a range of UUID points
qdredit -delete uuid0-uuid4

# Strip leading date from a UUID point
qdredit -stripdate uuid0

# Strip leading date from a range of UUID points
qdredit -stripdate uuid0-uuid5

# Add a new point (auto‑generated ID and timestamp)
qdredit -add --text "The party reached the Gilded Quill."

# Add a point with a specific integer ID and custom timestamp
qdredit -add --id 4000 --timestamp "05-10-2026 20:15:00" --text "Battle with the Hill Giants. Results: 14500 GP. One undentified magic ring."

# Add a point from file
qdredit -add --text point200.txt

# Add a point using stdin, by typing manually
qdredit -add --id 1038 --timestamp "12-12-2099 23:59:59.999"
# You will be prompted to enter text manually. CTRL+D or CTRL+Z ends.  

# Add a point using stdin and piping:
cat point75.txt | qdredit -add --id 2000 --timestamp "01-01-2075 00:00:00" --embedding-url http://embeddingserver:4567/api/embeddings

# Use a different collection (e.g., per‑character memory)
qdredit -c mem_Character -list

# Save full point data (including vectors) for IDs 0-100 to file
qdredit -fileprint 0-100

# Save all points (both integer and UUID) with vectors to file
qdredit -fileprint all

# Restore all points from the file (no embedding server needed)
qdredit -fileadd
```

---

## Final Notes:

* Integer IDs are stored as numbers, UUIDs as strings.
* The `-fileprint` output file is named 'qdrantpoints' in the current directory.
* Use `--no-color` if redirecting output to a file or pipe.
* The embedding server must be running for `-add` to work.

## License

This project is licensed under the GNU Affero General Public License v3.0 (AGPL-3.0). This means you are free to use, modify, and distribute this software, provided that any modified versions or network services utilizing this code are also made available under the same license. See the [LICENSE](LICENSE) file for full details.