# T085 — API+UI: Deteksi IP Kiosk Otomatis Sebelum Registrasi

## Depends on
Tidak ada — extend fitur existing `KioskGuard`/`kiosks` (ADR-021), tidak mengubah mekanisme whitelist IP itu sendiri.

## Context
- **App:** `apps/api` + `apps/web`
- **File:** `apps/api/src/auth/guards/kiosk.guard.ts`, `apps/web/src/app/(admin)/kiosk/kiosk-view.tsx`
- **Ref:** Insiden 2026-07-26 — admin daftarkan kiosk "test_laptop" dengan `allowedIp: 10.200.219.194` (dibaca dari `ipconfig` di laptop itu sendiri), tapi tap SELALU ditolak "IP tidak terdaftar". Log API menunjukkan request sebenarnya datang dari `192.168.123.2` — jaringan WiFi sekolah melakukan NAT/routing lintas-subnet yang mengubah source IP sebelum sampai ke server, sehingga IP yang diketik admin (dari sisi client) TIDAK PERNAH sama dengan IP yang benar-benar dilihat server. Solusi diusulkan user: server sendiri yang mendeteksi & tampilkan IP yang ia lihat, admin tinggal konfirmasi — menghilangkan sumber salah-ketik/asumsi topologi jaringan yang keliru.

## Spec Detail

### Masalah Inti
`KioskGuard` (baris 44-50, `kiosk.guard.ts`) membandingkan `clientIp` (dilihat SERVER) dengan `kiosk.allowedIp` (diinput ADMIN, biasanya dari `ipconfig`/`ifconfig` di device kiosk). Kedua nilai ini **HANYA identik kalau tidak ada NAT/proxy di antara device dan server** — begitu ada router yang melakukan NAT (jaringan sekolah kompleks, WiFi terpisah subnet, dst), keduanya berbeda dan whitelist SELALU gagal meski device-nya benar.

### Solusi: Endpoint "Deteksi IP" Sebelum Registrasi
1. **Endpoint baru `GET /kiosks/detect-ip`** (public/no-guard KHUSUS endpoint ini, TIDAK butuh token apapun — tapi TIDAK boleh mengembalikan data sensitif apapun selain echo IP) — cukup return `{ detectedIp: string }` hasil dari fungsi extract-IP yang SAMA PERSIS dengan yang dipakai `KioskGuard.extractClientIp()` (reuse logic yang sama, JANGAN duplikasi/rewrite terpisah supaya tidak ada divergensi antara deteksi dan validasi asli nanti)
2. **UI di halaman `/kiosk`** — form "Tambah Kiosk" (`kiosk-view.tsx`) ditambah section "Deteksi IP Otomatis":
   - Instruksi: "Buka link ini DARI DEVICE KIOSK yang akan didaftarkan: `<url_server>/api/kiosks/detect-ip`" — admin buka link tsb dari BROWSER DI DEVICE KIOSK itu sendiri (bukan dari laptop admin), supaya request benar-benar datang dari device yang mau didaftarkan
   - ATAU (lebih baik UX): admin buka halaman kiosk yang MEMANG akan didaftarkan (`apps/kiosk` yang sedang jalan di device itu), device itu sendiri fetch `detect-ip` dan TAMPILKAN hasilnya di layar kiosk (besar, mudah dibaca) — admin baca angka itu dari layar kiosk, lalu ketik/paste manual ke form Tambah Kiosk di dashboard admin (2 device berbeda, jadi tetap perlu ada langkah manual pemindahan angka, TAPI angkanya sekarang dijamin akurat karena berasal dari deteksi server, bukan tebakan `ipconfig` client)
   - Field `allowedIp` di form Tambah Kiosk TETAP text input manual (JANGAN auto-fill otomatis lintas device — device kiosk dan device admin BEDA, tidak ada cara aman untuk auto-share IP terdeteksi antar device tanpa mekanisme tambahan yang lebih kompleks dari yang diminta)

### Halaman Bantuan di `apps/kiosk` (Opsional tapi Direkomendasikan)
- Tambah halaman kecil `apps/kiosk/src/app/deteksi-ip/page.tsx` — device kiosk buka halaman ini, fetch `/api/kiosks/detect-ip` (lewat proxy kiosk yang sudah ada, konsisten pola `getClientIp` di `client-ip.ts`), tampilkan IP besar-besar di layar supaya gampang dibaca+dicatat manual oleh petugas yang berdiri di depan kiosk fisik

## JANGAN
- ❌ JANGAN buat endpoint deteksi ini butuh autentikasi kiosk token — device BELUM terdaftar saat proses ini jalan, tidak punya token untuk dikirim (chicken-and-egg problem kalau dipaksa pakai guard)
- ❌ JANGAN kembalikan info sensitif lain di response `detect-ip` selain IP — bukan endpoint untuk expose data kiosk manapun, murni echo IP
- ❌ JANGAN duplikasi logic ekstraksi IP (`extractClientIp` di `kiosk.guard.ts`, `getClientIp` di `apps/kiosk`) — kalau perlu dipakai di 2 tempat (guard asli + endpoint deteksi baru), extract jadi 1 helper function yang di-reuse, supaya IP yang "dideteksi" saat registrasi PASTI identik dengan IP yang "divalidasi" saat tap sungguhan nanti
- ❌ JANGAN hilangkan mekanisme whitelist IP existing — task ini MEMPERBAIKI AKURASI proses registrasi (supaya admin input IP yang benar), bukan mengganti/melemahkan validasi keamanannya

## Files
- **Modifikasi:** `apps/api/src/core/kiosk/kiosks.controller.ts` (atau lokasi setara) — endpoint baru `GET /kiosks/detect-ip`, extract logic IP ke helper shared dengan `KioskGuard`
- **Modifikasi:** `apps/web/src/app/(admin)/kiosk/kiosk-view.tsx` — tambah instruksi/section "Deteksi IP Otomatis" di form Tambah Kiosk
- **Buat (opsional, direkomendasikan):** `apps/kiosk/src/app/deteksi-ip/page.tsx` — halaman tampilan IP besar untuk device kiosk fisik

## Acceptance Criteria
- [ ] `GET /kiosks/detect-ip` (tanpa auth) return `{ detectedIp: "<ip yang sama seperti yang akan divalidasi KioskGuard>" }`
- [ ] Diuji dari device yang berbeda subnet/NAT dari server (skenario mirip insiden test_laptop) — IP yang di-echo endpoint ini SAMA PERSIS dengan IP yang muncul di log "IP mismatch" kalau device itu mencoba tap tanpa terdaftar dulu
- [ ] Form Tambah Kiosk di dashboard admin punya instruksi jelas cara pakai fitur deteksi ini
- [ ] Tidak ada regresi ke `KioskGuard` — validasi tap kiosk yang sudah terdaftar dengan benar tetap berfungsi seperti sebelumnya
