# Task-CORE-007 / WEB-010: Peta Lokasi antara Card Jadwal & Card Sesi (Jarak ke Sekolah)

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah diskusi dengan user (referensi visual peta "Ceklok KBM" JurnalePro). Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.
> Task gabungan backend (expose data lat/lng kampus) + frontend (render peta) — dipertahankan 1 file karena saling terkait erat.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** medium
**Alasan pemilihan:** Frontend menyentuh integrasi Leaflet (library baru di halaman ini meski sudah dipakai di modul lain), perlu GPS browser + kalkulasi jarak real-time. Backend murni expose field yang sudah ada di DB (low effort), tapi kombinasi keduanya + testing GPS permission flow butuh ketelitian medium.

## 2. Konteks & Tujuan Utama

Referensi user: halaman "Ceklok KBM" JurnalePro menampilkan peta interaktif (Leaflet/OSM) di ANTARA header jadwal dan list sesi — menunjukkan posisi guru (marker) relatif ke titik sekolah (ikon gedung), dengan label jarak (mis. "Jarak 61 m") muncul di popup marker.

**Keputusan user:** tambahkan peta serupa di halaman `/guru/jadwal`, diletakkan **di antara** card "Jadwal Mengajar Hari Ini" (header) dan card sesi pertama ("Mulai Mengajar").

**Temuan teknis (dicek Hermes sebelum menulis task ini):**
- Library `react-leaflet` + `leaflet` SUDAH terpasang di `apps/web/package.json` — TIDAK perlu instalasi baru.
- SUDAH ADA pola pemakaian peta di `apps/web/src/app/(admin)/kampus/kampus-map.tsx` (admin set titik lokasi kampus) — REUSE pola teknis yang sama (fix icon Leaflet+Next.js, TileLayer OSM, dll), TAPI itu untuk EDIT titik (klik-untuk-set), sedangkan kebutuhan di sini adalah TAMPILKAN posisi guru vs sekolah (read-only, 2 marker + circle radius geofence).
- `Kampus` model punya `lokasiLat`/`lokasiLng`/`radiusGeofenceMeter` (Decimal/Int, bisa null) — data SUDAH ADA di DB, tapi endpoint `GET /teaching-sessions/my-today` **BELUM** meng-expose field ini (cuma `kampus: string` nama saja, dicek di `SesiHariIniRow`/`teaching-sessions.service.ts`).
- Perhitungan jarak (`hitungJarakMeter`, Haversine) SUDAH ADA di `apps/api/src/common/geo.ts` — tapi itu untuk validasi SERVER SIDE saat submit. Untuk tampilan LIVE di map (jarak update saat guru bergerak), perlu hitung ULANG di CLIENT dari koordinat GPS browser vs koordinat kampus (JANGAN panggil API berulang kali hanya untuk hitung jarak — itu murni matematika, bisa di client).

## 3. Langkah Eksekusi Detail

### Backend — expose lokasi kampus ke endpoint my-today

1. Di `apps/api/src/teaching-sessions/teaching-sessions.service.ts`, method `getMyToday()` — tambahkan `kampusLat`, `kampusLng`, `kampusRadiusMeter` ke interface `SesiHariIni` DAN ke query `include: { kelas: { include: { kampus: true } } }` yang SUDAH ADA (baris 255) — data kampus SUDAH di-fetch, cuma perlu tambah field yang di-return (baris ~280 dan ~319, kedua cabang return — kasus `!jam` dan kasus normal).
2. Update `apps/web/src/lib/core-types.ts` — tambahkan field sepadan ke `SesiHariIniRow` (snake_case sesuai konvensi response API existing, cek field lain di interface yang sama untuk pola penamaan persis).
3. **Field boleh null** (`Kampus.lokasiLat`/`lokasiLng`/`radiusGeofenceMeter` semua nullable di schema) — kampus yang belum di-setup GPS-nya (lihat `kampus-view.tsx` admin, ada state "belum di-set") HARUS ditangani di frontend (lihat langkah 6).

### Frontend — render peta

4. Buat komponen baru `apps/web/src/app/(guru)/guru/jadwal/components/lokasi-map.tsx` — REUSE pola teknis `kampus-map.tsx` (fix icon Leaflet import, `MapContainer`/`TileLayer`/`Marker`/`Circle`), TAPI versi READ-ONLY (tidak ada `ClickToSetMarker`, tidak ada `draggable` marker) dengan 2 elemen:
   - **Marker sekolah** (icon gedung/`Building2` custom, atau default marker Leaflet — cek apakah ada custom icon lain di proyek untuk gedung, kalau tidak ada pakai default) di posisi `kampusLat`/`kampusLng`.
   - **Marker/titik guru** (posisi dari `navigator.geolocation`, REUSE fungsi `getGeolocation()` yang SUDAH ADA di `sesi-card.tsx` — cek apakah perlu diekstrak ke util shared kalau dipakai di 2 tempat sekarang, hindari duplikasi kode).
   - **Circle radius geofence** (sama pola `kampus-map.tsx` baris 84-90, radius dari `kampusRadiusMeter`).
   - **Popup/label jarak** — hitung `hitungJarakMeter` versi CLIENT (buat util JS sederhana yang sama formula Haversine, taruh di `apps/web/src/lib/geo.ts` — JANGAN import langsung dari `apps/api`, beda package) dari posisi guru vs kampus, tampilkan "Jarak X m" mengikuti referensi visual JurnalePro.

