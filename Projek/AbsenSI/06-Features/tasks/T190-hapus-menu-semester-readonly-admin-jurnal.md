# T190 — Web: Hapus Menu "Semester" Read-Only di Admin-Jurnal (Redundan Pasca-T188)

## Depends on
**WAJIB setelah T188** (Kalender Pendidikan full-CRUD di admin-jurnal) — menu ini baru jadi benar-benar redundan SETELAH T188 memberi admin_jurnal cara yang LEBIH LENGKAP (bukan cuma lihat) untuk hal yang sama.

## Objective
Hapus menu "Semester" (read-only murni, `(admin-jurnal)/admin-jurnal/semester/`) — REDUNDAN setelah T188 (halaman Kalender Pendidikan admin-jurnal menampilkan Tahun Ajaran+Semester dengan CRUD PENUH, jauh lebih lengkap dari tampilan read-only lama).

## Context — Keputusan User (2026-08-15)

User eksplisit: "hilangkan menu semester pada admin jurnal karena sudah ada di kalender pendidikan (semester yang tampil akan sesuai dengan tahun ajaran yang diaktifkan)". Halaman lama (`semester-view.tsx:16-19`, komentar "T054/T050 — read-only murni") HANYA tabel Tahun Ajaran+Semester+Tanggal, TANPA aksi apa pun.

## Spec Detail

### 1. Frontend — hapus menu

- Sidebar admin-jurnal — HAPUS entry menu "Semester" (yang mengarah ke path lama).
- Folder `(admin-jurnal)/admin-jurnal/semester/` — HAPUS filenya (page.tsx, semester-view.tsx) — INI HALAMAN YANG DIHAPUS TOTAL, BUKAN sekadar disembunyikan dari menu (user minta "hilangkan").
- **VERIFIKASI TIDAK ADA LINK internal lain** yang mengarah ke path `/admin-jurnal/semester` sebelum menghapus (grep menyeluruh) — kalau ADA, redirect ke halaman Kalender Pendidikan (T188) sebagai gantinya, JANGAN biarkan 404 mati begitu saja untuk link yang mungkin ter-bookmark.

## Edge Cases
- User/admin_jurnal yang PUNYA bookmark lama ke `/admin-jurnal/semester` — REKOMENDASI: tambah redirect otomatis ke path Kalender Pendidikan (T188) SUPAYA tidak 404 tiba-tiba, KONSISTEN prinsip UX yang baik untuk perubahan navigasi.

## Files
- **Hapus:** `apps/web/src/app/(admin-jurnal)/admin-jurnal/semester/` (folder+isinya).
- **Modifikasi:** sidebar admin-jurnal (hapus entry menu), TAMBAH redirect kalau ditemukan link internal lain yang masih mengarah ke path lama.
- **Jangan sentuh:** halaman Kalender Pendidikan T188 (sudah menggantikan fungsi ini sepenuhnya, tidak perlu diubah).

## Acceptance Criteria
- [x] Menu "Semester" TIDAK MUNCUL LAGI di sidebar admin-jurnal.
- [x] Path lama `/admin-jurnal/semester` TIDAK 404 mentah — redirect ke `/admin-jurnal/kalender`.
- [x] Halaman Kalender Pendidikan (T188) TETAP berfungsi normal, tidak disentuh.
- [x] Build + type-check hijau.

## Validasi Claudian
- [x] Grep menyeluruh SEBELUM hapus — hanya 1 referensi ditemukan (`admin-jurnal-sidebar.tsx`), tidak ada link internal lain.

## Status Eksekusi (2026-08-16)

**Selesai.**

- Grep `admin-jurnal/semester` di seluruh `apps/web/src` — HANYA 1 hit (entry sidebar itu sendiri), aman dihapus.
- `semester-view.tsx` dihapus total. `page.tsx` diganti jadi redirect ke `/admin-jurnal/kalender` (pola sama seperti redirect `jadwal-blok` di T189) — bukan disembunyikan, sesuai permintaan eksplisit user "hilangkan".
- Sidebar admin-jurnal — entry "Semester" dihapus, import `CalendarRange` (jadi unused) ikut dihapus.
- `tsc --noEmit` web bersih. Backend tidak disentuh (task murni frontend) — 443/443 test tetap lulus.
- Live-verify browser: belum dilakukan (konsisten pola T186-T189, verifikasi manual diserahkan ke user).
