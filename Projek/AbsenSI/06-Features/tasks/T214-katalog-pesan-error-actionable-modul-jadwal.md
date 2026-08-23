# T214 — API+Web: Audit & Katalog Pesan Error Actionable — Seluruh Modul Jadwal Baru

## Depends on
**REKOMENDASI dikerjakan SETELAH T206, T208, T209, T212, T213 selesai** (butuh SEMUA endpoint+UI modul baru sudah ada untuk diaudit menyeluruh) — TAPI task ini BISA JUGA dikerjakan SECARA BERTAHAP PARALEL dengan task lain (tiap task T206-T213 SUDAH diminta menulis pesan error jelas di masing-masing spec-nya) — task INI adalah **AUDIT FINAL+KONSOLIDASI** memastikan KONSISTENSI lintas semua titik, bukan menulis pesan error dari nol.

## Objective
User EKSPLISIT minta: **"perhatikan juga masalah UX terutama pesan error yang tampil agar user tidak bingung (berikan solusi apa yang harus dilakukan di pesan errornya)"** — task ini audit MENYELURUH semua pesan error di modul jadwal baru (T203-T213), pastikan SETIAP pesan error mengikuti pola: **[APA yang salah] + [APA yang harus dilakukan admin untuk memperbaiki]** — bukan sekadar "gagal" atau pesan teknis mentah.

## Prinsip Wajib — Pola Pesan Error (KONSISTEN Semua Titik)

**Format WAJIB**: `[Kondisi spesifik yang gagal] — [langkah konkret yang harus admin lakukan]`.

Contoh BENAR (SUDAH ditulis di beberapa task sebelumnya sebagai referensi pola):
- ❌ BURUK: "Gagal menyimpan jadwal."
- ✅ BAIK: "Guru Budi Santoso sudah mengajar di kelas X TKR 1 pada jam yang sama — pilih guru lain atau ubah jam."
- ❌ BURUK: "Mode tidak valid."
- ✅ BAIK: "Opsi Jadwal ini mode Normal, kolom Minggu harus dikosongkan — hapus isian di kolom Minggu untuk baris ini."
- ❌ BURUK: "Bentrok terdeteksi."
- ✅ BAIK: "Guru [Nama] sudah mengajar di kelas [Nama Kelas] jam ke-[X] hari [Y] — pilih jam lain atau guru pengganti."

## Spec Detail

### 1. Audit backend — katalog SEMUA titik validasi yang melempar exception di modul jadwal baru

Buat daftar LENGKAP (di laporan implementasi, BUKAN file terpisah) SEMUA titik `throw new BadRequestException/ConflictException/ForbiddenException/NotFoundException` di:
- `AlokasiWaktuService` (T206) — termasuk validasi hapus jamKe terpakai.
- `OpsiJadwalService` (T206) — mode permanen, overlap cakupan tingkat, dependency hapus.
- `JadwalSlotService` (T206) — SEMUA 5+ validasi (minggu, jamKe valid, duplikat, guru terdaftar, bentrok).
- `OpsiJadwalMingguGenerate`/Date Generator (T208) — selisih 7 hari, rentang di luar semester.
- Import (T213) — SEMUA kondisi tolak per baris.

UNTUK SETIAP titik — VERIFIKASI pesan SUDAH mengikuti format wajib di atas. KALAU BELUM, PERBAIKI di titik itu langsung (task ini BOLEH mengedit kode task lain, KARENA ini audit lintas-task yang baru bisa dilakukan setelah semuanya ada).

### 2. Audit frontend — pastikan pesan backend diteruskan APA ADANYA

