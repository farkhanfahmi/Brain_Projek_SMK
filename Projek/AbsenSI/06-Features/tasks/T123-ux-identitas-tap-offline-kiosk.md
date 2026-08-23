# T123 — Kiosk: Tampilkan Identitas (Nama/Foto) Saat Tap Offline, Bukan Cuma Angka

## Depends on
Disarankan dikerjakan SETELAH atau BERSAMAAN dengan T122 (fix sync silent-fail) — task ini menampilkan status tap yang tersimpan lokal, termasuk kasus `status: "failed"` yang baru ada setelah T122. Kalau T123 dikerjakan sendirian tanpa T122, cukup tampilkan identitas untuk tap yang statusnya `pending` (belum sync), tanpa bisa membedakan mana yang nanti gagal permanen.

## Objective
Saat kiosk offline dan tap tersimpan ke buffer lokal, operator/orang di gerbang bisa tahu SIAPA yang baru saja tap (nama, mungkin foto) — bukan cuma angka "3 tap tersimpan lokal" tanpa identitas apa pun.

## Context
- **App:** `apps/kiosk` — perluasan UI, bukan perubahan alur data.
- **Masukan user (2026-08-06)**: saat kondisi offline, tampilan kiosk TIDAK menunjukkan view tap yang biasa (nama, foto, status) seperti saat online.
- **KONFIRMASI KODE (Explore agent, 2026-08-06, bukan lagi dugaan)** — temuan lebih spesifik dan lebih bermasalah dari perkiraan awal:
  - `apps/kiosk/src/app/page.tsx` — saat `submitTap` gagal (offline), kode membuat `TapResponse` SINTETIS hardcode: `{ result: TapResult.ACCEPTED, message: "Tap tersimpan, akan disinkronkan otomatis" }` — TIDAK ADA `name`/`foto`/`identifier`/`kelas`/`time` sama sekali (field-field itu memang cuma ada di response ASLI dari server, yang tidak pernah didapat kiosk saat offline karena lookup identitas terjadi di server).
  - `feedback-screen.tsx` (siswa baris ±338, guru ±127) MENERIMA `result: ACCEPTED` ini dan me-render **kartu sukses yang SAMA PERSIS** seperti tap online berhasil — BUKAN layar berbeda/degraded. Karena field-field identitas itu `undefined`, yang tampil adalah: **nama kosong/blank**, **foto jadi siluet generik** (`KioskAvatar` fallback default), **NISN/kelas jadi tanda `-`**, **jam tap kosong**.
  - **Bug tambahan yang ditemukan**: pesan `"Tap tersimpan, akan disinkronkan otomatis"` yang sudah ada di kode itu **TIDAK PERNAH DITAMPILKAN** di kartu — template kartu `ACCEPTED` untuk siswa/guru menampilkan `response.time` di slot pesan, BUKAN `response.message`, jadi pesan penjelasan itu hilang begitu saja meski sudah ditulis di kode.
  - `OfflineIndicator` (banner merah "Offline — N tap tersimpan") HANYA elemen tambahan kecil di pojok bawah layar — TIDAK menggantikan kartu kosong yang membingungkan itu, keduanya tampil BERSAMAAN.
  - **Kesimpulan dampak nyata**: orang yang tap saat offline melihat kartu yang TERLIHAT seperti "berhasil normal" (layout sukses, avatar, dst) tapi datanya kosong semua tanpa penjelasan — ini BUKAN sekadar "kurang informasi", ini **visual yang menyesatkan** (terlihat OK padahal user tidak tahu apa yang sebenarnya terjadi).
- **Alur existing yang jadi acuan**: pola visual kartu sukses online (`feedback-screen.tsx`) dipakai sebagai basis, TAPI untuk kasus offline harus ada varian/state BERBEDA yang jelas menandakan "belum terverifikasi", bukan reuse kartu sukses apa adanya dengan data kosong.
- **Keterbatasan mendasar**: saat offline, kiosk TIDAK bisa memanggil API untuk resolve nama/foto dari `uid` kartu (itu justru alasan kenapa offline-first ada — kiosk tidak bisa mengandalkan server). Jadi identitas yang ditampilkan HARUS berasal dari data yang SUDAH ADA di kiosk secara lokal SEBELUM offline terjadi (lihat Spec Detail).

