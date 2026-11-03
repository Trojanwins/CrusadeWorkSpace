# CRUSADE DIGITAL NUSANTARA — SOURCE BACKUP, VERIFY & RESTORE

Owner: **Azis Syaiful Anam**. Scope: local development snapshot under
`D:\Crusade-AI-Migration\CrusadeOpsWorkspace` backed up to
`D:\Crusade-AI-Migration\CrusadeOpsBackups`. This is **never** production. No
database, WhatsApp, VPS, Mikrotik, OLT, ACS or bank system is ever contacted.

This document is the human-readable entry point. Step-by-step operator
instructions live in [`docs/BACKUP-RESTORE-RUNBOOK.md`](docs/BACKUP-RESTORE-RUNBOOK.md:1)
and the disaster-recovery plan (including database and secret backup design) in
[`docs/VPS-DISASTER-RECOVERY-PLAN.md`](docs/VPS-DISASTER-RECOVERY-PLAN.md:1).

---

## 1. What this backup does and does not cover

| Category | Covered? | Notes |
|---|---|---|
| Application source (3 apps + tools + docs + deployment) | **YES** | Copy-only, includes uncommitted & untracked files |
| Safe schema definitions (migrations, `.sql.example`) | **YES** | Captured under `01-APPLICATIONS` |
| Customer / production database dump | **NO — `NOT_BACKED_UP`** | Separate Owner-approved procedure required |
| `.env`, credentials, API keys, tokens, private keys | **NO — `NOT_BACKED_UP`** | Excluded by denylist |
| Real WhatsApp session (`auth_info_baileys`, `whatsapp_sessions`) | **NO — `NOT_BACKED_UP`** | Fake/mock transport only |

**A source snapshot is NOT a full disaster recovery.** Database, secrets and
WhatsApp sessions are deliberately excluded and remain `NOT_BACKED_UP`.

---

## 2. Master backup structure

```
D:\Crusade-AI-Migration\CrusadeOpsBackups\
├─ snapshots\YYYYMMDDTHHMMSSZ\        # timestamped copy-only source snapshot
│  ├─ 01-APPLICATIONS\                # crusade_team7_staging, ftth_work_order, whatsapp_bot_ai
│  ├─ 02-DOCUMENTATION\               # docs, top-level MD/HTML
│  ├─ 03-DEPLOYMENT\                  # deployment/**, host/infra config examples
│  ├─ 04-DATABASE-BACKUPS\README.md   # declares NOT_BACKED_UP
│  ├─ 05-SECURE-BACKUPS\README.md     # declares NOT_BACKED_UP
│  ├─ 06-RESTORE-TOOLS\               # copies of the backup/verify/restore scripts
│  ├─ 07-TEST-EVIDENCE\               # test/baseline artifacts
│  └─ manifests\
│     ├─ manifest.csv                 # relativePath, workspacePath, sizeBytes, sha256
│     ├─ manifest.json                # same, JSON form
│     ├─ exclusions.csv               # Path, Reason for every excluded file
│     ├─ path-mapping.csv             # workspacePath -> snapshotPath
│     └─ snapshot-summary.json        # timestamp, counts, git branch/HEAD, NOT_BACKED_UP flags
├─ archives\
│  └─ Crusade-Digital-Nusantara-Master-Source-<TS>.zip(+ .sha256)
├─ restore-tests\                     # isolated drill targets (disposable)
└─ LAST-BACKUP-REPORT.txt             # latest run summary
```

The snapshot preserves the **full source-relative sub-path** inside each bucket
(e.g. `02-DOCUMENTATION/docs/API-CONTRACT.md`), so no two files with the same
leaf name can collide.

---

## 3. The toolkit

| Script | Purpose |
|---|---|
| [`tools/run-backup.ps1`](tools/run-backup.ps1:1) | One entrypoint: snapshot → ZIP → verify → (optional) restore drills |
| [`tools/backup-source.ps1`](tools/backup-source.ps1:1) | Copy-only snapshot + manifests (never `git archive`, never deletes source) |
| [`tools/verify-backup.ps1`](tools/verify-backup.ps1:1) | Offline read-only integrity, path-safety, secret and ZIP checks |
| [`tools/restore-source.ps1`](tools/restore-source.ps1:1) | Restore snapshot or ZIP into an empty isolated folder; re-hashes every file |

Every script exits **nonzero on any error**, validates relative paths, rejects
traversal/absolute/reparse abuse, refuses a non-empty target, refuses
overlapping source/destination, and never touches the workspace source.

---

## 4. How to run

Backup only:

```powershell
powershell -ExecutionPolicy Bypass -File tools\run-backup.ps1
```

Backup **and** real restore drills (both snapshot and portable ZIP):

```powershell
powershell -ExecutionPolicy Bypass -File tools\run-backup.ps1 -RestoreDrill
```

Custom master root:

```powershell
powershell -ExecutionPolicy Bypass -File tools\run-backup.ps1 -MasterRoot E:\CrusadeBackups -RestoreDrill
```

Preview what would be included without writing anything:

```powershell
powershell -ExecutionPolicy Bypass -File tools\backup-source.ps1 -DryRun
```

Restore a snapshot into a fresh isolated folder:

```powershell
powershell -ExecutionPolicy Bypass -File tools\restore-source.ps1 `
  -Snapshot D:\Crusade-AI-Migration\CrusadeOpsBackups\snapshots\<TS> `
  -Target   D:\Crusade-AI-Migration\CrusadeOpsBackups\restore-tests\drill-<TS>
```

Restore from the portable ZIP:

```powershell
powershell -ExecutionPolicy Bypass -File tools\restore-source.ps1 `
  -Zip    D:\Crusade-AI-Migration\CrusadeOpsBackups\archives\Crusade-Digital-Nusantara-Master-Source-<TS>.zip `
  -Target D:\Crusade-AI-Migration\CrusadeOpsBackups\restore-tests\zip-drill-<TS>
```

Verify an existing snapshot and ZIP:

```powershell
powershell -ExecutionPolicy Bypass -File tools\verify-backup.ps1 `
  -Snapshot D:\Crusade-AI-Migration\CrusadeOpsBackups\snapshots\<TS> `
  -Zip      D:\Crusade-AI-Migration\CrusadeOpsBackups\archives\Crusade-Digital-Nusantara-Master-Source-<TS>.zip
```

---

## 5. Verified evidence (real run)

Run `20261103T040307Z` (UTC), executed on this workspace:

- Snapshot: **2480 files**, **25,169,100 bytes**; candidates 52,585; excluded 50,105.
- Git context: branch `agent/v17-six-phase`, HEAD `c13c8a6f…`, dirty workspace = true.
- Verification: **17 checks, 0 failures** — all SHA-256 match, 0 traversal offenders,
  0 secret/session hits, ZIP entries forward-slash separated, ZIP file-set equals snapshot.
- Portable ZIP: `Crusade-Digital-Nusantara-Master-Source-20261103T040307Z.zip`,
  SHA-256 `5CDE6B83A086B34CC9A6A6F27273B729302C882CD3CC0358BAE7367D5E6F71B6`.
- Restore drill (snapshot): **2480 restored / 2480 verified**.
- Restore drill (ZIP): **2491 entries extracted / 2480 verified** against manifest.
- Total pipeline: ~330s.

### Edge-case tests (all behaved as designed)

| Test | Result |
|---|---|
| Restore into a non-empty target | **REFUSED** (exit 1) |
| Restore into a path inside the live workspace | **REFUSED** (exit 1), no directory created |
| ZIP containing a `../escape.txt` entry | **REJECTED** (exit 1), nothing written outside target |
| Manifest SHA-256 mismatch | **DETECTED** by both restore and verify (exit 1) |

---

## 6. Portability note

The portable master ZIP is written with **forward-slash** entry names (ZIP spec
compliant) so it extracts correctly on Linux/VPS (`unzip`). Backslash-separated
archives produced by legacy Windows tooling are still accepted by the restore
path, which classifies entries tolerant of both separators and empty-directory
entries, and extracts via stream copy rather than `ExtractToFile`.

---

## 7. Governance guarantees

- No source file is modified, moved or deleted by any script.
- No Git operation (no reset/clean/checkout/stash); branch, HEAD and working tree untouched.
- No database connection, no migration, no real WhatsApp send, no VPS/SSH action.
- No secret, `.env`, key, token or WhatsApp session is ever placed in a snapshot or ZIP.
- Customer-facing or destructive actions require explicit Owner approval.

---

## 8. Disaster-Recovery add-on (`tools/dr/`) — encrypted DB, secrets & session backup

This is the **prepared but not-yet-executed** DR layer. It adds encrypted, streaming
backups for the database, secrets and WhatsApp session, all **approval-gated**. Nothing
in this section has been run against real data — see the status table below.

### 8.1 The toolkit

| Script | Purpose |
|---|---|
| [`tools/dr/dr-common.ps1`](tools/dr/dr-common.ps1:1) | Shared crypto + safety library: AES-256-CBC + HMAC-SHA256 (encrypt-then-MAC), PBKDF2-HMAC-SHA256, DPAPI key files, retention, approval gate, streaming process encryptor, canonical `New-Dir` |
| [`tools/dr/crusade-dr.ps1`](tools/dr/crusade-dr.ps1:1) | Single fail-closed entrypoint with 8 modes (incl. `RUNBOOK`) |
| [`tools/dr/restore-drill.ps1`](tools/dr/restore-drill.ps1:1) | Synthetic restore harness (11 checks); Tier B delegates to the isolated MariaDB harness when `CRUSADE_MARIADB_ISOLATED=1` |
| [`tools/dr/mariadb-restore-drill.ps1`](tools/dr/mariadb-restore-drill.ps1:1) | Isolated, disposable MariaDB restore harness (18 checks): fresh datadir, random port/DB, server-identity guard, schema + FK + row-count validation, fail-closed real-archive gate |
| [`tools/dr/collect-secrets.ps1`](tools/dr/collect-secrets.ps1:1) | In-memory secrets bundle → encrypted `.drsec` (allow-listed paths only) |

