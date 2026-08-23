# T108 — Realtime+Web: Ikon Notifikasi Piket (Kejadian Belum Tertangani, Lintas Halaman)

## Depends on
- **Disarankan setelah T107** (Surat Izin Masuk Kelas) selesai, karena salah satu kategori notif ("siswa terlambat") pakai kriteria tertangani = surat T107 sudah dicetak. Kalau T107 belum ada saat T108 dikerjakan, kategori "terlambat" di task ini boleh disesuaikan sementara (kriteria tertangani = piket sudah membuka detail baris itu, atau ditunda dulu sub-bagiannya) — **klarifikasi ke user** urutan pengerjaan sebelum mulai kalau T107 ternyata belum jalan.
- Tidak bentrok file dengan T106 (tabel/filter) — T108 murni provider baru + top-bar + beberapa `.emit()` backend tambahan.

## Objective
Piket melihat ikon lonceng notifikasi di top bar (semua halaman piket, bukan cuma Dashboard) yang menampilkan REALTIME semua "perilaku siswa tidak normal/tidak sesuai aturan" yang BELUM ditangani hari itu — supaya piket tidak harus berada di halaman Dashboard untuk tahu ada yang perlu dikerjakan.

## Context
- **App:** `apps/api` (tambah beberapa `socket.emit()` baru di service yang belum broadcast) + `apps/web` (provider baru + komponen ikon+dropdown di top bar)
- **Riset 2026-08-05 (Explore agent, baca kode langsung):**
  - `useAttendanceSocket` (`apps/web/src/lib/use-attendance-socket.ts`, 95 baris) — **TIDAK shared/context**, tiap pemanggilan bikin koneksi `io()` sendiri-sendiri (baris 66) dan `join:kampus` sendiri (baris 74). Dipakai saat ini HANYA di `piket-board-view.tsx:146` dan `tv/page.tsx:10`. **Task ini butuh provider BARU** yang memegang 1 koneksi socket di level layout, bukan reuse hook ini apa adanya di banyak tempat (itu akan bikin N koneksi terpisah kalau dipasang di tiap halaman).
  - `apps/web/src/app/(piket)/piket-content.tsx` — client wrapper yang SUDAH membungkus SEMUA halaman piket (dipanggil dari `layout.tsx`), sudah render `TopBarWithTitle`. **Ini lokasi yang tepat untuk mount provider notifikasi baru**, pola sama seperti `PiketDutyProvider` (`piket-duty-context.tsx`, 22 baris, `createContext` + `.Provider` + hook `usePiketOnDuty()`) — tapi provider baru ini CLIENT-side dan aktif (pegang state dari socket), bukan sekadar meneruskan boolean dari server seperti punya duty.
  - `apps/web/src/components/shell/top-bar.tsx` — tombol profil ada di baris ±63-78 (avatar bulat + nama/role), dalam grup flex kanan (`justify-between`, baris 49). Ikon lonceng ditaruh di grup yang sama, tepat SEBELUM tombol avatar.
  - **Backend BELUM broadcast apapun untuk lock/unlock atau konfirmasi izin** — `AttendanceGateway` (`apps/api/src/realtime/attendance.gateway.ts`) cuma emit `attendance:kampus:update`, `attendance:kiosk:update`, `attendance:today`, semuanya dari alur tap-in (`attendance.controller.ts`, `attendance-recorded.consumer.ts`). `students` (lock/unlock) dan `permits` (confirm-kembali) SAMA SEKALI tidak punya dependency ke gateway. **Task ini WAJIB menambah emit baru** di 2 service itu, bukan cuma kerja frontend.

## Keputusan Final (dikonfirmasi user 2026-08-05)

1. **Cakupan kejadian**: "semua perilaku siswa tidak normal/tidak sesuai aturan" — dipetakan ke 4 kategori konkret yang sudah ada representasinya di dashboard piket:
   - Siswa terlambat baru tap masuk
   - Siswa keluar belum kembali (lewat `jamKembaliDiharapkan`)
   - Siswa baru terkunci (2x-terlambat otomatis ATAU manual)
   - Siswa tidak absen pulang (belum tap keluar menjelang/lewat jam pulang)
2. **Interaksi**: klik ikon → dropdown kecil berisi list kejadian terbaru, tiap item bisa diklik untuk lompat ke halaman/section terkait.
3. **Kriteria "tertangani" (otomatis hilang dari list, BUKAN dropdown mark-as-read manual)**:
   - Terlambat → hilang setelah surat izin masuk kelas dicetak (T107, `POST /late-entry-slips` sukses untuk siswa itu hari ini).
   - Terkunci → hilang setelah di-`unlock()`.
   - Belum Kembali / Tidak Absen Pulang → hilang setelah diverifikasi (`confirmKembali`/`setPulang`, pola sudah ada di `piket-board-view.tsx`).
