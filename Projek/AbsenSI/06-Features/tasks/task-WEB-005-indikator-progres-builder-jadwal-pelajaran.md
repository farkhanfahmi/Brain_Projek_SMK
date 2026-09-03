# Task-WEB-005: Indikator Progres Pengisian Jadwal (Builder Jadwal Pelajaran)

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah diskusi kritis dengan user. Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** medium
**Alasan pemilihan:** Murni derived-state dari data yang SUDAH ada di client (`allSlots`, `alokasiWaktu.slots`) — tidak butuh API baru. Tapi logic "berapa slot yang SEHARUSNYA terisi" perlu ketelitian (beda mode Blok/Normal, exclude jam istirahat) supaya indikator tidak salah kasih sinyal ke admin.

## 2. Konteks & Tujuan Utama

Audit menu Jadwal Pelajaran (sesi diskusi 2026-09-02): builder `WorkspaceView` (`apps/web/src/app/(admin)/jadwal-pelajaran/[opsiJadwalId]/`) menampilkan accordion daftar SEMUA kelas/guru, tapi TIDAK ada indikator progres pengisian — admin harus buka satu-satu accordion untuk tahu kelas mana yang jadwalnya masih bolong. Untuk builder sekompleks ini (nested: Tab Kelas/Guru → accordion → Tab Hari → Tab Minggu A/B), ini menyulitkan admin memantau kelengkapan data di semester yang sedang disusun.

**Keputusan user:** implementasikan LANGSUNG kedua opsi yang didiskusikan:
1. **Badge status per-accordion-header** (KelasAccordionSection/GuruAccordionSection) — ringkasan "X/Y jam terisi" atau indikator warna per kelas/guru.
2. **Ringkasan agregat di level Workspace teratas** — mis. "8 dari 12 kelas sudah lengkap jadwalnya", ditampilkan di atas Tabs Kelas/Guru.

**Depends on:** Tidak ada.

## 3. Langkah Eksekusi Detail

### Langkah A — Definisikan "seharusnya terisi berapa slot" (WAJIB presisi, baca dulu sebelum coding)

1. **Pahami dulu variabel yang mempengaruhi "total slot seharusnya" per kelas:**
   - `hariAktifUrut` (hari yang DICENTANG admin di Checklist Hari, state global Workspace) — HANYA hari yang dicentang yang dihitung, BUKAN semua hari di Alokasi Waktu (kelas belum dianggap "kurang lengkap" untuk hari yang admin SENGAJA tidak centang untuk diisi sekarang).
   - Untuk TIAP hari aktif, jumlah `jamKe` yang tersedia di `alokasiWaktu.slots` untuk hari itu (`filter(s => s.hari === h && s.jamKe !== null)` — field `jamKe: null` adalah slot istirahat, HARUS di-exclude dari hitungan "seharusnya diisi mapel").
   - **Mode Blok**: kelas butuh entri TERPISAH untuk Minggu A DAN Minggu B (2x lipat) — total slot seharusnya = (jumlah jamKe per hari aktif) × 2. **Mode Normal**: cukup 1x (minggu selalu null).
   - `jamKeAkhirRentang` (T219, rentang jam yang di-expand jadi banyak baris `JadwalSlot`) — SATU rentang jam (mis. 1-3) menghasilkan BANYAK baris `JadwalSlot` (jamKe=1,2,3) untuk SATU sesi mengajar. Untuk hitungan "berapa jam terisi", hitung berdasarkan JUMLAH BARIS JadwalSlot yang match (bukan jumlah sesi unik) — supaya konsisten dengan "total slot tersedia" yang juga dihitung per-jamKe individual dari Alokasi Waktu, bukan per-sesi.

2. **Tulis fungsi hitung `hitungProgresKelas(kelasId, allSlots, alokasiWaktu, hariAktifUrut, isBlok): { terisi: number; total: number }`** (dan versi paralel untuk guru, `hitungProgresGuru`) — taruh di file util baru atau di dalam `WorkspaceView`/komponen accordion, sesuai konvensi kode existing di folder ini.

