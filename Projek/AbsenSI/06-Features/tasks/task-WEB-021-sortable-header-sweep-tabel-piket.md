# Task-WEB-021: Sortable Header Sweep — Semua Tabel Dashboard Piket

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah audit kejanggalan Dashboard Piket + diskusi kritis dengan user (2026-09-03). Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.

**Task Terbuat:** 2026-09-03
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** medium
**Alasan pemilihan:** Mekanis (pola `SortableHeader` sudah ada & sudah dipakai sebagian di semua file ini, tinggal diterapkan konsisten ke kolom yang tersisa), TAPI area terdampak lumayan luas (banyak file+tabel) sehingga effort medium bukan low — risiko utamanya scope creep/miss 1-2 kolom, bukan kesulitan teknis.

## 2. Konteks & Tujuan Utama

Aturan wajib proyek (`CLAUDE.md` §"Tabel Data & Mobile-First", skill `absensi`): *"Setiap kolom di tabel data (kecuali kolom 'No' dan kolom aksi/tombol) wajib sortable asc/desc via `SortableHeader`... bukan cuma sebagian kolom dalam tabel yang sama. Plus search box + kolom 'No' paling kiri."*

Audit Dashboard Piket (2026-09-03) menemukan pelanggaran aturan ini BERULANG di hampir semua tabel area piket — komponen `SortableHeader` (`apps/web/src/components/sortable-header.tsx`) sudah tersedia dan SUDAH dipakai untuk sebagian kolom di semua file berikut, tapi tidak diterapkan konsisten ke SEMUA kolom non-No/non-aksi. Task ini adalah "sapu bersih" 1 kali untuk seluruh area piket sekaligus — supaya tidak ditambal satu-satu berulang kali di task berbeda (riwayat proyek: itu pola yang bikin drift terus terjadi).

**Daftar lengkap yang perlu diperbaiki** (hasil audit, VERIFIKASI ULANG nomor baris saat implementasi karena file bisa sudah berubah):

1. **`apps/web/src/app/(piket)/piket/piket-board-view.tsx`** — 5 tabel:
   - Board utama "Murid Belum Hadir": kolom `Status`, `Keterangan`, `Jam Masuk`, `Jam Pulang` belum sortable (hanya Nama & Kelas).
   - "Belum Kembali": kolom `Jam Keluar`, `Diharapkan Kembali` belum sortable.
   - "Tidak Absen Pulang": kolom `Jam Masuk` belum sortable (Tanggal sudah).
   - "Perlu Ditinjau": kolom `Tanggal`, `Diharapkan Kembali` belum sortable.
   - "Murid Terkunci": kolom `Dikunci Sejak`, `Alasan` belum sortable.
2. **`apps/web/src/app/(piket)/piket/permintaan-izin/permintaan-izin-view.tsx`** — **TIDAK ADA kolom No, TIDAK ADA search box, TIDAK ADA satu pun kolom sortable sama sekali** (paling parah — pelanggaran total, bukan sebagian). Kolom existing: Nama Siswa, Kelas, Guru Pengaju, Alasan, Jam Keluar, Jam Kembali, Aksi.
3. **`apps/web/src/app/(piket)/piket/izin-keluar/izin-keluar-view.tsx`** — tabel "Izin dari Guru Kelas Hari Ini": kolom `Mapel`, `Diajukan Oleh`, `Alasan` belum sortable (Nama/Kelas/Jam Keluar sudah).
4. **`apps/web/src/app/(piket)/piket/riwayat-izin/riwayat-izin-view.tsx`** — kolom `Jenis`, `Kategori`, `Keterangan`, `Status Kembali` belum sortable.
5. **`apps/web/src/app/(piket)/piket/riwayat-aktivitas/riwayat-aktivitas-view.tsx`** — kolom `Keterangan` belum sortable. **PLUS** search/sort saat ini hanya beroperasi di dalam 1 halaman aktif (server tetap paginasi terpisah) — VERIFIKASI apakah ini perlu diperbaiki jadi server-side search/sort, ATAU cukup diberi keterangan UI eksplisit "mencari hanya di halaman ini" supaya tidak menyesatkan piket yang mengira sedang mencari di seluruh riwayat (KEPUTUSAN: pilih opsi yang TIDAK mengubah pagination/query backend — cukup tambah teks kecil di dekat search box, mis. "Mencari di halaman ini saja" — TIDAK PERLU migrasi ke server-side search, itu di luar scope task ini yang fokus soal sortable header).
6. **`apps/web/src/app/(piket)/piket/jurnal/jurnal-view.tsx`** (`JurnalRiwayatCard`) — kolom `Catatan` belum sortable.

## 3. Langkah Eksekusi Detail

1. Baca `apps/web/src/components/sortable-header.tsx` untuk pahami API-nya (props, cara pakai) — REPLIKASI pola yang SUDAH dipakai untuk kolom Nama/Kelas di file yang sama (jangan reinvent).

