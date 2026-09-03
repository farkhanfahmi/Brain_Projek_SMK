# Task-[MODUL-XXX]: [Nama Task]

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk). Contoh: task-CORE-001.
> Ditulis oleh Hermes (sesi Planning) setelah diskusi kritis dengan user. Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.
> **[2026-08-31] Konvensi penamaan**: format `task-MODUL-NNN` ini berlaku untuk task BARU mulai sekarang. 260+ task lama (T001-T263) tetap pakai format lama `T0xx-slug.md` (tanpa prefix modul, nomor polos) — TIDAK di-rename, kedua konvensi hidup berdampingan. Jangan bingung kalau menemukan file lama berformat beda.

**Task Terbuat:** [tanggal ditulis, isi otomatis saat file ini dibuat]
**Task Tereksekusi:** — [isi tanggal saat task dinyatakan selesai; kosongkan/— selama masih berjalan]

---

## 1. Info Eksekusi

**Rekomendasi Model:** [Haiku / Sonnet / Opus — default Sonnet kecuali terbukti perlu lebih/kurang]
**Tingkat Effort:** [low / medium / high / xhigh / max — default medium, naikkan hanya untuk task yang butuh reasoning berat: migrasi besar, refactor lintas-file, debugging root-cause licin]
**Alasan pemilihan:** [1 kalimat kenapa kombinasi ini, bukan yang lain — mis. "task ini murni tambah 1 endpoint CRUD sederhana, Sonnet+medium cukup, Opus berlebihan"]

## 2. Konteks & Tujuan Utama

[Penjelasan singkat: fitur/masalah apa yang diselesaikan, kenapa ini dibutuhkan, hasil diskusi kritis (skenario kegagalan yang sudah dipertimbangkan) yang melatarbelakangi keputusan desain task ini]

**Depends on:** [task-XXX-YYY jika ada dependency urutan. Tulis "Tidak ada" jika berdiri sendiri. JANGAN mulai eksekusi sebelum dependency selesai — relevan untuk fitur besar yang perlu urutan (mis. migrasi token dulu baru migrasi komponen).]

## 3. Langkah Eksekusi Detail

[Urutan instruksi teknis, bertahap, dengan path file EKSPLISIT dan potongan kode/pseudo-code kalau perlu presisi tinggi. Setiap langkah harus bisa dieksekusi tanpa Claude Code perlu menebak/berasumsi.]

1. ...
2. ...
3. ...

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Buat:** `apps/[app]/[path]`
- **Modifikasi:** `apps/[app]/[path]` — [apa yang diubah]
- **Jangan sentuh:** `apps/[app]/[path]`
- **⚠️ Kalau task ini butuh ubah `packages/types` (shared):** WAJIB stop dan minta konfirmasi user dulu — breaking change ke app lain.

**Dilarang dilakukan:**
- [mis. "Jangan ubah skema Prisma tanpa migration terpisah", "Jangan sentuh database production"]

**Skenario kegagalan yang WAJIB ditangani** (hasil analisa kritis sebelum spec ini ditulis):
- Kondisi: [mis. input kosong/null] → Perilaku yang benar: [...]
- Kondisi: [mis. race condition 2 request bersamaan] → Perilaku yang benar: [...]
- Kondisi: [mis. permission/role salah] → Perilaku yang benar: [...]
- **Kalau task ini butuh cek data production untuk memutuskan behavior** (mis. "apakah ada baris NULL", "berapa banyak record kondisi X") — Hermes WAJIB cek dulu via SSH read-only SEBELUM handoff (lihat `06-Features/akses-data-production.md`), tulis HASIL NYATA di sini (bukan instruksi "Claude Code cek sendiri" — Claude Code TIDAK PUNYA akses SSH production).

**Edge case:**
- [kondisi] → [behavior yang diharapkan]

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Kriteria 1
- [ ] Kriteria 2
- [ ] Kriteria 3

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini (dicek ulang oleh Hermes sebelum handoff)
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 300 baris perubahan; kalau lebih, pecah jadi beberapa task berurutan pakai "Depends on")
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada (cek `11-Decisions.md` / `DESIGN_SYSTEM_AGENT.md` bila relevan)
- [ ] Dependency (jika ada) sudah selesai sebelum task ini di-assign