### 8.2 Entrypoint modes

```powershell
# Read-only readiness report (touches no data, no secrets)
powershell -ExecutionPolicy Bypass -File tools\dr\crusade-dr.ps1 -Mode STATUS

# Existing source snapshot pipeline (delegates to run-backup.ps1)
powershell -ExecutionPolicy Bypass -File tools\dr\crusade-dr.ps1 -Mode SOURCE_ONLY

# Generate a reviewable DB backup plan (no DB contact, no data read)
powershell -ExecutionPolicy Bypass -File tools\dr\crusade-dr.ps1 -Mode DATABASE_PREPARE -Database whatsapp_bot

# Synthetic, isolated restore drill (safe; uses fake data only)
powershell -ExecutionPolicy Bypass -File tools\dr\crusade-dr.ps1 -Mode RESTORE_DRILL

# --- APPROVAL-GATED (refuse without an exact-scope Owner token) ---
powershell -ExecutionPolicy Bypass -File tools\dr\crusade-dr.ps1 -Mode DATABASE_BACKUP -Database whatsapp_bot -ApprovalToken <path>
powershell -ExecutionPolicy Bypass -File tools\dr\crusade-dr.ps1 -Mode SECURE_BACKUP  -ApprovalToken <path>

# Verify a CRUDRE01 encrypted archive (header + HMAC + optional decrypt)
powershell -ExecutionPolicy Bypass -File tools\dr\crusade-dr.ps1 -Mode VERIFY -Archive <file.drdb> -KeyFilePath <key> -ConfirmDecrypt

# Print the operator runbook (step order, gates, exact commands)
powershell -ExecutionPolicy Bypass -File tools\dr\crusade-dr.ps1 -Mode RUNBOOK
```

Isolated MariaDB restore tier (disposable instance; XAMPP 127.0.0.1:3306 is never contacted):

```powershell
# Tier B of the restore drill runs the live harness when this env var is set
$env:CRUSADE_MARIADB_ISOLATED = '1'
powershell -ExecutionPolicy Bypass -File tools\dr\crusade-dr.ps1 -Mode RESTORE_DRILL
```

The real-archive restore is **prepared, approval-gated and NOT run automatically** — see
[`docs/dr/RESTORE-REAL-ARCHIVE-COMMAND.md`](docs/dr/RESTORE-REAL-ARCHIVE-COMMAND.md:1).

### 8.3 Encryption & key separation

- Cipher: **AES-256-CBC** (PKCS7) over the body; **HMAC-SHA256** over the *ciphertext*
  (encrypt-then-MAC). Archive magic `CRUDRE01`.
- Key: **PBKDF2-HMAC-SHA256** (200000 iterations) → 64 bytes (32 AES + 32 HMAC). A wrong
  key is rejected by HMAC **before** any output file is created.
- Optional Windows **DPAPI** key file (`CurrentUser`/`LocalMachine`). DPAPI is
  machine/user bound — it is **not** portable across machines until tested; use a
  passphrase (portable) key for cross-VPS recovery.
- **The recovery key is stored OUTSIDE the backup root** (`-KeyRoot`,
  default `D:\Crusade-AI-Migration\CrusadeOpsRecoveryKeys`). Without the key the encrypted
  bundle is meaningless. Keep at least two key holders (Owner).

### 8.4 Current DR status (honest, as executed)

| Component | Status | Note |
|---|---|---|
| Source backup/restore | **READY / VERIFIED** | Existing pipeline, run `20261103T040307Z` |
| DR toolkit (crypto + modes) | **READY / TESTED** | Synthetic round-trip, wrong-key, tamper, misses |
| Synthetic restore drill | **PASS** | `restore-drill.ps1` exit 0, 11 pass / 0 fail / 0 not-run |
| Isolated MariaDB restore harness | **PASS** | `mariadb-restore-drill.ps1` exit 0, 18/18 (synthetic); MariaDB 10.4.32 |
| DATABASE_BACKUP | **READY (archive)** | Existing verified archive; real-archive restore approval-gated |
| SECURE_BACKUP | **NOT_READY** | Procedure prepared; requires Owner approval |
| Whatsapp session | **NOT_READY** | Procedure prepared; requires Owner approval |
| **FULL DR** | **NOT_READY** | Until secrets + session are backed up **and** a restore is verified |

**A source snapshot plus a tested DR toolkit is still NOT full disaster recovery.**
