---
tags: [absensi, infra, ssh, production, akses-data]
updated: 2026-09-02
---

# Akses Data Production — Siapa Bisa Apa

Baca ini kalau butuh cek data production dan bingung caranya, baik Anda (user) atau Claude Code yang sedang eksekusi task.

---

## Ringkasan Cepat

| Siapa | Akses ke Production | Cara |
|---|---|---|
| **Hermes** | ✅ Read-only (SELECT/SHOW/DESCRIBE/EXPLAIN saja) | SSH command-restricted, key khusus |
| **Claude Code** (di server manapun, termasuk sesi di server production) | ❌ **TIDAK PUNYA** kredensial SSH ke production dari sesi lain — kalau butuh data production, **JANGAN coba SSH sendiri**, minta ke user untuk diteruskan ke Hermes | — |
| **User** | ✅ Penuh (SSH admin `anunnaki`, akses `sudo`) | SSH biasa dengan key production |

## Kenapa Begini (Alasan Desain)

Ini bukan keterbatasan teknis — ini **keputusan keamanan yang disengaja** (lihat `workflow-2-sesi-diskusi-eksekusi.md`):

- **Hermes**: mitra diskusi + baca data untuk analisa, TIDAK PERNAH menulis kode/DB. Akses SSH-nya di-scope ketat lewat command-restricted whitelist (`task-INFRA-001-ssh-readonly-hermes.md`) — hanya bisa `SELECT` dan baca log/status service, tidak bisa `sudo` bebas atau tulis apapun.
- **Claude Code**: eksekutor kode, dipicu manual oleh user. TIDAK diberi kredensial SSH ke production sama sekali (Claude Code jalan di sesi terpisah setiap kali, tidak ada mekanisme aman untuk "titip" private key ke sesi Claude Code tanpa risiko kredensial bocor/tersebar).

## Kalau Task File Menyebut "Cek Data Production Dulu"

Kalau Anda (Claude Code) menemukan instruksi di task file seperti *"cek dulu apakah ada baris dengan kolom X NULL di production"* — **JANGAN mencoba SSH sendiri**. Kemungkinan besar:

1. **Hermes SUDAH mengecek ini** di sesi Planning sebelum task ditulis — cari bagian "Skenario Kegagalan" atau catatan `[VERIFIKASI SELESAI ...]` di task file, hasilnya biasanya sudah dituliskan LENGKAP di situ (lihat contoh: `task-CORE-005-restrukturisasi-jadwal-piket-per-kampus.md`, bagian 4).
2. **Kalau task file TIDAK punya hasil verifikasi** (task lama, ditulis sebelum akses SSH Hermes ada) — **STOP, jangan asumsikan/jangan coba akses sendiri**. Laporkan ke user: *"Task ini butuh verifikasi data production dulu — tolong minta Hermes cek dan update task file ini sebelum saya lanjutkan."*

## Cara Hermes Mengecek (untuk referensi, BUKAN untuk Claude Code jalankan)

```bash
ssh -i ~/.ssh/hermes_readonly hermes-readonly@10.10.10.198 'mysql -u hermes_readonly -e "<QUERY SELECT/SHOW/DESCRIBE/EXPLAIN>"'
```

- Key: `C:\Users\Administrator\.ssh\hermes_readonly` — **hanya ada di sisi Hermes/Windows**, tidak pernah dikirim ke server manapun.
- Whitelist command di server (`allowed-commands.sh`) hanya izinkan: `journalctl -u absensi-prod-*`, `systemctl status absensi-prod-*`, dan `mysql -u hermes_readonly -e "SELECT/SHOW/DESCRIBE/DESC/EXPLAIN ..."` — semua lainnya ditolak otomatis.
- Detail setup lengkap: `06-Features/tasks/task-INFRA-001-ssh-readonly-hermes.md`.

## Alur yang Benar Kalau Butuh Verifikasi Data Baru

1. User diskusi dengan Hermes (sesi Planning) tentang task yang butuh cek data production.
2. Hermes jalankan query read-only via SSH, dapatkan hasil nyata.
3. Hermes **tulis hasil VERBATIM ke task file** (bagian Skenario Kegagalan / Batasan) — bukan cuma bilang "sudah dicek", tapi hasil datanya persis, supaya Claude Code punya fakta konkret tanpa perlu akses sendiri.
4. User memicu Claude Code untuk eksekusi task — Claude Code baca hasil verifikasi itu dari file, TIDAK perlu SSH sendiri.

## Yang JANGAN Dilakukan

- ❌ Claude Code mencoba generate/reuse SSH key untuk akses production sendiri.
- ❌ Claude Code mengasumsikan data production berdasarkan data dev (`localhost:3306` atau `3307`) — dev dan production adalah database **terpisah total**, datanya bisa sangat berbeda.
- ❌ Menunda task tanpa laporan jelas — kalau butuh data production dan belum ada verifikasi di task file, LAPORKAN eksplisit ke user, jangan diam-diam skip validasi atau menebak.
