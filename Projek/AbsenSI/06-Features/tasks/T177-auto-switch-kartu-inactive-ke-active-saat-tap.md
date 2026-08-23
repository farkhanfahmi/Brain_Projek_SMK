# T177 — API+Web: Auto-Switch Kartu Nonaktif→Aktif Saat Tap (Ditandai untuk Ditinjau Admin)

## Depends on
Tidak ada dependency teknis wajib. Independen dari T167 (byte-reversal, TIDAK diubah — fallback itu tetap jalan LEBIH DULU sebelum logic task ini, urutan penting, lihat Spec Detail). Independen dari T165 (`activate()` publik TIDAK dipakai langsung di sini karena validasinya justru menghalangi alur — lihat Edge Cases).

## Objective
Kalau kartu di-tap berstatus `inactive`, **TAPI owner kartu itu (siswa/guru yang sama) sedang punya kartu LAIN yang `active`** — sistem OTOMATIS: aktifkan kartu yang baru saja di-tap ini, nonaktifkan kartu yang sebelumnya aktif, dan **terima tap ini sebagai berhasil** (bukan `rejected_inactive`). Kejadian ini WAJIB tercatat jelas untuk ditinjau admin (mitigasi risiko kartu hilang/tertukar dipakai orang lain).

## Context — Insiden yang Memicu Task Ini (2026-08-14)

User (admin) melaporkan kondisi darurat: banyak siswa tidak bisa tap karena UID yang BENAR (kartu fisik yang sungguh mereka pegang) berstatus `inactive` di sistem, sementara UID yang `active` adalah kartu YANG SALAH (kemungkinan besar admin salah pilih kartu saat proses "Ganti Kartu", T165/T166).

**Investigasi data production (bukan dugaan) mengonfirmasi pola ini nyata dan berulang**:
- Kasus contoh nyata: **MARCELL DIO ABANDA** (`student_id=2003`) — 2026-08-13 06:04 admin melakukan Ganti Kartu (`card.replace`): kartu lama `0114250778` dinonaktifkan, kartu baru `0441765638` diaktifkan. **Tapi kartu baru itu SALAH** — Marcell terus tap pakai kartu LAMA `0114250778` berulang kali selama ~19 JAM (`rejected_inactive`, BENAR sesuai kondisi saat itu, BUKAN bug), sampai admin sadar dan mengoreksi manual (revoke kartu salah, aktifkan kartu lama yang benar) — begitu dikoreksi, tap langsung `accepted` dalam hitungan detik.
- **Pola SAMA ditemukan pada minimal 4 siswa lain** (Leonanda Putra, Muhammad Fiki Adi Wijaya, M. Zalfa Kayla Al Basitha, Ahmad Luki Gunawan) — semua: kartu nonaktif TETAP di-tap terus-menerus oleh siswa (7-23 kali) sampai berjam-jam/berhari-hari setelah dicabut, sementara kartu "aktif" versi baru TIDAK PERNAH dipakai tap sama sekali. User eksplisit menyebut "masih ada banyak siswa mengalami hal yang sama" — tidak bisa dikumpulkan satu-satu untuk verifikasi manual, butuh solusi sistemik.

**Sinyal kunci yang jadi dasar task ini**: kartu yang BENAR-BENAR dipegang siswa akan TERUS di-tap meski gagal — kartu yang salah/hilang/tertukar TIDAK PERNAH muncul di tap manapun setelah didaftarkan. Ini pola perilaku nyata, bukan asumsi.

**Keputusan eksplisit user**: BUKAN "aktifkan semua kartu sekaligus" (ditolak — berisiko besar, akan menciptakan banyak siswa dengan 2 kartu aktif bersamaan, membuka celah kartu tercecer/dicuri dipakai orang lain tap atas nama siswa itu). Sebagai gantinya: **auto-switch OTOMATIS saat tap terjadi** (server yang memutuskan berdasarkan bukti tap nyata, bukan migrasi massal buta), DENGAN mitigasi wajib: kejadian auto-switch HARUS tercatat jelas untuk ditinjau admin (opsi "auto-switch tanpa pengecualian tanpa notifikasi" DITOLAK user karena risiko kartu curian tidak terdeteksi cepat).

