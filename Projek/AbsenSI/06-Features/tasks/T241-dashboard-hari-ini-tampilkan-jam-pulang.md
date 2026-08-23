# T241 — Web: Dashboard "Hari Ini" Wali Kelas — Tampilkan Jam Pulang di Kartu Hadir

## Depends on
Tidak ada dependency teknis. Independen, murni frontend `apps/web/src/app/(guru)/guru/wali-kelas/components/hari-ini-tab.tsx`. **TIDAK ADA perubahan backend diperlukan** — data sudah tersedia lengkap.

## Konteks — Kondisi Kode Saat Ini (dikonfirmasi via riset 2026-08-21, SETELAH T226 dieksekusi)

**Backend SUDAH 100% siap** — `waktuPulang` SUDAH ADA di response `GET /journal/kelas-wali-status-hari-ini` (field `PiketBoardRow.waktuPulang`, `core-types.ts`), SUDAH di-query+mapping benar di `AttendanceService.resolveStatusHarianSiswa()` (`attendance.service.ts:701-703`), dan SUDAH update REAL-TIME lewat Socket.IO (event `attendance:kampus:update` dengan `tapType: "pulang"`, sudah di-patch di state FE — `hari-ini-tab.tsx` baris 78 SUDAH punya logic `waktuPulang: event.tapType === "pulang" ? event.timestamp : row.waktuPulang`).

**YANG BELUM ADA: UI tidak menampilkan `waktuPulang` sama sekali.** `hari-ini-tab.tsx`, render daftar siswa dalam kartu yang di-expand (baris 160-171):
```tsx
<li key={s.studentId} ...>
  <span className="text-body text-ink">{s.nama}</span>
  <span className="text-caption text-ink-secondary">
    {s.waktuMasuk
      ? new Date(s.waktuMasuk).toLocaleTimeString("id-ID", { hour: "2-digit", minute: "2-digit" })
      : KATEGORI_LABEL[s.kategoriLive]}
  </span>
</li>
```
Hanya render `waktuMasuk` (atau label kategori kalau null) — `s.waktuPulang` TIDAK PERNAH dibaca di komponen ini meski sudah tersedia di data.

**Reset otomatis per-tanggal SUDAH BEKERJA by design** — TIDAK ADA cache/state manual yang perlu direset. `resolveStatusHarianSiswa()` hitung `today` fresh tiap request, filter `WHERE tanggal = today` — otomatis kosong lagi besok tanpa logic tambahan apa pun. **Ini BUKAN sesuatu yang perlu dibangun di task ini** — sudah otomatis benar.

## Keputusan Diminta User (2026-08-21)

Kartu **"Hadir"** (kategori `hadir`, salah satu dari 5 kartu status harian) — saat di-expand, tiap baris siswa TAMBAH info **jam pulang** (selain jam masuk yang sudah ada) — supaya wali kelas bisa pantau siapa yang sudah/belum pulang, bukan cuma siapa yang hadir.

## Spec Detail

### 1. Frontend — render `waktuPulang` di baris siswa

`apps/web/src/app/(guru)/guru/wali-kelas/components/hari-ini-tab.tsx` — UBAH render baris siswa (baris 160-171) — TAMBAH tampilan jam pulang, KHUSUS untuk kartu "Hadir" (kategori `hadir`) atau SEMUA kartu yang relevan (VERIFIKASI SAAT IMPLEMENTASI — REKOMENDASI: tampilkan jam pulang untuk SEMUA kategori yang punya `waktuMasuk` terisi, BUKAN cuma kartu "Hadir" secara ketat, karena siswa "Terlambat" juga tetap bisa pulang dan wali kelas mungkin ingin tahu — TAPI USER SPESIFIK SEBUT "menu Hadir", jadi MINIMAL pastikan kartu Hadir benar dulu, PERLUAS ke kartu lain kalau masuk akal tanpa menyalahi permintaan).

