# Task-WEB-008: Accordion Card Sesi — Tombol Mulai Mengajar/Izin Muncul Saat Card Diklik

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah diskusi dengan user. Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** low
**Alasan pemilihan:** Perubahan interaksi UI murni (toggle expand/collapse state lokal), tidak ada perubahan API/data. Perlu ketelitian supaya tombol yang sudah ada (Mulai Mengajar/Izin, dengan disabled state + tooltip) tetap berfungsi identik setelah dipindah ke area collapsible.

## 2. Konteks & Tujuan Utama

Saat ini di `SesiCard` (`apps/web/src/app/(guru)/guru/jadwal/components/sesi-card.tsx`), tombol "Mulai Mengajar" dan "Izin" SELALU tampil di setiap card, membuat halaman Jadwal Mengajar Hari Ini terasa penuh/ramai kalau guru punya banyak sesi hari itu (terlihat di screenshot referensi user — 2 card langsung menampilkan 2 tombol besar masing-masing).

**Keputusan user:** ubah jadi **accordion** — card menampilkan ringkasan (kelas, mapel, jam, ruangan, badge status) dalam kondisi collapsed; tombol Mulai Mengajar + Izin BARU muncul setelah card DIKLIK (expand), supaya tampilan awal halaman lebih ringkas terutama saat banyak sesi.

## 3. Langkah Eksekusi Detail

1. Di `sesi-card.tsx`, tambahkan state lokal `const [expanded, setExpanded] = useState(false)`.
2. Bungkus BAGIAN HEADER card (kelas+mapel, jam, ruangan+kampus, badge status — baris 142-170 existing) dengan elemen yang bisa diklik (`<button type="button" onClick={() => setExpanded(e => !e)}>` atau `<div role="button" tabIndex={0} onClick=... onKeyDown={handle Enter/Space}>` — WAJIB accessible, bukan cuma `onClick` di `div` polos tanpa keyboard support).
3. Tambahkan indikator visual expand/collapse (ikon chevron, `ChevronDown` dari `lucide-react` yang SUDAH dipakai di komponen accordion lain proyek ini — lihat pola `kelas-accordion-section.tsx` baris 147-150 sebagai REFERENSI PERSIS: `className={cn("h-5 w-5 shrink-0 text-ink-tertiary transition-transform", isOpen && "rotate-180")}`).
4. Area tombol (baris 172-212 existing: div "Mulai Mengajar" + "Izin" + tooltip disabled reason) dipindah ke DALAM blok conditional `{expanded && (...)}`, TIDAK diubah isinya sama sekali — SEMUA logic existing (disabled reason, tooltip hover, aria-describedby) dipertahankan PERSIS, cuma dibungkus kondisional render.
5. **Auto-expand default untuk sesi yang RELEVAN** — pertimbangkan card dengan status `bisa_dimulai` atau `sedang_berlangsung` (sesi yang butuh aksi SEKARANG) default `expanded = true` saat pertama render, sisanya (`belum_mulai`, `selesai`, `sudah_diizinkan`) default collapsed. Ini mencegah guru harus klik dulu untuk sesi yang justru paling urgent. **Diskusikan dengan user kalau preferensinya beda** (mis. user mungkin mau SEMUA default collapsed tanpa pengecualian) — implementasikan usulan ini sebagai default masuk akal, tapi laporkan pilihan ini di ringkasan hasil kalau ternyata perlu dikoreksi.
6. **Error state existing** (`error && <p>...</p>`, baris 214) dan **QR Scanner overlay** (`showScanner && <QrScanner .../>`, baris 216-225) TETAP tampil TANPA syarat `expanded` — ini adalah hasil AKSI dari tombol Mulai Mengajar (yang hanya bisa diklik saat expanded), jadi urutan render alami sudah benar (baru muncul setelah expanded=true DAN tombol diklik), tidak perlu perubahan tambahan di sini.

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/web/src/app/(guru)/guru/jadwal/components/sesi-card.tsx`
- **Jangan sentuh:** `jadwal-view.tsx` (pemanggil `SesiCard`) — tidak perlu prop baru, state expand murni internal per-card.

**Dilarang dilakukan:**
- Jangan ubah LOGIC tombol (disabled reason, urutan validasi GPS→kamera, dsb) — MURNI perubahan kapan tombol itu VISIBLE, bukan bagaimana dia bekerja.
- Jangan hilangkan accessibility existing (aria-describedby, role="tooltip") saat memindah blok tombol ke dalam kondisional.

**Skenario kegagalan yang WAJIB ditangani:**
- Card diklik saat scanner (`showScanner`) sedang terbuka → JANGAN biarkan klik card menutup/collapse area yang sedang menampilkan scanner aktif — pertimbangkan disable toggle expand SELAMA `showScanner === true` (guru sedang di tengah alur scan QR, tidak boleh area itu collapse tiba-tiba).
- Keyboard-only user (tab ke card, tekan Enter/Space) → HARUS bisa expand/collapse, bukan cuma mouse click.

**Edge case:**
- Sesi dengan `sudahDiizinkan: true` (badge "Diizinkan") → expand tetap menampilkan tombol "Izin" (yang jadi "Lihat Tugas Titipan" kalau `tugasSudahDiisi`) — perilaku existing dipertahankan, cuma visibility yang berubah.

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Card collapsed menampilkan ringkasan (kelas, mapel, jam, ruangan, kampus, badge status) TANPA tombol aksi
- [ ] Klik card (atau keyboard Enter/Space) toggle expand, menampilkan tombol Mulai Mengajar + Izin dengan SEMUA logic existing utuh
- [ ] Ikon chevron menunjukkan state expand/collapse, konsisten pola accordion lain di proyek (`kelas-accordion-section.tsx`)
- [ ] Sesi `bisa_dimulai`/`sedang_berlangsung` default expanded, sisanya default collapsed
- [ ] Toggle expand di-disable selama `showScanner` aktif
- [ ] Accessible via keyboard (tab + Enter/Space)

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 100 baris perubahan)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency (jika ada) sudah selesai sebelum task ini di-assign — tidak ada dependency
