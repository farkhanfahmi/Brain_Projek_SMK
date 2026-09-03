# Task-CORE-021 / WEB-025: Riwayat Izin — Tampilkan Pengajuan Guru Kelas yang Ditolak

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah audit kejanggalan Dashboard Piket + diskusi kritis dengan user (2026-09-03). Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.

**Task Terbuat:** 2026-09-03
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** low-medium
**Alasan pemilihan:** 1 endpoint baru (query sederhana, filter by status+rentang tanggal) + penggabungan tampilan di 1 tabel existing. Tidak ada logic bisnis baru, murni menambah visibilitas data yang sudah tersimpan.

## 2. Konteks & Tujuan Utama

Audit Dashboard Piket (2026-09-03) menemukan: saat piket klik "Tolak" di halaman "Permintaan Izin" (`ClassPermitRequestsService.tolak()`), baris `ClassPermitRequest` berstatus `ditolak` — datanya **TETAP TERSIMPAN** di database (bukan dihapus), TAPI **tidak ada satu pun halaman UI** yang menampilkannya kembali. Halaman "Riwayat Izin" (`riwayat-izin-view.tsx`) hanya membaca model `Permit` (dibuat SAAT "Izinkan", TIDAK PERNAH dibuat saat "Tolak") — jadi begitu piket klik Tolak, baris itu hilang dari state React lokal di halaman Permintaan Izin dan piket tidak punya cara lagi mengecek "kenapa izin si A kemarin ditolak".

**Keputusan yang sudah disepakati user (2026-09-03):**
1. Riwayat penolakan ditampilkan **digabung ke halaman "Riwayat Izin"** yang sudah ada — piket cukup 1 tempat untuk lihat semua riwayat izin (diterima maupun ditolak).
2. **TIDAK perlu** ditambahkan ke "Riwayat Catatan" (komponen `riwayat-catatan-table.tsx` di halaman detail siswa) — alasan didiskusikan: komponen itu isinya murni hal yang sudah FINAL/berefek nyata ke siswa (izin disetujui, dispen, terlambat, dst), sedangkan pengajuan ditolak justru TIDAK berefek apa pun ke siswa (tidak ada `Permit`, `ClassAttendanceMark` dihapus) — janggal dicampur di tabel yang sama. Komponen itu juga di-share ke wali kelas & admin, sementara keputusan user riwayat penolakan **cukup terlihat piket saja**.

**Depends on:** Tidak ada.

## 3. Langkah Eksekusi Detail

### Backend (`apps/api/src/class-permit-requests/`)

1. **Cek endpoint existing** — `listMenunggu()` (dipakai `GET /class-permit-requests?status=menunggu`) sudah ada tapi TIDAK filter rentang tanggal. `riwayat-izin-view.tsx` fetch `Permit` via `GET /permits?dari&sampai` (rentang tanggal, default hari ini). Task ini butuh endpoint SERUPA untuk `ClassPermitRequest` status `ditolak`.

2. **Tambah method `listDitolak(kampusId, dari, sampai)`** di `ClassPermitRequestsService` — query `classPermitRequest.findMany({ where: { status: 'ditolak', student: { kelas: { kampusId } }, session: { tanggal: { gte: dari, lte: sampai } } }, include: REQUEST_INCLUDE, orderBy: { decidedAt: 'desc' } })` — REPLIKASI pola scoping kampus yang SAMA dengan `listMenunggu()`/`findMenungguInKampus()` di file yang sama (jangan buat query scoping baru dari nol).

3. **Endpoint baru** `GET /class-permit-requests?status=ditolak&dari=YYYY-MM-DD&sampai=YYYY-MM-DD` di `class-permit-requests.controller.ts` — REPLIKASI pola parameter yang sudah dipakai `PermitsController` (`dari`/`sampai` query params) untuk konsistensi kontrak API antar endpoint riwayat. Role guard SAMA seperti endpoint `listMenunggu()` existing (`guru_piket`).

### Frontend (`apps/web/src/app/(piket)/piket/riwayat-izin/`)

4. **`page.tsx`** — tambah fetch paralel ke endpoint baru (`GET /class-permit-requests?status=ditolak&dari&sampai`, rentang tanggal SAMA dengan yang dipakai fetch `Permit`) — REPLIKASI pola `Promise.all` fetch paralel yang sudah dipakai di halaman piket lain (mis. `page.tsx` Izin Keluar T264 fetch `piket-board`+`izin-hari-ini` paralel).

