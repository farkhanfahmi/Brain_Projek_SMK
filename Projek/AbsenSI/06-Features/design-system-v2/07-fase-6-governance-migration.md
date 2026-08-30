---
tags: [absensi, design-system-v2, fase-6]
created: 2026-08-29
status: selesai
---

# Fase 6 -- Governance & Incremental Migration

Kembali ke 00-INDEX.md untuk konteks proyek revamp v2. INI FASE PENUTUP -- mengikat 6 fase sebelumnya jadi siap dieksekusi bertahap di dunia nyata.

## Tujuan Fase

Menjawab pertanyaan operasional yang belum dijawab fase manapun: bagaimana v2 hidup berdampingan dengan v1 selama migrasi bertahap? Bagaimana developer/AI agent mengajukan komponen baru tanpa kembali ke pola ad-hoc (Switch/Avatar/Accordion) yang jadi masalah v1? Bagaimana memastikan kepatuhan token terjaga otomatis, bukan cuma niat baik?

## Lokasi Artefak

- `C:\ProjekSMK\AbsenSI\DESIGN_SYSTEM_AGENT.md` -- instruksi wajib untuk AI coding agent, di ROOT REPO (bukan di packages/design-tokens/) supaya otomatis terbaca sebagai project context oleh Claude Code/Hermes.
- `C:\Users\Administrator\AppData\Local\hermes\scripts\absensi_ds_v2_linter.py` -- linter presisi v2, menggantikan/melengkapi drift-checker v1 lama.

## 1. Linter v2 -- Hasil Uji Nyata

BEDA dari `absensi_ds_drift_check.py` (linter v1 lama yang membandingkan kode vs dokumen markdown vault dengan heuristik longgar), linter v2 membandingkan kode vs TOKEN JSON terstruktur:

- Validasi broken alias semantic->primitive (deteksi otomatis, bukan manual seperti sebelumnya).
- Deteksi hardcode hex LEGACY yang sudah diketahui (11 warna spesifik dari temuan Fase 0), dengan saran token pengganti PERSIS.
- Deteksi pemakaian `text/tertiary` berisiko di teks kecil (restricted token dari Fase 2).
- Deteksi `<table>` tanpa `overflow-x-auto`/`overflow-auto` (risiko mobile dari kontrak DataTable Fase 3).

**Hasil uji jalan nyata (2026-08-29)**: 17 VIOLATION kritis (semua match dengan temuan Fase 0: `#F5841F` di kampus-map.tsx, 10 warna Material Design hardcode di rekap-view.tsx & rekap-guru-view.tsx), 24 WARNING (229 hex mentah tersebar, 23 file berisiko restricted-token, 1 file tabel tanpa overflow wrapper). Linter TERBUKTI bekerja dan menemukan masalah nyata, bukan sekadar skrip kosong.

Exit code 1 saat ada violation -- siap dipakai sebagai pre-commit hook atau CI check di masa depan (belum dipasang sebagai hook otomatis, baru dijalankan manual/on-demand).

## 2. Coexistence Strategy (v1 <-> v2)

Prinsip: token v2 TIDAK menimpa token v1 secara langsung. Keduanya hidup berdampingan sampai migrasi modul selesai.

- Token v1 tetap di `packages/ui/src/globals.css` + `packages/config/tailwind.config.ts` -- TIDAK disentuh sampai modul terkait dimigrasi.
- Token v2 di `packages/design-tokens/` -- generator ke CSS variables/Tailwind config BELUM dijalankan (item aksi masa depan, lihat Belum Dikerjakan).
- Modul yang belum dimigrasi TETAP memakai styling v1 apa adanya -- tidak ada "migrasi paksa" tanpa keputusan eksplisit.

## 3. Escape Hatch -- Proses Resmi Pengajuan Komponen/Varian Baru

Menjawab temuan awal: 5 komponen ad-hoc (Switch, Avatar, Checkbox Chip, Tooltip, Accordion) lahir karena TIDAK ADA jalur resmi mengajukan kebutuhan baru. Proses ini mencegah pola itu terulang.

### Kapan proses ini dipakai
- Developer/AI agent butuh komponen yang belum ada kontraknya di `component-contracts/`.
- Developer/AI agent butuh varian baru dari komponen yang SUDAH ada kontraknya (mis. Button perlu variant kelima).
- Developer/AI agent butuh token baru yang belum ada di semantic.tokens.json.

### Langkah proses
1. **Cek dulu**: apakah kebutuhan ini benar-benar baru, atau sebenarnya sudah ada kontrak yang cocok tapi belum ditemukan? Baca `component-contracts/` lengkap dulu.
2. **Kalau genuinely baru**: tulis draft kontrak mengikuti `_schema.json` (props, states, tokens, a11y, doAndDont) -- SAMA PERSIS format 14 kontrak yang sudah ada di Fase 3-4.
3. **Ajukan ke product owner** (user) untuk review -- sertakan: kenapa dibutuhkan, di mana akan dipakai, draft kontrak.
4. **Setelah disetujui**: simpan kontrak final ke `component-contracts/<nama>.json`, update dokumentasi Obsidian relevan (catat di fase mana kebutuhan ini muncul), bangun representasi visual di Figma page 'Design System v2' mengikuti pola yang sama seperti komponen lain.
5. **Kode boleh ditulis** HANYA SETELAH kontrak resmi ada -- bukan sebelumnya.