2. Untuk **setiap file di daftar bagian 2** — ganti `<TableHead>Label</TableHead>` polos jadi `<SortableHeader>` untuk SEMUA kolom yang disebutkan, ikuti pola sort-state (`useState` sort key+direction, fungsi compare) yang SUDAH ada di file itu untuk kolom yang sudah sortable — EXTEND accessor-nya untuk kolom baru, jangan bikin sistem sort terpisah.

3. **Khusus `permintaan-izin-view.tsx`** — perbaikan lebih besar dari sekadar sortable:
   - Tambah kolom "No" paling kiri (offset halaman kalau ada pagination, REPLIKASI pola file lain — kemungkinan file ini tidak dipaginasi karena datanya kecil, cukup index+1 dari array yang sudah difilter/sort).
   - Tambah search box (REPLIKASI pola search di `izin-keluar-view.tsx`/`riwayat-izin-view.tsx` — cari berdasar nama siswa minimal, boleh diperluas ke kelas/guru pengaju kalau mudah).
   - Semua kolom (Nama Siswa, Kelas, Guru Pengaju, Alasan, Jam Keluar, Jam Kembali) jadi sortable via `SortableHeader`, kecuali kolom "No" dan "Aksi".

4. Untuk `riwayat-aktivitas-view.tsx` — tambah teks kecil dekat search box (lihat poin 5 di bagian 2) menjelaskan scope pencarian saat ini, TANPA mengubah logic backend/pagination.

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/web/src/app/(piket)/piket/piket-board-view.tsx`
- **Modifikasi:** `apps/web/src/app/(piket)/piket/permintaan-izin/permintaan-izin-view.tsx` (perubahan lebih besar — kolom No + search + sort baru)
- **Modifikasi:** `apps/web/src/app/(piket)/piket/izin-keluar/izin-keluar-view.tsx`
- **Modifikasi:** `apps/web/src/app/(piket)/piket/riwayat-izin/riwayat-izin-view.tsx`
- **Modifikasi:** `apps/web/src/app/(piket)/piket/riwayat-aktivitas/riwayat-aktivitas-view.tsx`
- **Modifikasi:** `apps/web/src/app/(piket)/piket/jurnal/jurnal-view.tsx`
- **Jangan sentuh:** Backend/endpoint manapun — task ini MURNI frontend, semua data yang perlu sudah tersedia di response yang sudah diambil.

**Dilarang dilakukan:**
- Jangan tambah pagination baru ke tabel manapun (sudah didiskusikan dengan user — tabel snapshot harian/antrean kecil tidak butuh, tabel log yang sudah dipaginasi tidak diubah struktur paginasinya).
- Jangan migrasi `riwayat-aktivitas-view.tsx` ke server-side search — di luar scope, cukup transparansi UI.

**Skenario kegagalan yang WAJIB ditangani:**
- Kondisi: kolom berisi nilai `null`/kosong (mis. `alasanDetail` null, `Jam Kembali` tidak diisi) → Perilaku benar: sort tetap berfungsi tanpa error, baris dengan nilai null konsisten diletakkan di salah satu ujung (awal/akhir, VERIFIKASI SAAT IMPLEMENTASI konsistensi dengan kolom lain yang sudah ada di file yang sama).
- Kondisi: `permintaan-izin-view.tsx` kosong (tidak ada pengajuan menunggu) → search box tetap tampil tapi tidak error, tabel tampilkan empty state seperti biasa.

**Edge case:**
- Sekolah dengan jumlah pengajuan/riwayat besar → sort tetap client-side (data sudah di-fetch penuh di semua file ini kecuali Riwayat Aktivitas yang dipaginasi server — untuk itu sort HANYA berlaku di dalam halaman aktif, konsisten dengan search yang juga per-halaman, sudah dijelaskan di poin 5 bagian 2).

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Semua kolom non-No/non-Aksi di 5 tabel `piket-board-view.tsx` sudah sortable asc/desc.
- [ ] `permintaan-izin-view.tsx` punya kolom No, search box, dan semua kolom non-No/non-Aksi sortable.
- [ ] Tabel "Izin dari Guru Kelas Hari Ini" di `izin-keluar-view.tsx` — semua kolom sortable.
- [ ] `riwayat-izin-view.tsx` — semua kolom sortable.
- [ ] `riwayat-aktivitas-view.tsx` — kolom `Keterangan` sortable + teks penjelasan scope pencarian ditambahkan.
- [ ] `jurnal-view.tsx` (`JurnalRiwayatCard`) — kolom `Catatan` sortable.
- [ ] Build + typecheck bersih (`tsc --noEmit` web, `next build`).

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 300 baris perubahan; kalau lebih dari itu saat implementasi, PERTIMBANGKAN pecah jadi 2 task: board-view sendiri, sisanya gabung)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency: tidak ada