## Spec Detail

### 1. Backend — logic baru di `tap()`, TEPAT SEBELUM keputusan `rejected_inactive`

- `apps/api/src/attendance/attendance.service.ts` — titik lookup card SUDAH ADA di baris ~131-134 (lookup UID mentah) dan ~142-150 (fallback byte-reversal T167, urutan ini TIDAK BERUBAH — auto-switch task ini adalah langkah BERIKUTNYA, HANYA jalan setelah kedua lookup itu menghasilkan `card` yang DITEMUKAN tapi `status !== active`, BUKAN pengganti dari keduanya).
- Titik `rejected_inactive` SAAT INI di baris ~157-160 (`if (card.status !== CardStatus.active) { ... return rejected_inactive }`) — SISIPKAN logic baru TEPAT SEBELUM return itu:
  1. Cari kartu LAIN milik owner YANG SAMA (`studentId`/`teacherId` sama dengan `card` yang ditemukan) yang `status: active` — query `prisma.card.findFirst({ where: { OR: [{studentId: card.studentId}, {teacherId: card.teacherId}], status: active, NOT: {id: card.id} } })` (SESUAIKAN kondisi supaya HANYA match owner type yang relevan — kalau `card.studentId` ada, cari berdasar `studentId` SAJA, jangan campur `teacherId`; guru dengan multi-kartu aktif TIDAK PERNAH masuk kondisi ini karena guru TIDAK dibatasi 1 aktif — T119 — jadi logic INI HANYA RELEVAN UNTUK SISWA, guru selalu boleh multi-aktif sehingga tidak ada "kartu lain yang harus dinonaktifkan").
  2. **KALAU TIDAK ADA kartu aktif lain milik owner ini** → PERILAKU LAMA dipertahankan APA ADANYA, `return rejected_inactive` seperti sekarang (task ini TIDAK mengubah kasus ini).
  3. **KALAU ADA kartu aktif lain (SISWA SAJA, lihat poin 1)** → JALANKAN AUTO-SWITCH dalam `$transaction`:
     - Nonaktifkan kartu yang SEBELUMNYA aktif (`status: inactive, revokedAt: now()`) — REUSE pola field yang SAMA PERSIS dengan `CardsService.revoke()` (`cards.service.ts:242-249`), TAPI JANGAN panggil method publik itu langsung dari `attendance.service.ts` (hindari circular dependency yang tidak perlu untuk operasi sesederhana ini) — cukup `prisma.card.update()` langsung dalam transaksi yang SAMA dengan update kartu satunya, KONSISTEN pola field (status/revokedAt) yang dipakai `CardsService`.
     - Aktifkan kartu yang BARU SAJA di-tap ini (`status: active, revokedAt: null, issuedAt: new Date()`) — REUSE pola field PERSIS `reactivateCard()` (`cards.service.ts:234-240`), TAPI JANGAN panggil `CardsService.activate()` publik (validasinya `ensureOwnerExistsAndHasNoActiveCard()` akan MENOLAK karena kartu lama MASIH terlihat aktif kalau urutan query salah — task ini PERLU kontrol urutan transaksi sendiri: nonaktifkan-lama DAN aktifkan-baru dalam 1 `$transaction` atomik, TIDAK melalui 2 pemanggilan `CardsService` terpisah).
     - **WAJIB catat ke `activity_log`** — panggil `this.activityLog.record()` (SUDAH tersedia di constructor `attendance.service.ts:99`, KONSISTEN pola `applyLateStrikeLock()` yang sudah pakai `getSystemActorId()` untuk aksi otomatis sistem) — `action: "card.auto_switch"`, `targetType: "card"`, `targetId` = id kartu yang baru diaktifkan, `snapshotBefore`/`snapshotAfter` berisi KEDUA kartu (id, uid, status sebelum/sesudah) supaya admin yang meninjau tahu PERSIS kartu mana yang dinonaktifkan dan mana yang diaktifkan.
     - Lanjutkan alur `tap()` SEPERTI BIASA dengan kartu yang BARU diaktifkan ini sebagai `card` aktif (proses presensi normal — `determineStatus()`, dst — TIDAK ADA perubahan di logic setelah titik ini, kartu yang tadinya "gagal" sekarang diperlakukan seperti kartu aktif biasa yang baru saja tap).

