# T244 — Web+API: "Biodata Murid" Wali Kelas — Tampilkan Foto Siswa (Konsisten Menu Data Murid Admin)

## Depends on
Tidak ada. Independen — 3 lapis (backend select, tipe shared, frontend render).

## Objective
Halaman "Biodata Murid" (`/guru/wali-kelas/biodata`) tampilkan foto tiap siswa di daftar,
KONSISTEN dengan menu Data Murid admin (`/siswa`) yang sudah menampilkan foto.

## Konteks — Kondisi Kode Saat Ini (dikonfirmasi via riset 2026-08-25)

Bug ada di 3 lapis sekaligus, bukan cuma 1:

1. **Backend TIDAK query field foto sama sekali.** `JournalService.getKelasWaliSiswa()`
   (`apps/api/src/journal/journal.service.ts:302-313`):
   ```ts
   const students = await this.prisma.student.findMany({
     where: { OR: [{ kelasId }, { kelasTerakhirNama: kelas.nama }] },
     select: { id: true, nisn: true, nama: true, status: true, jenisKelamin: true },
     orderBy: { nama: "asc" },
   });
   ```
   `select` tidak menyertakan `foto` — field itu tidak pernah sampai ke response, terlepas
   apa pun yang dilakukan frontend.

2. **Tipe shared tidak punya field foto.** `KelasWaliSiswaRow`
   (`apps/web/src/lib/core-types.ts:920-926`):
   ```ts
   export interface KelasWaliSiswaRow {
     id: number;
     nisn: string;
     nama: string;
     status: PersonStatus;
     jenisKelamin: JenisKelamin | null;
   }
   ```
   Tidak ada `foto: string | null`.

3. **Frontend tidak render avatar sama sekali.** `DaftarSiswaTab`
   (`apps/web/src/app/(guru)/guru/wali-kelas/components/daftar-siswa-tab.tsx`) — kolom
   tabel cuma No/Nama/NISN/Status (baris 70-73), TIDAK ADA kolom/elemen foto di mana pun.

**Referensi pola yang SUDAH BENAR** (admin, `apps/web/src/app/(admin)/siswa/siswa-view.tsx`
baris 307, 777-791) — `StudentAvatar` component:
```tsx
function StudentAvatar({ foto, nama }: { foto: string | null; nama: string }) {
  if (!foto) {
    // fallback inisial nama (lihat kode asli untuk detail persis)
  }
  return (
    <img
      src={`/api/photo-proxy/students/${foto.split("/").pop()}`}
      ...
    />
  );
}
```
Pola URL foto SELALU lewat `/api/photo-proxy/students/<nama-file>` (bukan path langsung ke
storage) — konsisten proxy foto yang sudah dipakai di seluruh codebase (ADR-023).

## Spec Detail

1. **Backend** — tambah `foto: true` ke `select` di `getKelasWaliSiswa()`
   (`journal.service.ts:308`).
2. **Tipe shared** — tambah `foto: string | null;` ke `KelasWaliSiswaRow`
   (`core-types.ts:920-926`).
3. **Frontend** — `daftar-siswa-tab.tsx`: tambah kolom/elemen foto di tabel, REUSE
   `StudentAvatar` dari `siswa-view.tsx` KALAU bisa diimpor langsung (VERIFIKASI SAAT
   IMPLEMENTASI apakah komponen itu di-export dan aman diimpor lintas rute admin→guru, atau
   perlu di-extract ke lokasi shared seperti `components/` root — REKOMENDASI: extract ke
   `apps/web/src/components/student-avatar.tsx` kalau dipakai ≥2 tempat, supaya 1 sumber
   kebenaran, bukan duplikasi component identik).

## Edge Cases
- **Siswa tanpa foto** (`foto === null`) — tampilkan fallback yang SAMA dengan pola admin
  (biasanya inisial nama dalam avatar bulat) — JANGAN broken image icon.
- **Siswa nonaktif yang masih tampil** (kelasTerakhirNama match) — foto tetap ditampilkan
  kalau ada, tidak ada alasan disembunyikan.

## Files
- **Modifikasi:** `apps/api/src/journal/journal.service.ts` (`select` tambah `foto`).
- **Modifikasi:** `apps/web/src/lib/core-types.ts` (`KelasWaliSiswaRow` tambah `foto`).
- **Modifikasi:** `apps/web/src/app/(guru)/guru/wali-kelas/components/daftar-siswa-tab.tsx`
  (render avatar per baris).
- **Kemungkinan extract:** `StudentAvatar` dari `siswa-view.tsx` ke lokasi shared kalau
  dipakai lintas modul (lihat poin 3 Spec Detail).

## Acceptance Criteria
- [ ] Halaman Biodata Murid wali kelas tampilkan foto tiap siswa yang punya foto.
- [ ] Siswa tanpa foto tampil fallback konsisten pola admin (bukan broken image).
- [ ] Tidak ada duplikasi implementasi avatar kalau di-extract ke shared component.
- [ ] Build + type-check hijau (backend & frontend).

## Validasi Claudian
- [ ] Konfirmasi endpoint `/journal/kelas-wali-siswa` benar-benar mengembalikan `foto` di
      response setelah perubahan (test manual/curl, bukan cuma baca kode).
- [ ] Konfirmasi URL foto tetap lewat `/api/photo-proxy/students/...` (bukan path storage
      langsung) — konsisten ADR-023.
