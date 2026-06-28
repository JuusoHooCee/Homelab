# Portainer Admin Password Reset via BoltDB Hash Replacement

**Environment:** Any Linux system running Portainer CE in Docker  
**Use case:** Lost admin password, root/sudo access available on the host  
**Tested on:** Portainer CE 2.39.3 (applies to any version where `--reset-admin` is unavailable)

---

## Background

Portainer stores passwords as bcrypt hashes in a BoltDB key-value database (`portainer.db`).
Since the `--reset-admin` CLI flag was removed in newer versions, the password must be
replaced directly in the database binary using Python.

This same technique applies to **any application using BoltDB** as its storage backend.

---

## Step 1: Stop Portainer and locate the database

```bash
docker stop portainer

# Confirm the volume path
docker inspect portainer | grep -A5 Mounts
```

The database is typically located at:
```
/var/lib/docker/volumes/portainer_data/_data/portainer.db
```

---

## Step 2: Set up a Python environment for bcrypt

Use a virtual environment — the correct approach on Debian/Ubuntu systems:

```bash
python3 -m venv /tmp/bcryptenv
/tmp/bcryptenv/bin/pip install bcrypt
```

Generate a new bcrypt hash for your chosen password:

```bash
/tmp/bcryptenv/bin/python3 -c "
import bcrypt
password = b'YourNewPassword123!'
hashed = bcrypt.hashpw(password, bcrypt.gensalt())
print(hashed.decode())
"
```

Save the output hash. It will look like:
`$2b$12$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`

> **Critical:** bcrypt hashes are always exactly **60 characters** long.
> The replacement hash must match this length exactly, or the database will be corrupted.

---

## Step 3: Find the existing hash in the database

```bash
/tmp/bcryptenv/bin/python3 << 'EOF'
import re

db_path = "/var/lib/docker/volumes/portainer_data/_data/portainer.db"

with open(db_path, "rb") as f:
    data = f.read()

hashes = [(m.start(), m.group()) for m in re.finditer(rb'\$2[aby]\$\d+\$[./A-Za-z0-9]{53}', data)]
for offset, h in hashes:
    print(f"Offset {offset}: {h.decode()}")
    start = max(0, offset - 50)
    print(f"Context: {data[start:offset+80]}")
    print()
EOF
```

This will show the current hash and confirm the username (`admin`) next to it.

---

## Step 4: Replace the hash

Substitute `OLD_HASH` and `NEW_HASH` with the actual values:

```bash
/tmp/bcryptenv/bin/python3 << 'EOF'
import shutil

db_path = "/var/lib/docker/volumes/portainer_data/_data/portainer.db"

# Always take a backup first
shutil.copy(db_path, db_path + ".backup")
print("Backup created")

with open(db_path, "rb") as f:
    data = f.read()

old_hash = b"OLD_HASH_HERE"
new_hash = b"NEW_HASH_HERE"

if len(new_hash) != len(old_hash):
    print(f"ERROR: Hash length mismatch! Old: {len(old_hash)}, New: {len(new_hash)}")
    exit(1)

count = data.count(old_hash)
new_data = data.replace(old_hash, new_hash)

with open(db_path, "wb") as f:
    f.write(new_data)

print(f"Successfully replaced {count} hash occurrence(s)!")
EOF
```

> BoltDB keeps multiple internal copies of records — typically 3–4 occurrences.
> The script replaces all of them automatically.

---

## Step 5: Start Portainer and log in

```bash
docker start portainer
```

Log in at your Portainer URL:
- **Username:** `admin`
- **Password:** the password you used when generating the new hash

---

## Where else does this technique apply?

The same bcrypt hash replacement method works for any application that:
- Stores bcrypt-hashed passwords in a binary or database file
- Has lost its built-in password reset mechanism

| Application | Database | Notes |
|---|---|---|
| **Portainer** | BoltDB | Covered in this guide |
| **Gitea** | SQLite / MySQL / PostgreSQL | Use sqlite3 or SQL UPDATE instead of binary replacement |
| **Grafana** | SQLite | `UPDATE user SET password=... WHERE login='admin'` |
| **Nextcloud** | MySQL / PostgreSQL / SQLite | Use `occ user:resetpassword` or direct SQL |
| **NATS Streaming** | BoltDB | Same binary replacement approach |
| **Any app with a dump file** | Binary / JSON / BSON | Locate the hash pattern, replace in-place |

The core principle: **find the hash in storage → generate a new hash for a known password → replace it**.

---

## Notes

- A backup (`portainer.db.backup`) is always created before any changes.
- This method is non-destructive to surrounding data as long as hash lengths match.
- `venv` is the correct Python packaging approach on modern Debian/Ubuntu systems.
- If `--break-system-packages` is used instead of venv, it is generally safe for simple packages like `bcrypt`, but venv is the preferred habit to develop.
