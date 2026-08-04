# T097 — UI: Sidebar Guru Berkelompok (Guru / Wali Kelas / Ekstrakurikuler)

## Depends on
Tidak ada — murni restrukturisasi UI `GuruSidebar`, tidak menyentuh backend/RBAC. **BUKAN** bagian dari perubahan besar "guru_piket jadi peran tambahan" yang dibahas tapi SENGAJA DITUNDA (lihat Catatan Cofounder) — jangan gabungkan scope.

## Objective
Kelompokkan menu sidebar guru jadi 3 grup visual (Guru, Wali Kelas, Ekstrakurikuler) supaya guru yang punya banyak peran sekaligus (guru + wali kelas + pembina ekstra, semua dalam 1 akun) lebih mudah membedakan menu — bukan daftar menu datar seperti sekarang.

## Context
- **App:** `apps/web`
- **File utama:** `apps/web/src/app/(guru)/guru-sidebar.tsx`
- Diskusi lengkap 2026-07-30 — user awalnya minta 4 kelompok termasuk "Piket", tapi `guru_piket` ternyata ROLE AKUN TERPISAH (login beda, `PiketSidebar` beda, scoping `kampusId` dari JWT yang tidak dimiliki akun `guru` biasa) — mengubahnya jadi peran tambahan dalam 1 akun guru menyentuh RBAC inti (24 file memakai `role === "guru_piket"` langsung, ADR-015 scoping kampus, ADR-024 validasi bertugas). **User sepakat itu didiskusikan terpisah nanti, T097 ini HANYA untuk 3 kelompok yang sudah bisa hidup dalam 1 akun guru sekarang: Guru, Wali Kelas, Ekstrakurikuler.**

## Catatan Cofounder — Baca Sebelum Eksekusi

**Kalau nanti ada task lanjutan "Piket sebagai peran tambahan guru"**, ini bukan sekadar tambah grup ke-4 di sidebar — itu perubahan arsitektur RBAC:
- Semua endpoint piket (`permits`, board, dsb) bergantung pada `user.kampusId` di JWT (ADR-015) — akun `guru` biasa tidak punya `kampusId` terisi sama sekali (guru mengajar lintas kelas/kampus, `Teacher` model tidak punya field kampus).
- 24 file pakai `role === "guru_piket"` atau `@Roles(UserRole.guru_piket)` langsung (guard backend `RolesGuard`/`PiketOnDutyGuard`, middleware, layout, sidebar frontend) — semua perlu pola `isPiket` boolean turunan (seperti `isWaliKelas`/`isPembinaEkstra`), bukan cek role langsung.
- ADR-015 (scoping kampus piket) dan ADR-024 (validasi bertugas piket) perlu direvisi kalau fondasi role berubah.
- Perlu jalur migrasi akun `guru_piket` existing.

**JANGAN kerjakan hal di atas tanpa task terpisah yang eksplisit membahas ini** — kalau user minta lagi nanti, buat task baru (T098+), jangan diam-diam diperluas dari T097.

## Keputusan Final (dikonfirmasi user 2026-07-30)

