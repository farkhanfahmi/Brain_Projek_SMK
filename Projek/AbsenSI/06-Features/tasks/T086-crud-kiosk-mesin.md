# T086 — UI+Fix: Lengkapi CRUD Mesin (Edit) + Tambah @LogActivity yang Hilang

## Depends on
Tidak ada — API `PATCH /kiosks/:id` (update umum) SUDAH ADA dan lengkap (`UpdateKioskDto`: `nama`, `allowedIp`), murni pekerjaan UI + 1 fix kecil backend.

## Context
- **App:** `apps/api` + `apps/web`
- **File:** `apps/web/src/app/(admin)/kiosk/kiosk-view.tsx`, `apps/api/src/kiosks/kiosks.controller.ts`
- **Ref:** Ditemukan user 2026-07-26 — halaman `/kiosk` (Daftar Mesin) cuma punya Create (Tambah Mesin) dan Deactivate. TIDAK ADA cara edit `nama`/`allowedIp` dari UI meski endpoint backend-nya (`PATCH /kiosks/:id`, `kiosks.controller.ts:58-63`) sudah lengkap sejak awal. Kasus nyata yang memicu ini: admin salah input `allowedIp` kiosk "test_laptop" (karena NAT jaringan sekolah, lihat insiden T085), satu-satunya cara perbaiki sekarang adalah nonaktifkan + buat baru — seharusnya cukup Edit.

## Spec Detail

### Gap 1 — UI: Tombol Edit di Tabel Mesin
`kiosk-view.tsx:143-158` (kolom Aksi) — tambah 1 tombol icon `Pencil` (pola sama seperti tombol icon lain di tabel yang sama: `h-11 w-11 rounded-full hover:bg-surface-subtle`, konsisten dengan pola CRUD Guru T077), diletakkan sebelum tombol Link2 (URL) dan Deactivate.

Klik Edit → buka `Dialog` (BUKAN Sheet — form ini cuma 2-3 field: Nama, IP, mungkin Kampus/Tipe kalau memang bisa diubah, jauh di bawah ambang 6 field yang butuh Sheet+Tabs per `03-components.md`) berisi form edit:
- Reuse `KioskForm` yang sudah ada (`kiosk-view.tsx:174-286`) — refactor jadi dual-mode sama seperti `GuruForm` (T077): prop `mode: "create" | "edit"`, `initialData?: Kiosk`
- Field yang BISA diedit: `nama`, `allowedIp` (sesuai `UpdateKioskDto` yang sudah ada di backend)
- Field `kampusId` dan `tipe` — CEK dulu apakah `UpdateKioskDto` backend mendukungnya (saat ini TIDAK, cuma `nama`+`allowedIp`). Kalau user butuh ubah kampus/tipe juga, itu perlu extend `UpdateKioskDto` DULU (keputusan tambahan, konfirmasi dulu ke user sebelum extend scope) — untuk sekarang, cukup dukung field yang backend SUDAH terima
- Submit mode edit → `PATCH /kiosks/:id` dengan payload `{ nama, allowedIp }`

### Gap 2 — Backend: `@LogActivity` Hilang di Endpoint `update`
`kiosks.controller.ts:58-63` — method `update()` TIDAK punya decorator `@LogActivity`, padahal `create`, `deactivate`, `rotateToken` di controller yang sama semuanya punya. Ini pelanggaran aturan permanen (audit T067, 2026-07-22) — WAJIB ditambahkan:
```typescript
@Patch(":id")
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(UserRole.super_admin)
@LogActivity({
  action: "kiosk.update",
  targetType: "kiosk",
  prismaModel: "kiosk",
  sensitiveFields: ["deviceToken"],
})
update(@Param("id", ParseIntPipe) id: number, @Body() dto: UpdateKioskDto) {
  return this.kiosksService.update(id, dto);
}
```

## JANGAN
- ❌ JANGAN duplikasi `KioskForm` jadi 2 komponen — satu komponen dual-mode, pola sama seperti T077 (`GuruForm`)
- ❌ JANGAN extend `UpdateKioskDto` untuk field `kampusId`/`tipe` tanpa konfirmasi eksplisit dulu ke user — di luar scope minimal task ini kalau ternyata tidak diminta
- ❌ JANGAN hapus mekanisme Deactivate/Rotate Token yang sudah ada — Edit ini TAMBAHAN, bukan pengganti alur nonaktifkan/rotate yang sudah benar untuk kasus device hilang/dicuri

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/kiosk/kiosk-view.tsx` — tombol Edit + `KioskForm` dual-mode
- **Modifikasi:** `apps/api/src/kiosks/kiosks.controller.ts` — tambah `@LogActivity` ke method `update`

## Acceptance Criteria
- [ ] Tabel Daftar Mesin punya tombol Edit di tiap baris
- [ ] Klik Edit → Dialog terbuka dengan `nama`+`allowedIp` ter-pre-fill
- [ ] Ubah `allowedIp` → Simpan → `PATCH /kiosks/:id` terpanggil, tabel ter-update tanpa reload
- [ ] `activity_log` mencatat aksi `kiosk.update` setelah edit berhasil (cek tabel `activity_log` langsung, bukan asumsi)
- [ ] Kasus nyata yang memicu task ini teratasi: admin bisa perbaiki `allowedIp` yang salah TANPA nonaktifkan+buat kiosk baru