- Untuk SETIAP form/aksi di modul jadwal baru (Workspace T212, Menu Utama T210, Sub-Menu T211, Import T213) — VERIFIKASI error handling di frontend meneruskan `err.message` dari backend APA ADANYA ke UI (toast/inline error), **TIDAK di-generic-kan** jadi "Terjadi kesalahan" atau serupa — KONSISTEN aturan CLAUDE.md "Pesan error sesuai kondisi, bukan generik... Pesan dari backend diteruskan APA ADANYA ke UI".
- **KHUSUS dropdown Guru real-time (T212)** — badge merah "(Mengajar di Kelas [Nama Kelas])" untuk guru bentrok BUKAN exception/toast (itu preventif UI, bukan error state) — TAPI pastikan teksnya JELAS dan SPESIFIK (sebutkan kelas, BUKAN cuma icon/warna tanpa teks).

### 3. Skenario UX khusus yang WAJIB dicek — "kenapa saya tidak bisa X"

Untuk SETIAP kondisi berikut, pastikan admin MENDAPAT PENJELASAN JELAS (bukan tombol yang disabled tanpa alasan, atau silent no-op):
- Kenapa tombol "Buat Opsi Jadwal Baru" TIDAK BISA submit → field mode belum dipilih, ATAU field wajib lain kosong (validasi FE inline, JELAS per-field, bukan 1 pesan generic di bawah form).
- Kenapa 1 guru TIDAK MUNCUL di dropdown Guru saat pilih Mapel tertentu → karena guru itu BELUM terdaftar `MapelGuru` untuk Mapel itu — TAMBAHKAN teks bantuan di bawah dropdown ("Guru yang dicari tidak ada? Daftarkan dulu di menu Mata Pelajaran") KALAU dropdown kosong/list pendek — BUKAN cuma dropdown kosong tanpa penjelasan.
- Kenapa tombol Hapus Alokasi Waktu/Opsi Jadwal DISABLED → tooltip/pesan JELAS ("Masih dipakai N jadwal — hapus jadwal itu dulu" / "Opsi ini sedang aktif — nonaktifkan dulu").
- Kenapa Warning Banner (T210) muncul → SUDAH dirancang jelas di T210, VERIFIKASI ULANG di sini teksnya benar-benar actionable (bukan cuma informatif tapi TIDAK bilang apa yang perlu admin lakukan — REKOMENDASI tambah kalimat solusi eksplisit: "...aktifkan Semester/Tahun Ajaran terkait untuk membuat opsi ini berlaku").

## Edge Cases
- Error dari VALIDASI RACE CONDITION (2 admin submit hampir bersamaan, salah satu menang) — pesan HARUS bisa membedakan ini dari validasi normal ("Data ini baru saja diubah admin lain, coba muat ulang" — BUKAN pesan yang sama seperti validasi bentrok biasa, supaya admin tahu perlu REFRESH bukan GANTI PILIHAN).

## Files
- **Modifikasi:** SEMUA file service/controller/komponen dari T206, T208, T209, T210, T211, T212, T213 yang pesan errornya BELUM sesuai format wajib (daftar spesifik ditentukan SAAT audit, tidak bisa diprediksi sebelumnya).

## Acceptance Criteria
- [x] SEMUA pesan error backend modul jadwal baru mengikuti format `[apa yang salah] + [apa yang harus dilakukan]` — verified dengan daftar lengkap di laporan implementasi.
- [x] SEMUA frontend meneruskan pesan backend apa adanya, tidak di-generic-kan.
- [x] 4 skenario UX khusus (poin 3) diverifikasi punya penjelasan jelas, bukan silent/disabled tanpa alasan.
- [x] Build + type-check hijau (task ini murni edit teks pesan, TIDAK ada perubahan logic bisnis).

## Validasi Claudian
- [x] **WAJIB tulis daftar LENGKAP** semua titik pesan error yang diaudit — lihat Status Eksekusi di bawah.
- [x] Konfirmasi task ini TIDAK mengubah logic validasi apa pun — dikonfirmasi: SEMUA edit adalah literal string di dalam `throw new XException(...)` / JSX teks, TIDAK ADA perubahan kondisi `if`, urutan validasi, atau nilai yang dibandingkan. 565 test backend lulus tanpa perubahan assertion apa pun (test pakai `toBeInstanceOf`/`toMatchObject({status})`, bukan string-match pesan, jadi tidak ada test yang perlu disesuaikan — bukti tidak ada logic yang bergeser).

