# PRD.md — Product Requirements Document

**Produk:** Crusade Digital Nusantara — platform operasional FTTH, TransformBot AI, dan
Crusade Team 7 (Living Office).

**Workspace:** `CrusadeOpsWorkspace` — salinan pengembangan lokal, **bukan production**.

> **Penting:** Dokumen ini mencatat fitur dan alur pengguna **beserta status implementasi
> yang sesuai kenyataan**. Status tidak boleh dibesar-besarkan. Empat gate persetujuan
> (customer data, credentials, live gateway, production deployment) tetap **PENDING**.
> **RELEASE NO-GO / DEPLOY NO-GO.** Lihat [`RULES.md`](RULES.md:1) dan
> [`docs/CRUSADE-MASTER-MISSION-CONTINUITY.md`](docs/CRUSADE-MASTER-MISSION-CONTINUITY.md:1).

Legenda status:

| Status | Arti |
|---|---|
| ✅ **Implemented (local)** | Ada di sumber dan diuji lokal (sintetis/fake) |
| 🟡 **Partial** | Sebagian alur/edge-case belum lengkap atau uji terbatas |
| 🔵 **Design only** | Desain/proposal, belum ada implementasi |
| ⛔ **Blocked** | Menunggu keputusan Owner / schema authoritative / non-production hanya |

---

## 1. Tujuan Produk

Menyediakan satu ekosistem operasional yang mempertahankan sistem yang sudah ada:

1. **FTTH Work Order** — manajemen work order lapangan (customer, teknisi, penjadwalan,
   instalasi, maintenance, pembayaran teknisi, referral, laporan).
2. **TransformBot AI (Team 7)** — asisten/draft AI internal dengan human review, alur
   draft-only, dan kapabilitas Customer Solution.
3. **Crusade Living Office** — pusat koordinasi/visibilitas tim (V13–V17).

Prinsip lintas produk: **LANJUTKAN YANG SUDAH ADA — JANGAN MULAI DARI NOL**, AI hemat
token, dan human-in-the-loop untuk keputusan berisiko.

---

## 2. Persona & Peran

| Peran | Deskripsi |
|---|---|
| **Owner** | Pihak yang menyetujui perubahan, rilis, akses data, dan tindakan berisiko |
| **NOC** | Verifikasi teknis work order, approval teknis |
| **Finance** | Verifikasi/pembayaran teknisi |
| **Supervisor** | Review hasil kerja lapangan (approve / kembalikan revisi) |
| **Teknisi** | Menerima & mengerjakan work order, melaporkan hasil |
| **Customer Service (CS)** | Alur CS + master prompt CS (dilindungi) |
| **Customer / Calon Customer** | Pihak yang mengajukan permintaan layanan/proyek |

---

## 3. FTTH Work Order

Aplikasi Laravel mandiri di [`ftth_work_order/`](ftth_work_order/README.md:1). Port dev
**8011**, path `/ftth`.

### 3.1 Modul & status

| Fitur | Deskripsi | Status |
|---|---|---|
| Dashboard FTTH | Ringkasan operasional | ✅ Implemented (local) |
| Manajemen Work Order | Create, list, detail, status lifecycle | ✅ Implemented (local) |
| Verifikasi NOC | Antrean verifikasi teknis + approval | ✅ Implemented (local) |
| Verifikasi Finance | Antrean & otorisasi pembayaran | ✅ Implemented (local) |
| **Supervisor (review lapangan)** | Antrean supervisor; approve hasil atau kembalikan revisi | ✅ Implemented (local, owner-approved 2026-10-31) |
| Penjadwalan (scheduling) | Kapasitas, dispatch, jadwal instalasi | ✅ Implemented (local) |
| Laporan Instalasi | Report + ekspor XLSX (streaming, untrusted string typed) | ✅ Implemented (local) |
| Laporan Maintenance | Report maintenance | ✅ Implemented (local) |
| Pembayaran Teknisi | Payment report & web | ✅ Implemented (local) |
| Referral Teknisi | Skema referral & closing fee | ✅ Implemented (local) |
| Manajemen Teknisi | Data & aplikasi teknisi | ✅ Implemented (local) |
| Notifikasi (dispatcher) | Idempotent event claim, ACK uncertainty, retry-block | ✅ Implemented (fake transport) |
| Kanban/Komparasi Waktu | — | ⛔ Blocked (schema authoritative) |
| Pembatalan Work Order (cancellation) | Proposal skema pembatalan | 🔵 Design only (`docs/FTTH-CANCELLATION-SCHEMA-PROPOSAL.md`) |

