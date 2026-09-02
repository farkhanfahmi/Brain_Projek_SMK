# Task-CORE-002: Fix Silent-Fail Bentrok Guru di JadwalSlotService (Opsi B — Fail-Safe + Warning Eksplisit)

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah diskusi kritis dengan user. Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** medium
**Alasan pemilihan:** Perubahan response shape (breaking contract kecil) di endpoint yang dipakai real-time oleh FE (dropdown guru), plus perubahan UI untuk render badge warning baru. Butuh ketelitian supaya tidak mengubah behavior deteksi bentrok yang SUDAH benar (harus tetap conservative, cuma menambah kategori baru "tidak bisa dipastikan").

## 2. Konteks & Tujuan Utama

Audit menu Jadwal Pelajaran (sesi diskusi 2026-09-02) menemukan celah silent-fail di `JadwalSlotService.findBentrokConflict()` (`apps/api/src/jadwal-slot/jadwal-slot.service.ts`, sekitar baris 442-487).

**Masalah:** saat mengecek bentrok jadwal guru lintas Opsi Jadwal aktif, method ini memanggil `resolveJamForOpsiJadwal()` untuk tiap kandidat slot existing guru itu. Kalau resolve gagal (mis. `AlokasiWaktuSlot` terkait sudah dihapus/berubah di Opsi Jadwal lain), baris ini:
```ts
const jamCandidate = await this.resolveJamForOpsiJadwal(cSlot.opsiJadwalId, cSlot.hari, cSlot.jamKe);
if (!jamCandidate) continue;
```
**diam-diam melewati (skip)** kandidat itu — dianggap "tidak bentrok" padahal statusnya sebenarnya **tidak diketahui**. Konsekuensi: guru berpotensi ke-assign ganda ke 2 kelas pada jam yang sama tanpa terdeteksi sistem, kalau data Alokasi Waktu di salah satu Opsi Jadwal sedang tidak konsisten. Ini melanggar prinsip "pesan error non-generik / tidak silent fallback" yang sudah jadi aturan keras di proyek ini (lihat pola `logger.warn()` eksplisit yang sudah dipakai di `teaching-sessions.service.ts` untuk kasus serupa `resolveJamSesi()` return null).

**Keputusan user (Opsi B — fail-safe, bukan sekadar logging):** JANGAN anggap "tidak bentrok" saat resolve gagal. Tandai sebagai **kategori terpisah "tidak bisa dipastikan"**, JANGAN hard-block (guru tetap boleh dipilih — false-block juga bukan solusi), tapi tampilkan warning eksplisit di UI dropdown guru: *"⚠️ Jadwal lama guru ini tidak terbaca sempurna, cek manual"*.

User juga mengonfirmasi: **boleh perbaiki UI/UX kalau kurang baik** — jangan cuma tempel data mentah, desain badge yang jelas dan tidak mengganggu alur input existing.

**Depends on:** Tidak ada — independen dari task-WEB-001.

## 3. Langkah Eksekusi Detail

### Backend (`apps/api/src/jadwal-slot/`)

1. Di `jadwal-slot.service.ts`, ubah return type `findBentrokConflict()` (baris ~442-444) dari:
   ```ts
   Promise<{ teacherNama: string; jamKe: number; hari: number; kelasNama: string; opsiJadwalNama: string } | null>
   ```
   agar bisa membedakan 3 kondisi: **tidak bentrok** (`null`), **bentrok pasti** (object seperti sekarang), dan **tidak bisa dipastikan** (kondisi baru). Rekomendasi: ubah jadi return union type atau tambah field `pasti: boolean` di object konflik — pilih pendekatan yang paling minim breaking ke pemanggil lain (`ensureNoBentrok()` yang dipanggil `create()`/`update()`, HARUS tetap throw seperti sekarang untuk kasus bentrok pasti; untuk kasus "tidak bisa dipastikan", **JANGAN throw** — biarkan create/update tetap jalan, cukup dicatat).

