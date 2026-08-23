# T236 — Web: Fix Menu Ekstrakurikuler Guru — Tambah Sub-Menu Setting & Daftar Peserta yang Hilang

## Depends on
Tidak ada dependency teknis. Independen, murni frontend `apps/web/src/app/(guru)/components/nav-groups.ts`.

## Konteks — Root Cause Terkonfirmasi (2026-08-21)

User laporkan menu Ekstrakurikuler (untuk guru yang jadi pembina, `isPembinaEkstra`) sekarang cuma tersisa "Absensi" — sub-menu "Setting" dan "Ploting Siswa" hilang.

**BUKAN regresi T234** (dikonfirmasi via `git diff` commit T234 — `EKSTRAKURIKULER_GROUP` sama sekali tidak tersentuh). **Ini gap lama sejak T230** (commit `a9cd216`, saat `nav-groups.ts` pertama dibuat) — route `setting/` dan `peserta/` **SUDAH ADA dan BERFUNGSI PENUH** di backend+frontend sejak commit `705dda8` (jauh sebelum T230), TAPI tidak pernah didaftarkan ke `nav-groups.ts` saat file itu dibuat:

```ts
// apps/web/src/app/(guru)/components/nav-groups.ts — KONDISI SEKARANG
export const EKSTRAKURIKULER_GROUP: NavGroup = {
  groupLabel: "Ekstrakurikuler",
  items: [{ label: "Ekstrakurikuler", href: "/guru/ekstrakurikuler", icon: Sparkles }],
};
```

**Route yang SUDAH ADA tapi tidak ter-link** (`apps/web/src/app/(guru)/guru/ekstrakurikuler/`):
```
page.tsx                    ← Absensi/Presensi (SATU-SATUNYA yang ter-link sekarang)
peserta/page.tsx            ← "Daftar Peserta" (= Ploting Siswa yang dimaksud user)
setting/page.tsx            ← "Setting"
sesi/[sesiId]/page.tsx      ← detail sesi (dibuka dari dalam Absensi)
```

**Referensi pola yang SUDAH BENAR** untuk role `pembina_ekstra` (akun eksternal, route group PARALEL `(pembina-ekstra)`, REUSE komponen SAMA persis dari `apps/web/src/features/ekstrakurikuler/`) — `apps/web/src/app/(pembina-ekstra)/pembina-ekstra-sidebar.tsx`:
```ts
const NAV_ITEMS = [
  { label: "Presensi", href: "/ekstrakurikuler", icon: ClipboardList },
  { label: "Daftar Peserta", href: "/ekstrakurikuler/peserta", icon: Users },
  { label: "Setting", href: "/ekstrakurikuler/setting", icon: Settings },
];
```
**REPLIKASI PERSIS pola label+urutan ini** ke `EKSTRAKURIKULER_GROUP` guru — komponen backend SUDAH SAMA (reuse `PesertaView`/`SettingView`/`PresensiView`), HANYA prefix path beda (`/guru/ekstrakurikuler/*` vs `/ekstrakurikuler/*`).

**Verifikasi dampak production**: 3 akun guru sekolah (role `guru`) tercatat sebagai `pembinaId` ekstrakurikuler — SEMUA mengalami bug ini sekarang.

## Spec Detail

### 1. Tambah 2 item ke `EKSTRAKURIKULER_GROUP`

`apps/web/src/app/(guru)/components/nav-groups.ts`:
```ts
export const EKSTRAKURIKULER_GROUP: NavGroup = {
  groupLabel: "Ekstrakurikuler",
  items: [
    { label: "Absensi", href: "/guru/ekstrakurikuler", icon: ClipboardList },      // rename label dari "Ekstrakurikuler" → "Absensi", KONSISTEN label pola (pembina-ekstra)
    { label: "Daftar Peserta", href: "/guru/ekstrakurikuler/peserta", icon: Users },
    { label: "Setting", href: "/guru/ekstrakurikuler/setting", icon: Settings },
  ],
};
```
- **VERIFIKASI SAAT IMPLEMENTASI**: apakah T234 sudah menambah `children?` ke `NavGroupItem` (dari task T234, kalau sudah dieksekusi) — kalau grup ini SEBAIKNYA jadi expandable (icon grup + children, konsisten pola baru "Kehadiran"/"Laporan Guru" di Wali Kelas T234) ATAU cukup 3 item FLAT sejajar (konsisten pola LAMA sebelum T234 untuk grup ini) — REKOMENDASI: kalau `children` sudah ada infrastrukturnya dari T234, PAKAI itu untuk konsistensi (grup "Ekstrakurikuler" jadi expandable dengan 3 children), TAPI kalau belum, 3 item FLAT juga valid solusi minimal (tidak wajib nunggu T234 selesai untuk fix bug ini).
- Import ikon baru (`ClipboardList`, `Users`, `Settings` dari `lucide-react`) — CEK apakah sudah ada di import statement file ini atau perlu ditambah.

