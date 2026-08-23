# T234 — Web: Wali Kelas — Restrukturisasi dari Section Tab Menjadi Sub-Menu Sidebar

## Depends on
Tidak ada dependency teknis. Independen. **WAJIB dikerjakan SETELAH memahami kondisi terkini** (T224a-d, T226, T230 SEMUA sudah dieksekusi — struktur di task ini adalah kondisi TERKINI hasil task-task itu, BUKAN spec awal T224 lagi).

## Konteks — Kondisi Kode Saat Ini (dikonfirmasi via riset 2026-08-21)

Halaman Wali Kelas (`apps/web/src/app/(guru)/guru/wali-kelas/wali-kelas-view.tsx`) SAAT INI adalah **1 halaman dengan 6 SECTION TAB** (state `activeTab` di client, klik tombol pill untuk ganti tab, SEMUA data di-fetch sekaligus di `page.tsx` lalu diteruskan sebagai props ke `WaliKelasView`):

```ts
type TabKey = "hari-ini" | "ringkasan" | "rekap-mapel" | "catatan" | "daftar-siswa" | "rekap-detail";
```

| Tab existing | File komponen | Sumber data |
|---|---|---|
| Hari Ini | `components/hari-ini-tab.tsx` | fetch sendiri (T226) |
| Ringkasan Kehadiran | `components/ringkasan-kehadiran-tab.tsx` | `initialRekap` props |
| Rekap Detail | `components/rekap-detail-tab.tsx` | `academicYears` props, fetch `RekapView` scoped (T224d) |
| Daftar Siswa | `components/daftar-siswa-tab.tsx` | `siswaList` props (T224a) |
| Rekap Per Mapel | `components/rekap-mapel-tab.tsx` | `initialBreakdown`+`initialPermits` props |
| Catatan Guru Mapel | `components/catatan-tab.tsx` | `initialCatatan` props |

Plus halaman drill-down terpisah SUDAH ADA: `apps/web/src/app/(guru)/guru/wali-kelas/siswa/[id]/siswa-detail-wali-view.tsx` (biodata+riwayat siswa, T224b/d).

**Menu sidebar SAAT INI** (`apps/web/src/app/(guru)/components/nav-groups.ts`) — `WALI_KELAS_GROUP` cuma **1 item link tunggal** ke `/guru/wali-kelas`:
```ts
export interface NavGroupItem {
  label: string;
  href: string;
  icon: typeof Calendar;
}
export const WALI_KELAS_GROUP: NavGroup = {
  groupLabel: "Wali Kelas",
  items: [{ label: "Wali Kelas", href: "/guru/wali-kelas", icon: Users }],
};
```
Data model `NavGroupItem` FLAT (tidak ada nested children) — dipakai SAMA oleh `GuruSidebarDesktop` DAN `SidebarMobile` (T230, 1 sumber kebenaran shared).

**Pola sub-menu bertingkat (expand/collapse) SUDAH ADA di proyek ini** — `apps/web/src/components/shell/sidebar.tsx`, komponen `NavGroupSection` (dipakai sidebar ADMIN) — `useState(groupHasActiveItem)` untuk expand/collapse grup, auto-expand kalau salah satu child sedang aktif. **REPLIKASI pola ini**, JANGAN desain interaksi baru dari nol.

**Modul terkait yang SUDAH ADA** (untuk sub-menu "Laporan Guru"): `/guru/nilai` (`apps/web/src/app/(guru)/guru/nilai/`) — SAAT INI di `BASE_GROUP` (grup "Guru"), scope-nya SEMUA mapel yang diajar guru (BUKAN scoped ke kelas wali) — BEDA konteks dari "Nilai" yang diminta di sub-menu Laporan Guru wali kelas (lihat poin 3 Keputusan).

**Dikonfirmasi ULANG: TIDAK ADA sama sekali** model/fitur terkait pelanggaran/BK/SP1-3/penanganan siswa di seluruh codebase (47 model Prisma dicek, tidak ada satupun terkait ini).