4. **Background merah vs putih PER ITEM**: merah = kejadian itu SENDIRI belum ditangani (bukan soal piket sudah klik/lihat atau belum). Putih = sudah tertangani. **Tidak ada mekanisme "mark as read" terpisah dari status tertangani** — 1 sumber kebenaran (status data), bukan 2 (status data + status baca).
5. **Tidak ada fitur "clear manual"** — list otomatis reset kosong saat ganti hari (scope kejadian = HARI INI saja, sama seperti Piket Board yang sudah scoped per hari kampus). Tidak perlu tabel baru untuk track "sudah di-clear piket X".
6. **Audit trail**: setiap perubahan/eksekusi (verifikasi kembali, unlock, cetak surat, dst) SUDAH tercatat `activity_log` (insert-only, existing) dengan pelaku+waktu — **tidak perlu bikin mekanisme baru**, notifikasi ini murni MEMBACA status data existing secara agregat, bukan sumber data baru yang butuh log terpisah.
7. **Transport**: Socket.IO, koneksi di level `piket-content.tsx` (bukan polling), broadcast baru ditambahkan di backend untuk event yang belum ada (lock/unlock, confirm-kembali).

## Spec Detail

### Backend — Emit Baru
Tambah broadcast (reuse `AttendanceGateway` yang sudah ada — inject ke service yang butuh, atau extract method broadcast generik kalau `AttendanceGateway` dirasa nama yang terlalu sempit sekarang untuk dipakai modul lain; putuskan saat implementasi mana yang lebih bersih, JANGAN bikin gateway kedua yang terpisah tanpa alasan kuat):
- `apps/api/src/core/students/students.service.ts` — pada `lock()`/`unlock()` (baik manual maupun auto dari `applyLateStrikeLock()`), emit event baru (misal `attendance:kampus:locked` / `attendance:kampus:unlocked`) ke room kampus siswa yang bersangkutan, payload minimal: `studentId`, `nama`, `kelas`, `lockedAt`/`unlockedAt`.
- `apps/api/src/permits/permits.service.ts` — pada `confirmKembali()`/`setPulang()` (dan varian "Tidak Kembali" dari T098 kalau sudah ada saat ini dikerjakan), emit event baru (misal `attendance:kampus:izin-resolved`) dengan payload `permitId`, `studentId`, resolusi apa.
- Event tap-in yang SUDAH ada (`attendance:kampus:update`) sudah cukup untuk kategori "terlambat baru tap masuk" — filter `status === "terlambat"` di FRONTEND, tidak perlu event baru khusus untuk ini.
- "Tidak absen pulang" — TIDAK ada trigger event real-time alami (ini kejadian "non-event", ketiadaan tap keluar menjelang jam tertentu) — **kategori ini dihitung dari POLLING ringan periodik di frontend** (misal tiap 1-2 menit panggil endpoint agregat, BUKAN dari socket event), atau dari perhitungan client-side atas data board yang sudah di-fetch + jam sekarang. Putuskan pendekatan saat implementasi, dokumentasikan kenapa kategori ini beda mekanisme dari 3 lainnya (supaya developer berikutnya tidak bingung kenapa 1 dari 4 kategori tidak murni socket-driven).

### Backend — Endpoint Agregat (opsional tapi direkomendasikan)
- Endpoint baru `GET /piket/notifications` (atau serupa) yang mengembalikan snapshot AWAL semua kejadian belum-tertangani hari ini untuk kampus piket yang login — dipanggil SEKALI saat provider mount (supaya badge langsung terisi benar begitu piket login/refresh, bukan nunggu event baru masuk lewat socket). Socket event sesudahnya yang meng-update state secara incremental.
- Query: gabungan dari `AttendanceRecord` (status terlambat, tidak absen pulang), `Permit` (belum kembali lewat estimasi), `Student` (isLocked) — kemungkinan besar bisa reuse logic yang SAMA dengan `piketBoard()` (`attendance.service.ts`) yang sudah menghitung semua kategori ini untuk dashboard, JANGAN duplikasi query, panggil service yang sama atau extract method bersama.

