# T051 — UI: Dashboard Admin Jurnal — Kelola Izin Guru

## Depends on
T046 (API izin guru), T050 (layout dashboard admin_jurnal harus sudah ada)

## Objective
Buat halaman untuk `admin_jurnal` menginput status izin guru (per-sesi atau seharian penuh) berdasarkan laporan manual (WA/lisan), dan memantau status submit tugas titipan.

## Context
- **App:** `apps/web`
- **Route:** `/admin-jurnal/izin`
- **Role:** `admin_jurnal`
- **Ref:** `Projek/AbsenSI/06-Features/dashboard-guru-jurnal.md` — bagian "Izin Guru Tidak Mengajar", alur 5 langkah
- **⚠️ Ref WAJIB dibaca sebelum menulis UI:** `Projek/AbsenSI/06-Features/design-system/MASTER.md` + `01-colors.md`/`03-components.md` — TIDAK ADA emoji sebagai ikon UI (✅/⏳/⚠️ di tabel bawah HANYA ilustrasi konsep, ganti dengan Badge Pill + ikon `lucide-react`), dan warna badge terbatas pada token `success`/`danger`/`primary` (tidak ada "kuning")

## Spec Detail

### Halaman `/admin-jurnal/izin`

**Form "Input Izin Baru"** (di atas atau modal terpisah):
- Pilih guru (autocomplete dari daftar teachers)
- Pilih tanggal (date picker, default hari ini)
- Radio/toggle: **"Seharian Penuh"** vs **"Sesi Tertentu"**
  - Kalau "Sesi Tertentu" dipilih → tampilkan dropdown sesi (fetch jadwal guru itu pada tanggal yang dipilih — perlu endpoint bantu atau reuse `GET /teaching-sessions/my-today` dengan parameter guru+tanggal untuk admin, kemungkinan perlu endpoint tambahan kecil `GET /teaching-sessions?teacher_id=&tanggal=` khusus admin_jurnal kalau belum ada — cek apakah T041 bisa diperluas untuk menerima `teacher_id` dari query saat role adalah admin_jurnal, atau buat endpoint admin terpisah. Putuskan pendekatan paling konsisten saat implementasi.)
- Tombol "Setujui Izin" → `POST /teacher-permits`
- Setelah submit → tampilkan konfirmasi, form reset

**Tabel "Riwayat Izin"** (bawah form, atau tab terpisah) — header kolom `text-label` (13px, `--color-text-secondary`), sel `text-body` (14px):
| Guru | Tanggal | Cakupan | Status Tugas | Follow-Up |
|---|---|---|---|---|
| Ahmad S. | 2026-07-22 | XI-RPL-1 / Pemrograman Web | Badge Pill "Sudah Diisi" | — |
| Siti R. | 2026-07-22 | Seharian Penuh | Badge Pill "Belum Diisi" | Badge Pill "Perlu Ditindaklanjuti" |

- Filter: tanggal (default hari ini), guru (opsional)
- Kolom "Cakupan": tampilkan nama kelas+mapel kalau `session_id` terisi, atau "Seharian Penuh" kalau `null`
- Kolom "Status Tugas": Badge Pill (`03-components.md`) berdasarkan `submitted_at` — terisi → bg `--color-success-bg`/text `--color-success-text` "Sudah Diisi"; null → bg `--color-bg-surface-subtle`/text `--color-text-secondary` "Belum Diisi" (netral, BUKAN kuning — palet ini tidak punya token warning terpisah)
- Kolom "Follow-Up": Badge Pill dengan bg `--color-danger-bg`/text `--color-danger-text` "Perlu Ditindaklanjuti" HANYA kalau `follow_up_needed: true` (kolom kosong kalau tidak) — ini SATU-SATUNYA tempat warna danger dipakai di tabel ini, harus menonjol karena sinyal krusial ke admin_jurnal (dan nanti TV Piket)
- Klik baris → bisa lihat detail (file yang diupload guru, keterangan) kalau sudah submit

## JANGAN
- ❌ JANGAN buat form "request izin dari guru" di halaman ini — ini murni input SATU ARAH dari admin, sesuai keputusan alur (guru lapor manual di luar sistem)
- ❌ JANGAN izinkan admin_jurnal edit/hapus permit yang sudah dibuat — kalau salah input, buat baru dengan koreksi (tidak ada endpoint update/delete di scope T046, konsisten pola insert-only untuk data audit-sensitive seperti ini)
- ❌ JANGAN sembunyikan badge "Perlu Ditindaklanjuti" — ini krusial untuk fungsi piket memantau kelas kosong, harus menonjol secara visual (pakai `--color-danger-bg`/`--color-danger-text`, bukan teks kecil abu-abu)
- ❌ JANGAN pakai emoji (✅/⏳/⚠️) sebagai indikator status di tabel — pakai Badge Pill dengan token warna yang sesuai, ikon (kalau perlu) dari `lucide-react`

## Files
- **Buat:** `apps/web/app/(admin-jurnal)/izin/page.tsx`
- **Buat:** `apps/web/app/(admin-jurnal)/izin/components/izin-form.tsx`
- **Buat:** `apps/web/app/(admin-jurnal)/izin/components/izin-table.tsx`
- **Modifikasi (jika diperlukan):** `apps/api/src/teaching-sessions/teaching-sessions.controller.ts` — endpoint bantu untuk admin lihat jadwal guru tertentu di tanggal tertentu (lihat catatan di form "Sesi Tertentu" di atas)

## Acceptance Criteria
- [ ] Input izin "Seharian Penuh" untuk guru X tanggal tertentu → 1 baris `teacher_permits` dengan `session_id: null` tercipta, muncul di tabel riwayat
- [ ] Input izin "Sesi Tertentu" → dropdown hanya tampilkan sesi guru itu pada tanggal yang dipilih (bukan semua sesi semua guru)
- [ ] Setelah guru submit tugas (dari T049), refresh halaman ini → kolom "Status Tugas" berubah jadi "Sudah diisi", muncul link/preview file
- [ ] Permit yang lewat jam sesi tanpa tugas terisi (`follow_up_needed: true` dari job T044) → badge "Perlu Ditindaklanjuti" tampil jelas di tabel
- [ ] Filter tanggal & guru berfungsi, tabel ter-update sesuai filter