1. **3 kelompok**: "Guru" (Jurnal Mengajar, Riwayat Kehadiran), "Wali Kelas" (1 menu: Wali Kelas), "Ekstrakurikuler" (1 menu: Ekstrakurikuler) — sesuai flag yang SUDAH ADA di `GuruSidebar` (`isWaliKelas`, `isPembinaEkstra`, sudah dieksekusi lewat T096 sebelum task ini ditulis).
2. **Grup dengan 2+ menu → accordion collapsible** (bisa dilipat/dibuka via klik header). Saat ini HANYA grup "Guru" yang punya 2 menu (Jurnal Mengajar + Riwayat Kehadiran) — jadi ini satu-satunya grup yang benar-benar jadi accordion untuk sekarang.
3. **Grup dengan tepat 1 menu → tampil sebagai link biasa**, TANPA header collapsible terpisah — cukup label kecil di atas link tunggal itu (bukan tombol accordion yang bisa diklik untuk dilipat, karena tidak ada isi yang disembunyikan). Jadi "Wali Kelas" dan "Ekstrakurikuler" tampil sebagai: label kecil abu-abu "WALI KELAS" lalu langsung 1 baris link di bawahnya — TIDAK bisa diklik untuk collapse.
4. **Label grup SELALU tampil**, bahkan kalau guru cuma punya 1 kelompok aktif (guru biasa, bukan wali kelas, bukan pembina ekstra) — tetap tampilkan header "GURU" di atas 2 menu itu, demi konsistensi visual (jangan sembunyikan label untuk kasus paling umum).
5. **State accordion TIDAK disimpan** (bukan localStorage) — grup yang cocok dengan `pathname` aktif OTOMATIS terbuka saat render, grup lain default tertutup. Reset tiap kali pindah halaman/reload (state ditentukan murni dari `pathname`, bukan `useState` independen yang bisa nyangkut).

## Spec Detail

### Struktur data baru
Ganti flat array `navItems` jadi struktur berkelompok:
```typescript
interface NavGroup {
  groupLabel: string;
  items: { label: string; href: string; icon: LucideIcon }[];
}

const GROUPS: NavGroup[] = [
  {
    groupLabel: "Guru",
    items: [
      { label: "Jurnal Mengajar", href: "/guru/jurnal", icon: BookOpenCheck },
      { label: "Riwayat Kehadiran", href: "/riwayat", icon: FileClock },
    ],
  },
];
// Wali Kelas & Ekstrakurikuler ditambahkan kondisional (isWaliKelas/isPembinaEkstra),
// masing-masing groupLabel dengan items: [{ 1 menu }] -- tetap pakai struktur NavGroup
// yang sama, tapi dirender sebagai link biasa karena items.length === 1 (lihat poin render).
```

### Logic render per grup
- `items.length > 1` → render sebagai accordion: header grup (label + chevron icon, klik untuk toggle), lalu `items` di bawahnya HANYA render kalau grup dalam state terbuka.
- `items.length === 1` → render label grup (teks kecil, TANPA ikon chevron, TANPA onClick) langsung diikuti 1 baris link biasa (styling sama seperti item accordion lainnya untuk konsistensi).
- **Grup mana yang default terbuka**: cek apakah SALAH SATU `item.href` di grup itu match `pathname` aktif (pola cek match "aktif" yang SUDAH ADA di komponen ini — match terpanjang menang, lihat komentar existing soal T096). Grup yang mengandung item aktif → default terbuka. Grup lain (accordion, bukan link-tunggal) → default tertutup.
- User BISA klik header grup manapun untuk toggle manual (buka grup yang sedang tertutup, tutup grup yang sedang terbuka) — state ini `useState<Set<string>>` lokal per grup label, di-inisialisasi dari logic "grup aktif" di atas, TIDAK disinkronkan ulang otomatis saat pathname berubah dalam sesi yang sama kecuali re-mount (navigasi antar grup App Router tidak remount komponen sidebar, jadi state manual user harus dihormati sampai reload penuh — cek dulu apakah re-render terjadi natural saat pindah rute, sesuaikan kalau perilaku actual berbeda dari asumsi ini).

### Styling
- Ikuti pola desain existing (`text-body-medium`, `rounded-lg`, warna aktif `bg-primary text-white`) — JANGAN ciptakan skema warna baru, ini murni perubahan struktur bukan visual baru.
- Header grup: teks lebih kecil (`text-label` atau `text-caption`), warna `text-ink-tertiary` atau serupa (beda dari item link supaya jelas ini label kategori, bukan link yang bisa diklik navigasi — TAPI untuk grup accordion, tetap bisa diklik untuk toggle, jadi tetap butuh `cursor-pointer` + hover state ringan).
- Chevron icon (lucide `ChevronDown`/`ChevronRight`) di header grup accordion, arah berubah sesuai state terbuka/tertutup.

