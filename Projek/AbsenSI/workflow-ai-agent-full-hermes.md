---
tags: [absensi, workflow, ai-agent]
created: 2026-08-29
status: SUPERSEDED 2026-08-31
---

# Workflow AI Agent AbsenSI -- Full Hermes Orchestration

> ⚠️ DIBATALKAN 2026-08-31 -- user membalik keputusan ini secara eksplisit. Hermes TIDAK LAGI boleh memanggil Claude Code CLI sendiri; semua eksekusi Claude Code dipicu manual oleh user. Dokumen aktif sekarang: workflow-2-sesi-diskusi-eksekusi.md. Isi di bawah HANYA referensi historis (data biaya nyata, cara kerja umum Claude Code CLI) -- JANGAN diikuti sebagai panduan aktif.

> Keputusan resmi 2026-08-29 (SUDAH TIDAK BERLAKU): user memilih FULL menggunakan Hermes sebagai satu titik kontak untuk seluruh pekerjaan AbsenSI (riset, desain, DAN coding), bukan berpindah manual antara Hermes dan Claude Code di VS Code.

## Latar Belakang Keputusan

Sebelumnya development berjalan paralel: Hermes membangun Design System v2 (audit, token, Figma, dokumentasi) sementara user memakai Claude Code langsung di VS Code untuk coding. Setelah design system selesai, muncul pertanyaan strategi: pilih salah satu, atau tetap paralel?

**Evaluasi kritis yang dilakukan sebelum keputusan** (bukan asumsi):
- Dicek nyata: Claude Code JUGA punya self-improving memory (25 entri feedback/project di `D:\backup_claude_code\.claude\projects\...\memory\`) -- BUKAN pembeda unik Hermes seperti awalnya diasumsikan user.
- Perbandingan jujur dibuat: Hermes unggul di riset/browser/Figma/orkestrasi; Claude Code unggul di integrasi editor/graphify/agentic coding.
- Alasan asli user memilih full-Hermes: TIDAK mau bolak-balik 2 aplikasi, ingin satu history percakapan.

## Keputusan Final: Hermes sebagai Orkestrator

Hermes TIDAK menggantikan Claude Code -- Hermes MEMANGGIL Claude Code CLI di belakang layar via terminal saat butuh kekuatan coding, lalu melaporkan hasil ke user dalam chat yang sama.

```
User <--chat--> Hermes
                   |
                   +-- Riset, Figma, dokumentasi, audit -> Hermes kerjakan langsung
                   |
                   +-- Butuh nulis/refactor kode -> Hermes panggil:
                        claude -p "task" --allowedTools "Read,Edit,Bash" --max-turns N --output-format json
                        (workdir = repo target, hasil dibaca & dilaporkan Hermes ke user)
```

## Uji Coba Nyata (2026-08-29)

Dijalankan: audit read-only modul `(pembina-ekstra)` (7 file) via `claude -p` dengan `--allowedTools "Read,Grep,Glob"` (read-only, tidak ubah kode).

**Hasil membuktikan sistem 3-lapis bekerja end-to-end**: Claude Code OTOMATIS membaca `DESIGN_SYSTEM_AGENT.md` di root repo dan memakainya sebagai acuan audit -- tanpa perlu dijelaskan ulang oleh user atau Hermes.

**Temuan audit modul pembina-ekstra:**
- 0 hex hardcode, tapi 2 raw-color non-token (`bg-[hsl(var(--color-shadow)/0.6)]`, `hover:bg-white/20`)
- 3 dari 3 tabel tanpa `overflow-x-auto` -- risiko mobile
- 3 operasi destruktif (hapus sesi/peserta/kelompok) pakai `Dialog` generik, BUKAN `ConfirmationDialog` -- **blocker ditemukan**: komponen ConfirmationDialog baru ada sebagai kontrak JSON, belum ada implementasi React di packages/ui. Harus dibangun dulu sebelum modul manapun bisa patuh penuh.

**Data biaya nyata**: 147 detik, 23 turn, **$0.598** untuk 1 audit read-only modul kecil (7 file). Dicatat sebagai baseline -- migrasi modul besar (mis. admin, 98 file) akan jauh lebih mahal, perlu dipertimbangkan per kasus.

## Aturan Pemakaian Workflow Ini

1. **Mode print (`-p`) untuk tugas satu-kali** -- audit, migrasi 1 modul, fix bug spesifik. Selalu set `--max-turns` untuk mencegah biaya membengkak tanpa kendali.
2. **Mode interaktif tmux** hanya untuk kerja iteratif panjang yang butuh banyak keputusan di tengah jalan -- jarang dipakai, defaultnya mode print.
3. **Hermes SELALU melaporkan hasil ke user dalam chat**, termasuk biaya (`total_cost_usd` dari output JSON) dan ringkasan perubahan -- bukan cuma "sudah selesai".
4. **File konteks project (`DESIGN_SYSTEM_AGENT.md`, `CLAUDE.md`) WAJIB tetap dijaga akurat** -- ini yang membuat panggilan Claude Code oleh Hermes otomatis "tahu aturan" tanpa perlu dijelaskan ulang setiap kali.
5. **Klaim keberhasilan dari Claude Code adalah self-report** -- untuk perubahan kode nyata (bukan cuma audit read-only), Hermes tetap WAJIB verifikasi hasil (baca file yang diklaim diubah, jalankan linter v2) sebelum melapor "berhasil" ke user.

## Trade-off yang Diterima Sadar oleh User

- Tidak ada visual real-time diff editor (beda dari VS Code langsung) -- kompensasi: Hermes bisa baca & ringkas perubahan.
- Biaya per panggilan nyata (~$0.6 untuk tugas kecil) terpisah dari kuota langganan Claude Code Pro/Max yang mungkin sudah dibayar user -- BUKAN gratis meski user sudah subscribe Claude Code.
- Kurang cocok untuk sesi coding sangat iteratif cepat (ketik-lihat-ketik) dibanding memakai Claude Code langsung di terminal/VS Code.

## Kapan TIDAK Pakai Pola Ini

Kalau user sendiri ingin coding interaktif cepat langsung (bukan lewat Hermes), tetap boleh buka Claude Code di VS Code manual -- pola orkestrasi ini adalah DEFAULT preferensi, bukan larangan mutlak memakai Claude Code langsung.