## Status Eksekusi

**Selesai 2026-08-17 10:35** — cakupan T213 SUDAH termasuk (selesai duluan oleh sesi lain sebelum task ini dimulai, semula direncanakan skip tapi ternyata sudah ada).

### Daftar lengkap titik pesan error backend (throw exception)

**`AlokasiWaktuService` (T206)** — 3 titik, SEMUA SUDAH sesuai format, tidak diubah:
- `findOne()` not-found → sudah actionable ("pilih Alokasi Waktu lain atau buat yang baru")
- `update()` jamKe masih dipakai JadwalSlot → sudah actionable (sebut jam+hari+jumlah jadwal, "hapus/pindahkan jadwal itu dulu")
- `delete()` dirujuk OpsiJadwal → sudah actionable (sebut nama Opsi Jadwal, "hapus atau ubah Opsi Jadwal itu dulu")

**`OpsiJadwalService` (T206+T208)** — 9 titik, 4 DIPERBAIKI:
- `findOne()` not-found → sudah actionable, tidak diubah
- `create()` AlokasiWaktu not-found → **DIPERBAIKI** (tambah "pilih Alokasi Waktu lain atau buat dulu di menu Alokasi Waktu")
- `ensureNoTingkatOverlap()` overlap cakupan tingkat → sudah actionable, tidak diubah
- `delete()` masih aktif → sudah actionable, tidak diubah
- `delete()` masih punya JadwalSlot → sudah actionable (sebut jumlah), tidak diubah
- `generateMingguAB()` OpsiJadwal not-found → **DIPERBAIKI** (tambah "muat ulang halaman dan pilih Opsi Jadwal lain")
- `generateMingguAB()` mode bukan blok → sudah actionable, tidak diubah
- `generateMingguAB()` selisih bukan 7 hari → sudah actionable (sebut selisih aktual), tidak diubah
- `generateMingguAB()` rentang di luar semester → sudah actionable (sebut rentang semester valid), tidak diubah
- `deleteMingguGenerateTanggal()` not-found → **DIPERBAIKI** (tambah "muat ulang halaman untuk melihat daftar tanggal terkini")

**`JadwalSlotService` (T206+T213)** — 20 titik, 12 DIPERBAIKI:
- `findOne()` not-found → **DIPERBAIKI** (tambah "muat ulang halaman untuk melihat data terkini")
- `getOpsiJadwalOrThrow()` not-found → **DIPERBAIKI** (tambah "muat ulang halaman dan pilih Opsi Jadwal lain")
- Validasi #1 minggu-wajib-iff-blok (2 varian: blok-tanpa-minggu, normal-dengan-minggu) → **KEDUANYA DIPERBAIKI** (tambah instruksi eksplisit "pilih A/B" / "hapus isian kolom Minggu")
- Validasi #2 jamKe tidak valid → sudah actionable, tidak diubah
- Validasi #3 duplikat kelas-hari-jamKe → sudah actionable, tidak diubah
- Validasi #4 guru belum terdaftar MapelGuru → sudah actionable, tidak diubah
- Validasi #5 guru bentrok → sudah actionable (sebut nama guru+kelas+jam+Opsi Jadwal), tidak diubah
- Import: file Excel tanpa sheet → **DIPERBAIKI** (tambah "download template import dan isi ulang")
- Import: header kolom kosong → **DIPERBAIKI** (sama)
- Import baris: kolom wajib kosong → **DIPERBAIKI** ("lengkapi kolom yang kosong lalu upload ulang")
- Import baris: hari tidak valid → **DIPERBAIKI** ("ganti jadi salah satu dari Senin/.../Sabtu")
- Import baris: jam_ke tidak valid → **DIPERBAIKI** ("perbaiki nilai kolom jam_ke")
- Import baris: kolom minggu tidak valid → **DIPERBAIKI** ("isi dengan A/B, atau kosongkan")
- Import baris: kelas tidak ditemukan → **DIPERBAIKI** ("cek ejaan atau buat dulu di menu Kelas")
- Import baris: nama kelas ambigu → **DIPERBAIKI** ("ganti nama kelas supaya unik")
- Import baris: mapel tidak ditemukan → **DIPERBAIKI** ("cek ejaan atau buat dulu di menu Mata Pelajaran")
- Import baris: nama mapel ambigu → **DIPERBAIKI** (sama pola)
- Import baris: kolom guru >1 nama → **DIPERBAIKI** ("pisahkan tiap guru jadi baris terpisah")
- Import baris: guru tidak ditemukan → **DIPERBAIKI** ("cek ejaan atau daftarkan di menu Data Guru")
- Import baris: nama guru ambigu → **DIPERBAIKI** ("gunakan NIY atau nama lengkap yang lebih spesifik")
- Import baris: gagal simpan (exception dari `create()`) → REUSE pesan create() apa adanya (sudah actionable by construction, tidak diubah)

