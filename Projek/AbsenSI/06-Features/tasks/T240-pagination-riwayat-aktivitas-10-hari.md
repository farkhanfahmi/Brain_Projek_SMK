# T240 — API+Web: Pagination "Riwayat Aktivitas Saya" — 10 Hari per Halaman

## Depends on
Tidak ada dependency teknis. Independen, murni modul `attendance`+`activity-log`+frontend `apps/web/src/app/(guru)/riwayat/`.

## Konteks — Kondisi Kode Saat Ini (dikonfirmasi via riset 2026-08-21, SETELAH T235 dieksekusi)

**Data-fetching terjadi di `page.tsx` (server component), BUKAN `riwayat-view.tsx`**:
```tsx
// apps/web/src/app/(guru)/riwayat/page.tsx — KONDISI SEKARANG
const [history, activity] = await Promise.all([
  apiFetch<MyHistoryRow[]>("/attendance/my-history"),        // TANPA parameter, ambil SEMUA
  apiFetch<ActivityLogPage>("/activity-log/me?pageSize=100"), // HARDCODE pageSize=100, tidak ada page
]);
```

**`myHistory()` endpoint** (`apps/api/src/attendance/attendance.service.ts:570-589`) — HANYA terima `from`/`to` (date-range filter opsional, TIDAK dipakai FE saat ini), **TIDAK ADA** `page`/`limit`/`skip`/`take`/cursor sama sekali. Return array biasa (`MyHistoryRow[]`), BUKAN objek `{items, total, page, pageSize}`.

**`riwayat-view.tsx`** — SEMUA data digabung (`buildTimeline()`) jadi `Map<dateKey, FeedEntry[]>`, di-render SEMUA sekaligus tanpa slicing (`.map()` atas SEMUA `dateKeys`, tidak ada "muat lagi"/pagination UI).

**Pola pagination server-side SUDAH ADA dan matang** di modul `activity-log` — REPLIKASI PERSIS pola ini, JANGAN desain baru:
```ts
// apps/api/src/activity-log/activity-log.service.ts:87-124, findAll()
const page = filter.page ?? 1;
const pageSize = filter.pageSize ?? DEFAULT_PAGE_SIZE;
const [items, total] = await Promise.all([
  this.prisma.activityLog.findMany({ where, skip: (page - 1) * pageSize, take: pageSize, orderBy: ... }),
  this.prisma.activityLog.count({ where }),
]);
return { items, total, page, pageSize };
```
DTO: `apps/api/src/activity-log/dto/list-my-activity-log.dto.ts` (`page?: number`, `pageSize?: number`, `@Type(() => Number)`). Frontend type: `ActivityLogPage` (`core-types.ts:467-472`).

**Struktur "per hari" penting**: "10 hari" berarti **10 GRUP TANGGAL** (10 kartu `ActivityFeedCard`), BUKAN 10 baris entry mentah — karena UI-nya card-per-tanggal dengan N entries (tap masuk+pulang+log jurnal dst) di dalam tiap kartu.

## Keputusan Diminta User (2026-08-21)

Pagination Riwayat Aktivitas — **10 HARI (tanggal unik) per halaman**, bukan 10 baris data mentah.

## Spec Detail

### 1. Backend — `myHistory()` tambah pagination BERBASIS TANGGAL (bukan baris)

- **PENTING — pagination di sini BEDA dari pola `activity-log`** (yang paginate BARIS): task ini perlu paginate **TANGGAL UNIK**, karena 1 tanggal bisa punya beberapa baris (Masuk+Pulang). REKOMENDASI: pagination di LEVEL TANGGAL, bukan level `AttendanceRecord` — ambil `page`/`pageSize` (pageSize = 10 tanggal), resolve DULU daftar TANGGAL UNIK yang punya data (via `SELECT DISTINCT tanggal ... ORDER BY tanggal DESC LIMIT/OFFSET`), BARU query `AttendanceRecord` untuk tanggal-tanggal itu.
- **VERIFIKASI SAAT IMPLEMENTASI**: apakah lebih simpel resolve unique dates dulu via query terpisah (`Prisma.$queryRaw` atau `groupBy`), lalu `findMany({where: {tanggal: {in: uniqueDates}}})` — ATAU pendekatan lain yang lebih Prisma-idiomatic. Prisma `groupBy` bisa dipakai untuk `SELECT DISTINCT tanggal` dengan `orderBy`+`skip`+`take`.
- DTO baru/extend `MyHistoryQueryDto` — tambah `page?: number`, `pageSize?: number` (default `pageSize = 10`, KONSISTEN default value pola `activity-log`).
- Response BARU (BUKAN array polos lagi) — REPLIKASI shape `ActivityLogPage`:
  ```ts
  interface MyHistoryPage {
    items: MyHistoryRow[];  // SEMUA baris untuk tanggal-tanggal dalam halaman ini (bisa >10 baris kalau ada hari dengan masuk+pulang)
    totalDates: number;     // total TANGGAL UNIK (bukan total baris) — untuk hitung total halaman
    page: number;
    pageSize: number;
  }
  ```
- **BREAKING CHANGE untuk FE** — response shape berubah dari array polos ke object — `page.tsx` WAJIB disesuaikan (poin 3).

