# T246 — Web+API: Setting Kampus — Peta Interaktif untuk Titik GPS + Radius Geofence

## Depends on
Tidak ada. Independen — tapi jadi PRASYARAT PRAKTIS untuk T249-T251 (alur QR mulai
pembelajaran) supaya lapis validasi GPS-nya benar-benar aktif saat testing/pemakaian nyata.

## Objective
Admin bisa set titik GPS + radius geofence tiap kampus lewat peta interaktif di halaman
Setting Kampus — field ini SUDAH ADA di database dan SUDAH DIPAKAI validasi backend, tapi
TIDAK ADA UI sama sekali untuk mengisinya.

## Konteks — Kondisi Kode Saat Ini (dikonfirmasi via riset 2026-08-25)

**Temuan penting**: `Kampus.lokasiLat`/`lokasiLng`/`radiusGeofenceMeter` (`schema.prisma:16-30`)
sudah ada sejak lama, dengan komentar eksplisit di kode: `// null = gating geofence SKIP`.
Field ini SUDAH dipakai `TeachingSessionsService.startSession()`
(`teaching-sessions.service.ts:422-441`) untuk validasi geofence saat guru mulai mengajar —
TAPI `kampus-view.tsx` (`apps/web/src/app/(admin)/kampus/kampus-view.tsx`) cuma kelola field
`nama` lewat dialog sederhana. **Akibatnya: geofence secara diam-diam TIDAK PERNAH aktif di
seluruh sistem** — kalau field null, backend `logger.warn` lalu skip validasi (baris 427-430),
tidak ada error, tidak ada yang sadar fitur ini "mati" sejak awal karena tidak pernah ada
cara mengisinya.

## Keputusan Diminta User (2026-08-25)
Input GPS lewat **peta interaktif** (klik/drag pin + drag radius lingkaran), BUKAN input
angka lat/lng manual — jauh lebih mudah dan akurat untuk admin sekolah yang awam GPS.

## Spec Detail

### 1. Dependency baru: Leaflet + OpenStreetMap
`leaflet` + `react-leaflet` (gratis, tanpa API key, beda dari Google Maps yang butuh billing)
— TAMBAHKAN ke `apps/web/package.json`. Tile layer OpenStreetMap standar (`tile.openstreetmap.org`)
cukup untuk kebutuhan ini, TIDAK PERLU akun/API key apa pun.

### 2. Backend — endpoint update Kampus
`apps/api/src/core/kampus/` (atau modul yang menangani Kampus saat ini, VERIFIKASI lokasi
persis) — endpoint update Kampus SUDAH ADA untuk `nama` (dari `kampus-view.tsx` existing) —
EXTEND DTO-nya terima `lokasiLat`/`lokasiLng`/`radiusGeofenceMeter` (semua optional, nullable
— TIDAK WAJIB diisi, konsisten desain awal "null = skip validasi", sekolah yang belum
sempat setting geofence tetap bisa jalan seperti sekarang).

`@LogActivity` WAJIB dipasang di endpoint ini kalau belum (cek existing endpoint update
Kampus, aturan wajib project untuk endpoint mutasi).

### 3. Frontend — perluas dialog edit Kampus dengan peta

`kampus-view.tsx` — dialog Edit (yang sudah ada untuk field `nama`) tambah 1 section peta:
- `<MapContainer>` (react-leaflet) center di `lokasiLat`/`lokasiLng` existing (kalau ada)
  atau default center Indonesia/lokasi umum kalau kampus baru belum pernah di-set.
- Marker yang bisa di-drag (`draggable`) — posisi baru update state `lokasiLat`/`lokasiLng`.
- Lingkaran radius (Leaflet `Circle`) di sekitar marker, radius dari state
  `radiusGeofenceMeter` — BISA diubah lewat slider/input angka terpisah di samping peta
  (radius dalam meter TIDAK intuitif di-drag langsung di peta kecil, slider/input lebih
  presisi — VERIFIKASI SAAT IMPLEMENTASI kalau ternyata drag-radius-langsung lebih baik UX-nya).
- Klik di mana pun pada peta JUGA memindahkan marker ke titik itu (bukan cuma drag marker
  yang sudah ada) — UX lebih cepat untuk set posisi awal.
- Tombol "Simpan" (dialog existing) kirim SEMUA field sekaligus (nama + 3 field GPS baru).

### 4. Mobile-responsif
Peta harus tetap USABLE di layar sempit (mobile-first, aturan wajib project) — tinggi peta
adaptif (`h-64` mobile, lebih besar di `sm:`/`md:` kalau perlu), kontrol zoom Leaflet default
sudah touch-friendly, TIDAK PERLU kustomisasi khusus mobile kecuali ternyata bermasalah saat
implementasi.

## Edge Cases
- **Kampus tanpa GPS di-set** (field null) — TETAP bisa dipakai normal, geofence skip
  seperti sekarang (behavior lama TIDAK berubah, cuma nambah CARA mengisinya).