3. **Verifikasi definisi ini dengan cara BACA komentar T219 di `jadwal-slot.service.ts`** (sudah dibaca Hermes saat audit, rujuk ke situ) — pastikan pemahaman `jamKeAkhirRentang` benar sebelum implementasi, JANGAN asumsi tanpa cek ulang komentar itu.

### Langkah B — Badge per-accordion (Opsi 1)

4. Di `KelasAccordionSection` (`apps/web/src/app/(admin)/jadwal-pelajaran/[opsiJadwalId]/components/kelas-accordion-section.tsx`) — tambahkan badge kecil di header accordion (baris ~140-151, `headerLabel(kelas)`), render di sebelah `ChevronDown`:
   - Hitung `{ terisi, total } = hitungProgresKelas(kelas.id, allSlots, alokasiWaktu, hariAktifUrut, isBlok)`.
   - Tampilkan badge: `${terisi}/${total} jam` dengan warna indikator (hijau = `terisi === total && total > 0`, kuning/amber = `0 < terisi < total`, abu-abu = `terisi === 0`). Pakai token warna EXISTING (`success-bg`/`status-shipped-bg`/`surface-subtle`, KONSISTEN pola warna proyek, JANGAN token baru).
   - Kalau `total === 0` (mis. belum ada hari dicentang / Alokasi Waktu kosong) → JANGAN tampilkan badge menyesatkan "0/0" seolah lengkap, tampilkan badge netral/tidak tampil sama sekali dengan kondisi ini.

5. Terapkan pola SAMA di `GuruAccordionSection` (`guru-accordion-section.tsx`) — progres dihitung per-guru (total slot seharusnya untuk guru itu SECARA KONSEP beda dari kelas: guru bisa punya total slot custom tergantung berapa banyak dia mengajar, BUKAN dari 1 Alokasi Waktu tunggal seperti kelas). **Diskusikan/tentukan definisi "total slot seharusnya untuk 1 guru" sebelum implementasi** — kemungkinan besar TIDAK ADA angka pasti "guru ini seharusnya ngajar berapa jam" (beda dari kelas yang punya definisi jelas dari Alokasi Waktu) — pertimbangkan badge guru cukup MENAMPILKAN JUMLAH JAM YANG SUDAH TERISI SAJA (tanpa pembagi "total seharusnya"), BUKAN format X/Y seperti kelas. **Ini keputusan desain yang perlu dikonfirmasi user SEBELUM diimplementasikan** kalau berbeda dari asumsi Hermes di atas — laporkan pemahaman ini ke user saat mulai eksekusi, jangan diam-diam pilih salah satu tanpa konfirmasi kalau ternyata ambigu.

### Langkah C — Ringkasan agregat di Workspace (Opsi 2)

6. Di `WorkspaceView` (`workspace-view.tsx`), tambahkan ringkasan agregat DI ATAS Tabs Kelas/Guru (baris ~198 area, sebelum `<Tabs value={section}>`):
   ```
   "8 dari 12 kelas sudah lengkap jadwalnya"
   ```
   Dihitung dari agregasi `hitungProgresKelas()` untuk SEMUA `kelasScopeCocok`, tampil sebagai teks ringkas + mungkin progress bar kecil (opsional, sesuaikan dengan komponen UI yang sudah ada di `@absensi/ui`, jangan bikin komponen progress bar baru kalau belum ada — cek dulu library komponen existing).
   
   Untuk Section Guru, ringkasan agregat serupa TAPI sesuaikan dengan keputusan langkah 5 (kalau guru tidak punya "total seharusnya" pasti, ringkasan agregat guru bisa berupa "X guru sudah punya minimal 1 jadwal" alih-alih X/Y).

