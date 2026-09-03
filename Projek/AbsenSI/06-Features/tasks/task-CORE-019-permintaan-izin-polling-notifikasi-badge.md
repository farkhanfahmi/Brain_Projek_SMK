# Task-CORE-019 / WEB-023: "Permintaan Izin" — Tambah Polling Realtime + Badge Notifikasi

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah audit kejanggalan Dashboard Piket + diskusi kritis dengan user (2026-09-03). Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.
> Menyentuh backend (badge count) DAN frontend (polling) — 1 file karena saling terkait, TIDAK dipecah kecuali terbukti besar saat eksekusi.

**Task Terbuat:** 2026-09-03
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** low-medium
**Alasan pemilihan:** Pola polling SUDAH ADA di file tetangga (`IzinGuruKelasSection` di halaman Izin Keluar), tinggal dipindah/direplikasi ke halaman yang benar. Tambahan badge count di notification context butuh 1 field baru di response backend — kecil tapi menyentuh 2 layer.

## 2. Konteks & Tujuan Utama

Audit Dashboard Piket (2026-09-03) menemukan kejanggalan: halaman **"Permintaan Izin"** (`apps/web/src/app/(piket)/piket/permintaan-izin/permintaan-izin-view.tsx`) — yang fungsinya sebagai antrean approval untuk `ClassPermitRequest` status `menunggu` dari guru kelas — **TIDAK melakukan polling/refresh otomatis sama sekali**. State `requests` hanya diisi sekali dari SSR (`page.tsx`), piket harus manual refresh browser untuk melihat pengajuan baru.

Ironisnya, section `IzinGuruKelasSection` di halaman **Izin Keluar** (`izin-keluar-view.tsx` baris ~81-94) — yang BUKAN halaman review utamanya — justru SUDAH melakukan polling 30 detik (`setInterval(refresh, 30_000)`) untuk data terkait `ClassPermitRequest`.

Tambahan: `piket-notifications-context.tsx` saat ini hanya punya 4 kategori badge (terlambat/terkunci/belum-kembali/tidak-absen-pulang) — TIDAK ADA kategori "menunggu izin dari guru kelas", sehingga piket tidak dapat sinyal apa pun di sidebar/topbar bahwa ada pengajuan baru menumpuk, kecuali sedang membuka halaman Permintaan Izin secara manual.

**Depends on:** Tidak ada.

## 3. Langkah Eksekusi Detail

### Backend (`apps/api/src/class-permit-requests/`)

1. **Cek endpoint existing** — `GET /class-permit-requests?status=menunggu` (dipakai `listMenunggu()`) sudah ada, dipakai SSR halaman ini. Task ini TIDAK perlu endpoint baru untuk polling data tabel (frontend cukup panggil ulang endpoint yang sama secara berkala).

2. **Tambah count "menunggu" ke response notifikasi piket** — cari endpoint yang dipakai `piket-notifications-context.tsx` untuk snapshot 4 kategori badge existing (kemungkinan di `attendance.controller.ts`/`attendance.service.ts`, VERIFIKASI SAAT IMPLEMENTASI lokasi persis), tambahkan 1 field baru `permintaanIzinMenunggu: number` — hitung dari `classPermitRequest.count({ where: { status: 'menunggu', student: { kelas: { kampusId } } } })`, scope kampus SAMA seperti kategori lain di endpoint itu (REPLIKASI pola scoping existing, jangan query baru terpisah kalau bisa digabung ke query count yang sudah ada di endpoint yang sama demi hindari request tambahan).

### Frontend

3. **`permintaan-izin-view.tsx`** — tambah `useEffect` dengan `setInterval(refresh, 30_000)` (REPLIKASI PERSIS pola `IzinGuruKelasSection` di `izin-keluar-view.tsx` baris ~81-94, termasuk cleanup interval saat unmount) untuk refresh data tabel `requests` secara berkala. Refresh HARUS tidak mengganggu state `busyId`/dialog yang sedang terbuka (kalau piket sedang proses 1 request, refresh polling tidak boleh menutup dialog Tolak yang terbuka atau reset state di tengah aksi — VERIFIKASI SAAT IMPLEMENTASI, cek pola guard yang sama di `IzinGuruKelasSection` kalau ada).

4. **`piket-notifications-context.tsx`** — tambah kategori ke-5 "menunggu izin" dari field `permintaanIzinMenunggu` baru di response (langkah 2). Tambahkan badge count ini ke tampilan sidebar untuk item menu "Permintaan Izin" (cek `piket-sidebar.tsx` — apakah item menu individual sudah support badge count seperti pola item lain yang mungkin sudah punya badge, REPLIKASI kalau ada, ATAU tambahkan pola badge count sederhana kalau belum ada satupun — VERIFIKASI SAAT IMPLEMENTASI pola badge existing di sidebar).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** endpoint snapshot notifikasi piket (backend, lokasi VERIFIKASI SAAT IMPLEMENTASI — kemungkinan `attendance.service.ts`/`attendance.controller.ts`)
- **Modifikasi:** `apps/web/src/app/(piket)/piket/permintaan-izin/permintaan-izin-view.tsx` — tambah polling
- **Modifikasi:** `apps/web/src/app/(piket)/piket-notifications-context.tsx` — tambah kategori badge baru
- **Modifikasi:** `apps/web/src/app/(piket)/piket-sidebar.tsx` — render badge count baru (kalau pola badge belum ada, tambahkan minimal)
- **Jangan sentuh:** `IzinGuruKelasSection` di `izin-keluar-view.tsx` (hanya direplikasi polanya, file itu sendiri TIDAK diubah — section itu tetap ada karena scope-nya beda: itu untuk visibilitas piket di alur izin keluar existing, bukan duplikat dari halaman ini).

**Dilarang dilakukan:**
- Jangan hapus polling di `IzinGuruKelasSection` (izin-keluar-view.tsx) — section itu tetap relevan, task ini menambah polling di halaman yang SEHARUSNYA punya, bukan memindahkan/menghapus yang sudah ada.
- Jangan buat endpoint count terpisah kalau bisa digabung ke query snapshot notifikasi yang sudah ada — hindari request tambahan yang tidak perlu ke server.

**Skenario kegagalan yang WAJIB ditangani:**
- Kondisi: piket sedang mengisi dialog "Tolak" (form alasan) SAAT polling refresh berjalan → Perilaku benar: dialog tidak tertutup paksa/state form tidak hilang (refresh hanya update data tabel di belakang, bukan reset UI aktif).
- Kondisi: request gagal (network error) saat polling → Perilaku benar: silent retry di interval berikutnya (SAMA seperti pola existing `IzinGuruKelasSection`/`piket-notifications-context`), TIDAK menampilkan error mengganggu untuk kegagalan sesaat, TAPI JANGAN sampai badge count "nyangkut" salah selamanya kalau server down berkepanjangan — VERIFIKASI SAAT IMPLEMENTASI apakah perlu indikator halus (dicatat sebagai temuan audit terpisah, opsional untuk task ini kalau scope membengkak).

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Halaman "Permintaan Izin" auto-refresh data tabel setiap 30 detik tanpa perlu reload manual.
- [ ] Refresh tidak mengganggu dialog/aksi yang sedang berlangsung.
- [ ] Badge count "menunggu izin" muncul di sidebar untuk item menu "Permintaan Izin", scope kampus benar.
- [ ] Build + typecheck bersih, test unit backend untuk field count baru (kalau logic count cukup kompleks untuk butuh test).

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 200 baris perubahan)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency: tidak ada