- **Radius 0 atau negatif** — validasi FE+BE tolak, radius wajib > 0 kalau lat/lng diisi.
- **Isi lat/lng tapi radius kosong (atau sebaliknya)** — VERIFIKASI SAAT IMPLEMENTASI:
  REKOMENDASI keduanya wajib terisi bersamaan (all-or-nothing) supaya tidak ada state
  "GPS ada tapi radius tidak ada" yang membingungkan backend (`geofenceLengkap` di
  `teaching-sessions.service.ts:424-425` sudah cek KETIGA field harus non-null).

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/kampus/kampus-view.tsx` (tambah peta di dialog).
- **Modifikasi:** endpoint+DTO update Kampus di `apps/api/src/core/` (VERIFIKASI lokasi
  persis saat implementasi — cek `core.module.ts` atau serupa untuk struktur modul Core).
- **Modifikasi:** `apps/web/package.json` (tambah `leaflet`+`react-leaflet`).
- **Jangan sentuh:** `teaching-sessions.service.ts` geofence check — logic validasi SUDAH
  BENAR, task ini cuma nambah CARA MENGISI data yang dipakainya.

## Acceptance Criteria
- [x] Dialog Edit Kampus tampilkan peta interaktif, marker bisa di-drag/klik-pindah.
- [x] Radius geofence bisa diubah (input angka meter, tervisualisasi sebagai lingkaran
      oranye di peta).
- [x] Simpan berhasil update `lokasiLat`/`lokasiLng`/`radiusGeofenceMeter` di database —
      diverifikasi live via curl.
- [x] Kampus tanpa GPS di-set tetap berfungsi normal (regresi check — geofence skip seperti
      sebelumnya, tidak error) — diverifikasi live (hapus lokasi jadi null berhasil).
- [x] Peta tetap usable di layar mobile sempit (`h-64` mobile, `sm:h-80` desktop).
- [x] `@LogActivity` terpasang di endpoint update (sudah ada sejak sebelumnya, tidak
      disentuh).
- [x] Build + type-check hijau (`tsc --noEmit` api+web bersih).

## Validasi Claudian
- [x] Konfirmasi TIDAK ADA breaking change ke `startSession()` — field/nama/tipe database
      tidak berubah, method itu query Prisma langsung (tidak lewat `KampusService.serialize()`
      baru), TIDAK terpengaruh.
- [x] Test live via curl: PATCH kampus dengan GPS+radius berhasil, GET kembalikan number
      murni (bukan string Decimal — lihat bug ditemukan di bawah). Validasi all-or-nothing
      dan radius<=0 keduanya ditolak dengan pesan jelas sesuai spec.

## Bug Ditemukan + Diperbaiki Saat Implementasi (2026-08-26)

**Prisma `Decimal` ter-serialize JSON sebagai STRING, bukan number** — `lokasiLat`/
`lokasiLng` (`@db.Decimal(10,7)`) kalau dikembalikan APA ADANYA dari Prisma akan jadi
`"-0.3823"` (string) di response JSON, BUKAN `-0.3823` (number) — perilaku default
`Decimal.prototype.toJSON()` Prisma Client. Ditemukan via test curl langsung (bukan
tebakan), field ini SATU-SATUNYA tempat proyek pertama kali mengirim `Decimal` mentah ke
frontend (`TeachingSession.lokasiLat` lain cuma dipakai backend, tidak pernah diserialize
ke client). Kalau dibiarkan, komponen peta Leaflet (`lat: number`) akan dapat string
secara diam-diam — bug runtime halus (`.toFixed()` gagal, perbandingan koordinat salah).
Diperbaiki: `KampusService.serialize()` transform eksplisit `Number(kampus.lokasiLat)`
sebelum dikirim, dipanggil di SEMUA method publik (`findAll`/`create`/`update`).
`startSession()` (`teaching-sessions.service.ts`) TIDAK terdampak — query Prisma
langsung, sudah pakai `Number()` sendiri sejak awal, tidak lewat service ini.

## Implementasi (2026-08-26)

- Backend: `UpdateKampusDto` extend `lokasiLat`/`lokasiLng`/`radiusGeofenceMeter` (semua
  optional). `KampusService.validateGeofence()` baru — all-or-nothing (isi 1 field wajib
  isi ketiganya) + radius harus > 0, cek silang antar-field (tidak cukup 1 decorator DTO).
- Dependency baru: `leaflet`+`react-leaflet`+`@types/leaflet` di `apps/web`.
- Komponen baru `kampus-map.tsx` — `MapContainer` (react-leaflet) dengan klik-pindah
  marker + drag marker + lingkaran radius (`Circle`, warna oranye aksen tunggal DESIGN.md).
  Fix icon default Leaflet (404 di bundler modern, pola umum leaflet+Next.js). Custom hook
  `RecenterOnPositionChange` (Leaflet tidak auto re-center saat prop `center` berubah
  post-mount, dokumentasi resmi react-leaflet).
- `kampus-view.tsx` — import `KampusMap` via `next/dynamic({ssr:false})` (Leaflet akses
  `window` saat import, crash kalau di-SSR). Peta HANYA muncul di mode edit (bukan create)
  — alur wajar: buat kampus dulu (nama saja), baru set lokasi lewat dialog Ubah.
  `handleHapusLokasi()` reset ketiga field ke null sekaligus (konsisten all-or-nothing).
  Tombol Simpan disabled kalau lat terisi tapi radius <=0 (defense-in-depth FE, backend
  tetap validator utama).
- Frontend type `Kampus` (`core-types.ts`) tambah 3 field baru.
