# T251 — Web: Guru — Integrasi Scanner QR ke Tombol "Mulai Mengajar" Existing

## Depends on
**T249** (backend terima `qrToken`, tabel pesan error) — WAJIB selesai dulu. Sebaiknya
**T250** juga sudah ada (butuh siswa yang bisa generate QR untuk testing end-to-end),
walau task ini SECARA TEKNIS bisa dikerjakan paralel dengan T250.

## Objective
Tombol "Mulai Mengajar" yang SUDAH ADA di jadwal guru (`sesi-card.tsx`) — sisipkan langkah
scan kamera SEBELUM commit, urutan permission: GPS dulu baru kamera, pesan error spesifik
per kondisi kegagalan (penekanan eksplisit user).

## Konteks — Kode Existing yang Diubah

`apps/web/src/app/(guru)/guru/jadwal/components/sesi-card.tsx`, fungsi
`handleMulaiMengajar()` (baris 57-77) SAAT INI:
```ts
async function handleMulaiMengajar() {
  setError(null);
  setStarting(true);
  try {
    const position = await getGeolocation();
    const result = await apiClientFetch<StartSessionResponse>(
      `/teaching-sessions/${sesi.session_id}/start`,
      { method: "POST", body: JSON.stringify({ lat: position.coords.latitude, lng: position.coords.longitude }) },
    );
    router.push(`/guru/sesi/${result.session_id}`);
  } catch (err) {
    setError(err instanceof Error ? err.message : "Gagal memulai sesi mengajar");
    setStarting(false);
  }
}
```
`getGeolocation()` (baris 21-31) SUDAH ADA, sudah handle GPS gagal dengan pesan
"Aktifkan lokasi untuk memulai sesi mengajar" — INI PESAN LAMA, task ini perlu pesan yang
LEBIH SPESIFIK sesuai tabel di T249 ("Aktifkan lokasi (GPS) di HP Anda untuk memulai
pembelajaran" — VERIFIKASI SAAT IMPLEMENTASI apakah perlu diseragamkan persis dengan
kalimat itu atau pesan existing sudah cukup jelas, KEDUANYA secara makna sudah benar,
prioritaskan konsistensi lintas T249-T251 dalam 1 kalimat yang SAMA).

## Spec Detail

### 1. Urutan baru: GPS → buka kamera → scan → submit
Ubah `handleMulaiMengajar()` jadi 2 tahap:
1. **Dapatkan GPS DULU** (`getGeolocation()`, SUDAH ADA, TIDAK diubah logic-nya) — kalau
   gagal, STOP di sini, JANGAN buka kamera sama sekali (hemat 1 permission prompt kalau
   GPS saja sudah gagal).
2. **GPS berhasil → buka overlay kamera** (komponen baru, lihat poin 2) — scan sukses
   dapat `qrToken` → submit ke `startSession()` (endpoint SAMA, body TAMBAH field `qrToken`).

### 2. Komponen scanner kamera baru
Dependency baru: library scan QR dari kamera browser (rekomendasi `html5-qrcode` atau
`jsqr`+`getUserMedia` manual — VERIFIKASI SAAT IMPLEMENTASI mana yang lebih ringan/stabil
lintas browser mobile, `html5-qrcode` lebih siap pakai dengan viewfinder bawaan).

Overlay full-screen (atau modal besar, VERIFIKASI SAAT IMPLEMENTASI mana yang lebih baik
di mobile) — viewfinder kamera + teks instruksi "Arahkan kamera ke QR Ketua/Wakil Ketua
Kelas" + tombol "Batal" (kembali ke card, tidak jadi mulai).

### 3. Tabel pesan error — REUSE PERSIS dari T249, TIDAK BOLEH beda kalimat

| Kondisi | Pesan yang tampil di UI guru |
|---|---|
| GPS ditolak/gagal (browser) | "Aktifkan lokasi (GPS) di HP Anda untuk memulai pembelajaran" |
| Kamera ditolak (browser) | "Izinkan akses kamera untuk scan QR Ketua Kelas" + instruksi singkat buka setting browser kalau permission ke-block permanen |
| Semua error dari `startSession()` (existing + baru T249) | Pesan backend DITERUSKAN APA ADANYA (aturan wajib project — JANGAN di-generic-kan ulang di frontend) |