## Keputusan Diminta User (2026-08-21)

**Ganti dari section-tab (1 halaman) menjadi SUB-MENU sidebar sungguhan** (URL/route berbeda per sub-menu, BUKAN state tab client) dengan struktur:

1. **Biodata Murid** → Daftar Nama Murid (= tab "Daftar Siswa" existing, DIPINDAH jadi sub-menu sendiri)
2. **Kehadiran** → 4 sub-sub-menu: Hari Ini, Ringkasan Kehadiran, Rekap Detail, Rekap Per Mapel (SEMUA existing, DIPINDAH)
3. **Laporan Guru** → 3 sub-sub-menu: Catatan (= tab "Catatan Guru Mapel" existing), Kehadiran Guru (BARU, KOSONG), Nilai (BARU, KOSONG — BUKAN link ke `/guru/nilai` yang sudah ada, itu scope beda/semua-mapel-guru, ini KHUSUS nilai siswa kelas wali)
4. **Penanganan Murid** — KOSONG SEMUA (placeholder, fitur BK/SP1-3 belum ada modelnya sama sekali, DI LUAR SCOPE task ini — user eksplisit: "untuk sekarang tampilkan saja dulu sub menu kosong")
5. **Murid Mutasi** — BARU, KOSONG (rencana: daftar siswa nonaktif kelas wali — DATA-nya SEBENARNYA sudah bisa diambil dari endpoint T224a yang include siswa nonaktif, TAPI task ini HANYA buat placeholder halaman, TIDAK PERLU implementasi filter nonaktif-saja sekarang — user eksplisit "tidak perlu anda selesaikan semua")
6. **Sub Menu Graf** — BARU, KOSONG (rencana: grafik-grafik, di luar scope task ini)

**INSTRUKSI EKSPLISIT USER**: "tidak perlu anda selesaikan semua, cukup ganti struktur sub menunya, pecah dari section tab menjadi sub menu. biarkan sub menu yang belum memiliki tampilan menjadi submenu kosong untuk sementara waktu." — task ini **MURNI RESTRUKTURISASI NAVIGASI**, BUKAN membangun fitur baru (Kehadiran Guru, Nilai versi wali, Penanganan Murid, Murid Mutasi, Grafik SEMUA jadi halaman placeholder kosong, BUKAN dikerjakan penuh).

## Spec Detail

### 1. Extend `NavGroupItem` untuk dukung nested children

`apps/web/src/app/(guru)/components/nav-groups.ts` — TAMBAH opsional `children` ke interface, REPLIKASI pola data admin sidebar kalau sudah ada struktur serupa (VERIFIKASI SAAT IMPLEMENTASI struktur data `sidebar.tsx` admin untuk grup+children, KONSISTEN bentuk):
```ts
export interface NavGroupItem {
  label: string;
  href: string;
  icon: typeof Calendar;
  children?: NavGroupItem[]; // BARU — sub-item bertingkat, kalau ada berarti item ini EXPANDABLE bukan link langsung
}
```

### 2. Struktur `WALI_KELAS_GROUP` baru — 6 item, sebagian dengan children