### 2. Verifikasi kedua render — desktop + mobile (SATU sumber, otomatis ikut benar)

Karena `nav-groups.ts` adalah SATU sumber kebenaran dipakai `GuruSidebarDesktop` DAN `SidebarMobile` (T230) — fix di 1 file ini OTOMATIS memperbaiki KEDUA tampilan, TIDAK PERLU perubahan terpisah di komponen sidebar itu sendiri.

## Edge Cases

- **Guru BUKAN pembina ekstra** (`isPembinaEkstra === false`) — grup ini TETAP TIDAK MUNCUL sama sekali (behavior existing sudah benar, TIDAK diubah task ini).
- **Halaman `setting`/`peserta` sendiri** — SUDAH fungsional penuh (backend+frontend, dipakai role `pembina_ekstra` eksternal tanpa masalah) — task ini MURNI nav-groups.ts, TIDAK PERLU sentuh halaman `setting/page.tsx`/`peserta/page.tsx` itu sendiri.

## Files
- **Modifikasi:** `apps/web/src/app/(guru)/components/nav-groups.ts` (`EKSTRAKURIKULER_GROUP`).
- **Jangan sentuh:** `apps/web/src/app/(guru)/guru/ekstrakurikuler/setting/page.tsx`, `peserta/page.tsx` (sudah fungsional, tidak perlu diubah), `apps/web/src/app/(pembina-ekstra)/*` (role eksternal, tidak terdampak/tidak perlu diubah).

## Eksekusi (2026-08-21)

**Koreksi kecil dari narasi spec**: contoh kode di spec poin 1 menulis label "Absensi",
tapi referensi asli `pembina-ekstra-sidebar.tsx` (yang spec sendiri minta direplikasi
PERSIS) pakai label **"Presensi"** — mengikuti file referensi asli (sumber kebenaran
eksplisit menurut spec), BUKAN contoh kode narasi yang keliru ketik.

`EKSTRAKURIKULER_GROUP` (`nav-groups.ts`) — infrastruktur `children` dari T234 SUDAH
ADA, TAPI dipakai sebagai 3 item FLAT (bukan 1 item expandable + children) — groupLabel
"Ekstrakurikuler" sudah cukup jelas konteksnya di sidebar, konsisten pola item flat
lain di `WALI_KELAS_GROUP` ("Biodata Murid" dst yang tidak di-nest lagi di bawah 1
induk). Icon `Sparkles` (lama, groupLabel-level) diganti `ClipboardList`/`Users`/
`Settings` (item-level, REPLIKASI PERSIS `pembina-ekstra-sidebar.tsx`).

Karena `nav-groups.ts` SATU sumber kebenaran dipakai `GuruSidebarDesktop` DAN
`SidebarMobile` (T230), fix di 1 file ini otomatis benar di kedua tampilan — tidak ada
perubahan lain diperlukan. `setting/page.tsx`/`peserta/page.tsx` TIDAK disentuh sama
sekali (dikonfirmasi sudah fungsional penuh, dipakai role `pembina_ekstra` eksternal
tanpa masalah).

tsc bersih, full suite jest 42/610 pass (0 regresi, perubahan murni frontend
navigasi/data statis).

## Acceptance Criteria
- [x] Guru pembina ekstra — sidebar (desktop+mobile) tampilkan 3 item: Presensi, Daftar Peserta, Setting (label "Presensi" bukan "Absensi", mengikuti referensi asli yang lebih otoritatif dari contoh kode narasi spec).
- [x] Klik "Daftar Peserta" — buka halaman `peserta/page.tsx` yang SUDAH ADA (href `/guru/ekstrakurikuler/peserta`, route dikonfirmasi ada sebelum implementasi).
- [x] Klik "Setting" — buka halaman `setting/page.tsx` yang SUDAH ADA (href `/guru/ekstrakurikuler/setting`, route dikonfirmasi ada).
- [x] Guru BUKAN pembina ekstra — grup ini tetap tidak muncul (behavior kondisional `isPembinaEkstra` di `guru-sidebar-desktop.tsx`/`sidebar-mobile.tsx` TIDAK disentuh, hanya isi `EKSTRAKURIKULER_GROUP` yang berubah).
- [x] Build + type-check hijau.

## Validasi Claudian
- [x] Konfirmasi 2 halaman (`setting`/`peserta`) TIDAK diubah sama sekali — hanya `nav-groups.ts` yang di-modifikasi, diverifikasi `git diff` scope.
- [x] Konfirmasi urutan+label item KONSISTEN dengan pola `pembina-ekstra-sidebar.tsx` (referensi yang sudah benar) — label PERSIS sama ("Presensi", bukan "Absensi" dari contoh kode narasi yang typo), urutan PERSIS sama (Presensi → Daftar Peserta → Setting).
