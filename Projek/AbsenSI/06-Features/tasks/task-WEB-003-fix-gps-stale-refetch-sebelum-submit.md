# Task-WEB-003: Fix GPS Stale — Re-fetch Lokasi Tepat Sebelum Submit Mulai Mengajar

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah diskusi kritis dengan user. Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** low
**Alasan pemilihan:** Perubahan lokal ke 1 komponen (`SesiCard`), memindahkan titik pemanggilan 1 fungsi (`getGeolocation()`) yang sudah ada — tidak ada perubahan skema/API baru, murni penyesuaian urutan eksekusi di FE.

## 2. Konteks & Tujuan Utama

Audit menu Jadwal (sesi diskusi 2026-09-02) menemukan celah di alur "Mulai Mengajar" guru (`apps/web/src/app/(guru)/guru/jadwal/components/sesi-card.tsx`):

Urutan saat ini (T251):
1. Guru klik "Mulai Mengajar" → `handleMulaiMengajar()` ambil GPS SEKALI (`getGeolocation()`), simpan ke state `position`.
2. Kamera QR dibuka (`showScanner: true`).
3. Guru scan QR (bisa ada jeda — cari Ketua Kelas, tunggu dia buka layar QR, dst).
4. `handleScanDecode()` submit ke backend pakai `position` yang **sudah diambil di langkah 1** — BUKAN lokasi terkini saat submit.

**Celah:** kalau ada jeda waktu antara langkah 1 dan langkah 4, koordinat yang dikirim ke backend (dipakai validasi geofence) sudah tidak representatif terhadap lokasi guru SAAT INI. Guru bisa klik tombol dalam radius sekolah, lalu pergi, baru scan QR — validasi backend tetap lolos pakai koordinat lama.

**Keputusan user (Opsi A):** re-fetch GPS tepat sebelum submit ke backend — paling menutup celah, trade-off UX minor (2x prompt lokasi, tapi browser biasanya sudah cache izin permission setelah yang pertama sehingga cuma re-fetch posisi, bukan re-ask permission dari user).

**Depends on:** Tidak ada.

## 3. Langkah Eksekusi Detail

1. Di `sesi-card.tsx`, pisahkan concern **"minta izin lokasi + buka kamera"** (langkah 1-2, TETAP seperti sekarang — validasi awal cepat sebelum buka kamera, supaya guru yang GPS-nya gagal total tidak perlu buka kamera dulu) dari **"ambil koordinat final yang dikirim ke backend"** (harus re-fetch, bukan reuse).

2. Ubah `handleScanDecode()` (baris ~90-115) — SEBELUM memanggil `apiClientFetch` submit start-session, tambahkan re-fetch GPS:
   ```ts
   async function handleScanDecode(qrToken: string) {
     if (submitting) return;
     setSubmitting(true);
     setScanError(null);
     try {
       // T-baru — re-fetch GPS TEPAT sebelum submit, JANGAN reuse `position` lama
       // dari saat tombol "Mulai Mengajar" diklik (bisa sudah lama/stale).
       const freshPosition = await getGeolocation();
       const result = await apiClientFetch<StartSessionResponse>(
         `/teaching-sessions/${sesi.session_id}/start`,
         {
           method: "POST",
           body: JSON.stringify({
             lat: freshPosition.coords.latitude,
             lng: freshPosition.coords.longitude,
             qrToken,
           }),
         },
       );
       router.push(`/guru/sesi/${result.session_id}`);
     } catch (err) {
       // getGeolocation() sekarang BISA gagal di titik ini juga (GPS mati/ditolak
       // setelah izin pertama) — pesan error HARUS beda dari kegagalan network/backend,
       // pakai pesan yang SAMA seperti getGeolocation() reject sebelumnya (T251 sudah
       // punya pesan baku: "Aktifkan lokasi (GPS) di HP Anda untuk memulai pembelajaran").
       if (err instanceof TypeError) {
         setScanError("Gagal terhubung ke server, periksa koneksi internet Anda");
       } else {
         setScanError(err instanceof Error ? err.message : "Gagal memulai sesi mengajar");
       }
       setSubmitting(false);
     }
   }
   ```

3. **Pertahankan `handleMulaiMengajar()` (langkah 1) apa adanya** — TETAP ambil GPS di awal untuk validasi cepat (fail-fast kalau GPS mati total, guru tidak perlu buka kamera dulu baru tahu GPS-nya gagal). State `position` dari sini SEKARANG hanya dipakai sebagai **pre-check** (gerbang awal sebelum buka kamera), BUKAN lagi dikirim ke backend — nilai final yang dikirim SELALU dari re-fetch di `handleScanDecode()`.