5. **Update posisi guru secara berkala** — GPS browser via `navigator.geolocation.watchPosition()` (BUKAN `getCurrentPosition()` sekali saja) supaya titik guru di peta bergerak kalau dia berpindah, konsisten dengan sifat "live map" referensi. Cleanup `clearWatch()` saat unmount.

6. **Tangani kasus data lokasi kampus NULL** (kampus belum di-setup GPS admin) — JANGAN crash/render peta kosong membingungkan, tampilkan pesan jelas ("Lokasi kampus belum diatur admin — hubungi Admin Jurnal") menggantikan area peta, KONSISTEN pola pesan error proyek ini (jelas, actionable, bukan generic).

7. **Tangani kasus GPS guru gagal/ditolak** — REUSE pesan error PERSIS yang sudah ada di `sesi-card.tsx` (`"Aktifkan lokasi (GPS) di HP Anda untuk memulai pembelajaran"`, KONSISTEN kalimat, jangan buat pesan baru) — tampilkan di area peta sebagai pengganti (peta tetap render TANPA marker guru kalau GPS gagal, hanya marker sekolah yang tampil, atau tampilkan pesan penuh — pilih salah satu, dokumentasikan keputusan).

8. Render `<LokasiMap />` di `jadwal-view.tsx`, diletakkan PERSIS di antara header card (`Jadwal Mengajar Hari Ini`) dan list `sesi.map(...)`.

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/api/src/teaching-sessions/teaching-sessions.service.ts` — expose lat/lng/radius kampus
- **Modifikasi:** `apps/web/src/lib/core-types.ts` — field baru `SesiHariIniRow`
- **File baru:** `apps/web/src/app/(guru)/guru/jadwal/components/lokasi-map.tsx`
- **Kemungkinan file baru:** `apps/web/src/lib/geo.ts` (util Haversine client-side)
- **Modifikasi:** `apps/web/src/app/(guru)/guru/jadwal/jadwal-view.tsx` — render `<LokasiMap />`
- **Jangan sentuh:** `apps/web/src/app/(admin)/kampus/kampus-map.tsx` (versi admin EDIT, biarkan terpisah dari versi guru READ-ONLY ini — REUSE POLA, bukan REUSE KOMPONEN, karena behaviornya beda: draggable vs read-only).

**Dilarang dilakukan:**
- Jangan hitung jarak lewat panggilan API berulang — murni matematika client-side dari 2 koordinat, tidak butuh round-trip server.
- Jangan tampilkan peta kalau kampus belum punya lat/lng — fallback pesan jelas, JANGAN peta kosong/blank yang membingungkan.
- Jangan lupa `watchPosition` di-clear saat unmount — GPS watch yang tidak dibersihkan menguras baterai HP guru terus-menerus di background.

**Skenario kegagalan yang WAJIB ditangani:**
- Kampus `lokasiLat`/`lokasiLng` null → pesan "Lokasi kampus belum diatur admin".
- GPS guru gagal/ditolak → pesan sama persis `sesi-card.tsx`.
- Guru berada SANGAT JAUH dari sekolah (device testing/simulasi) → peta tetap render dengan zoom yang masuk akal (jangan auto-zoom super jauh sampai kedua marker tidak terlihat sama sekali — pertimbangkan `map.fitBounds()` supaya kedua marker selalu masuk viewport).
- Sesi yang statusnya sudah "selesai" atau "sudah_diizinkan" (tidak butuh Mulai Mengajar lagi) — peta TETAP tampil (posisi header→peta→list card konsisten terlepas status sesi individual, ini bukan kondisional per-sesi tapi elemen tetap 1x di halaman).

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Peta tampil di antara header card dan list sesi, menunjukkan marker sekolah + marker posisi guru + circle radius geofence
- [ ] Label jarak (mis. "Jarak 61 m") ditampilkan, dihitung client-side, update saat guru bergerak
- [ ] Posisi guru update berkala via `watchPosition`, dibersihkan saat unmount
- [ ] Kasus kampus belum ada lokasi → pesan jelas, bukan peta kosong
- [ ] Kasus GPS gagal → pesan identik `sesi-card.tsx`
- [ ] Tidak ada pemborosan API call untuk hitung jarak

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini (KECUALI pilihan render saat GPS gagal — peta tanpa marker guru vs pesan penuh, didokumentasikan keputusan yang diambil)
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 250 baris perubahan gabungan BE+FE)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency (jika ada) sudah selesai sebelum task ini di-assign — tidak ada dependency