### 2. Backend — notifikasi/tinjauan admin (mitigasi wajib, DIKONFIRMASI user)

- **Broadcast Socket.IO real-time** — REUSE pola `broadcastStudentLock()`/`broadcastPermitResolved()` (`attendance.gateway.ts` baris ~224-231, method serupa di `AttendanceGateway`) — tambah method baru `broadcastCardAutoSwitch(event)`, emit ke room `attendance:kampus:{kampusId}` (kampus siswa terkait) ATAU room admin baru yang lebih sesuai (PUTUSKAN saat implementasi — REKOMENDASI: room `admin:card-review` baru, KONSISTEN pola `admin:kiosk-health` yang sudah ada sebagai room khusus admin, supaya notifikasi ini tidak tercampur ke dashboard piket yang scope-nya beda).
- **Daftar "Perlu Ditinjau" untuk admin** — endpoint baru `GET /cards/auto-switch-log` (atau nama serupa) — query `activity_log` filter `action: "card.auto_switch"`, urutkan terbaru dulu, role `super_admin, card_admin` — TIDAK PERLU field baru di model `Card` (activity_log SUDAH cukup sebagai sumber kebenaran "riwayat kejadian", KONSISTEN prinsip append-only proyek — field boolean `needsReview` di `Card` TIDAK diperlukan karena ini soal RIWAYAT KEJADIAN bukan STATE PERMANEN yang perlu di-reset admin).
- **UI baru** — section/halaman kecil (mis. di halaman `/kartu` yang sudah ada, T166, TAMBAH tab/badge "Auto-Switch Perlu Ditinjau" — BUKAN halaman terpisah baru, REUSE shell yang sudah ada) menampilkan daftar kejadian auto-switch: nama siswa, kartu lama (dinonaktifkan) vs kartu baru (diaktifkan), waktu kejadian — supaya admin BISA cepat cek "apakah ini benar (siswa memang salah kartu) atau curiga (kartu dicuri dipakai orang lain)" dan BATALKAN manual via tombol Ganti Kartu/Aktifkan Kembali existing kalau ternyata salah.

### 3. Toleransi debounce — TIDAK BERUBAH

- Auto-switch terjadi SEBELUM logic debounce (`rejected_duplicate`) — urutan pemrosesan lain di `tap()` (paling atas: lookup card → byte-reversal fallback → **[AUTO-SWITCH BARU]** → status check lain → debounce → determineStatus) TIDAK diubah kecuali penyisipan poin 1 di atas.

### 4. Backend+Frontend — daftar "UID Benar-Benar Tidak Terdaftar" (untuk panggil manual siswa)

**Konteks tambahan (2026-08-14, sesi diskusi lanjutan)**: auto-switch (poin 1-3) HANYA menolong kasus kartu `inactive` yang owner-nya PUNYA kartu lain `active`. Untuk siswa yang UID kartunya **BENAR-BENAR TIDAK ADA DI `Card` SAMA SEKALI** (bukan nonaktif — betul-betul tidak terdaftar, `rejected_unknown` bahkan setelah fallback byte-reversal T167) — auto-switch TIDAK BISA membantu (tidak ada "kartu lain" untuk dibandingkan). User butuh **daftar deteksi** kejadian ini supaya bisa memanggil siswa yang bersangkutan secara manual keesokan harinya.

**Batasan struktural PENTING yang WAJIB dipahami sebelum implementasi**: `tap_events` untuk `result: rejected_unknown` punya `card_id: NULL` — sistem TIDAK TAHU siapa siswanya dari baris tap itu sendiri (tidak ada link ke `Student`). Fitur ini HANYA BISA menampilkan **UID mentah + waktu + kiosk** yang gagal — BUKAN nama siswa (mustahil secara data, JANGAN coba "tebak" siswa dari data lain, itu di luar scope dan berisiko salah tuduh). Admin yang tahu identitas siswa lewat proses manual (tanya langsung/lihat siapa yang mengantre di kiosk itu).

