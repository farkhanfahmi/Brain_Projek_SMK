# Task-WEB-013: Frontend — UI Checklist Presensi + Popup Alasan Alfa + Tombol Massal

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) — lihat `06-Features/desain-redesign-presensi-izin-keluar.md` §2-3. Dieksekusi oleh Claude Code.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** medium
**Alasan pemilihan:** Perombakan pola interaksi `AttendanceTable` dari instant-PATCH ke local-state-lalu-submit — komponen dipakai di 2 tempat (`/guru/sesi/[id]` dan nanti edit presensi task-WEB-014), perlu desain ulang yang tetap reusable.

## 2. Konteks & Tujuan Utama

Baca `06-Features/desain-redesign-presensi-izin-keluar.md` §2-3. `AttendanceTable` (`apps/web/src/app/(guru)/guru/sesi/[sessionId]/components/attendance-table.tsx`) saat ini toggle 1 status per klik, langsung PATCH. Desain baru: 3 status (Hadir/Izin/Alfa), state lokal dulu, popup wajib keterangan untuk Alfa, tombol massal "Hadir Semua"/Reset, submit sekali di akhir.

**Depends on:** task-CORE-011 (endpoint backend harus sudah terima payload+validasi baru).

## 3. Langkah Eksekusi Detail

1. **Redesain state `AttendanceTable`** — dari update-server-per-klik jadi state lokal dulu:
   ```ts
   const [localMarks, setLocalMarks] = useState<Record<number, { status: ClassAttendanceStatus; keterangan?: string }>>(...)
   ```
   Inisialisasi dari `initialSiswa` (status awal + keterangan kalau ada).

2. **3 tombol status per baris siswa** (bukan 2 seperti sekarang) — Hadir / Izin / Alfa, ikon compact terinspirasi JurnalePro TAPI styling KONSISTEN Design System v1 proyek (`primary`/`success`/`danger` token, BUKAN warna baru). Klik salah satu ubah state lokal `localMarks[studentId]`.

3. **Popup wajib keterangan saat pilih Alfa** — komponen `Dialog` (REUSE `@absensi/ui` Dialog yang sudah dipakai di proyek lain, mis. `create-assessment-dialog.tsx` sebagai referensi pola) muncul begitu guru klik tombol Alfa untuk 1 siswa — field textarea keterangan WAJIB diisi sebelum popup bisa ditutup dengan "Simpan" (tombol disabled kalau kosong). Kalau guru batal (klik luar/Escape/tombol Batal), status siswa itu TIDAK berubah (tetap status sebelumnya, BUKAN otomatis jadi Alfa tanpa keterangan).

4. **Tombol massal** di atas tabel — "Hadir Semua" (set semua `localMarks` jadi `hadir`, HANYA state lokal, belum submit) dan "Reset" (kembalikan ke state awal `initialSiswa`). REPLIKASI referensi JurnalePro (2 tombol berdampingan) tapi styling token v1.

5. **Indikator visual siswa belum tap gerbang** — badge kecil (SUDAH ADA di kode existing, `AlertCircle` icon) TETAP dipertahankan, tapi TAMBAHKAN styling row/border yang lebih jelas untuk siswa berstatus default Alfa (belum tap gerbang) — supaya guru langsung notice tanpa harus baca ikon kecil.

6. **Tombol "Simpan" di bagian bawah** (BARU — belum ada, sebelumnya tidak ada submit terpisah) — saat diklik, kirim SEMUA `localMarks` sebagai 1 payload `PATCH` ke endpoint (task-CORE-011). Tampilkan loading state, dan kalau backend return 400 (belum terabsen), **tampilkan pesan itu APA ADANYA** dari response (jangan generic "gagal menyimpan"), KONSISTEN pola error handling proyek (pesan actionable, bukan generik).

7. **`readOnly` prop existing DIPERTAHANKAN** — kalau `readOnly=true` (dipakai `/guru/presensi` saat ini), SELURUH UI checklist+tombol+popup TIDAK dirender sama sekali (murni tampilan, sama seperti sekarang) — task-WEB-014 nanti yang akan mengaktifkan mode edit terkontrol untuk kasus tertentu, BUKAN task ini.

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/web/src/app/(guru)/guru/sesi/[sessionId]/components/attendance-table.tsx` — redesain total interaksi

**Dilarang dilakukan:**
- Jangan hapus prop `readOnly` — masih dipakai `/guru/presensi`, task-WEB-014 yang akan menyesuaikan pemakaiannya di sana.
- Jangan submit otomatis tanpa tombol "Simpan" eksplisit — SEMUA perubahan HARUS local-state dulu, ini keputusan arsitektur inti redesign ini.
- Jangan izinkan popup Alfa ditutup dengan keterangan kosong — validasi FE + backend (task-CORE-011) harus dobel-lapis.

**Skenario kegagalan yang WAJIB ditangani:**
- Guru klik Simpan tapi ada siswa belum di-checklist → backend 400 dengan nama siswa, tampilkan pesan itu di UI, JANGAN biarkan guru bingung kenapa gagal tanpa penjelasan.
- Guru sudah isi popup keterangan Alfa untuk siswa A, lalu ganti pikiran klik Hadir untuk siswa A sebelum submit → keterangan lama otomatis dibuang (tidak perlu disimpan lagi karena status bukan Alfa lagi), state bersih.
- Koneksi gagal saat submit → error ditampilkan, state lokal TIDAK di-reset (guru tidak perlu checklist ulang dari nol, bisa retry submit langsung).

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] 3 tombol status (Hadir/Izin/Alfa) per baris siswa, state lokal (belum ke server) sampai Simpan diklik
- [ ] Popup wajib keterangan muncul untuk Alfa, tidak bisa ditutup tanpa isi
- [ ] Tombol "Hadir Semua" dan "Reset" berfungsi di state lokal
- [ ] Tombol "Simpan" mengirim semua perubahan sekaligus, tampilkan pesan error backend apa adanya
- [ ] `readOnly` mode tetap berfungsi seperti sebelumnya (tidak regresi)

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 250 baris perubahan)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency: task-CORE-011 WAJIB selesai dulu (endpoint backend harus terima payload baru)