7. Ringkasan ini HARUS reaktif — update otomatis begitu `allSlots` berubah (setelah save accordion manapun), TIDAK perlu refresh manual. Karena `allSlots` sudah di-refetch otomatis lewat `refetchSlots()` existing tiap kali ada save, ringkasan re-kalkulasi otomatis kalau di-derive langsung dari `allSlots` (bukan cache terpisah).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/web/src/app/(admin)/jadwal-pelajaran/[opsiJadwalId]/components/kelas-accordion-section.tsx` — tambah badge progres per header
- **Modifikasi:** `apps/web/src/app/(admin)/jadwal-pelajaran/[opsiJadwalId]/components/guru-accordion-section.tsx` — tambah badge progres per header (definisi disesuaikan, lihat langkah 5)
- **Modifikasi:** `apps/web/src/app/(admin)/jadwal-pelajaran/[opsiJadwalId]/workspace-view.tsx` — tambah ringkasan agregat
- **Kemungkinan file baru:** util helper `hitungProgresKelas`/`hitungProgresGuru` — taruh di lokasi yang konsisten dengan konvensi util lain di folder ini (cek dulu apakah ada folder `lib/` lokal di sini atau reuse `apps/web/src/lib/`).
- **Jangan sentuh:** `tab-hari-input.tsx`, `guru-hari-input.tsx` (form input per hari) — di luar scope, cuma tampilan ringkasan progres yang berubah.

**Dilarang dilakukan:**
- Jangan hitung progres dengan asumsi SEMUA hari Alokasi Waktu (harus HANYA hari yang dicentang admin di Checklist Hari) — kalau salah, admin akan melihat kelas "belum lengkap" padahal dia memang belum berniat isi hari itu sekarang.
- Jangan buat komponen progress bar/badge baru dari nol kalau `@absensi/ui` sudah punya primitive yang bisa dipakai — cek dulu.

**Skenario kegagalan yang WAJIB ditangani:**
- Kondisi: admin ganti Checklist Hari (uncheck beberapa hari) SETELAH sebagian jadwal sudah terisi untuk hari yang di-uncheck itu → progres HARUS re-kalkulasi otomatis mengikuti `hariAktifUrut` terbaru (total jadi lebih kecil, badge kelas yang tadinya "5/10" bisa jadi "5/6" kalau beberapa hari di-uncheck) — JANGAN progres nyangkut di angka lama.
- Kondisi: mode Blok, kelas baru terisi Minggu A tapi Minggu B masih kosong sama sekali → badge harus mencerminkan total gabungan A+B (mis. "6/12" bukan "6/6" yang menyesatkan seolah lengkap padahal cuma 1 minggu).
- Kondisi: `alokasiWaktu.slots` kosong sama sekali (belum ada AlokasiWaktu diisi) → total = 0 untuk semua kelas, badge/ringkasan JANGAN tampil menyesatkan, tampilkan state kosong yang jelas atau sembunyikan indikator sama sekali (konsisten dengan warning existing "Alokasi Waktu ini belum punya slot jam pelajaran sama sekali").

**Edge case:**
- Kelas yang TIDAK termasuk `tingkatScopes` Opsi Jadwal (sudah difilter di `kelasScopeCocok`) — TIDAK dihitung sama sekali di ringkasan agregat, konsisten dengan filter existing.

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Tiap accordion Kelas menampilkan badge progres (mis. "X/Y jam") yang akurat terhadap hari aktif tercentang + mode Blok/Normal
- [ ] Tiap accordion Guru menampilkan indikator progres (format disesuaikan hasil langkah 5, dikonfirmasi ke user kalau ambigu)
- [ ] Ringkasan agregat tampil di level Workspace ("X dari Y kelas lengkap")
- [ ] Progres re-kalkulasi otomatis setelah save (tanpa refresh manual) DAN setelah ganti Checklist Hari
- [ ] Tidak ada sinyal menyesatkan (badge "lengkap" padahal ada jam kosong yang harusnya terisi, atau sebaliknya)

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini (dicek ulang oleh Hermes sebelum handoff) — KECUALI poin definisi progres guru (langkah 5) yang eksplisit perlu konfirmasi user saat eksekusi kalau interpretasi Hermes ternyata tidak sesuai maksud
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 300 baris perubahan; kalau logic hitung progres + 3 komponen UI ternyata lebih besar, pertimbangkan pecah jadi task terpisah per komponen saat eksekusi)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada (konsisten pola T219 jamKeAkhirRentang)
- [ ] Dependency (jika ada) sudah selesai sebelum task ini di-assign — tidak ada dependency