**Filter WAJIB untuk menghindari noise** — riset data production (2026-08-14) menemukan BANYAK baris `rejected_unknown` adalah noise pembacaan reader (goyang/sentuhan sebentar), BUKAN kartu siswa sungguhan — contoh nyata ditemukan: UID pendek 1-3 digit (`48`, `684`, `999`, `8`, `33`), UID sangat panjang 17-20+ digit (`01142031144210196298`, `33553580621960020684`) — pola ini TIDAK PERNAH cocok format kartu RFID sekolah yang SELALU 10 digit desimal (konsisten SEMUA UID valid di `Card` yang sudah diverifikasi sepanjang investigasi T167/T177, contoh `0114250778`, `0793501459`). **WAJIB filter**: HANYA tampilkan UID yang PERSIS 10 karakter digit (`^\d{10}$`) — buang UID di luar panjang itu dari daftar (noise, bukan kartu sungguhan, JANGAN ditampilkan supaya admin tidak memanggil siswa yang kartunya sebenarnya baik-baik saja).

**Filter tambahan REKOMENDASI** — HANYA tampilkan UID yang tap **BERULANG** (misal ≥2 kali dalam rentang beberapa hari terakhir, BUKAN sekali doang) — UID 10-digit yang cuma muncul 1x juga BISA noise (reader kadang salah baca 1 digit dari kartu yang sebenarnya terdaftar) — pola nyata di data production: UID yang benar-benar "siswa mencoba berkali-kali karena memang tidak bisa" muncul 4-11x dari kiosk yang sama (`0114387645` 10x, `0114194681` 11x, `0114319418` 4x, `0114306758` 9x) — BEDA JELAS dari UID sekali-doang yang kemungkinan salah baca sesaat. PUTUSKAN ambang jumlah percobaan minimal saat implementasi (REKOMENDASI ≥2), TAPI JANGAN terlalu ketat sampai menyembunyikan kasus nyata — tampilkan JUMLAH PERCOBAAN di UI supaya admin sendiri yang menilai mana yang prioritas.

**Endpoint baru** — `GET /cards/unregistered-uid-log?days=` (nama final diputuskan saat implementasi) — role `super_admin, card_admin`. Query `tap_events` WHERE `result = 'rejected_unknown'` AND `uid` REGEXP `'^[0-9]{10}$'`, GROUP BY `uid`, HAVING `COUNT(*) >= 2` (ambang dari poin di atas), dalam rentang `days` terakhir (default beberapa hari, PUTUSKAN default wajar misal 3-7 hari saat implementasi). Per baris: `uid`, `jumlahPercobaan`, `pertamaKali`, `terakhirKali`, `kioskIds` (daftar kiosk mana saja UID ini dicoba — membantu admin tahu di gerbang mana siswa itu biasa tap, PETUNJUK tambahan untuk identifikasi manual).

**UI** — TAMBAH SEBAGAI TAB/SECTION TERPISAH di halaman yang SAMA dengan "Auto-Switch Perlu Ditinjau" (poin 2 di atas, `apps/web/src/app/(admin)/kartu/kartu-view.tsx`) — BEDA MAKNA jelas dari tab itu (ini "kartu TIDAK ADA sama sekali di sistem", bukan "kartu ada tapi salah status") — beri judul jelas seperti "UID Belum Terdaftar (Perlu Dipanggil Manual)" supaya admin tidak bingung dengan tab auto-switch. Tampilkan tabel: UID, Jumlah Percobaan, Kiosk, Waktu Pertama/Terakhir — SESUAI aturan tabel permanen proyek (search+sort+kolom No, meski tabel ini kemungkinan pendek).

