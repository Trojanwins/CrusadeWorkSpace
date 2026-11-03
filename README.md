# CrusadeOpsWorkspace

Workspace pengembangan lokal untuk **Crusade Digital Nusantara** — mencakup aplikasi
**FTTH Work Order**, runtime **TransformBot AI (Crusade Team 7)**, dan **Crusade Living
Office V13–V17**.

> ⚠️ **Ini adalah salinan pengembangan lokal — BUKAN production.**
> Jangan terhubung ke `C:\xampp\htdocs` asli, database customer, WhatsApp live, Mikrotik,
> OLT, ACS, bank, atau VPS produksi. Status rilis: **RELEASE NO-GO / DEPLOY NO-GO.**
> Empat gate persetujuan (customer data, credentials, live gateway, production deployment)
> tetap **PENDING**. Selalu baca [`AGENTS.md`](AGENTS.md:1) dan [`RULES.md`](RULES.md:1) lebih dulu.

Dokumentasi terkait:

| Dokumen | Isi |
|---|---|
| [`RULES.md`](RULES.md:1) | Aturan kode, arsitektur, keamanan, dan AGENTS.md |
| [`DESAIN.md`](DESAIN.md:1) | Standar visual, UI/UX, komponen, responsivitas |
| [`PRD.md`](PRD.md:1) | Fitur & alur pengguna seluruh modul + status implementasi |
| [`AGENTS.md`](AGENTS.md:1) | Governance keselamatan workspace (WAJIB) |
| [`docs/API-CONTRACT.md`](docs/API-CONTRACT.md:1) | Kontrak API internal (HMAC, nonce, idempotency) |
| [`docs/LOCAL-DEVELOPMENT.md`](docs/LOCAL-DEVELOPMENT.md:1) | Profil development terisolasi (sumber acuan utama) |
| [`docs/CRUSADE-MASTER-MISSION-CONTINUITY.md`](docs/CRUSADE-MASTER-MISSION-CONTINUITY.md:1) | Checkpoint misi & evidence |

---

## 1. Struktur Proyek

```
CrusadeOpsWorkspace/
├── ftth_work_order/         # Aplikasi Laravel FTTH Work Order (dev port 8011)
├── crusade_team7_staging/   # Runtime TransformBot / Team 7 (dev port 8012)
├── whatsapp_bot_ai/         # Sumber WhatsApp Bot AI + wa-gateway (gateway disabled)
├── deployment/              # Template staging & production (placeholder-only)
│   ├── staging/
│   └── production/
├── docs/                    # Kontrak, keputusan owner, checkpoint, evidence
│   └── test-results/        # Hasil verifikasi & evidence JSON
├── tools/                   # Harness verifikasi, inventori, generator, testing
├── Crusade_Living_Office_V13..V17_*.html
├── AGENTS.md                # Governance keselamatan (WAJIB dibaca)
├── RULES.md  DESAIN.md  PRD.md  README.md
└── 0                        # (file penanda existing — jangan dihapus)
```

Kedua aplikasi Laravel **terpisah**: punya `vendor/` sendiri, database sendiri, dan
**tidak** berbagi runtime database atau session cookie.

---

## 2. Persyaratan

| Perangkat | Versi |
|---|---|
| PHP | **8.2.12** (development). Workstation contoh: `D:\xampp\php\php.exe` |
| Composer | via `tools/composer.phar` (bukan instalasi global) |
| Extensions | `pdo_sqlite`, `pdo_mysql` (harness MariaDB opsional) |
| Python | 3.x (harness/verifikasi) |
| Node.js | hanya untuk Baileys gateway (tidak diperlukan untuk uji aplikasi) |
| Framework | Laravel **12.69.0**, PHPUnit **11.5.56** (terkunci di lockfile) |

---

## 3. Instalasi & Menjalankan Aplikasi

### 3.1 Menjalankan (dari workspace, dua terminal terpisah)

```powershell
python -X utf8 tools/run-development.py transformbot
python -X utf8 tools/run-development.py ftth
```

- TransformBot: `http://127.0.0.1:8012/crusade-team7`
- FTTH: `http://127.0.0.1:8011/ftth`
- Login dummy (fixture publik lokal, **bukan** kredensial nyata):
  `owner@example.invalid` / `Dummy-Only-2026`
- Launcher hanya mengikat **loopback**, menyetel `TMP/TEMP` ke `tools/tmp`, dan
  `Ctrl+C` hanya menghentikan child yang ia jalankan. Tidak ada gateway/queue/scheduler/
  Night Shift yang dijalankan.

**Prasyarat:** database dummy development harus sudah diinisialisasi (lihat §3.2).

### 3.2 Dependensi & inisialisasi (hanya jika `vendor/` belum ada)

Hanya pada checkout lokal terpisah yang sudah direview:

```powershell
$env:COMPOSER_HOME = Join-Path (Get-Location) 'tools/composer-home'
$env:COMPOSER_CACHE_DIR = Join-Path (Get-Location) 'tools/composer-cache'
$env:COMPOSER = 'composer.json'
php tools/composer.phar install --working-dir=ftth_work_order --no-scripts --no-plugins --prefer-dist --no-interaction
$env:COMPOSER = 'composer-development.json'
php tools/composer.phar install --working-dir=crusade_team7_staging --no-scripts --no-plugins --prefer-dist --no-interaction
$env:COMPOSER = 'composer.json'
php tools/prepare-ftth-manifest.php
php tools/prepare-transformbot-manifest.php
php tools/init-development.php ftth
php tools/init-development.php transformbot
```