```ts
export const WALI_KELAS_GROUP: NavGroup = {
  groupLabel: "Wali Kelas",
  items: [
    { label: "Biodata Murid", href: "/guru/wali-kelas/biodata", icon: ... },
    {
      label: "Kehadiran", href: "/guru/wali-kelas/kehadiran", icon: ...,
      children: [
        { label: "Hari Ini", href: "/guru/wali-kelas/kehadiran/hari-ini", icon: ... },
        { label: "Ringkasan Kehadiran", href: "/guru/wali-kelas/kehadiran/ringkasan", icon: ... },
        { label: "Rekap Detail", href: "/guru/wali-kelas/kehadiran/rekap-detail", icon: ... },
        { label: "Rekap Per Mapel", href: "/guru/wali-kelas/kehadiran/rekap-mapel", icon: ... },
      ],
    },
    {
      label: "Laporan Guru", href: "/guru/wali-kelas/laporan-guru", icon: ...,
      children: [
        { label: "Catatan", href: "/guru/wali-kelas/laporan-guru/catatan", icon: ... },
        { label: "Kehadiran Guru", href: "/guru/wali-kelas/laporan-guru/kehadiran-guru", icon: ... },
        { label: "Nilai", href: "/guru/wali-kelas/laporan-guru/nilai", icon: ... },
      ],
    },
    { label: "Penanganan Murid", href: "/guru/wali-kelas/penanganan", icon: ... },
    { label: "Murid Mutasi", href: "/guru/wali-kelas/mutasi", icon: ... },
    { label: "Sub Menu Graf", href: "/guru/wali-kelas/grafik", icon: ... },
  ],
};
```
- **VERIFIKASI SAAT IMPLEMENTASI**: ikon SPESIFIK per item (`lucide-react`) — PUTUSKAN yang masuk akal secara visual, KONSISTEN gaya ikon existing (`Users` untuk biodata, dst).
- **PENTING**: "Biodata Murid", "Penanganan Murid", "Murid Mutasi", "Sub Menu Graf" adalah item TANPA children (link langsung 1 halaman) — HANYA "Kehadiran" dan "Laporan Guru" yang expandable (punya children, karena masing-masing ada beberapa sub-halaman).

### 3. Route baru — pecah `/guru/wali-kelas` jadi banyak sub-route

Struktur folder Next.js App Router baru di `apps/web/src/app/(guru)/guru/wali-kelas/`:
```
biodata/page.tsx                          → render komponen `DaftarSiswaTab` (RENAME jadi tidak perlu, cukup dipindah/wrap)
kehadiran/hari-ini/page.tsx                → render `HariIniTab`
kehadiran/ringkasan/page.tsx               → render `RingkasanKehadiranTab`
kehadiran/rekap-detail/page.tsx            → render `RekapDetailTab`
kehadiran/rekap-mapel/page.tsx             → render `RekapMapelTab`
laporan-guru/catatan/page.tsx              → render `CatatanTab`
laporan-guru/kehadiran-guru/page.tsx       → BARU, placeholder kosong (lihat poin 5)
laporan-guru/nilai/page.tsx                → BARU, placeholder kosong
penanganan/page.tsx                        → BARU, placeholder kosong
mutasi/page.tsx                            → BARU, placeholder kosong
grafik/page.tsx                            → BARU, placeholder kosong
```
- **VERIFIKASI SAAT IMPLEMENTASI**: apakah `/guru/wali-kelas` (root, tanpa sub-path) perlu `redirect()` ke salah satu sub-halaman default (KONSISTEN aturan CLAUDE.md "route lama yang dikonsolidasi → redirect server-side" — di sini ARAHNYA TERBALIK, dari 1 route jadi banyak, TAPI prinsip yang sama berlaku: JANGAN biarkan `/guru/wali-kelas` polos jadi 404 atau halaman kosong tanpa arah, REKOMENDASI: `redirect("/guru/wali-kelas/biodata")` sebagai default masuk pertama).
- Komponen tab EXISTING (`daftar-siswa-tab.tsx`, `hari-ini-tab.tsx`, dst) — **REUSE APA ADANYA** (pindahkan import ke `page.tsx` masing-masing sub-route, JANGAN tulis ulang logic-nya), HANYA `wali-kelas-view.tsx` (state tab lama) yang DIHAPUS/tidak dipakai lagi.
- **Fetch data** — SAAT INI `page.tsx` tunggal fetch SEMUA data sekaligus untuk 6 tab (boros, semua di-load walau user cuma buka 1 tab) — task ini JADI KESEMPATAN memperbaiki ini SEKALIAN: tiap `page.tsx` sub-route HANYA fetch data yang relevan untuk sub-halaman itu sendiri (KONSISTEN prinsip Next.js App Router server component per-halaman) — VERIFIKASI SAAT IMPLEMENTASI endpoint apa yang perlu dipanggil di tiap sub-route berdasarkan props yang SEBELUMNYA diterima komponen tab itu.

