# Task-WEB-019: Legend Badge H/I/A + Riwayat 5 Pertemuan Terakhir di Tabel Presensi

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah user tunjukkan screenshot referensi JurnalePro presisi, 2026-09-03. task-WEB-013 (redesign checklist presensi) SUDAH DIEKSEKUSI & selesai — task ini TAMBAHAN terpisah untuk 2 elemen visual yang terlewat saat spec awal ditulis, BUKAN revisi task lama. Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.

**Task Terbuat:** 2026-09-03
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** low-medium
**Alasan pemilihan:** Legend badge murni styling (low). Riwayat 5 pertemuan butuh query baru (join TeachingSession+ClassAttendanceMark scoped mapel+kelas) — sedikit lebih kompleks, dorong ke medium bagian itu.

## 2. Konteks & Tujuan Utama

User menunjukkan screenshot presisi halaman Presensi JurnalePro (2026-09-03) yang punya 2 elemen visual BELUM ada di `AttendanceTable` AbsenSI (`apps/web/src/app/(guru)/guru/sesi/[sessionId]/components/attendance-table.tsx`, sudah di-redesign lengkap oleh task-WEB-013 — 3 tombol status Hadir/Izin/Alfa, popup wajib keterangan, tombol massal, submit tunggal — SEMUA itu SUDAH SELESAI, task ini TIDAK menyentuh ulang bagian itu):

1. **Legend badge di header tabel** — JurnalePro punya 4 badge (H/I/S/A) di pojok kanan atas daftar siswa sebagai legenda warna. AbsenSI HANYA punya 3 status (`hadir`/`izin`/`tidak_ada_di_kelas` — TIDAK ADA status Sakit terpisah, keputusan disepakati sebelumnya di redesign presensi) — jadi versi AbsenSI HANYA 3 badge: **H** (hijau/`success`), **I** (biru — token `info` atau `primary`, TENTUKAN konsisten dengan warna tombol status existing), **A** (merah/`danger`).

2. **Riwayat 5 pertemuan terakhir** — JurnalePro menampilkan 3 titik kecil di bawah nama tiap siswa menunjukkan histori kehadiran. **Keputusan user**: AbsenSI pakai **5 pertemuan terakhir** (bukan 3, supaya pola lebih kelihatan), diambil dari `ClassAttendanceMark` untuk **mapel+kelas yang SAMA** dengan sesi saat ini (bukan lintas mapel — riwayat matematika tidak relevan dicampur ke riwayat sejarah).

**Depends on:** Tidak ada dependency teknis baru — `ClassAttendanceMark.status` (3 nilai) dan `keterangan` SUDAH ADA di skema (dari task-CORE-010, sudah selesai).

## 3. Langkah Eksekusi Detail

### Legend Badge (Frontend Murni)

1. Di `AttendanceTable`, header tabel (area yang sama dengan tombol "Hadir Semua"/"Reset" atau baris `TableHeader` — TENTUKAN posisi paling pas secara visual saat implementasi, REKOMENDASI: sejajar kanan dengan judul "Presensi Kelas") — tambahkan 3 badge bulat kecil (diameter ~24px, label huruf tunggal putih di tengah):
   - **H** — `bg-success-text` (WARNA SAMA dengan token yang dipakai badge status "Hadir" existing di tabel, JANGAN token baru)
   - **I** — warna SAMA dengan badge status "Izin" existing (`text-primary`/`bg-primary` sesuai kode saat ini)
   - **A** — `bg-danger-text` (SAMA dengan badge status "Alfa" existing)
   - MURNI dekoratif/legenda — TIDAK ADA `onClick`, tidak interaktif.

### Riwayat 5 Pertemuan Terakhir (Backend + Frontend)

2. **Backend** — tambahkan field baru ke response `GET /teaching-sessions/:id/detail` (`JournalController.getDetail()`/`JournalService.getDetail()`) ATAU endpoint terpisah — TENTUKAN saat implementasi mana yang lebih murah (REKOMENDASI: tambah ke `getDetail()` existing, supaya 1 request saja, tidak perlu N+1 call per siswa).

