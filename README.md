# MongoDB Atlas → Google Drive Backups (GitHub Actions)

This repository contains a **serverless** backup job that dumps your MongoDB Atlas cluster with `mongodump` and uploads the compressed archive to **Google Drive** using `rclone`. It runs **every day at 02:30 Asia/Colombo** (21:00 UTC). You can also trigger it manually from the GitHub Actions tab.

---

## What you'll set up

- A GitHub repository with this workflow
- GitHub Secrets:
  - `MONGO_URI` – your Atlas connection string (read-only user recommended)
  - `RCLONE_TOKEN` – JSON token for a configured `gdrive` remote from rclone
  - *(optional)* `MONGO_DB` – dump a **single** database instead of all
  - *(optional)* `DRIVE_PATH` – e.g., `gdrive:Backups/Mongo/MyApp`
  - *(optional)* `RETENTION_DAYS` – e.g., `14` (defaults to 14 if not set)

---

## 1) Create a read-only user in Atlas

Give it `readAnyDatabase` (or per-db read roles).
Then build a connection string (SRV is OK) like:

```
mongodb+srv://backup_reader:YOURPASS@cluster0.xxxxx.mongodb.net
```

Put this in GitHub Secrets as **`MONGO_URI`**.

> **Network access note:** GitHub runner IPs are dynamic. In Atlas, temporarily allow access from `0.0.0.0/0` or use a stable runner behind a fixed IP. You can restrict by username/password and time-bound allowlisting if needed.

---

## 2) Get an rclone Google Drive token

On your local machine:

1. Install rclone from https://rclone.org
2. Run `rclone config`
   - `n` → new remote
   - name: `gdrive`
   - storage: `drive`
   - Use auto config and log into the Google account that owns the target Drive
3. Grab the generated token JSON at `~/.config/rclone/rclone.conf`, section `[gdrive]`:
   ```
   [gdrive]
   type = drive
   token = {"access_token":"ya29...","token_type":"Bearer",...}
   ```
4. Copy the entire **`token = {...}`** JSON (only the JSON value) into a new GitHub Secret named **`RCLONE_TOKEN`**.

> Alternative (advanced): use a Google **Service Account** and set up rclone with `service_account_file`. This avoids expiring user tokens but requires Drive sharing to the SA. Not covered here to keep it simple.

---

## 3) Optional secrets

- **`MONGO_DB`**: Only dump this DB (`mongodump --db`). If not set, all databases are dumped.
- **`DRIVE_PATH`**: Drive path (default: `gdrive:Backups/Mongo`).
- **`RETENTION_DAYS`**: Files older than this on Drive are deleted (default 14).

---

## 4) Schedule

The workflow's cron is `0 21 * * *` (21:00 UTC), which equals **02:30 Asia/Colombo** the next day. Adjust if you like.

---

## How restore works

1. Download your `.tar.gz` from Drive.
2. Extract:
   ```bash
   tar -xzf mongo-backup-YYYY-MM-DDTHH-MM-SS.tar.gz -C /tmp/mongo-restore
   ```
3. Restore **all DBs**:
   ```bash
   mongorestore --uri="mongodb+srv://USER:PASS@cluster0.xxxxx.mongodb.net" --gzip /tmp/mongo-restore
   ```
4. Or restore a **single DB** from the extracted folder:
   ```bash
   mongorestore --uri="mongodb+srv://USER:PASS@cluster0.xxxxx.mongodb.net" --db target_db --gzip /tmp/mongo-restore/your_db
   ```

> Tip: Always test a restore on a staging cluster.

---

## Security best practices

- Use a **read-only** backup user.
- Limit Atlas network access (time-boxed allowlisting when the job runs).
- Consider **encrypting archives** before upload if data is sensitive (e.g., add a `gpg -c` step).
- Keep your GitHub Secrets updated; rotate credentials periodically.

---

## Trigger a manual run

Go to **Actions → MongoDB Atlas → Google Drive Backup → Run workflow**.

---

## Troubleshooting

- **Auth / IP errors:** Check Atlas Network Access rules; GitHub runner egress IPs are dynamic.
- **rclone "invalid_grant":** Recreate `RCLONE_TOKEN` by re-running `rclone config` locally.
- **Large dumps:** Consider increasing runner disk or excluding heavy collections with mongodump flags.
