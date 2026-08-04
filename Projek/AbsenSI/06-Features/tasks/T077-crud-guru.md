# T077 — UI: Lengkapi CRUD Guru (Edit, Detail, Nonaktifkan)

## Depends on
Tidak ada — API sudah lengkap (`teachers.controller.ts` sudah punya `PATCH /teachers/:id` dengan `@Roles(super_admin, card_admin)`), murni pekerjaan UI.

## Context
- **App:** `apps/web`
- **File:** `apps/web/src/app/(admin)/guru/guru-view.tsx`
- **Ref:** Ditemukan 2026-07-24 — halaman `/guru` hanya punya Create (`GuruForm` di Sheet+Tabs) dan tabel read-only (NIY, Nama, Status). Tidak ada cara edit data guru atau ubah status dari UI meski API-nya sudah ada.

## Spec Detail

### Masalah
`guru-view.tsx:100-129` — tabel hanya render 3 kolom tanpa kolom Aksi. Tidak ada tombol Edit di baris manapun, tidak ada halaman/dialog detail.

### Solusi
1. **Tambah kolom Aksi** di tabel (`guru-view.tsx`) — 1 tombol icon "Edit" (pensil, pola sama seperti `kartu-view.tsx` yang sudah punya tombol icon Repeat/XCircle di kolom Aksi).
2. **Reuse `GuruForm`** yang sudah ada — extract jadi mode `create`/`edit`:
   - Tambahkan prop `mode: "create" | "edit"`, `initialData?: Teacher`
   - Saat `mode="edit"`: pre-fill semua field dari `initialData`, submit ke `PATCH /teachers/:id` bukan `POST /teachers`
   - Field NIY tetap bisa diedit (tidak ada constraint di skema yang melarang, tapi TAMPILKAN warning kecil di bawah field NIY saat edit: "Mengubah NIY tidak memengaruhi riwayat presensi yang sudah tercatat" — supaya admin sadar konsekuensinya)
3. **Status guru** — sudah ada field `status` (aktif/nonaktif/cuti) di form yang sama, jadi "nonaktifkan guru" cukup lewat Edit form ubah status ke `nonaktif`, TIDAK perlu tombol/dialog terpisah.
4. **JANGAN buat endpoint DELETE** — tidak ada di controller, dan `PersonStatus.nonaktif` sudah cukup mewakili "guru keluar/nonaktif" tanpa hapus baris (histori presensi guru terhubung by FK, hapus akan merusak riwayat).

### UI Detail
- Kolom Aksi baru di kanan tabel (setelah kolom Status), lebar `w-16`, isi 1 tombol icon `Pencil` (lucide-react) — pola identik dengan tombol icon di `kartu-view.tsx:200-217` (`h-11 w-11 rounded-full`, `hover:bg-surface-subtle`)
- Klik Edit → buka `Sheet` yang sama (reuse `GuruForm` dalam mode edit), title berubah jadi "Edit Guru"
- Setelah submit sukses → update state lokal `teachers` (replace item by id), tutup Sheet

## JANGAN
- ❌ JANGAN buat endpoint atau tombol DELETE — gunakan status `nonaktif`
- ❌ JANGAN buat halaman detail terpisah `/guru/[id]` — cukup Edit via Sheet, konsisten dengan pola form guru yang sudah ada (tidak seperti siswa yang punya halaman detail dengan riwayat)
- ❌ JANGAN duplikasi `GuruForm` jadi 2 komponen berbeda (Create/Edit) — satu komponen dengan prop `mode`

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/guru/guru-view.tsx` — tambah kolom Aksi, refactor `GuruForm` jadi dual-mode

## Acceptance Criteria
- [ ] Tabel Guru punya kolom Aksi dengan tombol Edit di tiap baris
- [ ] Klik Edit → Sheet terbuka dengan semua field ter-pre-fill dari data guru tsb
- [ ] Ubah nama/status/dll → Simpan → `PATCH /teachers/:id` terpanggil, tabel ter-update tanpa reload halaman
- [ ] Ubah status ke "Nonaktif" via Edit → guru tsb tampil dengan `StatusBadge` neutral "Nonaktif" di tabel