### 3.2 Alur pengguna — Work Order (aktual)

```
Customer/CS  ->  Create Work Order (ASSIGNED)  ->  Teknisi mengerjakan
     ->  Laporan hasil lapangan (SUBMITTED)
     ->  Supervisor review  ->  APPROVED  (audit SUPERVISOR, tanpa payment)
                            ->  REVISION  (kembali ke work order yang sama, revision_notes)
     ->  Verifikasi NOC     ->  approval teknis (+ coupling pembayaran)
     ->  Finance            ->  otorisasi & pembayaran teknisi
```

Catatan penting:

- **Supervisor** memakai ulang `work_order_approvals` dengan diskriminator
  `approval_type = SUPERVISOR`; **tidak** membuat `TechnicianPayment`.
- Supervisor **dilarang** mengakses approval NOC, payment finance, dan antrean verifikasi NOC.
- Approve/revisi bersifat **single-use** (aksi berulang ditolak; status tidak berubah).
- **Pembatalan work order belum diimplementasikan** — pemegang kewenangan pembatalan
  adalah keputusan Owner yang belum final.

### 3.3 FTTH API (service-to-service)

Dua label protokol yang **koeksis** (lihat [`docs/API-CONTRACT.md`](docs/API-CONTRACT.md:24)):

| Label | Metode | Status |
|---|---|---|
| `FTTH-API-V1` | `GET /api/v1/technicians`, `GET /api/v1/work-orders`, `POST /api/v1/work-orders` | ✅ Read implemented; write **local/testing only, default off** |
| `FTTH-BRIDGE-V1` | `POST /api/internal/team7/draft`, `POST /api/internal/technician-replies`, `POST /api/internal/ftth/notifications` | ✅ Implemented (development, signed, fake transport) |

- Write route `POST /api/v1/work-orders` hanya terdaftar di `local`/`testing`, butuh
  `FTTH_API_ENABLED` **dan** `FTTH_API_LOCAL_WRITE_ENABLED` (default `false`).
- Idempotency: key disimpan sebagai SHA-256; klaim + baris domain + receipt + audit
  dalam satu transaksi; replay identik mengembalikan receipt tersimpan.
- Tidak memicu notifikasi/pembayaran/approval.

### 3.4 FTTH-001 — `sendFieldOnce`

- `FtthSchedulingService::sendFieldOnce` adalah wrapper tipis dengan whitelist event
  (`PSB_CAPACITY_REQUEST`, `PSB_CUSTOMER_ASSIGNED`) yang mendelegasikan ke
  `NotificationDispatcher::send()`.
- Coverage: ✅ event `PSB_CUSTOMER_ASSIGNED`, event tak didukung, replay non-confirmed,
  dan throwable in-try (via `SendFieldOnceCoverageTest`).
- Gap tersisa: ⛔ integrasi berbasis **MariaDB** (`NOT_RUN`, butuh persetujuan Owner).

---

## 4. TransformBot AI (Crusade Team 7)

Runtime Laravel di [`crusade_team7_staging/`](crusade_team7_staging/README.md:1). Port dev
**8012**, path `/crusade-team7`. Profil development terisolasi (bootstrap/provider/routes
khusus development); CS Master Prompt V4, Shifting Rolling, dan provider asli **tidak diubah**.

### 4.1 Modul & status

| Fitur | Deskripsi | Status |
|---|---|---|
| Bot WhatsApp AI | Multi-nomor, auto-reply, broadcast, riwayat chat | ✅ Implemented (source); gateway **disabled** di staging |
| Draft bridge Team 7 | `ANALYZE → DRAFT → VERIFY → PENDING_APPROVAL` (human review) | ✅ Implemented (fake provider) |
| Command Center Team 7 | Panel operasional Team 7 | ✅ Implemented (local) |
| Crusade Office | Koordinasi workflow/assignment/event lintas scope | ✅ Implemented (local) |
| **Customer Project Intake V1** | Penerimaan permintaan proyek, draft-only | ✅ Implemented (local) |
| **Customer Requirements & Project Brief V1** | Brief deterministik, pemisahan terkonfirmasi vs asumsi | ✅ Implemented (local) |
| **Proposal Workbench V1** | Rekomendasi + draf proposal internal (owner-only, draft-only) | ✅ Implemented (local) |
| **Customer-Project Handoff V1** | Link proposal→workflow, authorization, release approval | ✅ Implemented (local) |
| AI Provider asli | Gemini/Groq dsb. | ⛔ Disabled — gunakan `DevelopmentAiProvider` (dummy text) |
| WhatsApp live | Gateway Baileys | ⛔ Disabled — fake transport only |