## Spec Detail

### Sumber Data Identitas Saat Offline
- Cek apakah kiosk punya CACHE lokal data kartu→nama/foto (misal di-download saat online untuk mempercepat lookup, atau di-refresh berkala) — KALAU ADA, dipakai untuk resolve `uid` yang di-tap saat offline jadi nama tampilan.
- KALAU TIDAK ADA cache lokal sama sekali — ini KEPUTUSAN BESAR yang perlu dikonfirmasi ke user: apakah perlu membangun cache lokal baru (kartu→nama, di-refresh tiap kali online, disimpan di IndexedDB yang sama dengan offline buffer), ATAU cukup tampilkan UID mentah saat offline (kurang ideal tapi tidak butuh cache baru) sampai tersinkron dan barulah nama sebenarnya diketahui. **JANGAN putuskan sendiri tanpa konfirmasi** — ini scope yang signifikan (cache lokal baru = state management tambahan, perlu strategi refresh/invalidasi) vs scope kecil (tampilkan UID + badge "menunggu sync" saja).

### Tampilan Saat Tap Offline (Berhasil Disimpan ke Buffer)
- **JANGAN reuse kartu sukses `ACCEPTED` apa adanya untuk kasus offline** — ini akar masalah yang dikonfirmasi (kartu kosong terlihat seperti sukses). Buat VARIAN TAMPILAN BARU khusus offline (atau tambahkan mode berbeda di `feedback-screen.tsx`), yang SECARA VISUAL JELAS beda dari kartu sukses biasa — bukan cuma data kosong dengan layout sama.
- Tap yang di-buffer offline TETAP menampilkan layar feedback ke orang yang tap — bukan diam saja atau cuma update angka counter di banner. Layar itu menunjukkan (tergantung keputusan sumber data di atas): nama (kalau ada cache) ATAU UID + pesan jelas "Tersimpan, akan disinkronkan" (kalau tidak ada cache), DENGAN indikator visual jelas bahwa ini BELUM terverifikasi 100% oleh server (beda dari tampilan sukses online yang sudah pasti) — warna/ikon berbeda dari kartu sukses hijau biasa (kuning/abu "pending", BUKAN merah "gagal" — ini beda kondisi dari benar-benar ditolak).
- **WAJIB perbaiki bug pesan yang hilang**: `response.message` ("Tap tersimpan, akan disinkronkan otomatis") HARUS benar-benar tampil di kartu — saat ini tertulis di kode tapi template render `response.time` di slot itu, bukan `.message`, jadi tidak pernah terlihat. Pastikan varian tampilan offline baru punya slot pesan yang benar.
- Setelah T122 selesai (opsional untuk T123 kalau dikerjakan terpisah): tap yang akhirnya GAGAL PERMANEN saat sync (status `failed`) perlu punya cara ditampilkan/diketahui — pertimbangkan indikator di layar kiosk (misal badge merah di banner "1 tap GAGAL, perlu tap ulang manual") supaya operator tahu ada yang perlu ditindaklanjuti, BUKAN cuma tersimpan diam di IndexedDB tanpa sinyal apa pun.

## Edge Cases
- Kiosk offline dalam waktu SANGAT LAMA (misal berjam-jam, seperti insiden 646 tap sebelumnya) — daftar identitas yang menumpuk bisa panjang, pertimbangkan UX yang wajar (list scroll, atau cukup tampilkan tap TERBARU + counter untuk sisanya, bukan wajib tampilkan semua riwayat penuh di 1 layar kiosk kecil).
- Tap offline untuk kartu yang TIDAK ADA di cache lokal (kalau opsi cache dipilih) — kartu baru yang belum sempat ter-cache sebelum offline terjadi, fallback ke tampilan UID mentah untuk kasus ini secara spesifik.