## Edge Cases
- **Kartu yang di-tap TERNYATA milik owner yang SUDAH TIDAK PUNYA kartu aktif lain (kasus normal `rejected_inactive` biasa, misal kartu memang sengaja dicabut permanen tanpa pengganti)** — TIDAK terpengaruh, tetap `rejected_inactive` seperti sekarang (poin 1.2 di atas).
- **GURU** (bukan siswa) — TIDAK PERNAH masuk logic auto-switch ini SAMA SEKALI (T119: guru boleh multi-kartu aktif tanpa batas, jadi tidak ada "kartu lain yang harus dinonaktifkan" — kalaupun guru punya kartu inactive, dia mungkin memang sengaja tidak dipakai, BUKAN kasus "salah pilih kartu" seperti siswa). Query poin 1 WAJIB eksplisit exclude guru (`studentId` bukan null), JANGAN generalisasi ke `teacherId` sekalipun terlihat "konsisten" — ini KEPUTUSAN SENGAJA, bukan kelalaian.
- **Race condition**: 2 tap nyaris bersamaan (kartu lama dan kartu baru di-tap dalam detik yang sama oleh 2 orang berbeda, skenario kartu benar-benar tertukar 2 siswa berbeda pegang kartu masing-masing salah) — transaksi `$transaction` MySQL dengan row-level lock pada update `Card` seharusnya menangani ini (satu tap akan menunggu transaksi lain selesai) — TIDAK PERLU logic tambahan di luar transaksi standar, TAPI WAJIB diuji skenario ini saat implementasi (test concurrent tap).
- **Kartu yang di-auto-switch AKTIF, TERNYATA memang kartu curian** (skenario risiko yang mendasari keputusan mitigasi) — task ini TIDAK MENCEGAH kejadian itu terjadi SEKALI (tap pertama kartu curian akan tetap diterima+switch), tapi MEMASTIKAN admin tahu SECEPATNYA (broadcast realtime+daftar tinjauan) untuk membatalkan manual (revoke kartu curian, aktifkan kembali kartu asli) sebelum kerugian berlanjut — INI BATASAN YANG DISADARI DAN DITERIMA user, bukan celah yang perlu ditutup di task ini.
- **Sesi/absensi yang SUDAH tercatat dengan `card_id` LAMA (sebelum auto-switch)** — `tap_events.card_id`/`AttendanceRecord` histori TIDAK diubah retroaktif (konsisten prinsip insert-only, T167 juga begitu) — hanya tap BARU sejak auto-switch terjadi yang pakai kartu yang sudah di-switch.

## Files
- **Modifikasi:** `apps/api/src/attendance/attendance.service.ts` (`tap()`, sisipkan logic auto-switch), `apps/api/src/realtime/attendance.gateway.ts` (`broadcastCardAutoSwitch()` baru), `apps/api/src/cards/cards.controller.ts` (endpoint baru `GET /cards/auto-switch-log` DAN `GET /cards/unregistered-uid-log`), `apps/api/src/cards/cards.service.ts` (query activity_log/tap_events untuk kedua endpoint baru), `apps/web/src/app/(admin)/kartu/kartu-view.tsx` (2 tab baru: "Auto-Switch Perlu Ditinjau" dan "UID Belum Terdaftar").
- **Jangan sentuh:** `CardsService.activate()`/`revoke()`/`reactivateCard()` (TIDAK dipanggil langsung dari `attendance.service.ts`, direplikasi field-nya via `prisma.card.update()` langsung dalam transaksi lokal — hindari circular dependency & validasi `ensureOwnerExistsAndHasNoActiveCard()` yang tidak cocok untuk alur ini), urutan lookup card existing (byte-reversal T167 TIDAK diubah, auto-switch murni langkah TAMBAHAN setelahnya), logic keterlambatan guru (T175, tidak terkait).