### Frontend
- **Provider baru** `apps/web/src/app/(piket)/piket-notifications-context.tsx` (pola sama seperti `piket-duty-context.tsx`, tapi client-active): buka 1 koneksi socket (mirip `useAttendanceSocket` tapi TIDAK pakai hook itu langsung di banyak tempat — extract logic koneksi jadi dipakai SEKALI di sini), fetch snapshot awal dari endpoint agregat, subscribe ke event-event di atas, expose `usePiketNotifications()` → `{ items: NotificationItem[], count: number }`.
- Mount provider ini di `apps/web/src/app/(piket)/piket-content.tsx`, di dalam/sejajar `PiketDutyProvider` yang sudah ada.
- **Komponen ikon** — tambah di `apps/web/src/components/shell/top-bar.tsx`, sebelum tombol avatar (sekitar baris 62): ikon lonceng (`lucide-react`, ikuti aturan DESIGN.md — TIDAK emoji) + badge angka merah kalau `count > 0` (pola badge sama seperti yang lain di proyek, cek existing badge style). **Komponen top-bar ini dipakai lintas route group** (admin/guru/piket) — pastikan ikon HANYA muncul untuk role piket (`guru_piket`), cek prop yang sudah ada (`roleLabel`/atau tambah prop baru eksplisit) supaya tidak muncul di dashboard admin/guru yang tidak punya context ini.
- **Dropdown**: klik ikon → daftar item, tiap item background merah/putih sesuai status tertangani (lihat Keputusan #4), klik item → navigasi ke halaman terkait dengan anchor/scroll ke section relevan (kalau memungkinkan) atau minimal ke halaman yang benar (Dashboard untuk terlambat/terkunci/belum-kembali, atau langsung buka modal yang relevan kalau desain memungkinkan — putuskan level kedalaman ini saat implementasi, minimal "ke halaman yang benar" sudah cukup untuk versi pertama).

## Edge Cases
- Piket pindah kampus/logout-login ulang → provider reconnect dengan `kampusId` yang benar, tidak nyangkut data kampus lama.
- Ganti hari (lewat tengah malam) saat piket masih login → list notifikasi harus reset kosong tanpa perlu piket refresh manual (cek ulang tanggal saat menghitung kategori, atau reconnect terjadwal tengah malam — putuskan mekanisme paling sederhana saat implementasi).
- Banyak tab/device piket yang sama login bersamaan → tidak masalah, tiap tab independen (sama seperti pola socket existing).

## Files
- **Buat:** `apps/web/src/app/(piket)/piket-notifications-context.tsx`, endpoint backend baru untuk snapshot agregat (lokasi: kemungkinan `apps/api/src/attendance/` atau modul baru kecil, putuskan saat implementasi).
- **Modifikasi:** `apps/api/src/core/students/students.service.ts` (emit lock/unlock), `apps/api/src/permits/permits.service.ts` (emit confirm-kembali/set-pulang), `apps/api/src/realtime/attendance.gateway.ts` (method broadcast baru kalau perlu), `apps/web/src/app/(piket)/piket-content.tsx` (mount provider), `apps/web/src/components/shell/top-bar.tsx` (ikon + dropdown).
- **Jangan sentuh:** `useAttendanceSocket` yang sudah ada — biarkan tetap dipakai apa adanya oleh `piket-board-view.tsx` dan `tv/page.tsx`, TIDAK perlu diganti/dipaksa reuse provider baru ini (beda kebutuhan: board butuh data board penuh, notifikasi cuma butuh ringkasan kejadian).

## Acceptance Criteria
- [ ] Ikon lonceng muncul konsisten di semua halaman piket (bukan cuma Dashboard), dengan badge angka kalau ada kejadian belum tertangani.
- [ ] Klik ikon menampilkan dropdown berisi kejadian real (bukan dummy), item merah untuk belum tertangani, putih untuk sudah.
- [ ] Siswa terlambat baru tap masuk muncul di notif dalam hitungan detik (realtime via socket, bukan perlu refresh).
- [ ] Siswa yang dikunci (manual atau auto) muncul di notif segera setelah lock terjadi.
- [ ] Item terlambat hilang dari notif setelah surat T107 dicetak untuk siswa itu; item terkunci hilang setelah unlock; item belum-kembali hilang setelah diverifikasi.
- [ ] List notifikasi kosong kembali otomatis di hari berikutnya, tanpa aksi manual piket.
- [ ] Ikon TIDAK muncul untuk role selain `guru_piket` (admin/guru tidak melihat ini).
- [ ] Build + type-check `apps/api` dan `apps/web` hijau.

## Validasi Claudian
- [ ] Konfirmasi ke user pendekatan "Tidak Absen Pulang" (polling vs client-computed) sebelum eksekusi — ini satu-satunya kategori yang tidak murni event-driven, potensi kebingungan desain kalau tidak diklarifikasi dulu.
- [ ] Pastikan tidak membuat 2 mekanisme "status tertangani" yang berbeda (satu di backend, satu dihitung ulang di frontend) yang bisa saling tidak sinkron — idealnya backend jadi satu-satunya sumber kebenaran status, frontend murni menampilkan.
- [ ] Cek dulu apakah `top-bar.tsx` benar dipakai lintas route group sebelum menambah role-gating — kalau ternyata `(piket)` sudah punya top bar sendiri yang terpisah dari admin/guru, sesuaikan lokasi perubahan.
- [ ] Kalau T107 belum selesai saat T108 mulai dikerjakan, konfirmasi ke user dulu bagaimana kategori "terlambat" ditangani sementara (lihat catatan di Depends on).