### 4.2 Alur pengguna — Customer Solution Capability (aktual)

```
Calon Customer
  -> Customer Project Intake V1        (PENDING -> DRAFT_READY)
  -> Customer Requirements & Project Brief V1
  -> Proposal Workbench V1             (draft internal, versioning, review single-use)
  -> Customer-Project Handoff V1       (link ke workflow + authorization + release approval)
```

Aturan alur (tegas):

- Seluruh modul bersifat **draft-only / internal / owner-only**. Tidak ada penawaran harga final,
  kontrak, komitmen tenggat, work order, atau pengiriman WhatsApp.
- Setiap proposal menampilkan banner: `NON-BINDING DRAFT — BUKAN PENAWARAN, BUKAN KONTRAK`.
- Idempotency intake via SHA-256 `inquiry_key` (permintaan identik tidak membuat baris ganda).
- Review bersifat **single-use**; hanya status `DRAFT_READY` yang dapat direview.
- Akses dibatasi allowlist `team7.owner_user_ids`; non-owner menerima `403/404`, dan
  upaya terlarang tidak membuat baris.

### 4.3 Alur permintaan customer (gambaran baseline)

```
Calon Customer -> CS -> Sales & Digital Marketing -> Analisis Teknis -> Proposal
  -> Persetujuan Owner -> Development -> QA & Security -> Persetujuan Rilis -> Penyerahan
```

Sesuai [`docs/CRUSADE-TEAM7-MASTER-BASELINE-V1.md`](docs/CRUSADE-TEAM7-MASTER-BASELINE-V1.md:46).

---

## 5. Crusade Living Office (V13–V17)

Berkas HTML di root: `Crusade_Living_Office_V13_Meeting.html` hingga
`Crusade_Living_Office_V17_Development.html`.

| Versi | Fokus | Status |
|---|---|---|
| V13 | Meeting | ✅ Presentasi/dev |
| V14 | Development | ✅ Presentasi/dev |
| V15 | Development | ✅ Presentasi/dev |
| V16 | Development (lease/fencing runtime terkait) | ✅ Presentasi/dev |
| V17 | Development (agent layer, skills, event protocol) | ✅ Presentasi/dev (lihat `tools/crusade_v17_*`) |

Catatan: komponen runtime V16/V17 (`tools/crusade_v16_runtime.py`,
`tools/crusade_v17_*`) diuji dengan proses test-owned & SQLite sementara; hasilnya
**bukan** sertifikasi concurrency tingkat produksi. Jadwal/rolling di dalam Living Office
**tidak boleh diubah**.

---

## 6. Keamanan & Privasi

- Semua kredensial, contact, dan data customer **tidak** disimpan di repo; template
  memakai marker `__PROVISION_*`.
- Transpor notifikasi default `fake`; WhatsApp live diarsitekturkan mustahil di profil
  staging (tiga kontrol independen: default config, override staging, guard fail-closed).
- Log disanitasi (tanpa payload/secret/auth/customer field).
- Signed bridge/HMAC, nonce replay protection, rate limit, dan scope check diterapkan.
- AP-8: provisioning tiga contact key staging (NOC phone, finance phone, finance name)
  — mekanisme **READY**; nilai sebenarnya menunggu **pasokan Owner** via secret store.

---

## 7. Status Rilis (ringkas)

| Area | Status |
|---|---|
| Aplikasi FTTH | ✅ Local functional; schema produksi ⛔ |
| TransformBot / Team 7 | ✅ Local functional (draft-only) |
| Living Office V13–V17 | ✅ Presentasi/dev |
| AP-8 provisioning | ✅ READY (values pending Owner) |
| D-8 staging execution | ⛔ Menunggu persetujuan Owner terpisah |
| Release package | ⛔ `NON_DEPLOYABLE_PREVIEW` |
| Empat gate approval | ⛔ **PENDING** |

**RELEASE NO-GO / DEPLOY NO-GO.**

---

## 8. Non-Goals (saat ini)

- Tidak ada deployment, migrasi produksi, atau akses VPS dari workspace ini.
- Tidak ada WhatsApp live, gateway aktif, atau AI model live.
- Tidak ada penawaran harga, kontrak, atau pengembangan berbayar otomatis untuk customer.
- Tidak mengubah CS Master Prompt Unified V4, Shifting Rolling, atau tugas agent aktif.