3. Query: untuk SETIAP siswa di `detail.siswa`, ambil 5 `ClassAttendanceMark` TERAKHIR (join `TeachingSession`) dengan `mapelId` DAN `kelasId` SAMA dengan sesi saat ini, `tanggal < session.tanggal` (BUKAN termasuk sesi ini sendiri), `orderBy: tanggal desc`, `take: 5`. **Optimasi**: JANGAN query per-siswa dalam loop (N+1) — REPLIKASI pola batch query yang sudah dipakai proyek ini di tempat lain (mis. `resolveJamBatch()` di `AlokasiWaktuService`, sebagai referensi pola "1 query besar, map di memori" bukan query per-item).

4. Tambahkan field baru `riwayat: Array<{ tanggal: string; status: ClassAttendanceMarkStatus }>` per siswa di `SessionDetail`/`SessionDetailResponse` (FE types).

5. **Frontend** — di bawah nama tiap siswa di `AttendanceTable` (baris kecil, `text-caption`), render 5 titik kecil (`<span>` bulat kecil ~6px) berwarna sesuai status (hijau=hadir/warna-Izin=izin/merah=alfa, KONSISTEN warna legend langkah 1) untuk tiap entry `riwayat`. Urutkan kronologis (lama→baru dari kiri ke kanan, KONSISTEN 1 arah).

6. **Riwayat kurang dari 5** (siswa baru/awal semester) — render HANYA sejumlah entry yang benar-benar ada (mis. cuma 2 titik kalau baru 2 pertemuan) — JANGAN render titik placeholder/kosong untuk pertemuan yang belum terjadi (bisa menyesatkan seolah ada data kosong, bukan "belum ada").

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/api/src/journal/journal.service.ts` — `getDetail()`, interface `SessionDetail`
- **Modifikasi:** `apps/api/src/journal/journal.controller.ts` — expose `riwayat` di response
- **Modifikasi:** `apps/web/src/lib/core-types.ts` — `SessionDetailResponse` +field `riwayat` per siswa
- **Modifikasi:** `apps/web/src/app/(guru)/guru/sesi/[sessionId]/components/attendance-table.tsx` — legend badge + render riwayat

**Dilarang dilakukan:**
- Jangan tambah status ke-4 (Sakit) — KEPUTUSAN SUDAH FINAL sebelumnya, AbsenSI tetap 3 status.
- Jangan sentuh ulang logic checklist/popup/submit yang SUDAH SELESAI di task-WEB-013 — task ini MURNI tambahan visual di luar itu.
- Jangan query N+1 per siswa untuk riwayat — WAJIB batch query.

**Skenario kegagalan yang WAJIB ditangani:**
- Siswa baru pindah kelas (belum ada riwayat sama sekali untuk mapel+kelas ini) → riwayat kosong, tidak ada titik dirender, tidak error.
- Sesi ini adalah pertemuan PERTAMA untuk mapel+kelas ini di semester (tidak ada histori sebelumnya untuk siswa manapun) → semua siswa riwayat kosong, tabel tetap render normal tanpa titik.
- Siswa dengan riwayat campuran (mis. dari kelas lama sebelum pindah, `kelasId` beda) → TIDAK ikut terhitung (filter `kelasId` sesi saat ini secara ketat, riwayat dari kelas lain tidak relevan).

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] 3 badge legend (H/I/A) tampil di header tabel, warna konsisten dengan badge status per-baris
- [ ] Riwayat 5 pertemuan terakhir (mapel+kelas sama, sebelum sesi ini) tampil sebagai titik kecil berwarna di bawah nama siswa
- [ ] Riwayat kurang dari 5 → tampilkan hanya yang ada, tanpa placeholder kosong
- [ ] Query riwayat batch (bukan N+1 per siswa)
- [ ] Tidak ada perubahan pada logic checklist/submit yang sudah selesai (task-WEB-013)

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 150 baris perubahan gabungan BE+FE)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada (3 status final, bukan 4)
- [ ] Dependency: tidak ada — task-WEB-013 (checklist presensi) SUDAH SELESAI, ini murni tambahan independen