### 4. Sidebar desktop + mobile — render children (expand/collapse)

- `apps/web/src/app/(guru)/components/guru-sidebar-desktop.tsx` — TAMBAH logic render `children` kalau item punya itu, REPLIKASI PERSIS pola `NavGroupSection` dari `apps/web/src/components/shell/sidebar.tsx` (admin) — auto-expand kalau child sedang aktif (`groupHasActiveItem`), collapse/expand manual via klik.
- `apps/web/src/app/(guru)/components/sidebar-mobile.tsx` (T230) — SAMA POLA, REUSE logic yang SAMA (idealnya extract komponen `NavGroupSection` SHARED dipakai desktop+mobile guru KEDUANYA, supaya TIDAK ADA 2 implementasi expand/collapse berbeda yang bisa drift — KONSISTEN pelajaran dari bug T230 sebelumnya soal drift 2 array menu).
- `resolveActiveGroup()`/`isItemActive()` (`nav-groups.ts`) — VERIFIKASI SAAT IMPLEMENTASI apakah logic ini perlu di-extend untuk mendukung matching PATHNAME terhadap children (bukan cuma `items` flat level pertama) — highlight harus benar baik untuk grup TANPA children (Biodata Murid dst) maupun grup DENGAN children (Kehadiran → Hari Ini dst, kedua level harus ter-highlight benar).

### 5. Halaman placeholder kosong — pola konsisten

Untuk `Kehadiran Guru`, `Nilai` (dalam Laporan Guru), `Penanganan Murid`, `Murid Mutasi`, `Sub Menu Graf` — SEMUA halaman ini BARU dan **KOSONG** (belum ada fitur), tapi TETAP harus:
- Route valid, tidak 404.
- Tampilkan state "Fitur belum tersedia" / "Segera hadir" yang JELAS (BUKAN halaman blank putih tanpa penjelasan — konsisten prinsip UX proyek, pesan actionable meski untuk placeholder) — REKOMENDASI: komponen shared `ComingSoonPlaceholder` kecil, dipakai ke-5 halaman ini, supaya konsisten visual DAN gampang diganti nanti kalau fitur asli mulai dibangun (tinggal replace isi 1 file per halaman).
- `usePageTitle()` tetap dipanggil dengan judul yang benar (KONSISTEN pola existing tiap halaman guru).

### 6. Catatan untuk masa depan (DICATAT, TIDAK dikerjakan task ini)

- **Penanganan Murid** — nanti akan terhubung ke akun guru BK untuk lihat SP1-SP3/peringatan lintas-role (guru pengajar, wali kelas, BK semua bisa lihat) — INI PERLU MODUL BARU TOTAL (model Prisma baru, role/akses baru) — TASK TERPISAH DI MASA DEPAN, TIDAK ADA DI SCOPE T234.
- **Murid Mutasi** — rencana isi: daftar siswa nonaktif kelas wali (data SUDAH bisa diambil dari endpoint T224a yang sudah include siswa nonaktif dengan badge) — TASK TERPISAH untuk mengisi kontennya nanti, T234 HANYA buat placeholder.
- **Sub Menu Graf** — rencana isi: visualisasi grafik kehadiran dsb — TASK TERPISAH.
- **Laporan Guru > Kehadiran Guru** — rencana isi: kemungkinan berbeda dari `/guru/nilai` existing (yang scope semua mapel guru) — perlu KLARIFIKASI LEBIH LANJUT dari user saat task pengisian konten ini ditulis nanti (apakah maksudnya kehadiran GURU-GURU LAIN yang mengajar di kelas wali ini, dilihat dari sudut pandang wali kelas — BEDA dari kehadiran siswa).
- **Laporan Guru > Nilai** — SAMA, perlu klarifikasi scope (nilai siswa di kelas wali, across semua mapel — BEDA dari `/guru/nilai` yang nilai per-mapel-yang-diajar-guru-itu-sendiri).

