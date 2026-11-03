# DESAIN.md — Crusade Digital Nusantara

Standar visual, UI/UX, layout, komponen, responsivitas, dan konsistensi desain
antar-modul di dalam `CrusadeOpsWorkspace`.

Dokumen ini **mendokumentasikan** konvensi yang **sudah ada di sumber** (Blade views),
bukan menetapkan desain baru. Perubahan visual yang menghadap customer wajib disetujui
Owner, dan tetap mengikuti [`RULES.md`](RULES.md:1) serta [`AGENTS.md`](AGENTS.md:1).

Sumber acuan:

- FTTH shell/layout: [`ftth_work_order/resources/views/ftth/layouts/app.blade.php`](ftth_work_order/resources/views/ftth/layouts/app.blade.php:1)
- Team 7 shell: [`crusade_team7_staging/resources/views/team7/inquiries/layout.blade.php`](crusade_team7_staging/resources/views/team7/inquiries/layout.blade.php:1) dan [`crusade_team7_staging/resources/views/team7/proposals/layout.blade.php`](crusade_team7_staging/resources/views/team7/proposals/layout.blade.php:1)
- Living Office: `Crusade_Living_Office_V13..V17_*.html`

---

## 1. Prinsip Desain

Aplikasi tetap **terpisah** tetapi memakai **identitas visual yang konsisten**
(lihat [`docs/CRUSADE-TEAM7-MASTER-BASELINE-V1.md`](docs/CRUSADE-TEAM7-MASTER-BASELINE-V1.md:15)).

1. **Dark-first** — seluruh antarmuka menggunakan tema gelap (network/NOC operational).
2. **Data-dense tapi tenang** — tabel, kartu statistik, dan badge status dominan.
3. **Status selalu terlihat** — warna semantik untuk sukses, peringatan, dan bahaya.
4. **Draft-only aman** — modul yang belum final selalu menampilkan banner eksplisit.
5. **Read-only saat ragu** — aksi berisiko diberi konfirmasi/penolakan jelas, bukan silent.

---

## 2. Token Warna

### 2.1 Tema operasional FTTH (neon/cyber-tech)

Berasal dari `:root` FTTH layout. Dipakai untuk dashboard operasional, work order,
verifikasi NOC/Finance, testing/scheduling.

| Token | Nilai | Penggunaan |
|---|---|---|
| `--cyber` | `#00e5c3` | Aksen utama, garis aktif, ikon |
| `--cyber2` | `#00b8ff` | Aksen sekunder/gradien |
| `--cyber-soft` | `rgba(0,229,195,.10)` | Latar lembut elemen aktif |
| `--bg` | `#050a12` | Latar halaman |
| `--bg2` | `#08111d` | Latar lapisan |
| `--panel` / `--panel2` | `#0b1522` / `#0e1b2b` | Panel & kartu |
| `--sidebar` | `#050b14` | Sidebar |
| `--ink` | `#e5f4f2` | Teks utama |
| `--muted` | `#78909c` | Teks sekunder |
| `--line` | `rgba(100,180,190,.14)` | Garis pemisah |
| `--success` | `#00e59b` | Status sukses / NOC |
| `--warning` | `#ffb020` | Status peringatan |
| `--danger` | `#ff4d6d` | Status bahaya / logout |
| `--info` | `#00b8ff` | Informasi |

### 2.2 Tema Team 7 (mint enterprise)

Berasal dari `:root` layout inquiries/proposals. Dipakai untuk Customer Project Intake,
Proposal Workbench, dan modul Customer Solution Capability.

| Token | Nilai | Penggunaan |
|---|---|---|
| `--bg` | `#090f1b` | Latar halaman |
| `--card` | `#152137` | Panel/kartu |
| `--line` | `#283951` | Garis pemisah & border |
| `--ink` | `#edf4ff` | Teks utama |
| `--muted` | `#9cacc6` | Teks sekunder |
| `--mint` | `#31d2ad` | Aksen utama & tombol primary |
| `--warn` | `#ffcc78` | Peringatan |
| `--red` | `#ff969e` | Bahaya/error |

### 2.3 Aturan warna