### Kenapa proses ini penting
Draft kontrak WAJIB ditulis sebelum kode, bukan sesudah -- mencegah developer/AI agent menulis implementasi dulu lalu "mendokumentasikan belakangan" (yang biasanya tidak pernah benar-benar terjadi, persis pola yang menciptakan 5 komponen ad-hoc v1).

## 4. Rencana Migrasi Per Modul (Urutan DITENTUKAN Product Owner, Belum Final)

Berdasarkan data Fase 0 (bukan keputusan final -- product owner yang menentukan urutan eksekusi sesungguhnya):

| Modul | File Count | Kompleksitas Migrasi | Catatan |
|---|---|---|---|
| (pembina-ekstra) | 7 | Rendah | Kandidat pilot pertama -- modul terkecil, risiko regresi minimal |
| (siswa) | 5 | Rendah | Kandidat pilot alternatif |
| (piket) | 22 | Sedang, TAPI prioritas tinggi | Paling kritis dari audit responsif awal -- role paling sering akses mobile |
| (admin-jurnal) | 28 | Sedang | |
| (guru) | 63 | Tinggi | |
| (admin) | 98 | Tinggi, TAPI mengandung 2 bug hardcode konkret (kampus-map.tsx, rekap-view.tsx) yang linter v2 sudah deteksi | Modul terbesar, effort terbesar |

**Status migrasi aktual saat ini: BELUM ADA MODUL yang dimigrasi ke v2.** Seluruh Fase 0-5 adalah pembangunan fondasi (token, aturan, kontrak, representasi Figma) -- eksekusi migrasi kode sesungguhnya adalah keputusan terpisah yang menunggu arahan product owner.

## 5. Checklist Migrasi Per Modul (Template untuk Dipakai Nanti)

Saat product owner memutuskan modul X mulai dimigrasi:

- [ ] Jalankan linter v2, catat baseline violation/warning untuk modul tsb.
- [ ] Ganti import token v1 (`globals.css` var) ke v2 (`design-tokens` semantic) di file terkait.
- [ ] Untuk setiap komponen dipakai, cek kontraknya di `component-contracts/` -- pastikan props/states sesuai.
- [ ] DataTable: set `mobileLayout` eksplisit sesuai kontrak (default 'card').
- [ ] Ganti hardcode hex yang terdeteksi linter dengan token semantic yang disarankan.
- [ ] Operasi destruktif: pastikan pakai ConfirmationDialog, bukan Dialog generik/confirm() native.
- [ ] Jalankan linter v2 lagi, verifikasi violation berkurang/hilang untuk modul tsb.
- [ ] Update STATUS.md di vault + tabel migrasi di dokumen ini (tandai modul selesai).
- [ ] Commit dengan pesan jelas menyebut migrasi design-system-v2 untuk modul X.

## Keputusan Arsitektur

1. `DESIGN_SYSTEM_AGENT.md` sengaja ditaruh di ROOT REPO (bukan di dalam packages/design-tokens/) -- ini konvensi yang sama seperti CLAUDE.md/AGENTS.md, dibaca otomatis sebagai project context oleh AI coding agent (Claude Code, Hermes) begitu bekerja di repo ini.
2. Linter v2 dibuat sebagai script TERPISAH dari drift-checker v1 lama (`absensi_ds_v2_linter.py` vs `absensi_ds_drift_check.py`) -- bukan menimpa, supaya v1 tetap bisa dipakai untuk modul yang belum migrasi, v2 dipakai untuk modul yang sudah/sedang migrasi.
3. Escape Hatch mewajibkan draft kontrak SEBELUM kode -- desain sengaja friction tinggi di sisi proses, supaya developer/agent berpikir dua kali sebelum menulis komponen ad-hoc, bukan mempermudah jalan pintas.
4. Urutan migrasi modul SENGAJA tidak diputuskan dalam dokumen ini -- product owner yang berhak menentukan prioritas bisnis, dokumen ini hanya menyediakan data (ukuran, kompleksitas, bug konkret) sebagai bahan keputusan.

## Belum Dikerjakan (Item Aksi Masa Depan, di Luar 7 Fase Ini)

- Generator otomatis token JSON v2 -> CSS variables/Tailwind config -- saat ini token v2 baru "hidup" di JSON dan Figma, belum bisa langsung dipakai kode React tanpa migrasi manual per modul.
- Pemasangan linter v2 sebagai git pre-commit hook otomatis (saat ini manual run).
- Kontrak untuk 13 komponen v1 yang belum dikontrak presisi (Textarea, Skeleton, Label, Tabs, Popover, Sheet, Calendar, DatePicker, Form, Badge, StatusBadge, Pagination, ActivityFeed) plus 5 ad-hoc (Switch, Avatar, CheckboxChip, Tooltip, Accordion) -- akan dilengkapi via proses Escape Hatch saat modul yang memakainya mulai dimigrasi.
- Keputusan dark mode -- belum diputuskan scope-nya sejak Fase 1.
- Eksekusi migrasi modul pertama -- menunggu keputusan product owner urutan mana yang dimulai.