2. Di titik `if (!jamCandidate) continue;` (baris ~471) — tambahkan:
   - `logger.warn()` eksplisit menyebutkan slot mana yang gagal diresolve (opsiJadwalId, hari, jamKe, teacherId) — untuk observability/debugging.
   - Kumpulkan kandidat ini ke daftar terpisah "tidak bisa dipastikan" alih-alih langsung `continue` tanpa jejak.

3. Update `cekKetersediaanGuru()` (baris ~303-340) — response shape ditambah field baru, MISALNYA:
   ```ts
   {
     teacherIdsTersedia: number[],
     teacherIdsBentrok: number[],
     bentrokDetail: [...],
     teacherIdsTidakBisaDipastikan: number[],       // BARU
     tidakBisaDipastikanDetail: [{ teacherId, alasan }], // BARU — alasan: mis. "Jadwal lama di Opsi Jadwal X tidak terbaca (Alokasi Waktu berubah/dihapus)"
   }
   ```
   Guru dalam kategori ini **TIDAK masuk** `teacherIdsBentrok` (tidak hard-block) tapi juga sebaiknya tidak murni `teacherIdsTersedia` polos — pertimbangkan tetap masukkan ke `teacherIdsTersedia` (secara teknis dia "tersedia untuk dipilih") DAN sekaligus ke `teacherIdsTidakBisaDipastikan` (union — dropdown baca kedua field untuk decide badge mana yang tampil). Diskusikan/tetapkan kontrak final ini secara konsisten sebelum ubah DTO.

4. Pastikan `ensureNoBentrok()` (dipanggil `create()`/`update()`) — kondisi "tidak bisa dipastikan" TIDAK melempar `ConflictException` (guru tetap boleh disimpan), tapi WAJIB tercatat di `activityLog` snapshot (`jadwal_slot.create`/`jadwal_slot.update`) sebagai metadata tambahan, misal field `bentrokTidakPasti: true` di snapshot — supaya kalau nanti ada masalah nyata, ada jejak audit bahwa slot ini dibuat dalam kondisi validasi bentrok tidak lengkap.

### Frontend (`apps/web/src/app/(admin)/jadwal-pelajaran/[opsiJadwalId]/components/`)

5. Cari komponen yang consume `cekKetersediaanGuru` (kemungkinan `guru-dropdown-realtime.tsx` — verifikasi dulu nama file & lokasi pemakaian API ini).

6. Tambahkan badge/indicator baru di dropdown guru untuk guru berstatus `tidakBisaDipastikan`:
   - Style: badge kuning/amber (token `status-shipped-bg`/`status-shipped-text` — KONSISTEN dengan warning banner yang SUDAH ada di `jadwal-pelajaran-view.tsx` baris 256-270, jangan bikin token warna baru).
   - Teks: **"⚠️ Jadwal lama guru ini tidak terbaca sempurna, cek manual"** (sesuai yang dikonfirmasi user) — atau versi lebih actionable kalau Hermes menilai perlu (mis. sebutkan Opsi Jadwal mana yang bermasalah, ambil dari `tidakBisaDipastikanDetail[].alasan`). Prioritaskan actionable message konsisten aturan proyek "pesan error/warning harus jelas, bukan generik".
   - Badge ini **TIDAK disable** opsi guru di dropdown (guru tetap bisa dipilih) — beda dari badge bentrok existing (badge merah, kemungkinan disable/warning keras).