**Setelah scan sukses TAPI backend menolak** (misal QR ternyata sudah expired pas sampai
ke server) — kamera TETAP terbuka/scanner tetap aktif, tampilkan pesan error di atas/bawah
viewfinder (BUKAN nutup overlay paksa) — supaya guru bisa langsung coba scan ulang tanpa
klik "Mulai Mengajar" dari awal lagi (UX lebih cepat, sesuai semangat "error harus
actionable" — beri kesempatan retry cepat, bukan cuma kasih tahu lalu buntu).

### 4. Sukses
Scan+submit berhasil → overlay kamera tertutup, state SAMA seperti existing
(`router.push(`/guru/sesi/${result.session_id}`)`, TIDAK berubah).

### 5. Mobile-first
Ini SUDAH mobile-context by nature (guru pegang HP di kelas) — pastikan overlay kamera
proporsional di layar kecil, tombol Batal mudah dijangkau ibu jari (posisi bawah, bukan
pojok atas kecil).

## Edge Cases
- **Guru buka overlay kamera tapi device TIDAK PUNYA kamera** (jarang tapi mungkin di
  laptop tanpa webcam kalau guru pakai laptop, bukan HP) — pesan jelas "Perangkat Anda
  tidak memiliki kamera, gunakan HP untuk memulai pembelajaran" BUKAN error teknis mentah.
- **Guru scan QR yang BUKAN untuk sesi ini** (misal salah kelas, QR basi dari kemarin kalau
  somehow ke-cache di layar siswa — seharusnya tidak mungkin karena token selalu baru, tapi
  VERIFIKASI defensif) — backend HARUS tolak (token tidak cocok di Redis key sesi itu),
  pesan "QR tidak valid atau sudah kadaluarsa" (sama seperti T249).
- **Koneksi internet guru putus saat proses submit** — pesan jaringan JELAS ("Gagal
  terhubung ke server, periksa koneksi internet Anda"), BUKAN pesan yang menyiratkan QR-nya
  salah (beda akar masalah, jangan disamakan).

## Files
- **Modifikasi:** `apps/web/src/app/(guru)/guru/jadwal/components/sesi-card.tsx`
  (`handleMulaiMengajar()`, sisip langkah scan).
- **Buat:** komponen scanner kamera baru (lokasi VERIFIKASI SAAT IMPLEMENTASI — kemungkinan
  `apps/web/src/components/qr-scanner.tsx` kalau cukup generik dipakai ulang, atau lokal ke
  folder `jadwal/components/` kalau spesifik konteks ini saja).
- **Modifikasi:** `apps/web/package.json` (tambah library scan QR kamera).
- **Modifikasi:** `POST /teaching-sessions/:id/start` body terima `qrToken` (koordinasi
  dengan T249, endpoint sama, TIDAK bikin endpoint baru).

## Acceptance Criteria
- [x] Klik "Mulai Mengajar" — GPS diminta LEBIH DULU (`handleMulaiMengajar()`, logic
      `getGeolocation()` TIDAK diubah), kamera baru dibuka (`setShowScanner(true)`) SETELAH
      GPS sukses — GPS gagal langsung STOP di card, kamera tidak pernah diminta.
- [x] Scan QR valid + semua validasi lain lolos → sesi dimulai, behavior SAMA seperti
      sebelumnya (`router.push(/guru/sesi/${result.session_id})`, TIDAK diubah).
- [x] SETIAP kondisi gagal (tabel spec) tampil pesan SPESIFIK — GPS ditolak (kalimat PERSIS
      disamakan dgn T249), kamera ditolak/tidak ada (dibedakan via regex `NotFoundError`),
      error jaringan murni (`err instanceof TypeError`) vs error backend (pesan backend APA
      ADANYA) — tidak ada 1 pun jalur baru yang jatuh ke pesan generik lama.
- [x] Scan gagal karena backend tolak — scanner TETAP terbuka (`showScanner` TIDAK diubah
      di catch block `handleScanDecode()`, cuma `scanError` diisi + `submitting` direset)
      untuk retry cepat.
- [x] Overlay kamera responsif — full-screen fixed inset-0, video `object-cover` mengisi
      penuh, tombol Batal besar (`h-11`) di bawah (dekat ibu jari). **Test HP fisik BELUM
      dilakukan** (lihat Validasi Claudian).
- [x] Build + type-check hijau (`tsc --noEmit` bersih, `next build` sukses).

## Validasi Claudian
- [x] Konfirmasi TIDAK ADA regresi ke alur existing — SEMUA validasi lama (izin guru, tap
      gerbang, jendela waktu, geofence) ada di backend `startSessionInternal()` yang SAMA
      SEKALI TIDAK disentuh task ini (task ini murni sisi frontend); `mulaiMengajarDisabledReason()`
      dan `STATUS_BADGE` tidak diubah; SATU-SATUNYA perubahan alur adalah menyisipkan
      layar scan antara GPS sukses dan submit — dikonfirmasi 49/49 test backend
      `teaching-sessions.service.spec.ts` tetap lulus tanpa modifikasi.
- [ ] Test manual dengan kamera HP ASLI (Android+iOS) — BELUM dilakukan di sesi ini
      (keterbatasan environment: tidak ada device fisik tersambung ke sesi eksekusi ini).
      `html5-qrcode` dipilih justru KARENA reputasinya sudah battle-tested lintas browser
      mobile (viewfinder+permission handling bawaan), tapi klaim ini TETAP perlu diverifikasi
      manual oleh user sebelum dianggap benar-benar selesai secara UX.
- [ ] Test end-to-end PENUH bersama T250 (2 device sungguhan) — BELUM dilakukan, sama
      alasan di atas (butuh 2 device fisik + kamera nyata, di luar kemampuan verifikasi
      sesi eksekusi ini). Diverifikasi sejauh ini murni lewat code review: alur
      join/leave/token/gagal/berhasil socket (T250) dan validasi urutan+pesan error
      (T249) sudah cocok secara kontrak (nama event, bentuk payload, endpoint yang dipanggil).

## Implementasi (2026-08-27)

**Dependency baru**: `html5-qrcode@2.3.8` (rekomendasi spec — viewfinder+permission
handling bawaan, TypeScript types sudah dibundel di paket `esm/index.d.ts`, tidak perlu
`@types/*` terpisah).

**Komponen baru** `apps/web/src/components/qr-scanner.tsx` (`QrScanner`) — ditaruh di
lokasi shared (bukan lokal ke `guru/jadwal/components/`) karena desainnya generik ("scan
QR dari kamera manapun"), walau baru 1 pemakai saat ini. Overlay full-screen (`fixed
inset-0 z-50`), `Html5Qrcode.start({facingMode:"environment"}, ...)` sekali per mount
(deps array kosong sengaja — `disabled`/`onDecode` dibaca via `useRef` supaya callback
`html5-qrcode` yang di-passing SEKALI saat `start()` selalu baca nilai TERBARU tanpa perlu
restart kamera tiap render, pola sama seperti closure-ref di komponen lain project ini).
Pesan error kamera dibedakan via regex terhadap STRING hasil reject `html5-qrcode`
(BUKAN `instanceof Error` — dikonfirmasi baca source `html5-qrcode` versi ini: promise
`start()` reject dengan STRING template `"Error getting userMedia, error = <DOMException>"`,
bukan objek Error — `NotFoundError`/`DevicesNotFoundError` di dalam string itu → pesan
"tidak ada kamera", selain itu → pesan "izinkan akses kamera").

**`sesi-card.tsx`**: `handleMulaiMengajar()` dipecah — GPS tetap logic sama, TAPI sukses
GPS sekarang `setShowScanner(true)` (bukan langsung submit). Handler baru
`handleScanDecode(qrToken)` (dipanggil `QrScanner.onDecode`) yang menaruh logic submit LAMA
+ field baru `qrToken` di body. `submitting` state dipakai sebagai prop `disabled` ke
`QrScanner` — cegah decode QR yang SAMA dari frame yang sama memicu submit ganda selagi
request pertama masih diproses (guru masih mengarahkan kamera, `html5-qrcode` terus decode
tiap frame). Error jaringan murni dibedakan dari error backend via `err instanceof
TypeError` (native `fetch()` reject dgn `TypeError` kalau request tidak pernah sampai
server — BEDA dari `apiClientFetch()` yang `throw new Error(body.message)` untuk response
non-OK yang MEMANG sampai ke server).

**Verifikasi**: `tsc --noEmit` api+web bersih, `next build` web sukses (route `/guru/jadwal`
ter-build normal, `html5-qrcode` aman di-import karena `sesi-card.tsx` sudah `"use client"`
— tidak ada crash SSR), 49/49 test `teaching-sessions.service.spec.ts` tetap lulus (backend
sama sekali tidak disentuh task ini, regresi nol dijamin secara struktural). **Keterbatasan
KRITIKAL**: TIDAK ADA test manual kamera HP fisik maupun end-to-end 2-device sungguhan di
sesi eksekusi ini (lihat Validasi Claudian) — SANGAT DIREKOMENDASIKAN user test langsung di
HP Android+iOS asli sebelum fitur ini dianggap production-ready, mengingat riwayat proyek
soal isu kompatibilitas kamera browser mobile yang tidak kelihatan dari code review/devtools.