- Warna **hanya** diambil dari token di atas; jangan membuat hex baru tanpa alasan.
- Gunakan `color-scheme:dark` sehingga kontrol native (select, date) tampil gelap.
- Warna semantik tidak boleh ditukar: hijau/mint = aman, kuning = perlu perhatian,
  merah = gagal/berbahaya.

---

## 3. Tipografi

| Konteks | Font | Catatan |
|---|---|---|
| FTTH body | `'Plus Jakarta Sans', sans-serif` | UI umum |
| FTTH teknis/angka | `'JetBrains Mono', monospace` | Kode, angka statistik, label sistem, badge |
| Team 7 | `system-ui,-apple-system,Segoe UI,sans-serif` | Ringan, tanpa CDN font |
| Living Office | mengikuti file HTML masing-masing | — |

Skala ukuran yang dipakai di sumber:

- Judul halaman FTTH: `~1.08rem`, bobot 800.
- Sub-judul teknis: `~0.61rem`, monospace, `letter-spacing`.
- Body Team 7: `14px/1.55`; `h1` `clamp(22px,3vw,30px)`.
- Angka statistik: monospace `~1.55rem`, bobot 700.

> Catatan: FTTH memuat font dari CDN opsional. Permintaan fungsional/API **tidak**
> bergantung pada aset CDN ini (lihat [`docs/LOCAL-DEVELOPMENT.md`](docs/LOCAL-DEVELOPMENT.md:46)).

---

## 4. Layout

### 4.1 FTTH — sidebar tetap + topbar

- Sidebar kiri tetap (`275px`) berisi brand, menu bergrup, dan box user.
- Topbar sticky (`76px`, `backdrop-filter: blur`) berisi judul halaman dan badge NOC.
- Konten utama: `margin-left:275px`, padding `26px 30px 60px`.
- Latar dekoratif: grid halus + scanline pada `body:before`/`body:after` (pointer-events none).

### 4.2 Team 7 — satu kolom terpusat

- Kontainer `.main`: `max-width:1100px`, `margin:0 auto`, padding `27px clamp(18px,3vw,42px)`.
- Tanpa sidebar; navigasi lewat tautan kembali ("← Kembali ke ... Command Center").
- Header standar: eyebrow → h1 (dengan sub-label versi) → paragraf `.sub`.

### 4.3 Status & isolasi

- Modul yang belum final diberi penanda eksplisit di judul (mis. "V1 · draft-only").
- Modul internal/draft memakai **banner** permanen (lihat §6.4).

---

## 5. Komponen

Daftar komponen dan kelas yang sudah dipakai di sumber:

| Komponen | Kelas/kunci | Sumber |
|---|---|---|
| Kartu statistik | `.stat-card`, `.stat-icon`, `.stat-number`, `.stat-label` | FTTH |
| Kartu umum | `.card`, `.card-header`, `.card-body` | FTTH |
| Navigasi samping | `.ftth-nav`, `.ftth-nav.active`, `.ftth-nav.disabled` | FTTH |
| Badge NOC | `.noc-badge`, `.noc-dot` (animasi `nocPulse`) | FTTH |
| Badge peran/user | `.user-box`, `.user-avatar`, `.user-name`, `.user-role` | FTTH |
| Form field | `.form-control`, `.form-select`, `.field` | FTTH & Team 7 |
| Tombol | `.btn`, `.btn.primary`, `.btn.danger` | Team 7 |
| Notifikasi | `.notice`, `.notice.warn`, `.notice.err` | Team 7 |
| Tag status | `.tag`, `.tag.ready/.approved/.rejected/.failed/.draft` | Team 7 |
| Panel | `.panel` | Team 7 |
| Hasil teks | `.textResult` (`white-space:pre-wrap`) | Team 7 |

### 5.1 Tag/badge status

Warna tag dipetakan ke makna status. Untuk pipeline Team 7
(`PENDING → DRAFT_READY → APPROVED/REJECTED`, atau `FAILED`):