## Business Rules
- Tidak ada perubahan backend/guard/API sama sekali — T097 murni presentational.
- Proteksi akses tetap di backend (endpoint kelas-wali, endpoint ekstra) — pengelompokan sidebar TIDAK mengubah apapun soal siapa yang boleh akses apa, cuma tampilan menu yang SUDAH ditentukan boleh tampil (via `isWaliKelas`/`isPembinaEkstra` yang sudah ada).

## Edge Cases
- Guru biasa (bukan wali kelas, bukan pembina ekstra) — hanya 1 grup "Guru" tampil (accordion, karena 2 item), TIDAK ada grup Wali Kelas/Ekstrakurikuler sama sekali (bukan ditampilkan kosong/disabled).
- Guru yang jadi wali kelas SEKALIGUS pembina ekstra — 3 grup semua tampil: "Guru" (accordion 2 item), "Wali Kelas" (link tunggal), "Ekstrakurikuler" (link tunggal).
- Guru buka halaman `/guru/wali-kelas` — grup "Wali Kelas" (link tunggal) otomatis ter-highlight sebagai aktif, grup "Guru" (accordion) default TERTUTUP karena tidak ada item aktif di dalamnya.
- Guru klik header grup "Guru" untuk membukanya secara manual walau sedang di halaman Wali Kelas — grup terbuka menampilkan 2 sub-menu, TIDAK otomatis pindah halaman (accordion cuma expand/collapse, bukan navigasi).

## Files
- **Modifikasi:** `apps/web/src/app/(guru)/guru-sidebar.tsx` — restrukturisasi total dari flat array ke grouped, tambah accordion state + logic render kondisional (accordion vs link tunggal).
- **Jangan sentuh:** backend apapun, `apps/web/src/lib/current-user.ts`, endpoint `GET /users/me` (flag `isWaliKelas`/`isPembinaEkstra` sudah benar, tidak perlu diubah), `PiketSidebar`/`Sidebar` admin (di luar scope, walau punya bug `isActive` serupa yang sudah dicatat di task lain — T094 untuk piket).

## Acceptance Criteria
- [ ] Guru biasa (tanpa wali kelas/ekstra) — sidebar tampil 1 grup "Guru" berlabel, berisi Jurnal Mengajar + Riwayat Kehadiran, bisa dilipat/dibuka via klik header.
- [ ] Guru + Wali Kelas — 2 grup: "Guru" (accordion) dan "Wali Kelas" (link tunggal berlabel, tidak collapsible).
- [ ] Guru + Wali Kelas + Ekstrakurikuler — 3 grup semua tampil sesuai aturan di atas.
- [ ] Buka halaman yang match salah satu grup — grup itu default terbuka, grup accordion lain default tertutup.
- [ ] Klik header grup accordion — toggle buka/tutup berfungsi, tidak memicu navigasi.
- [ ] Tidak ada regresi pada highlight menu aktif (bug serupa T094/T096 — pastikan match tetap benar per item, bukan grup yang salah ter-highlight).
- [ ] Build + type-check `apps/web` hijau.
- [ ] Verifikasi visual (Playwright atau manual) untuk ketiga kombinasi peran di atas.

## Validasi Claudian
- [ ] Scope TIDAK melebar ke perubahan role `guru_piket` — itu didiskusikan tapi SENGAJA DITUNDA, task terpisah kalau nanti dilanjutkan (baca Catatan Cofounder di atas).
- [ ] Tidak menyentuh backend/RBAC sama sekali — murni restrukturisasi 1 komponen frontend.
- [ ] Baca ulang `guru-sidebar.tsx` versi TERKINI sebelum eksekusi (T096 sudah menambah `isPembinaEkstra` sejak dokumen ini ditulis, mungkin ada perubahan lanjutan lain sebelum T097 benar-benar dikerjakan).
