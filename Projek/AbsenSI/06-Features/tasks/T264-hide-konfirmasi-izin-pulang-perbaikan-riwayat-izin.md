# T264 — Piket: Hide "Konfirmasi Izin Pulang" (Sub-alur B) + Sort Riwayat Izin (Belum Kembali di Atas)

## Depends on
Tidak ada dependency teknis. Murni frontend (1 hide + 1 perubahan sort), tidak ada perubahan backend/schema.

## Konteks — Diskusi 2026-08-31

User melaporkan menu **"Perizinan Keluar"** (`/piket/izin-keluar`) menampilkan **885 baris** di tabel "Konfirmasi Izin Pulang (Sub-alur B)" — dikira data lama yang belum dieksekusi piket dari hari-hari sebelumnya.

**Investigasi (sudah diverifikasi langsung ke DB production 2026-08-31)** membuktikan ini BUKAN bug data lama:
- Tabel ini query `attendance_records` filter `tanggal = hari ini` + scope kampus petugas — **benar, tidak ada data lama nyangkut**.
- 885 = jumlah PERSIS siswa **Kampus 2** yang sudah tap pulang **hari itu juga** (`kampus_id=1` → 562, `kampus_id=2` → 885, dicek via `SELECT COUNT(*) FROM attendance_records WHERE waktu_pulang IS NOT NULL AND tanggal=CURDATE()` di-join kelas→kampus).
- Fitur ini (`apps/web/src/app/(piket)/piket/izin-keluar/izin-keluar-view.tsx:83-207`, section "Konfirmasi Izin Pulang (Sub-alur B)") **by design** menampilkan **SEMUA** siswa yang sudah tap pulang hari itu (kondisi filter cuma `s.waktuPulang !== null`, `izin-keluar-view.tsx:48`) — tujuannya piket **mencari manual** siapa di antara mereka yang pulangnya sebenarnya izin (bukan pulang sekolah biasa), lalu label ulang tap yang SUDAH terjadi via `POST /attendance/confirm-izin-pulang/:attendanceRecordId` (`attendance.service.ts:975`, comment: "Tidak mengubah waktu_pulang yang sudah tercatat dari tap, hanya label pulang_via").
- **Tidak ada logika perbandingan jadwal pelajaran sama sekali** di fitur ini — dugaan user "karena jadwal pelajaran belum diinput, semua dianggap pulang lebih awal" **tidak berlaku** (dikonfirmasi baca kode `confirmIzinPulang()` full, tidak ada perbandingan jam apa pun, murni re-label).

**Keputusan user**: fitur ini terlalu tidak praktis untuk piket (scroll 885 baris manual) — **HIDE dulu** sampai ada desain ulang yang lebih baik (kemungkinan nanti: filter otomatis berbasis jadwal pelajaran terakhir vs jam tap, di luar scope task ini).

**Kesalahpahaman yang perlu diluruskan** (penting dibaca sebelum eksekusi): user berpikir halaman **"Riwayat Izin"** (`/piket/riwayat-izin`) HANYA berisi riwayat inputan izin dari piket, TANPA kemampuan konfirmasi "sudah kembali" — **INI TIDAK AKURAT**. Investigasi kode (`riwayat-izin-view.tsx:203-250`) membuktikan halaman ini **SUDAH PUNYA** tombol per-baris **"Sudah Kembali"** dan **"Dianggap Pulang"** (muncul kalau `permit.jenis === "keluar" && permit.statusKembali === "belum"`), memanggil endpoint **PERSIS SAMA** dengan yang dipakai section "Belum Kembali" di Papan Piket utama (`PATCH /permits/:id/confirm-kembali` dan `/set-pulang`, keduanya sudah ada penuh di `permits.service.ts:485-626`). **TIDAK PERLU membangun fitur konfirmasi dari nol** — fitur itu sudah berfungsi. Yang benar-benar baru dari task ini HANYA soal urutan tampilan (lihat Spec di bawah).

## Spec Detail

### 1. Hide "Konfirmasi Izin Pulang (Sub-alur B)"

File: `apps/web/src/app/(piket)/piket/izin-keluar/izin-keluar-view.tsx`.

