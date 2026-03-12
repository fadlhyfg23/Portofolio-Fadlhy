# Solusi Bot Otomasi Laporan Uang Operasional

Dokumen ini merangkum solusi praktis agar input laporan operasional proyek tidak lagi manual ke spreadsheet.

## Tujuan
- Upload foto struk/nota → nominal, tanggal, vendor, dan kategori terbaca otomatis.
- Satu input bisa dicatat ke **database lama** dan **database baru** (dual-write).
- Bisa tambah **proyek baru** langsung dari bot.
- Bisa input **modal proyek** dari bot tanpa edit spreadsheet manual.

## Arsitektur yang Direkomendasikan (MVP Cepat)
1. **Channel bot**: Telegram Bot / WhatsApp API.
2. **Orchestrator**: n8n (mudah drag-and-drop) atau Make.
3. **OCR + ekstraksi data**:
   - Google Document AI / OCR.space / Tesseract + LLM parser.
   - LLM dipakai untuk normalisasi format nominal, tanggal, dan kategori.
4. **Database utama**: PostgreSQL (Supabase/Neon).
5. **Sink ke legacy**: Google Sheets lama tetap diupdate otomatis.
6. **Dashboard**: Looker Studio/Metabase (opsional).

## Alur Kerja Bot
### 1) Input struk/nota
User kirim:
- Foto struk
- Atau teks singkat: `#project A #kategori transport #catatan tol`

Flow:
1. Bot terima media.
2. OCR baca teks struk.
3. Parser ekstrak:
   - tanggal transaksi
   - total nominal
   - nama merchant
   - item (jika ada)
4. Classifier tentukan kategori (transport, konsumsi, material, dll).
5. Bot kirim preview:
   - "Terdeteksi: Rp120.000 | Transport | Proyek X | 11-03-2026. Simpan?"
6. Jika user balas **Ya**, data ditulis ke 2 target:
   - `db_new.expenses`
   - `legacy_sheet_expenses`

### 2) Dual database (lama + baru)
Gunakan pola **outbox + retry** agar aman:
- Tulis dulu ke DB baru (`status_sync_legacy = pending`).
- Worker sinkronisasi ke DB/sheet lama.
- Jika gagal, retry bertahap + logging error.
- Tujuan: input tidak hilang walau salah satu target down.

### 3) Tambah proyek baru via bot
Contoh command:
- `/add_project Proyek A | Klien ABC | 2026-03-01`

Flow:
- Validasi nama unik.
- Simpan ke `projects` (DB baru).
- Optional sink ke sheet master proyek lama.
- Bot balas ID proyek + status aktif.

### 4) Input modal proyek via bot
Contoh command:
- `/set_modal PRJ-001 15000000`

Flow:
- Catat ke `project_capital`.
- Update saldo awal proyek.
- Kirim ringkasan:
  - modal
  - total terpakai
  - sisa budget

## Skema Data Minimal
### Tabel `projects`
- `id`
- `project_code`
- `project_name`
- `client_name`
- `start_date`
- `status`

### Tabel `expenses`
- `id`
- `project_id`
- `date`
- `merchant`
- `amount`
- `category`
- `notes`
- `receipt_url`
- `source` (bot/manual/import)
- `status_sync_legacy` (pending/success/failed)

### Tabel `project_capital`
- `id`
- `project_id`
- `capital_amount`
- `set_by`
- `set_at`

## Kategori Otomatis (Contoh Rule)
- Kata kunci "tol, parkir, bensin" → `transport`
- Kata kunci "makan, snack, kopi" → `konsumsi`
- Kata kunci "kabel, adaptor, bracket" → `material`
- Jika confidence rendah → `uncategorized` + minta konfirmasi user

## Fitur Penting yang Disarankan
- **Approval mode**: transaksi di atas nominal tertentu wajib approval atasan.
- **Audit trail**: siapa input, edit, approve.
- **Duplicate detection**: hindari struk sama masuk dua kali.
- **Budget alert**: notifikasi jika sisa modal < 20%.
- **Export bulanan**: otomatis PDF/Excel per proyek.

## Roadmap Implementasi (Praktis)
### Minggu 1 (MVP)
- Bot + OCR + input ke DB baru.
- Kategori otomatis + konfirmasi user.

### Minggu 2
- Sink ke DB/sheet lama (dual-write).
- Command tambah proyek + set modal.

### Minggu 3
- Dashboard ringkasan & alert budget.
- Hardening retry, logging, dan backup.

## Rekomendasi Stack Paling Cepat
- **Bot**: Telegram Bot API
- **Workflow**: n8n
- **DB**: Supabase Postgres
- **OCR**: Google Vision/Document AI
- **Storage struk**: Supabase Storage / Google Drive

Dengan stack ini, biasanya MVP bisa jalan dalam 5–10 hari kerja.