**`JadwalPelajaranService` (T211)** — 0 titik throw (read-only, delegasi edit ke `JadwalSlotService.update()` apa adanya) — tidak ada yang diaudit di sini.

### Audit frontend — passthrough pesan backend

Grep menyeluruh SEMUA `catch (err)` di route `jadwal-pelajaran/**` (7 titik: `jadwal-pelajaran-view.tsx` x3, `edit-inline-dialog.tsx`, `tab-hari-input.tsx`, `date-generator-panel.tsx`, `import-dialog.tsx` x2) — SEMUA sudah pola `err instanceof Error ? err.message : "<fallback>"`, TIDAK ADA yang men-generic-kan pesan backend. Tidak ada perbaikan diperlukan di titik ini.

### 4 skenario UX khusus (spec poin 3)

1. **Tombol "Buat Opsi Jadwal Baru" disabled tanpa alasan** → **DIPERBAIKI**: tambah `title="Pilih Tahun Ajaran dan Semester dulu sebelum membuat Opsi Jadwal"` (pola sama seperti tombol Hapus yang sudah benar).
2. **Guru tidak muncul di dropdown karena belum MapelGuru** → **DIPERBAIKI di 3 titik** (`tab-hari-input.tsx` mobile+desktop card kosong, `guru-dropdown-realtime.tsx` popover kosong): tambah "— daftarkan dulu di menu Mata Pelajaran".
3. **Tombol Hapus Opsi Jadwal/Alokasi Waktu disabled** → SUDAH BENAR sebelum audit (`title={opsi.isActive ? "Nonaktifkan dulu sebelum hapus" : undefined}` di `jadwal-pelajaran-view.tsx`), tidak diubah.
4. **Warning Banner (T210) informatif tapi tidak actionable** → **DIPERBAIKI**: tambah kalimat solusi eksplisit "aktifkan {Semester/Tahun Ajaran} terkait di menu Kalender Pendidikan/Tahun Ajaran supaya Opsi Jadwal ini benar-benar berlaku".

### Temuan LOGIC terpisah (BUKAN diperbaiki di task ini, sesuai instruksi eksplisit)

**Edge case race condition (spec bagian Edge Cases) BELUM diimplementasikan** — saat ini SEMUA exception (validasi normal ATAU race 2-admin-submit-bersamaan) pakai jalur pesan yang SAMA (mis. bentrok guru). Tidak ada mekanisme untuk membedakan "tolakan normal karena pilihan salah" vs "tolakan karena data baru saja berubah di request lain". Memperbaiki ini butuh perubahan LOGIC (optimistic locking / version check / re-fetch-and-compare sebelum throw), BUKAN sekadar teks — di luar scope task ini. **Direkomendasikan sebagai task terpisah kalau prioritas** (kemungkinan kecil terjadi di skala pemakaian AbsenSI saat ini, tapi tetap gap nyata dari spec).