Sembunyikan section ini (baris 93-137, `<div className="rounded-xl bg-surface p-6 shadow-elevated">...Konfirmasi Izin Pulang (Sub-alur B)...</div>`) — REKOMENDASI: comment-out blok JSX-nya (bukan hapus permanen, supaya gampang di-restore/redesain nanti) dengan komentar jelas alasan+tanggal+link ke task ini, MISALNYA:
```tsx
{/* T264 (2026-08-31) — di-hide sementara: menampilkan SEMUA siswa tap pulang hari itu
    (885 baris di Kampus 2, tidak praktis untuk piket scroll manual). Perlu desain ulang
    (kemungkinan filter otomatis berbasis jadwal pelajaran terakhir vs jam tap) sebelum
    diaktifkan lagi — lihat 06-Features/tasks/T264-*.md untuk detail investigasi. */}
```
- State terkait (`sudahTapPulang`, `handleTandaiIzinPulang`, komponen `SubAlurBRow`) **BOLEH tetap ada di file** (tidak dipakai lagi sementara tapi tidak mengganggu) ATAU dihapus kalau linter/tsc komplain unused-variable — **VERIFIKASI SAAT IMPLEMENTASI** mana yang lebih bersih (kemungkinan besar perlu comment-out juga bagian state-nya kalau `noUnusedLocals` aktif di tsconfig, supaya build tidak gagal).
- **JANGAN sentuh backend** (`POST /attendance/confirm-izin-pulang/:id` TETAP ADA, tidak dihapus — cuma UI-nya yang disembunyikan, endpoint tidak divalidasi perlu dihapus di task ini).
- Setelah di-hide, halaman "Perizinan Keluar" HANYA menyisakan form "Izin Keluar Sementara (Sub-alur A)" (baris 209+, TIDAK disentuh).

### 2. Riwayat Izin — Sort 2-Tingkat: Belum Kembali SELALU di Atas

File: `apps/web/src/app/(piket)/piket/riwayat-izin/riwayat-izin-view.tsx`, comparator di `useMemo` sekitar baris 56-81.

**Aturan sort baru** (menggantikan urutan default `createdAt desc` polos dari backend):
- **Primary key**: `statusKembali === "belum"` → tampil DI ATAS grup `statusKembali !== "belum"` (`"sudah"`/`"pulang"`) — SELALU, terlepas dari kolom apa yang sedang di-klik user untuk sort.
- **Secondary key**: kalau user klik salah satu `SortableHeader` (nama/kelas/tanggal/jamKeluar) TETAP berlaku sort itu, tapi HANYA di dalam masing-masing grup (belum kembali disortir sendiri, sudah kembali/pulang disortir sendiri) — jangan sampai klik kolom lain "mencampur" 2 grup itu lagi.
- Kalau user BELUM klik kolom manapun (`sort === null`, state awal) — default HARUS tetap: grup belum-kembali di atas (diurut apa? REKOMENDASI: `jamKeluar` ascending di dalam grup ini, supaya yang paling lama menunggu tampil duluan — VERIFIKASI SAAT IMPLEMENTASI preferensi urutan sekunder default, bisa juga `tanggal` ascending kalau lebih masuk akal untuk workflow piket), grup sudah-kembali di bawah diurut `createdAt desc` (perilaku lama, terbaru duluan).
- **Implementasi teknis**: ubah comparator di `useMemo` — bikin fungsi `groupRank(permit)` yang return `0` kalau `statusKembali === "belum"`, `1` kalau bukan — bandingkan `groupRank` DULU sebagai primary sort key SEBELUM `accessor(sort.field)` dipanggil sebagai secondary — REPLIKASI pola comparator JS standar "sort by A, then by B" (`(a,b) => groupRank(a)-groupRank(b) || existingCompare(a,b)`).
- **`SortableHeader` TIDAK PERLU diubah** — komponennya cuma UI+callback, semua logic ada di comparator pemanggil (dikonfirmasi baca kode `sortable-header.tsx`, 36 baris, tidak ada logic sort di situ).

### 3. TIDAK Perlu Dikerjakan (supaya tidak salah scope)