4. **Hapus dependency `position` yang tidak lagi dipakai** di `handleScanDecode` — cek signature function, parameter/guard `if (!position || submitting) return;` (baris 91) perlu disesuaikan karena `position` lama tidak lagi jadi syarat submit (diganti re-fetch).

5. **Pertimbangkan UX**: re-fetch GPS di `handleScanDecode` terjadi SETELAH scan QR berhasil di-decode (async, ada jeda beberapa ratus ms - beberapa detik tergantung device GPS) — pastikan ada indikator loading yang jelas selama proses ini (`submitting: true` sudah ada, cek apakah UI overlay scanner menampilkan state ini dengan jelas, mis. teks "Memvalidasi lokasi..." bukan cuma spinner generik).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/web/src/app/(guru)/guru/jadwal/components/sesi-card.tsx` — `handleMulaiMengajar()`, `handleScanDecode()`
- **Jangan sentuh:** backend `startSession()`/`startSessionInternal()` (`teaching-sessions.service.ts`) — validasi geofence backend SUDAH benar, ini murni memastikan data yang DIKIRIM ke backend akurat, bukan mengubah cara backend memvalidasi.
- **Jangan sentuh:** `QrScanner` component — behavior kamera/decode QR tidak berubah, hanya titik pemanggilan submit setelahnya.

**Dilarang dilakukan:**
- Jangan hapus validasi GPS awal di `handleMulaiMengajar()` — itu tetap berguna sebagai fail-fast UX (guru tahu GPS-nya bermasalah sebelum buka kamera, bukan setelah repot-repot scan QR).
- Jangan biarkan submit berjalan dengan koordinat lama sebagai fallback kalau re-fetch gagal — kalau re-fetch GPS gagal saat submit, TOLAK submit dan tampilkan error jelas, JANGAN fallback diam-diam ke `position` lama (itu justru mengembalikan celah yang sedang diperbaiki).

**Skenario kegagalan yang WAJIB ditangani:**
- Kondisi: re-fetch GPS di `handleScanDecode` gagal (GPS mati/izin dicabut SETELAH kamera terbuka) → Perilaku yang benar: `scanError` tampil di overlay scanner (BUKAN keluar dari overlay), guru bisa retry tanpa perlu ulang dari awal (tutup kamera, klik Mulai Mengajar lagi) — cek apakah `handleCameraError` (untuk kegagalan KAMERA) perlu dibedakan alurnya dari kegagalan GPS-saat-submit ini (kegagalan GPS TIDAK PERLU menutup kamera, cukup tampil scanError dan guru retry scan).
- Kondisi: guru scan QR berkali-kali cepat (double-tap/QR ke-scan 2x sebelum request pertama selesai) → guard `submitting` (sudah ada) harus tetap efektif mencegah 2 request paralel, verifikasi guard ini masih benar setelah re-fetch GPS ditambahkan ke alur (re-fetch sendiri butuh waktu, pastikan `submitting: true` di-set SEBELUM re-fetch GPS dimulai, bukan setelah).
- Kondisi: browser TIDAK cache izin lokasi (kasus jarang, browser tertentu/mode privat) → re-fetch kedua akan re-prompt izin ke user — pastikan pesan error kalau user MENOLAK re-prompt ini sama jelasnya dengan pesan T251 existing, bukan pesan generik baru.

**Edge case:**
- Jeda antara `handleMulaiMengajar` (GPS awal) dan `handleScanDecode` (GPS submit) SANGAT SINGKAT (guru langsung scan QR dalam hitungan detik) → re-fetch tetap dilakukan (konsisten, tidak ada exception "kalau cepat skip re-fetch") — overhead-nya minimal dan konsistensi logic lebih penting daripada micro-optimization ini.

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Koordinat yang dikirim ke backend saat submit SELALU hasil re-fetch tepat sebelum request, BUKAN GPS yang diambil saat klik tombol "Mulai Mengajar"
- [ ] Validasi GPS awal (`handleMulaiMengajar`) tetap ada sebagai fail-fast sebelum kamera dibuka
- [ ] Kegagalan re-fetch GPS saat submit menampilkan error jelas DI DALAM overlay scanner, guru bisa retry tanpa mengulang dari awal
- [ ] Tidak ada fallback diam-diam ke koordinat lama kalau re-fetch gagal
- [ ] Guard `submitting` tetap mencegah double-submit meski re-fetch GPS menambah latensi ke alur

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini (dicek ulang oleh Hermes sebelum handoff)
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 300 baris perubahan — task ini jauh di bawah itu, 1 komponen)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada (pesan error T251 tetap konsisten dipakai ulang)
- [ ] Dependency (jika ada) sudah selesai sebelum task ini di-assign — tidak ada dependency