### 2. Backend — `/activity-log/me` — tambah `page` (SUDAH support `pageSize`, tinggal dipakai)

- Endpoint ini SUDAH mendukung pagination penuh (`page`+`pageSize`) — TIDAK PERLU perubahan backend, HANYA frontend yang perlu MULAI MENGIRIM parameter `page` (sekarang hardcode `pageSize=100` tanpa `page`, artinya SELALU ambil 100 item pertama tanpa jeda).
- **VERIFIKASI SAAT IMPLEMENTASI**: `activity_log` di-gabung dengan `attendance_record` dalam 1 timeline per-tanggal di FE — pagination KEDUA sumber ini perlu SINKRON secara tanggal (bukan independen per masing-masing `page`/`pageSize`) — REKOMENDASI: pagination utama berbasis TANGGAL dari `myHistory()` (poin 1), lalu `activity-log` di-fetch dengan rentang TANGGAL yang SAMA (pakai parameter `from`/`to` kalau endpoint activity-log punya itu, ATAU fetch pageSize besar tetap tapi filter di FE ke rentang tanggal yang relevan — VERIFIKASI mana yang lebih efisien/benar).

### 3. Frontend — UI pagination + fetch bertahap

- `apps/web/src/app/(guru)/riwayat/page.tsx` — UBAH jadi CLIENT-SIDE fetch (atau server component dengan `searchParams` untuk `page` dari URL query string, REKOMENDASI: URL-based pagination `?page=1` supaya bookmark-able, KONSISTEN prinsip halaman lain di proyek) — fetch `myHistory()` dengan `page`+`pageSize=10`.
- `riwayat-view.tsx` — TAMBAH kontrol pagination (tombol "Sebelumnya"/"Selanjutnya", atau nomor halaman — REPLIKASI pola pagination UI YANG SUDAH ADA di proyek ini kalau ada komponen shared, VERIFIKASI SAAT IMPLEMENTASI apakah ada `Pagination` component reusable di `packages/ui` atau tempat lain).
- **Loading state** — saat pindah halaman, tampilkan loading indicator (KONSISTEN pola loading state lain di proyek).

## Edge Cases

- **Tanggal dengan 0 entri gabungan tap+activity_log** (tidak mungkin muncul karena query berbasis tanggal yang PUNYA data) — TIDAK RELEVAN, pagination hanya iterate tanggal yang benar-benar punya minimal 1 baris.
- **Guru baru, riwayat < 10 hari total** — halaman 1 tampil apa adanya (kurang dari 10 kartu), TIDAK ADA halaman 2 (tombol "Selanjutnya" disabled/tidak muncul).
- **`activity_log` dan `attendance_record` untuk tanggal yang SAMA tapi salah satu sumber "kehabisan" pagination duluan** (edge case sinkronisasi 2 sumber data) — VERIFIKASI SAAT IMPLEMENTASI skenario ini tidak menyebabkan data hilang/duplikat, KEMUNGKINAN BESAR aman kalau pendekatan "resolve tanggal dulu, baru fetch kedua sumber untuk rentang tanggal yang sama" (poin 1-2) diikuti konsisten.

## Files
- **Modifikasi:** `apps/api/src/attendance/attendance.service.ts` (`myHistory()` pagination), `apps/api/src/attendance/dto/my-history-query.dto.ts` (`page`/`pageSize`), `apps/web/src/app/(guru)/riwayat/page.tsx`+`riwayat-view.tsx` (UI pagination, adaptasi response shape baru).
- **Jangan sentuh:** `apps/api/src/activity-log/` (endpoint SUDAH support pagination, tidak perlu perubahan backend).

## Acceptance Criteria
- [ ] Halaman Riwayat Aktivitas — tampilkan MAKSIMAL 10 tanggal unik per halaman (bisa lebih dari 10 baris data kalau ada hari dengan masuk+pulang).
- [ ] Kontrol pagination (Sebelumnya/Selanjutnya atau nomor halaman) berfungsi, URL bookmark-able (`?page=N`).
- [ ] Guru dengan riwayat < 10 hari — tidak ada halaman 2, kontrol pagination tidak membingungkan.
- [ ] `activity_log` (jurnal/log lain) dan `attendance_record` (tap gerbang) — SINKRON tampil untuk tanggal yang sama dalam 1 halaman, tidak ada data "bocor" ke halaman lain karena pagination independen.
- [ ] Build + type-check hijau, jest baru untuk `myHistory()` pagination (halaman 1, halaman 2, kurang dari 10 hari total).

## Validasi Claudian
- [ ] Konfirmasi pagination berbasis TANGGAL UNIK, bukan baris `AttendanceRecord` mentah (1 tanggal bisa 2 baris masuk+pulang, jangan sampai halaman 1 dan 2 memecah 1 tanggal jadi 2 halaman berbeda).
- [ ] Konfirmasi `activity-log` dan `attendance-record` di-fetch untuk RENTANG TANGGAL YANG SAMA per halaman (bukan pagination independen yang bisa membuat 1 tanggal punya data tap tapi tidak ada data log atau sebaliknya).
- [ ] Konfirmasi response `myHistory()` yang berubah shape (array → object `{items,totalDates,page,pageSize}`) tidak merusak consumer LAIN dari endpoint ini kalau ada (grep pemanggil lain sebelum ubah shape).
