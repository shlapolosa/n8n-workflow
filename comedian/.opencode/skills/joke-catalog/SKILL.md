---
name: joke-catalog
description: Manages joke storage and retrieval with JSON file persistence
license: MIT
allowed-tools:
  - read
  - write
  - edit
  - grep
  - glob
metadata:
  version: "1.0.0"
  category: "entertainment"
---

# Joke Catalog Skill

Storage: `jokes.json` in project root.

## JSON Schema

```json
{
  "jokes": [
    {
      "id": "number",
      "joke": "string",
      "category": "pun|one-liner|knock-knock|dad-joke",
      "timestamp": "ISO 8601",
      "rating": "number|null"
    }
  ],
  "metadata": {
    "lastUpdated": "ISO 8601",
    "totalCount": "number"
  }
}
```

## Operations

### Add Joke
1. Read jokes.json (create if missing)
2. Next ID = max(IDs) + 1
3. Append joke with timestamp
4. Update metadata
5. Write with pretty formatting

### List/Count
1. Read jokes.json
2. Return formatted list or count

## Empty Init

```json
{
  "jokes": [],
  "metadata": { "lastUpdated": "", "totalCount": 0 }
}
```
