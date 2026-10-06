# AGENT INSTRUCTION & ARCHITECTURAL HANDOFF SPECIFICATION
## System: Autonomous AI Creator Management & Social Scheduler
**Target Audience:** Autonomous Coding Agents, Tech Leads, Engineering Pods  
**Status:** Canonical Implementation Rules  
**Last Updated:** October 2026  

---

## 1. MISSION & HIGH-LEVEL GOAL
Membangun platform aplikasi web **Creator Management** yang memproses jadwal konten dari berbagai sumber input (Upload Excel, Manual Web Form, Google Sheets Sync), secara mandiri memicu pembuatan visual berbasis AI (termasuk carousel dengan pemisah `||`), melakukan loop approval interaktif dua arah via WhatsApp, secara mandiri memfasilitasi revisi berbasis chat tanpa campur tangan manusia, dan menerbitkan konten ke jejaring sosial sesuai kolom platform yang dikonfigurasi.

Semua implementasi wajib mematuhi [PRD_FSD_CREATOR_MANAGEMENT.md](file:///d:/_RAHMAD/_2026/Creator%20Management/PRD_FSD_CREATOR_MANAGEMENT.md).

---

## 2. STRICT BUSINESS LOGIC & CONSTRAINTS

### A. Tri-Modal Ingestion & Data Parsing
1. **Excel Parsing (`.xlsx`, `.csv`)**:
   - Kolom wajib: `task_name`, `caption`, `image_prompt`, `scheduled_at`, `platforms`.
   - Kolom opsional: `whatsapp_recipient`, `brand_id`.
   - **Carousel Parsing**: Deteksi string separator `||` pada `image_prompt`. Jika ada, split menjadi array slide berurutan. Set `is_carousel = true` dan `total_slides = N`.
2. **Immediate Pre-Generation**:
   - Jangan tunda generasi gambar hingga hari-H. Segera setelah baris data berhasil divalidasi, buat status `GENERATING` dan masukkan ke antrean Image Generator Worker.
   - Begitu selesai di-generate, ubah status menjadi `WAITING_APPROVAL_WINDOW`.

### B. WhatsApp Approval Timing & Dynamic Settings
1. **Perhitungan Waktu Trigger**:
   - Jangan hard-code jam pengiriman WA.
   - Ambil nilai dari `brands.approval_window_minutes` (misal 60 menit, 120 menit, atau 0 untuk langsung kirim).
   - Cron pemantau memeriksa tiap menit:
     $$\text{current\_time} \ge (\text{scheduled\_at} - \text{approval\_window\_minutes})$$
   - Ketika kondisi terpenuhi, kirim gambar + preview caption + petunjuk respon ke WhatsApp, lalu ubah status menjadi `WAITING_APPROVAL_RESPONSE`.
2. **Fallback Nomor WhatsApp**:
   - Prioritas 1: `posts.override_wa_phone` (jika kolom di Excel/Form terisi).
   - Prioritas 2: `brands.default_wa_phone` (nomor default PIC brand).

### C. WhatsApp Webhook State Machine & Autonomous Revision
1. **Respon Persetujuan (`APPROVE` / `OKE` / `SETUJU` / `ACC`)**:
   - Ubah status menjadi `APPROVED`.
   - Kirim balasan konfirmasi bahwa konten telah disetujui dan siap tayang pada jadwalnya.
2. **Respon Permintaan Revisi (`REVISI: [catatan]`)**:
   - Tangkap catatan setelah kata `REVISI:`.
   - Catat entri baru pada tabel `post_revisions`.
   - Ubah status menjadi `REVISING`.
   - **Autonomous Re-generation**: Panggil AI LLM untuk memutasi prompt asli digabung dengan masukan revisi, lalu masukkan kembali ke Image Generator Worker.
   - Begitu gambar baru jadi, kirimkan preview gambar hasil revisi kembali ke nomor WhatsApp yang sama dengan instruksi approval ulang.
3. **Penanganan Kasus Expired (Jadwal Terlewat Tanpa Respon)**:
   - Jika $\text{current\_time} \ge \text{scheduled\_at}$ dan status masih `WAITING_APPROVAL_RESPONSE`, ubah status menjadi `EXPIRED`.
   - Kirimkan peringatan darurat ke WhatsApp:
     - Balas `UPLOAD` $\rightarrow$ Ubah ke `APPROVED` dan segera posting saat itu juga.
     - Balas `LEWATKAN` $\rightarrow$ Ubah status ke `CANCELED`.

### D. Omnichannel Publishing Rules
1. **Penerbitan Sesuai Kolom `platforms`**:
   - Parse string/array `platforms` (contoh: `Instagram, TikTok, LinkedIn`).
   - Konten hanya di-upload ke platform yang tertera pada baris tersebut.
2. **Multi-Brand Credential Isolation**:
   - Setiap request ke API sosial media wajib mengambil token dari tabel `social_accounts` yang sesuai dengan `brand_id` dari konten tersebut.
   - Sediakan Mock Driver untuk lingkungan pengujian/demo agar alur dapat diuji 100% tanpa bergantung pada token aktif sosial media.

---

## 3. RECOMMENDED STACK & REPOSITORY STRUCTURE

### Stack Rekomendasi:
- **Backend / Fullstack**: Node.js / TypeScript (Next.js 14/15 App Router atau NestJS / Express + React).
- **Database**: PostgreSQL (Prisma ORM / Drizzle ORM) atau SQLite WAL untuk prototipe cepat.
- **Queue Engine**: BullMQ (Redis) atau in-memory background worker dengan persistence log.
- **AI Integrasi**:
  - Image Gen: OpenAI DALL-E 3 / Replicate Flux API / Mock Generator untuk mode offline.
  - Prompt Mutator: OpenAI GPT-4o-mini / Gemini Flash.
- **WhatsApp Bridge**: Fonnte API / Twilio WhatsApp / Baileys WebSocket / Mock WhatsApp Simulator.

### Struktur Direktori yang Disarankan:
```
├── app/                        # Next.js App Router (Dashboard & API)
│   ├── api/
│   │   ├── posts/              # Ingestion endpoints (upload-excel, manual)
│   │   ├── webhooks/
│   │   │   └── whatsapp/       # Inbound WhatsApp message receiver
│   │   └── brands/             # Brand and social credentials settings
│   ├── dashboard/              # Halaman tabel, kanban, template download
│   └── settings/               # Halaman integrasi akun & nomor WA
├── lib/
│   ├── parsers/                # Excel/CSV parser & delimiter '||' splitter
│   ├── ai/                     # Image generation driver & prompt mutator
│   ├── whatsapp/               # Outbound WhatsApp messenger & message formatter
│   ├── social/                 # Driver Instagram, TikTok, LinkedIn & Mock Publisher
│   └── queue/                  # Background worker coordinator & cron scheduler
├── prisma/ atau db/            # Database schema & migrations
├── docs/                       # PRD_FSD_CREATOR_MANAGEMENT.md, FUNCTIONAL_SPEC.md
└── AGENT.md                    # File petunjuk ini
```

---

## 4. CODE IMPLEMENTATION GUIDELINES FOR CODING AGENTS

1. **Idempotency & Safe Retries**:
   - Jangan pernah melakukan pengiriman pesan WhatsApp atau publikasi ke sosial media ganda jika terjadi restart sistem. Gunakan status transaksi database.
2. **Defensive Excel Parsing**:
   - Tangani spasi di nama header (misal `Task Name ` atau `task_name`). Gunakan sanitasi lowercase trim.
   - Format tanggal Excel bertipe serial number wajib dikonversi dengan benar ke format JavaScript `Date`.
3. **Graceful WhatsApp Message Parsing**:
   - Tangani pesan masuk dengan case-insensitive trim.
   - Berikan balasan panduan yang ramah jika user mengirim pesan yang tidak dimengerti oleh sistem.
4. **Environment Variables Configuration**:
   - Selalu pisahkan kunci rahasia ke dalam `.env`:
     ```env
     DATABASE_URL=postgresql://...
     AI_IMAGE_API_KEY=...
     WHATSAPP_GATEWAY_TOKEN=...
     WHATSAPP_GATEWAY_URL=...
     ENCRYPTION_MASTER_KEY=...
     MOCK_MODE=true # Sangat berguna untuk dev & testing instan
     ```

---

## 5. VERIFICATION CHECKLIST (DEFINITION OF DONE)
- [ ] Berhasil mengunggah file Excel dengan delimiter `||` dan menghasilkan beberapa entri `post_media` (carousel).
- [ ] Background worker berhasil men-generate gambar dan menyimpannya di storage.
- [ ] Cron berhasil mendeteksi jadwal `scheduled_at - approval_window_minutes` dan mengirim pesan WA simulasi/real.
- [ ] Balasan WA `OKE` mengubah status menjadi `APPROVED`.
- [ ] Balasan WA `REVISI: [catatan]` berhasil men-generate gambar baru dan mengirim chat approval ulang.
- [ ] Jika waktu tayang lewat tanpa balasan, status menjadi `EXPIRED` dan mengirimkan prompt pemulihan (`UPLOAD` / `LEWATKAN`).
- [ ] Konten berhasil diunggah ke platform yang sesuai (atau dicatat sukses di mock dispatcher).
