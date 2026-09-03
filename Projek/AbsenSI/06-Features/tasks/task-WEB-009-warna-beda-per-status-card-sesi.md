# Task-WEB-009: Warna Berbeda per Status Card Sesi (Terapkan Token Semantic v2)

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah diskusi dengan user (referensi visual 8-status JurnalePro). Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** low
**Alasan pemilihan:** Murni penyesuaian styling (ganti className token warna), tidak ada logic baru. Perlu ketelitian memilih token semantic yang tepat per status supaya konsisten dengan Design System v2, bukan asal comot warna.

## 2. Konteks & Tujuan Utama

Referensi user: JurnalePro membedakan 8 status pelaksanaan KBM dengan warna badge berbeda (Izin=oranye, Sakit=merah, Diahiri=merah tua, Terlewati=merah terang, Terlaksana=hijau, Berikutnya=ungu, Saatnya KBM=cyan, Berlangsung=biru).

**PENTING — perbedaan cakupan status:** AbsenSI HANYA punya **5 status** (`belum_mulai`, `bisa_dimulai`, `sedang_berlangsung`, `selesai`, `sudah_diizinkan`) — BUKAN 8 seperti JurnalePro. JurnalePro punya status lebih granular (Izin/Sakit terpisah, Diahiri/Terlewati sebagai 2 kondisi gagal berbeda) karena sistem presensi guru mereka berbeda. **Task ini TIDAK menambah status baru** — murni memberi warna BERBEDA untuk 5 status yang SUDAH ADA, saat ini beberapa di antaranya masih terlihat serupa (`STATUS_BADGE` di `sesi-card.tsx` saat ini pakai `bg-surface-subtle` untuk 2 status berbeda: `belum_mulai` dan `selesai`).

**Keputusan user:** beri warna beda tiap status card, mengikuti SEMANGAT visual JurnalePro (warna sebagai sinyal cepat) — TAPI diadaptasi ke 5 status AbsenSI dan token Design System v2 yang SUDAH ADA (bukan warna baru di luar sistem).

## 3. Langkah Eksekusi Detail

1. Di `apps/web/src/app/(guru)/guru/jadwal/components/sesi-card.tsx`, redesain `STATUS_BADGE` map (baris 14-20) supaya **5 status punya kombinasi warna berbeda satu sama lain**, menggunakan token semantic v2 yang SUDAH TERSEDIA (cek `apps/web` — proyek sudah punya `success-bg`/`success-text`, `warning-bg`/`warning-text`, `danger-bg`/`danger-text`, `info-bg`/`info-text`, `primary-soft`, `primary-tint`, `surface-subtle` — semua sudah dipakai di kode existing lain, JANGAN buat token/hex baru):

   | Status | Makna | Token yang Diusulkan | Alasan |
   |---|---|---|---|
   | `belum_mulai` | Netral, belum waktunya | `bg-surface-subtle text-ink-secondary` | Netral, tidak perlu perhatian (SAMA seperti sekarang, tidak berubah) |
   | `bisa_dimulai` | **Aksi dibutuhkan sekarang** | `bg-warning-bg text-warning-text` | Kuning/amber = sinyal "perlu tindakan", beda dari netral, BEDA dari sedang_berlangsung |
   | `sedang_berlangsung` | Sedang aktif | `bg-primary-soft text-primary-hover` | TETAP sama seperti sekarang (sudah oranye spotlight, ini benar — status paling aktif memang seharusnya paling menonjol pakai brand color) |
   | `selesai` | Selesai, tidak perlu aksi | `bg-success-bg text-success-text` | Hijau = selesai/beres, BEDA dari belum_mulai (sekarang keduanya sama abu-abu, membingungkan) |
   | `sudah_diizinkan` | Diizinkan tidak mengajar | `bg-info-bg text-info-text` | Biru info = kondisi khusus/informasional, BEDA dari status lain |

   **Ini USULAN, bukan keputusan final** — kalau Claude Code menemukan token yang lebih pas secara kontras/aksesibilitas (cek `foundation-rules.json` kontras minimum WCAG AA di Design System v2), boleh disesuaikan SELAMA tetap dari palet token yang SUDAH ADA, bukan warna baru.

2. **Verifikasi kontras** — semua kombinasi bg+text WAJIB pass WCAG AA (4.5:1 teks normal) sesuai `06-Features/design-system-v2/03-fase-2-foundation-rules.md` bagian Kontrak Aksesibilitas — token semantic v2 SUDAH dirancang pass ini secara default (`success-text`/`warning-text`/dst sudah computed AA-safe), tapi tetap verifikasi kombinasi akhirnya bukan menduga.

3. **Pertimbangkan warna JUGA diterapkan ke border/left-accent card** (bukan cuma badge kecil) untuk efek visual yang lebih terasa "beda warna per status" seperti maksud JurnalePro (badge kecil saja mungkin kurang terasa) — mis. `border-l-4` dengan warna status di sisi kiri card. **Diskusikan dulu preferensi user** sebelum menerapkan border tebal (bisa terasa terlalu ramai kalau 5+ card berjajar) — implementasikan badge dulu sebagai MINIMUM viable, border sebagai enhancement opsional yang bisa ditambah kalau user minta setelah lihat hasil badge saja.

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/web/src/app/(guru)/guru/jadwal/components/sesi-card.tsx` (map `STATUS_BADGE`)

**Dilarang dilakukan:**
- Jangan tambah token warna baru di luar Design System v2 (tidak ada hex hardcode, tidak ada Tailwind color class default seperti `bg-blue-500` — SEMUA harus dari token semantic proyek).
- Jangan ubah teks label badge (Belum Waktunya/Bisa Dimulai/Sedang Berlangsung/Selesai/Diizinkan) — murni perubahan warna, bukan copy.
- Jangan tambah status baru (Izin/Sakit terpisah dsb) — di luar scope task ini, itu perubahan data model yang jauh lebih besar (perlu didiskusikan terpisah kalau memang diinginkan).

**Skenario kegagalan yang WAJIB ditangani:**
- Kombinasi warna baru harus TETAP readable di viewport mobile kecil (font badge sudah `text-caption`, kecil) — verifikasi tidak ada kombinasi warna yang nge-blur/susah dibaca di ukuran kecil.

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] 5 status punya kombinasi warna VISUAL BERBEDA satu sama lain (tidak ada 2 status share warna yang sama seperti sekarang belum_mulai+selesai)
- [ ] Semua token dari Design System v2 yang sudah ada, tidak ada hex/warna baru
- [ ] Kontras WCAG AA terpenuhi untuk semua kombinasi
- [ ] Label teks badge tidak berubah

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini (KECUALI pilihan border-left-accent yang eksplisit didiskusikan dulu ke user sebelum diterapkan)
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 30 baris perubahan)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada (single-accent-utama tetap dipegang untuk primary brand, feedback/status memang sudah dirancang multi-warna di v2)
- [ ] Dependency (jika ada) sudah selesai sebelum task ini di-assign — tidak ada dependency