- **JANGAN** tambah fitur konfirmasi "sudah kembali"/"dianggap pulang" — **SUDAH ADA DAN BERFUNGSI** di `riwayat-izin-view.tsx:231-250`, endpoint sudah lengkap. Kalau saat implementasi ternyata tombol ini TIDAK MUNCUL atau error di UI sungguhan, itu bug TERPISAH yang perlu dilaporkan balik (bukan diasumsikan "belum ada fitur ini" lalu dibangun ulang dari nol — cek dulu `needsAction` logic dan `onDuty` state kenapa tombol tidak muncul, JANGAN duplikasi endpoint yang sudah ada).
- **JANGAN** ubah `orderBy` di backend (`permits.service.ts:85`, `findAll()`) — grouping ini murni presentasi FE, tidak perlu ubah query database (data yang di-fetch backend tetap sama, cuma urutan tampilnya diubah di client).
- **JANGAN** sentuh section "Belum Kembali"/"Perlu Ditinjau" di Papan Piket utama (`piket-board-view.tsx`) — task ini scope-nya HANYA halaman "Perizinan Keluar" (hide) dan "Riwayat Izin" (sort), bukan Papan Piket.

## Edge Cases
- **Semua permit di halaman "Riwayat Izin" sudah kembali** (tidak ada yang belum) — grouping tetap jalan normal, tinggal 1 grup (grup belum-kembali kosong), tidak perlu penanganan khusus.
- **User klik kolom sort lalu klik lagi (toggle asc→desc→off)** — pastikan grouping (belum kembali di atas) TETAP bertahan bahkan saat sort di-"off"-kan (state `sort === null`) — jangan sampai toggle off balik ke urutan lama `createdAt desc` polos tanpa grouping.
- **Filter tanggal (`dari`/`sampai`) diubah user** — fetch ulang dari backend (sudah ada, `riwayat-izin-view.tsx:112-139`), comparator grouping harus tetap berlaku ke data baru hasil fetch itu, bukan cuma data awal `initialPermits`.

## Files
- **Modifikasi:** `apps/web/src/app/(piket)/piket/izin-keluar/izin-keluar-view.tsx` (hide Sub-alur B), `apps/web/src/app/(piket)/piket/riwayat-izin/riwayat-izin-view.tsx` (comparator grouping).
- **Jangan sentuh:** backend apa pun (`permits.service.ts`, `permits.controller.ts`, `attendance.service.ts`), `piket-board-view.tsx`.

## Acceptance Criteria
- [ ] Halaman "Perizinan Keluar" tidak lagi menampilkan section "Konfirmasi Izin Pulang (Sub-alur B)" — hanya form "Izin Keluar Sementara" yang tersisa.
- [ ] Endpoint `POST /attendance/confirm-izin-pulang/:id` TETAP ada di backend (tidak dihapus), cuma UI-nya yang hilang.
- [ ] Halaman "Riwayat Izin" — permit dengan `statusKembali: "belum"` SELALU tampil di atas permit yang sudah `"sudah"`/`"pulang"`, dalam KONDISI APAPUN (default load, setelah klik sort kolom manapun, setelah toggle sort off, setelah ganti filter tanggal).
- [ ] Dalam grup yang sama, sort per-kolom (nama/kelas/tanggal/jamKeluar) tetap berfungsi normal seperti sebelumnya.
- [ ] Tombol "Sudah Kembali"/"Dianggap Pulang" di Riwayat Izin tetap berfungsi seperti sebelumnya (tidak boleh regresi) — setelah permit dikonfirmasi (statusKembali berubah jadi "sudah"/"pulang"), baris itu HARUS otomatis pindah ke grup bawah tanpa perlu reload halaman (state lokal `handleAction()` sudah update in-place, tinggal pastikan re-sort ikut ter-trigger).
- [ ] tsc + build hijau.

## Validasi Claudian
- [ ] Konfirmasi TIDAK ADA fitur konfirmasi baru yang dibangun ulang dari nol — grep dulu pastikan `confirmKembali()`/`setPulang()` di `permits.service.ts` benar-benar tidak disentuh sama sekali.
- [ ] Konfirmasi grouping bertahan di SEMUA state sort (null/asc/desc), bukan cuma default load — test manual klik tiap kolom sortable + toggle off, screenshot/verifikasi urutan belum-kembali tetap di atas.
- [ ] Konfirmasi hide Sub-alur B tidak menyebabkan error TypeScript unused-variable/import yang menggagalkan build — cek `tsc` bersih setelah comment-out.