Contoh render:
```tsx
<li key={s.studentId} className="flex items-center justify-between rounded-lg bg-surface-subtle px-3 py-2">
  <span className="text-body text-ink">{s.nama}</span>
  <span className="flex flex-col items-end text-caption text-ink-secondary">
    <span>
      Masuk: {s.waktuMasuk
        ? new Date(s.waktuMasuk).toLocaleTimeString("id-ID", { hour: "2-digit", minute: "2-digit" })
        : KATEGORI_LABEL[s.kategoriLive]}
    </span>
    {s.waktuMasuk && (
      <span>
        Pulang: {s.waktuPulang
          ? new Date(s.waktuPulang).toLocaleTimeString("id-ID", { hour: "2-digit", minute: "2-digit" })
          : "Belum Pulang"}
      </span>
    )}
  </span>
</li>
```
- **"Belum Pulang" WAJIB tampil eksplisit** kalau `waktuPulang` masih `null` (siswa hadir tapi belum tap pulang) — JANGAN kosong/blank, supaya wali kelas langsung tahu status tanpa ambigu.
- **VERIFIKASI SAAT IMPLEMENTASI**: layout 2 baris (Masuk+Pulang) per siswa mungkin butuh penyesuaian spacing/ukuran kartu — KONSISTEN styling existing (`text-caption`, `text-ink-secondary`), JANGAN desain visual baru dari nol.

### 2. TIDAK PERLU perubahan lain

- Backend — TIDAK disentuh sama sekali (data sudah lengkap).
- Socket.IO real-time — TIDAK disentuh (sudah ada logic patch `waktuPulang` di state, tinggal dirender).
- Reset per-tanggal — TIDAK PERLU logic tambahan (sudah otomatis by design query date-scoped).

## Edge Cases

- **Siswa hadir tapi belum tap pulang** (kasus NORMAL untuk jam pelajaran masih berlangsung) — tampil "Belum Pulang", BUKAN kosong/error.
- **Siswa kategori lain (Terlambat/Izin/dst) yang KEBETULAN juga punya `waktuMasuk`+`waktuPulang` terisi** (misal siswa terlambat tapi sudah pulang juga) — VERIFIKASI SAAT IMPLEMENTASI keputusan poin 1 (tampilkan jam pulang di SEMUA kartu yang relevan, bukan cuma "Hadir" secara sempit).
- **Real-time update pulang saat kartu SEDANG di-expand** — tampilan jam pulang HARUS berubah otomatis dari "Belum Pulang" jadi jam sebenarnya TANPA reload (infrastruktur real-time sudah ada, PASTIKAN render baru ini ikut re-render saat state ter-patch).

## Files
- **Modifikasi:** `apps/web/src/app/(guru)/guru/wali-kelas/components/hari-ini-tab.tsx` (render `waktuPulang` di baris siswa).
- **Jangan sentuh:** backend (`attendance.service.ts`, `journal-kelas-wali.controller.ts`), Socket.IO gateway — SEMUA sudah benar, tidak perlu perubahan.

## Acceptance Criteria
- [ ] Expand kartu "Hadir" — tiap baris siswa tampilkan jam Masuk DAN jam Pulang (atau "Belum Pulang" kalau belum tap).
- [ ] Siswa yang belum tap pulang — tampil "Belum Pulang" eksplisit, tidak kosong.
- [ ] Update real-time — begitu siswa tap pulang, tampilan berubah otomatis dari "Belum Pulang" ke jam sebenarnya TANPA reload halaman.
- [ ] Data otomatis reset besok (TIDAK PERLU verifikasi tambahan, sudah bekerja by design — cukup pastikan TIDAK ADA regresi ke behavior ini).
- [ ] Build + type-check hijau.

## Validasi Claudian
- [ ] Konfirmasi TIDAK ADA perubahan backend sama sekali — task ini MURNI render field yang sudah tersedia.
- [ ] Konfirmasi real-time update jam pulang benar-benar ter-render (bukan cuma state ter-update tapi UI tidak re-render) — test manual: tap pulang siswa, cek kartu yang sedang di-expand berubah otomatis.
- [ ] Konfirmasi "Belum Pulang" tampil jelas untuk `waktuPulang === null`, bukan string kosong atau `undefined` mentah.
