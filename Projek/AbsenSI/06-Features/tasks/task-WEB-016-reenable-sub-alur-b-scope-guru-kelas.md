# Task-WEB-016: Re-enable + Scope Ulang Sub-alur B (Konfirmasi Izin Pulang)

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) — lihat `06-Features/desain-redesign-presensi-izin-keluar.md` §8, dan `06-Features/tasks/T264-hide-konfirmasi-izin-pulang-perbaikan-riwayat-izin.md` untuk konteks kenapa fitur ini di-hide sebelumnya. Dieksekusi oleh Claude Code.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** medium
**Alasan pemilihan:** Un-hide UI existing (low effort) TAPI filter query-nya berubah total (dari "semua siswa tap pulang" jadi "hanya yang izin dari guru kelas") DAN butuh auto-cleanup real-time — kompleksitas menengah di sinkronisasi state.

## 2. Konteks & Tujuan Utama

Baca `06-Features/desain-redesign-presensi-izin-keluar.md` §8 dan `T264-*.md` (histori hide). **Root cause T264**: section "Konfirmasi Izin Pulang (Sub-alur B)" di `izin-keluar-view.tsx` menampilkan SEMUA 885 siswa yang tap pulang hari itu (over-inklusif, tidak selektif) — di-hide 2026-08-31.

**Keputusan baru user**: AKTIFKAN LAGI, tapi filter HANYA siswa yang statusnya "Izin" berasal dari **inputan guru kelas** (hasil task-CORE-014/CORE-015 — bukan lagi query generik "semua yang tap pulang"). Auto-cleanup: entry hilang otomatis saat ganti hari ATAU guru ubah status Izin→Hadir.

**Depends on:** task-CORE-014, task-CORE-015 (data `ClassAttendanceMark.status = izin` dengan `permitId` harus sudah bisa terjadi).

## 3. Langkah Eksekusi Detail

1. **Un-comment** blok JSX "Konfirmasi Izin Pulang (Sub-alur B)" di `apps/web/src/app/(piket)/piket/izin-keluar/izin-keluar-view.tsx` (baris ~93-137, di-comment-out oleh T264 dengan komentar penanda jelas — cari komentar itu sebagai lokasi pasti).

2. **GANTI SUMBER DATA QUERY** — backend endpoint yang dipakai section ini SEBELUMNYA murni filter `waktuPulang !== null` (semua yang tap pulang). Buat/ubah endpoint BARU yang filter: siswa dengan `ClassAttendanceMark.status = "izin"` DAN `permitId IS NOT NULL` (dari alur guru kelas, task-CORE-014/015) UNTUK SESI HARI INI, JOIN ke `AttendanceRecord.waktuPulang` (opsional, kalau mau tetap tunjukkan info jam pulang aktual).

3. **Auto-cleanup 1 — ganti hari**: query SUDAH otomatis ter-scope "hari ini" (pola existing proyek untuk data harian — VERIFIKASI query baru di langkah 2 juga scoped tanggal `= CURDATE()`, konsisten pola lain), jadi TIDAK perlu mekanisme cleanup manual — besok otomatis kosong karena filter tanggal.

4. **Auto-cleanup 2 — guru ubah Izin→Hadir**: ini BUTUH mekanisme real-time. Opsi:
   - **Polling** (sederhana, KONSISTEN pola beberapa fitur piket lain di proyek — cek `usePiketOnDuty`/polling existing) — refetch list tiap interval pendek (mis. 30-60 detik).
   - **Socket.IO event** (lebih responsif, REUSE infrastruktur `AttendanceGateway` yang SUDAH ADA — emit event baru saat `ClassAttendanceMark` status berubah dari `izin` ke lainnya, dashboard piket subscribe event ini untuk refresh list).
   - **REKOMENDASI**: Socket.IO kalau `AttendanceGateway` sudah punya infrastruktur room yang cocok (cek scope kampus, REPLIKASI pola broadcast existing seperti `broadcastQrMulaiBerhasil`), supaya konsisten dengan pola real-time proyek lain — TAPI kalau ternyata terlalu kompleks untuk task ini, polling sederhana JUGA ACCEPTABLE sebagai MVP, catat di ringkasan hasil kalau memilih ini.

5. **Tombol aksi per baris** — REPLIKASI style existing (`SubAlurBRow` component, `handleTandaiIzinPulang`) TAPI sesuaikan label/copy supaya jelas ini "izin dari guru kelas" (bukan generic "tandai izin pulang") — mis. tampilkan kolom tambahan "Diajukan oleh: <nama guru>" dan "Alasan: <keterangan>" untuk konteks yang lebih jelas dari sebelumnya (perbaikan atas versi lama yang minim info).

6. **State lokal `sudahTapPulang`/`handleTandaiIzinPulang`** yang SEBELUMNYA di-comment (dicek T264 poin "boleh tetap ada tapi tidak dipakai") — sekarang AKTIF dipakai lagi, VERIFIKASI tidak ada sisa kode usang/tidak konsisten dari masa di-hide.

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/web/src/app/(piket)/piket/izin-keluar/izin-keluar-view.tsx` — un-hide + ganti sumber data
- **Modifikasi (kemungkinan):** endpoint backend baru/diubah untuk query filter baru (lokasi tergantung struktur — cek modul `attendance`/`journal` mana yang lebih tepat)
- **Modifikasi (kemungkinan):** `apps/api/src/realtime/attendance.gateway.ts` — event baru kalau pilih Socket.IO (langkah 4)

**Dilarang dilakukan:**
- Jangan kembalikan ke query lama (`waktuPulang !== null` generik) — itu PERSIS root cause T264, sudah terbukti tidak praktis.
- Jangan hapus endpoint `POST /attendance/confirm-izin-pulang/:id` — masih dipakai (T264 eksplisit bilang jangan disentuh backend-nya).

**Skenario kegagalan yang WAJIB ditangani:**
- Tidak ada siswa izin dari guru kelas hari ini → section tampil kosong dengan pesan jelas (BUKAN dihilangkan total dari halaman — tetap ada sebagai section, cuma isinya "Belum ada murid izin dari guru kelas hari ini").
- Guru ubah status TEPAT SAAT piket sedang lihat halaman (race condition) → auto-cleanup (langkah 4) harus terjadi tanpa piket perlu refresh manual, KONSISTEN maksud "otomatis terhapus".

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Section "Konfirmasi Izin Pulang (Sub-alur B)" aktif kembali
- [ ] HANYA menampilkan siswa dengan status Izin dari alur guru kelas (task-CORE-014/015), BUKAN semua yang tap pulang
- [ ] Otomatis kosong di hari berikutnya (scope tanggal)
- [ ] Otomatis hilang dari list kalau guru ubah status Izin→Hadir (real-time atau polling, tanpa piket refresh manual)
- [ ] Info tambahan (guru pengaju, alasan) ditampilkan untuk konteks lebih jelas dari versi lama

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini (KECUALI pilihan Socket.IO vs polling di langkah 4, boleh dipilih sesuai kompleksitas saat implementasi)
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 150 baris perubahan)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada — VERIFIKASI tidak mengulang masalah T264 (query harus SELEKTIF, bukan generik lagi)
- [ ] Dependency: task-CORE-014, task-CORE-015 WAJIB selesai dulu (butuh data `permitId` di `ClassAttendanceMark`)