7. Cek existing pattern badge bentrok merah di dropdown (`bentrokDetail`) sebagai referensi styling — badge warning baru harus visually berbeda tapi masih dalam 1 sistem desain yang sama (jangan bikin desain baru dari nol, ikuti pola yang sudah ada).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/api/src/jadwal-slot/jadwal-slot.service.ts` — `findBentrokConflict()`, `ensureNoBentrok()`, `cekKetersediaanGuru()`
- **Modifikasi:** DTO response terkait (`apps/api/src/jadwal-slot/dto/cek-ketersediaan-guru.dto.ts` kalau ada response DTO eksplisit, atau cek shared types `packages/types` — **kalau perubahan menyentuh `packages/types`, WAJIB stop dan minta konfirmasi user dulu** sesuai aturan template ini)
- **Modifikasi:** komponen FE dropdown guru realtime (lokasi persis dikonfirmasi saat eksekusi, kemungkinan `guru-dropdown-realtime.tsx`)
- **Jangan sentuh:** logic `ensureMingguSesuaiMode`, `ensureJamKeValid`, `ensureNoDuplikatSlot`, `ensureGuruTerdaftarMapel` — 4 validasi lain di file yang sama, di luar scope task ini.

**Dilarang dilakukan:**
- Jangan ubah behavior kasus bentrok PASTI (yang sudah bisa diresolve jamnya) — itu harus tetap throw `ConflictException` seperti sekarang, TIDAK BOLEH melonggarkan validasi yang sudah benar.
- Jangan hard-block guru kategori "tidak bisa dipastikan" — sesuai keputusan user, mereka tetap harus bisa dipilih.

**Skenario kegagalan yang WAJIB ditangani:**
- Kondisi: SEMUA kandidat slot guru gagal resolve (mis. AlokasiWaktu yang dipakai banyak Opsi Jadwal dihapus massal) → Perilaku yang benar: guru tetap masuk `teacherIdsTersedia` + flag `tidakBisaDipastikan`, TIDAK crash, TIDAK melempar 500.
- Kondisi: campuran — sebagian slot bentrok PASTI, sebagian tidak bisa dipastikan → guru itu harus tetap masuk `teacherIdsBentrok` (bentrok pasti menang/prioritas lebih tinggi dari sekadar warning ketidakpastian) — JANGAN downgrade bentrok pasti jadi cuma warning.
- Kondisi: perubahan response DTO breaking FE lain yang consume endpoint sama tapi tidak diupdate → grep semua pemanggil `cek-ketersediaan-guru` di FE sebelum ubah shape, pastikan semua consumer sudah disesuaikan dalam task ini juga (jangan pecah setengah).

**Edge case:**
- Guru yang statusnya "tidak bisa dipastikan" TAPI TIDAK PERNAH benar-benar bentrok (false positive dari sisi observability murni) → tetap tampilkan warning (fail-safe conservative sesuai keputusan user), bukan disembunyikan — biar admin yang putuskan cek manual atau tidak.

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] `resolveJamForOpsiJadwal()` gagal resolve → dicatat via `logger.warn()`, TIDAK lagi silent `continue` tanpa jejak
- [ ] Response `cekKetersediaanGuru` punya field baru untuk kategori "tidak bisa dipastikan" (guru tetap masuk tersedia, tidak hard-block)
- [ ] `ensureNoBentrok()` TIDAK throw untuk kasus "tidak bisa dipastikan" — hanya tercatat di activityLog snapshot
- [ ] Dropdown guru FE menampilkan badge warning (amber, token existing) untuk guru kategori ini, dengan pesan actionable
- [ ] Behavior bentrok PASTI (existing) TIDAK berubah — masih throw `ConflictException` seperti sebelumnya
- [ ] Tidak ada regresi di test existing `jadwal-slot.service.spec.ts` (kalau ada test untuk `cekKetersediaanGuru`/`ensureNoBentrok`, update sesuai kontrak baru)

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini (dicek ulang oleh Hermes sebelum handoff)
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 300 baris perubahan; kalau ternyata FE-nya kompleks, pertimbangkan pecah jadi task-CORE-002 (backend) + task-WEB-002 (FE) terpisah saat eksekusi)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada (cek pola `logger.warn()` fallback-aman lain di `teaching-sessions.service.ts` sebagai referensi konsistensi gaya)
- [ ] Dependency (jika ada) sudah selesai sebelum task ini di-assign