| Status | Gaya |
|---|---|
| `DRAFT_READY` | `.tag.ready` (hijau) |
| `APPROVED` | `.tag.approved` (biru) |
| `REJECTED` | `.tag.rejected` (merah) |
| `FAILED` | `.tag.failed` (kuning) |
| `DRAFT` | `.tag.draft` (kuning pucat) |

---

## 6. UI/UX

### 6.1 Prinsip interaksi

- Aksi destruktif/berisiko (cancel, revoke, hapus) diberi gaya `.btn.danger` dan
  wajib konfirmasi; jangan pernah silent.
- Review/approve **single-use**: setelah diproses, kontrol dinonaktifkan dan status
  ditampilkan sebagai teks, bukan tombol aktif.
- Idempotency harus terlihat: submit ganda tidak membuat data ganda.

### 6.2 Umpan balik & pesan

- Sukses: `.notice` (`role="status"`).
- Peringatan: `.notice.warn` (`role="alert"`).
- Error/validasi: `.notice.err` (`role="alert"`), tampilkan pesan pertama.
- Semua pesan memakai bahasa Indonesia yang jelas dan spesifik.

### 6.3 Aksesibilitas

- Gunakan elemen semantik (`<main>`, `<h1>`, `<table>`, `<label for>`).
- Setiap input punya `<label>` eksplisit.
- Kontras teks terhadap latar gelap harus memadai (token `--ink`/`--muted`).
- Gunakan atribut `role` pada notifikasi. FTTH menyetel
  `<meta name="robots" content="noindex,nofollow,noarchive,nosnippet,noimageindex">`.

### 6.4 Banner draft-only (WAJIB)

Setiap modul yang menghasilkan draf non-final menampilkan banner permanen. Contoh
yang sudah dipakai Proposal Workbench:

```html
<div class="banner" role="note">NON-BINDING DRAFT — BUKAN PENAWARAN, BUKAN KONTRAK</div>
```

Banner harus menyatakan secara eksplisit: bukan harga final, bukan kontrak, bukan
komitmen tenggat, dan tidak memicu work order/pembayaran/WhatsApp.

---

## 7. Responsivitas

- **Viewport wajib**: setiap layout menyetel `<meta name="viewport" content="width=device-width, initial-scale=1">`.
- **Fluid spacing Team 7**: gunakan `clamp()` untuk padding & ukuran judul.
- **FTTH mobile**: terdapat kontrol `.mobile-menu` (`display:none` di desktop) untuk
  membuka/menutup sidebar pada layar kecil; sidebar tetap `275px` di desktop.
- **Tabel**: gunakan padding sel proporsional (`10px 8px`) dan font `13px`; hindari
  overflow horizontal — bungkus dalam kontainer yang dapat menyesuaikan.
- **Gambar/logo**: `place-items:center` pada wadah grid; jangan menyetel ukuran piksel
  tetap pada konten yang perlu menyesuaikan.

---

## 8. Konsistensi Antar-Modul

Agar modul tetap terasa satu produk meski aplikasinya terpisah:

1. **Identitas brand**: eyebrow `Crusade Digital Nusantara / <kapabilitas>` pada
   header modul Team 7; brand block pada sidebar FTTH.
2. **Pola header sama**: eyebrow kecil → judul besar (+ label versi) → deskripsi singkat.
3. **Pola navigasi kembali**: tautan di bawah konten menuju panel/command center induk.
4. **Bahasa**: antarmuka memakai Bahasa Indonesia; istilah teknis/status dalam huruf
   kapital (mis. `DRAFT_READY`, `APPROVED`).
5. **Tidak menyalin aset berat** antar aplikasi CDN; gunakan token warna yang setara,
   bukan file CSS yang sama, untuk menjaga isolasi runtime.
6. **Living Office** (V13–V17) memakai identitas visual Crusade dan dipakai sebagai
   presentasi/meeting; jangan mengubah jadwal/rolling di dalamnya.

---

## 9. Batasan

- Dokumen ini **tidak** mengubah view, CSS, atau aset apa pun; statusnya dokumentasi.
- Uji visual/browser penuh (aksesibilitas, load) tetap **NOT_RUN** dan bukan bagian
  dari klaim dokumen ini.
- Perubahan desain yang menghadap customer memerlukan persetujuan Owner.