5. **`riwayat-izin-view.tsx`** — terima prop baru `initialDitolak: ClassPermitRequestRow[]` (tipe SUDAH ADA dari task-CORE-014, cek `core-types.ts`). Gabungkan tampilan ke 1 tabel yang sama dengan `Permit`:
   - **Pendekatan tampilan**: satu tabel gabungan (BUKAN 2 tabel terpisah) — tambahkan baris `ClassPermitRequest` status `ditolak` ke array yang sama dengan `permits`, dengan bentuk data yang dinormalisasi supaya cocok dengan struktur kolom existing (Nama, Kelas, Jenis, Kategori, Keterangan, Tanggal, Jam Keluar, Status Kembali, Aksi). Untuk baris `ditolak`: kolom "Jenis" → tampilkan label khusus **"Ditolak"** (bukan "Tidak Masuk"/"Keluar" — beda makna, ini pengajuan yang gagal, bukan permit final), kolom "Kategori" → "-" (tidak ada kategori izin/sakit pasti untuk request yang ditolak, atau tampilkan alasan pengajuan asli kalau ada field-nya, VERIFIKASI SAAT IMPLEMENTASI field apa yang tersedia di `ClassPermitRequest`), kolom "Keterangan" → tampilkan `alasanTolak` (alasan piket menolak — ini yang paling penting buat piket lihat), kolom "Status Kembali" → "-" (tidak relevan untuk baris ditolak), kolom "Aksi" → kosong (tidak ada aksi lanjutan untuk baris ditolak, request sudah final).
   - **VERIFIKASI SAAT IMPLEMENTASI**: sort/filter existing (`accessors` di `filtered` useMemo, baris ~65-70) perlu extend untuk field campuran 2 sumber data — pastikan sort tetap jalan benar untuk kombinasi `Permit`+`ClassPermitRequest` yang dinormalisasi ke bentuk sama.
   - Tambahkan **badge/indikator visual jelas** untuk baris "Ditolak" (warna beda dari baris permit normal, mis. `bg-danger-bg`/`text-danger-text` — konsisten token warna "hal yang tidak disetujui" yang sudah dipakai di tempat lain proyek ini) supaya piket langsung bisa bedakan sekilas mana yang permit final vs pengajuan ditolak.

6. **Search** — pastikan search nama murid (`search` state existing) ikut mencakup baris `ditolak` yang baru ditambahkan (karena digabung ke array yang sama, seharusnya otomatis ikut ter-filter, VERIFIKASI SAAT IMPLEMENTASI).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/api/src/class-permit-requests/class-permit-requests.service.ts` — tambah `listDitolak()`
- **Modifikasi:** `apps/api/src/class-permit-requests/class-permit-requests.controller.ts` — tambah handling query `status=ditolak&dari&sampai`
- **Modifikasi:** `apps/web/src/app/(piket)/piket/riwayat-izin/page.tsx` — fetch tambahan
- **Modifikasi:** `apps/web/src/app/(piket)/piket/riwayat-izin/riwayat-izin-view.tsx` — gabung tampilan
- **Jangan sentuh:** `apps/web/src/components/riwayat-catatan-table.tsx` (KEPUTUSAN EKSPLISIT: TIDAK ditambahkan ke sini, lihat bagian 2), `PermitsService`/model `Permit` (data `ditolak` TETAP di model `ClassPermitRequest`, TIDAK dikonversi jadi `Permit` — itu justru desain yang sudah benar/sengaja, lihat komentar `tolak()` di `class-permit-requests.service.ts`).

**Dilarang dilakukan:**
- Jangan buat `Permit` baru untuk merepresentasikan penolakan — filosofi existing SUDAH BENAR (Permit hanya untuk izin yang benar-benar disetujui/final), task ini murni soal TAMPILAN gabungan, bukan ubah model data.
- Jangan tampilkan riwayat `ditolak` ini ke role selain `guru_piket` (endpoint baru WAJIB guard `guru_piket` saja, konsisten keputusan user "cukup di piket saja").

**Skenario kegagalan yang WAJIB ditangani:**
- Kondisi: tidak ada pengajuan ditolak di rentang tanggal yang dipilih → tabel gabungan tetap tampil normal (hanya berisi `Permit` seperti sebelumnya), tidak ada section/error kosong yang aneh.
- Kondisi: `alasanTolak` kosong/null (piket tolak tanpa isi alasan, kalau field ini opsional) → tampilkan "-" atau "Tidak ada alasan", BUKAN kosong/undefined mentah di UI.
- Kondisi: rentang tanggal yang dipilih piket sangat panjang, jumlah gabungan data besar → tetap client-side sort/search seperti pola existing (tidak perlu pagination baru, sesuai keputusan sebelumnya soal tabel piket).

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Halaman "Riwayat Izin" menampilkan baris pengajuan izin guru kelas yang ditolak, digabung dalam tabel yang sama dengan riwayat `Permit`.
- [ ] Baris "Ditolak" punya indikator visual jelas berbeda dari baris permit normal.
- [ ] Kolom "Keterangan" pada baris ditolak menampilkan `alasanTolak` dari piket.
- [ ] Search nama murid tetap berfungsi mencakup baris gabungan baru.
- [ ] Sort kolom tetap berfungsi benar untuk data campuran 2 sumber.
- [ ] Scope kampus & role `guru_piket` tegak di endpoint baru (tidak bocor ke role lain/kampus lain).
- [ ] `riwayat-catatan-table.tsx` TIDAK disentuh sama sekali (sesuai keputusan eksplisit).
- [ ] Build + typecheck bersih, test unit baru untuk `listDitolak()`.

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 200 baris perubahan)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency: tidak ada