## Edge Cases

- **User buka `/guru/wali-kelas` langsung (root, bookmark lama)** — redirect ke sub-halaman default (`/guru/wali-kelas/biodata`), JANGAN 404.
- **URL lama di-bookmark user** (kalau ada yang sempat pakai state tab lama, TIDAK ADA URL berbeda untuk tab lama karena itu state client — jadi TIDAK ADA bookmark lama yang rusak, aman).
- **Sidebar mobile — grup "Kehadiran"/"Laporan Guru" dengan banyak children** — pastikan TIDAK memenuhi layar mobile secara berlebihan saat di-expand (scroll internal kalau perlu, KONSISTEN pola admin sidebar kalau sudah menangani kasus serupa).

## Files
- **Modifikasi:** `apps/web/src/app/(guru)/components/nav-groups.ts` (struktur `WALI_KELAS_GROUP` baru + `NavGroupItem.children`), `guru-sidebar-desktop.tsx`+`sidebar-mobile.tsx` (render children, REPLIKASI `NavGroupSection` admin).
- **Buat:** 11 folder route baru di `apps/web/src/app/(guru)/guru/wali-kelas/` (lihat poin 3), komponen `ComingSoonPlaceholder` shared.
- **Hapus/tidak dipakai lagi:** `apps/web/src/app/(guru)/guru/wali-kelas/wali-kelas-view.tsx` (state-tab lama, PASTIKAN tidak ada import tersisa yang mengarah ke file ini setelah migrasi selesai).
- **Jangan sentuh:** isi LOGIC komponen tab existing (`daftar-siswa-tab.tsx` dkk) — HANYA dipindah lokasi importnya, konten/fetch internal komponen TIDAK berubah.

## Acceptance Criteria
- [ ] Sidebar (desktop+mobile) — grup "Wali Kelas" tampil 6 item, 2 di antaranya (Kehadiran, Laporan Guru) expandable dengan children yang benar.
- [ ] Semua 6 tab existing (Biodata/Daftar Siswa, Hari Ini, Ringkasan, Rekap Detail, Rekap Per Mapel, Catatan) — BERPINDAH jadi sub-menu terpisah dengan URL sendiri, FUNGSI TIDAK BERUBAH (data yang sama, tampilan yang sama, cuma cara aksesnya beda).
- [ ] 5 sub-menu baru (Kehadiran Guru, Nilai [Laporan Guru], Penanganan Murid, Murid Mutasi, Sub Menu Graf) — route valid, tampil placeholder "belum tersedia" yang jelas, TIDAK 404.
- [ ] `/guru/wali-kelas` (root) — redirect ke sub-halaman default, tidak blank/404.
- [ ] Highlight item aktif di sidebar — benar untuk SEMUA level (item tanpa children maupun children di dalam grup expandable).
- [ ] Halaman drill-down `siswa/[id]/` (detail biodata+riwayat) TIDAK regresi (masih bisa diakses dari sub-menu Biodata Murid).
- [ ] Build + type-check hijau.

## Validasi Claudian
- [ ] Konfirmasi komponen tab existing DI-REUSE APA ADANYA (dipindah, bukan ditulis ulang) — verifikasi diff HANYA di level routing/import, bukan logic internal komponen.
- [ ] Konfirmasi pola expand/collapse sidebar SHARED antara desktop+mobile guru (bukan 2 implementasi terpisah yang bisa drift, pelajaran dari T230).
- [ ] Konfirmasi 5 halaman placeholder BENAR-BENAR kosong (tidak ada implementasi fitur nyata di baliknya) — task ini murni restrukturisasi navigasi sesuai instruksi eksplisit user, BUKAN kesempatan diam-diam membangun fitur baru yang belum diminta detailnya.