## Acceptance Criteria
- [x] Siswa tap kartu `inactive`, owner-nya PUNYA kartu lain `active` → tap DITERIMA (bukan `rejected_inactive`), kartu yang di-tap jadi `active`, kartu lama jadi `inactive`.
- [x] Siswa tap kartu `inactive`, owner-nya TIDAK PUNYA kartu aktif lain → PERILAKU LAMA dipertahankan, tetap `rejected_inactive`.
- [x] GURU tap kartu `inactive` (meski punya kartu lain aktif) → TIDAK PERNAH auto-switch, perilaku LAMA dipertahankan (`rejected_inactive` seperti sekarang, T119 tetap berlaku).
- [x] Setiap auto-switch tercatat di `activity_log` dengan snapshot before/after KEDUA kartu, actor = system.
- [x] Broadcast Socket.IO terkirim ke admin saat auto-switch terjadi — verified live (koneksi socket nyata, room `admin:card-review`, payload lengkap diterima).
- [x] Endpoint `GET /cards/auto-switch-log` mengembalikan riwayat kejadian, role dibatasi `super_admin`/`card_admin`.
- [x] UI halaman Kartu menampilkan daftar "Auto-Switch Perlu Ditinjau" dengan info lengkap (nama, kartu lama vs baru, waktu).
- [x] Test concurrent tap (2 kartu owner sama, hampir bersamaan) tidak menghasilkan state korup (2 kartu aktif sekaligus ATAU 0 kartu aktif) — LOLOS, 4 request paralel hasil akhir tepat 1 kartu aktif.
- [x] Endpoint `GET /cards/unregistered-uid-log` HANYA menampilkan UID 10-digit yang tap ≥2 kali — noise (UID pendek/panjang, sekali-doang) TERBUKTI TERFILTER (verified dengan pola data nyata yang sebelumnya ditemukan mengandung noise).
- [x] UI "UID Belum Terdaftar" tampil terpisah jelas dari "Auto-Switch Perlu Ditinjau" — TIDAK ADA nama siswa ditampilkan (mustahil secara data, hanya UID+waktu+kiosk).
- [x] Build + type-check hijau, jest baru untuk skenario auto-switch (18 test attendance.service.spec.ts) DAN untuk filter UID belum terdaftar+auto-switch-log (8 test cards.service.spec.ts) ditambahkan.

## Validasi Claudian
- [x] **WAJIB verifikasi eksplisit**: logic auto-switch HANYA untuk siswa (`studentId`) — DIVERIFIKASI live (guru dengan kartu lain aktif tetap `rejected_inactive`, kartu tidak berubah sama sekali).
- [x] **WAJIB verifikasi**: fallback byte-reversal (T167) tetap jalan LEBIH DULU dan TIDAK DIUBAH — auto-switch disisipkan SETELAH kedua lookup card (baris ~131-150), 0 diff di logic byte-reversal.
- [x] **WAJIB verifikasi**: operasi nonaktifkan-lama+aktifkan-baru terjadi dalam **1 transaksi atomik** (`prisma.$transaction([update, update])`) — `CardsService.activate()`/`revoke()`/`reactivateCard()` TIDAK disentuh sama sekali (0 diff), field direplikasi langsung via `prisma.card.update()`.
- [x] **WAJIB verifikasi**: mitigasi notifikasi admin — broadcast Socket.IO diverifikasi via koneksi client nyata (bukan cuma unit test mock), endpoint riwayat diverifikasi via curl live, UI tinjauan diverifikasi via Playwright browser nyata.
- [x] Test concurrent tap DIJALANKAN nyata (4 request paralel via `curl ... &` + `wait`) — hasil dilaporkan: semua 4 `accepted`, 0 error, state akhir konsisten (1 kartu aktif), HANYA 1 entri `activity_log` baru (request pertama switch, 3 lainnya melihat kartu sudah aktif dan diproses sebagai tap normal).

## Status Eksekusi (2026-08-14)

**Selesai.**

**Backend**:
- `apps/api/src/attendance/attendance.service.ts` — `tap()` cabang `card.status !== active` disisipkan logic auto-switch (siswa+punya kartu aktif lain → `autoSwitchCard()`, selain itu perilaku lama `rejected_inactive`). Method baru `autoSwitchCard()`: `prisma.$transaction([update, update])` atomik, `activityLog.record()` (action `card.auto_switch`, actor sistem, snapshot before/after kedua kartu), broadcast via `AttendanceGateway.broadcastCardAutoSwitch()`.
- `apps/api/src/realtime/attendance.gateway.ts` — `CardAutoSwitchEvent` type baru, `join:card-review` subscribe message (pola SAMA `join:kiosk-health`), `broadcastCardAutoSwitch()` emit ke room `admin:card-review`.
- `apps/api/src/cards/cards.service.ts` — `autoSwitchLog()` (query activity_log action=card.auto_switch, join nama/nisn siswa dari kartu baru), `unregisteredUidLog()` (query tap_events rejected_unknown, filter regex `^\d{10}$` + ambang ≥2 tap, group by uid, kumpulkan nama kiosk).
- `apps/api/src/cards/cards.controller.ts` — `GET /cards/auto-switch-log`, `GET /cards/unregistered-uid-log` (role `super_admin, card_admin` dari `@Roles` level controller, sudah ada).
- `apps/api/src/cards/dto/unregistered-uid-log-query.dto.ts` — `days` opsional (default 7, maks 30).
- Test baru: `attendance.service.spec.ts` (+2 describe block, 18 test total di file itu: switch atomik, log activity, broadcast, guru tidak pernah trigger broadcast); `cards.service.spec.ts` (baru, 8 test: autoSwitchLog mapping, unregisteredUidLog filter noise dengan pola data production nyata).