## Files
- **Modifikasi:** `apps/kiosk/src/app/page.tsx` (atau komponen tampilan tap terkait), `apps/kiosk/src/lib/offline-buffer.ts` (kalau perlu tambah field cache identitas), `apps/kiosk/src/lib/tap-client.ts` (kalau perlu logic tambahan resolve nama sebelum simpan ke buffer).
- **Kemungkinan buat baru**: mekanisme cache kartu→identitas (endpoint baru untuk fetch daftar kartu ringan, atau reuse data yang sudah di-fetch kiosk saat online untuk keperluan lain — cek dulu apakah kiosk SUDAH fetch daftar siswa/guru untuk keperluan lain yang bisa di-reuse, sebelum bikin endpoint baru).

## Acceptance Criteria
- [x] Saat kiosk offline, tap yang tersimpan ke buffer menampilkan identitas (nama kalau ada cache, atau UID + pesan jelas kalau tidak) — bukan cuma angka counter generik. **Diputuskan: UID mentah** (lihat Validasi Claudian) — `response.identifier = uid`, ditampilkan besar+jelas di kartu baru.
- [x] Tampilan tap offline secara visual jelas BEDA dari tap sukses online (indikator "belum terverifikasi"/"menunggu sync") — TIDAK LAGI reuse kartu `ACCEPTED` hijau dengan data kosong. 2 varian baru (guru+siswa) di `feedback-screen.tsx`, warna amber/kuning-coklat (`#8A6800`/`#5C4600`, konsisten dengan palet `justLocked` yang sudah ada sebagai preseden "perlu perhatian" di kiosk ini), ikon jam (`Clock`, bukan centang/silang), judul "Menunggu Sinkronisasi"/"Belum Terverifikasi".
- [x] Pesan "Tap tersimpan..." BENAR-BENAR tampil di layar (bug lama: pesan ada di kode tapi template render `.time`, bukan `.message`, di slot itu). **Fix**: kedua varian offline baru render `{response.message}` di slot khusus, diverifikasi lewat pembacaan kode langsung (bukan `.time`).
- [x] Tap yang gagal permanen saat sync (T122) punya sinyal visual jelas ke operator kiosk. `OfflineIndicator` diperluas dengan badge kedua (amber, `AlertTriangle`) terpisah dari badge pending merah — "N tap GAGAL, perlu tap ulang manual", data dari `getFailedTaps()` (baru, `offline-buffer.ts`) di-poll bareng siklus sync 5 detik yang sudah ada.
- [x] Build + type-check `apps/kiosk` hijau. `tsc --noEmit` bersih, `next build` sukses (`✓ Compiled successfully`, route `/` 26.3 kB).

## Validasi Claudian
- [x] **Dikonfirmasi user 2026-08-06**: pilih opsi SCOPE KECIL — tampilkan UID mentah + pesan jelas, TIDAK membangun cache lokal kartu→nama baru. Nama sebenarnya baru diketahui setelah sync berhasil (bisa dicek di histori/riwayat setelah online).
- [x] Dicek: kiosk TIDAK punya mekanisme fetch/cache data siswa-guru yang bisa direuse — `useKioskRecent` (satu-satunya kandidat) butuh koneksi Socket.IO aktif, tidak berfungsi offline sama sekali. Ini memperkuat alasan kenapa opsi scope-kecil dipilih (tidak ada fondasi existing untuk di-reuse, harus dibangun dari nol kalau pilih opsi cache).

## Status Eksekusi — SELESAI (2026-08-06)
`apps/kiosk/src/app/page.tsx` — `runTap()` tidak lagi mengarang `TapResponse` palsu ber-`result: ACCEPTED` untuk tap offline; sekarang `identifier: uid` + flag boolean `feedbackOffline` baru, di-thread ke `FeedbackScreen` via prop `offline`. `apps/kiosk/src/components/feedback-screen.tsx` — 2 branch baru (guru + siswa) DI ATAS pengecekan `ACCEPTED`/`justLocked` biasa, supaya offline selalu diprioritaskan sebelum klasifikasi lain. `apps/kiosk/src/components/offline-indicator.tsx` — `failedCount` prop opsional baru, badge terpisah. Tidak ada perubahan skema `offline-buffer.ts` di luar yang sudah dibuat T122 (`getFailedTaps()` sudah ada dari situ, cukup dipakai di sini).