- Composer PHAR diambil dari HTTPS resmi dengan verifikasi SHA256.
- **Jangan** melakukan Composer update global/tanpa review.
- `init-development` memeriksa path SQLite khusus, menjalankan migrasi yang sesuai, dan
  `firstOrCreate` fixture dummy. Ia **tidak** memuat `.env`, tidak mereset/menghapus DB.
- Database development terpisah:
  `ftth_work_order/database/ftth_work_order_dev.sqlite` dan
  `crusade_team7_staging/database/transformbot_dev.sqlite`.

---

## 4. Konfigurasi Aman

- **Jangan** menaruh secret/contact/kredensial nyata di repo. Template memakai marker
  `__PROVISION_*`.
- Notifikasi default `fake` (`FTTH_TRANSPORT=fake`); WhatsApp live **disabled** di staging.
- API write lokal **default off**: `FTTH_API_LOCAL_WRITE_ENABLED=false` dan pada
  TransformBot `TRANSFORMBOT_FTTH_WRITE_API_ENABLED=0`. Route write hanya terdaftar di
  `local`/`testing`.
- AP-8 provisioning contact staging (mekanisme READY, nilai menunggu Owner):

  ```powershell
  powershell -File tools/ap8-provision-contacts.ps1   # intake input tersembunyi, menulis di luar repo
  powershell -File tools/ap8-verify-contacts.ps1      # validasi, tidak mengirim apa pun
  ```

  Lihat [`docs/AP8-PROVISIONING-RUNBOOK.md`](docs/AP8-PROVISIONING-RUNBOOK.md:1).
- Identitas signing/encryption statis hanya fixture development (loopback-guarded) dan
  **tidak layak** untuk staging/production.

---

## 5. Testing

### 5.1 Unit/Feature test (dari root aplikasi)

```powershell
# FTTH (dari ftth_work_order)
D:\xampp\php\php.exe vendor\bin\phpunit -c phpunit.xml

# TransformBot — profil development (dari crusade_team7_staging)
D:\xampp\php\php.exe vendor\bin\phpunit tests/Development --no-coverage
```

- FTTH `phpunit.xml`: `APP_ENV=testing`, SQLite `:memory:`, `FTTH_BRIDGE_SECRET` sintetis.
- Test **wajib** dijalankan dari root aplikasi yang sesuai (bukan dari workspace root).
- TransformBot: `tests/Development` **tidak** terdaftar di `<testsuite>` default, jadi
  panggil dengan path eksplisit (bukan `phpunit` polos).
- Suite `tests/Feature/ExampleTest` (default `Tests\TestCase`) memerlukan `APP_KEY` dan
  tidak memuat `bootstrap/development.php`; ini adalah artefak baseline, bukan regresi.

### 5.2 Harness verifikasi (dari workspace root)

```powershell
python -X utf8 tools/verify-development.py
python -X utf8 tools/verify-concurrency.py
python -X utf8 tools/verify-loopback.py
python -X utf8 tools/verify-security-remediation.py
python -X utf8 tools/verify-mariadb-development.py            # butuh MariaDB dummy terisolasi
```

- Hasil disimpan di `docs/test-results/`.
- Verifikasi menjalankan dua suite PHPUnit terisolasi, route discovery, syntax check, dan
  perbandingan hash sumber asli.
- Harness MariaDB menolak port terisi, memverifikasi `@@datadir`, dan memakai allowlist
  database test-only. **Jangan** mengarahkannya ke instance lain.

### 5.3 Lint/syntax PHP

```powershell
D:\xampp\php\php.exe -l path\to\file.php
```

---

## 6. Inisialisasi Database (development)

- SQLite file terpisah per aplikasi (§3.2); test memakai in-memory.
- Concurrency test memakai `ftth_work_order/database/ftth_concurrency_testing.sqlite`
  dengan event dummy saja.
- MariaDB dummy: instance terisolasi via `tools/` dengan datadir di `tools/tmp`; **bukan**
  database bersama/staging/production.

---

## 7. Catatan Penting

1. **NO DATA LOSS / NO REWRITE / NO AUDIT LOOP.** Pertahankan semua perubahan yang belum
   di-commit (modified & untracked).
2. **Test-first** untuk perbaikan bug; jangan melemahkan assertion.
3. Semua status harus jujur: `PASS`, `FAIL`, `NOT_RUN`, atau `BLOCKED`.
4. Jangan jalankan migrasi/deployment, jangan akses VPS, jangan aktifkan WhatsApp asli.
5. Code path yang dilindungi (CS Master Prompt V4, Shifting Rolling, tugas agent aktif)
   tidak boleh diubah tanpa otorisasi khusus.
6. Skrip `tools/build-*`, `extract-*`, dan transformasi lain adalah **riwayat
   implementasi**, bukan perintah instalasi — jangan dijalankan ulang atas sumber ter-review.
7. Baileys gateway `whatsapp_bot_ai/wa-gateway` **disabled** di staging dan bukan
   komponen rilis lengkap saat ini.

---

## 8. Dukungan & Rujukan

- Perubahan alur/kontrak: lihat keputusan Owner di `docs/OWNER-DECISION-PACKAGE.md`.
- Kontrak API: [`docs/API-CONTRACT.md`](docs/API-CONTRACT.md:1).
- Operasi staging: [`docs/STAGING-OPERATIONS.md`](docs/STAGING-OPERATIONS.md:1).
- Kesiapan AP-8/D-8: [`docs/AP8-COMPLETION-D8-READINESS.md`](docs/AP8-COMPLETION-D8-READINESS.md:1).