**Frontend**:
- `apps/web/src/lib/use-card-review-socket.ts` (baru) — hook Socket.IO pola sama `useKioskHealthSocket`, join `admin:card-review`, terima event `card:auto-switch`.
- `apps/web/src/lib/core-types.ts` — `AutoSwitchLogEntry`, `UnregisteredUidEntry` baru.
- `apps/web/src/app/(admin)/kartu/kartu-view.tsx` — 2 tab baru "Auto-Switch Perlu Ditinjau" (badge merah kalau ada event realtime baru, refetch otomatis saat event masuk) dan "UID Belum Terdaftar" (search+sort semua kolom+kolom No, sesuai aturan tabel permanen proyek).

**Verifikasi live** (dev DB port 3307, dev API+web port 3101/3100, production tidak disentuh):
1. Siswa tap kartu inactive, owner punya kartu lain aktif → `result: accepted`, kartu lama `active→inactive`, kartu baru `inactive→active`, `activity_log` tercatat lengkap dengan snapshot.
2. Siswa tap kartu inactive, TIDAK ADA kartu aktif lain (kedua kartu owner inactive) → `result: rejected_inactive`, TIDAK ADA perubahan status kartu — perilaku lama dipertahankan persis.
3. GURU tap kartu inactive dengan kartu lain aktif → `result: rejected_inactive`, kartu TIDAK berubah — dikonfirmasi guru tidak pernah auto-switch meski secara data mirip kondisi siswa yang di-switch.
4. Concurrent test: 4 request paralel (`curl ... &` x4 + `wait`) ke UID yang sama → semua `accepted`, 0 error di log server, state akhir `jumlah_aktif=1` (tidak korup), hanya 1 activity_log baru (idempotent secara efektif karena re-query per-request).
5. Broadcast Socket.IO — koneksi client Node nyata (socket.io-client), join `admin:card-review`, trigger tap dari terminal lain → event `card:auto-switch` diterima dengan payload lengkap (studentId, nama, nisn, kelas, kampusId, oldCardUid, newCardUid, switchedAt).
6. `GET /cards/auto-switch-log` (curl+JWT) → array riwayat dengan nama+nisn siswa ter-join dari kartu baru.
7. `GET /cards/unregistered-uid-log` (curl+JWT, data uji UID 10-digit 3x tap dari 2 kiosk + noise UID pendek/panjang/sekali-doang) → HANYA UID valid yang muncul, noise semua terfilter, `kioskNama` berisi 2 nama kiosk berbeda.
8. UI browser (Playwright) — tab "Auto-Switch Perlu Ditinjau" menampilkan baris lengkap (nama+nisn, kartu lama, kartu baru, waktu format id-ID) setelah trigger tap nyata dari luar; tab "UID Belum Terdaftar" menampilkan search box+kolom sortable, hanya UID valid muncul, noise `48` terverifikasi tidak tampil.
9. Semua data uji (cards, attendance_records, tap_events, activity_log, semester test, kiosk allowed_ip override, password hash admin) dibersihkan/dikembalikan, dikonfirmasi via re-query — dev DB kembali ke state semula.
10. `tsc --noEmit` bersih `apps/api` dan `apps/web`, `jest` — 24 suite / 302 test lulus 100%.
