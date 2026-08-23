---
tags: [template, design-system, meta]
updated: 2026-08-05
---

# TEMPLATE: Design System Reference

> Dokumen ini GENERIK — dipakai ulang di proyek apa pun, bukan spesifik 1 proyek.
> Isi tiap `___` di bawah, hapus baris yang tidak relevan untuk proyek Anda, lalu
> simpan hasilnya sebagai `06-Features/design-system/MASTER.md` (atau setara) di
> proyek yang bersangkutan. Setelah diisi, file itu jadi **sumber kebenaran desain**
> yang dibaca Claude Code sebelum menulis UI apa pun — makin detail & tidak ambigu
> tiap nilai di sini, makin konsisten hasil desain antar sesi/komponen/halaman.
>
> Prinsip: **tulis nilai eksak, bukan deskripsi kualitatif.** "Biru yang enak dilihat"
> tidak bisa dieksekusi konsisten; `#2563EB` bisa. Kalau ragu satu nilai, lebih baik
> pilih placeholder sementara daripada dibiarkan kosong — kosong = Claude akan
> menebak sendiri dan tebakan itu tidak akan konsisten antar sesi.

---

## 0. Ringkasan Produk (konteks yang mengarahkan semua keputusan di bawah)

- **Jenis produk**: ___ (mis. SaaS dashboard internal, e-commerce publik, aplikasi konsumen, landing page marketing, tool internal engineering)
- **Siapa penggunanya**: ___ (mis. staf internal terlatih vs publik awam — ini menentukan seberapa "aman"/konvensional desain harus)
- **Mood/kepribadian visual dalam 3 kata**: ___ (mis. "hangat, profesional, tenang" vs "berani, playful, energik")
- **Referensi visual acuan** (kalau ada): ___ (nama produk/screenshot/link yang jadi acuan gaya — mis. "gaya EzMart dashboard", "mirip Linear", "mirip Stripe Dashboard")
- **Dark mode dibutuhkan?**: ___ (ya/tidak/nanti — kalau ya, semua token warna di bawah butuh pasangan light+dark)

---

## 1. Warna

### 1.1 Warna Dasar (Surface & Background)
- Page background: `___` (hex) — ___ (alasan/mood, mis. "beige hangat, bukan putih polos")
- Card/panel surface: `___`
- Surface sekunder (mis. sidebar, header berbeda dari card): `___`
- Border/divider default: `___`
- Overlay/backdrop (modal, drawer): `___` (warna + opacity, mis. `rgba(0,0,0,0.4)`)

### 1.2 Warna Aksen
- **Aksen primer** (`--color-primary`): `___` — dipakai untuk: ___ (tombol utama, link aktif, ikon terpilih, dst — sebutkan eksplisit)
- Berapa banyak warna aksen diizinkan?: ___ (Rekomendasi: **1 aksen tunggal** kecuali ada alasan kuat — kalau lebih dari 1, sebutkan aksen ke-2/3 dan ATURAN kapan masing-masing dipakai, jangan biarkan ambigu)
- Varian kepekatan aksen (untuk state hover/soft-bg/disabled tanpa nambah hue baru): `___-hover`, `___-soft`, `___-tint` — isi tiap nilai hex-nya
- Warna aksen di atas background gelap vs terang (kalau ada dark mode): ___

### 1.3 Warna Status/Semantik
- Success — bg: `___`, text: `___`, border: `___`
- Danger/Error — bg: `___`, text: `___`, border: `___`
- Warning — bg: `___`, text: `___`, border: `___`
- Info — bg: `___`, text: `___`, border: `___`
- **Kontras WCAG dicek?**: ___ (target minimal AA 4.5:1 untuk teks normal, 3:1 untuk teks besar/UI komponen — kalau warna diubah nanti demi kontras, WAJIB update dokumen ini juga, bukan cuma kode)

### 1.4 Warna Kategorikal Tambahan (>2 kategori non-binary, mis. status workflow multi-state)
- Kapan boleh dipakai?: ___ (mis. "hanya di kolom status tabel data, TIDAK untuk chart/KPI/toggle")
- Daftar warna kategori (isi sebanyak dibutuhkan, minimal beri nama+hex+kapan dipakai):
  - Kategori 1: nama `___`, bg `___`, text `___`
  - Kategori 2: nama `___`, bg `___`, text `___`
  - Kategori 3: nama `___`, bg `___`, text `___`
- Untuk kategori TANPA makna status (mis. varian A/B, minggu ganjil/genap): pakai kepekatan aksen tunggal, atau hue terpisah? ___

### 1.5 Warna Teks
- Teks primer: `___`
- Teks sekunder: `___`
- Teks tersier/disabled: `___`
- Teks di atas warna aksen (mis. teks putih di tombol oranye): `___`
- Placeholder input: `___`
- Link (default/hover/visited kalau relevan): `___` / `___` / `___`

### 1.6 Larangan Eksplisit Warna
- Warna yang TIDAK BOLEH dipakai: ___ (mis. "tidak ada abu netral/biru dingin untuk background", "tidak ada gradient")
- Kondisi khusus yang boleh menyimpang (kalau ada): ___

---

## 2. Tipografi

- Font keluarga utama: `___` (nama font + fallback stack lengkap, mis. `"Plus Jakarta Sans", Inter, sans-serif`)
- Font untuk angka/data tabular (kalau beda dari body): `___`
- Font monospace (kode/UID/ID, kalau relevan): `___`
- Sumber font: ___ (self-host / Google Fonts via `next/font` / CDN — sebutkan metode loading supaya konsisten)

### Skala Ukuran (isi tiap step yang dipakai, hapus yang tidak)
| Nama token | Ukuran (px/rem) | Line-height | Font-weight | Dipakai untuk |
|---|---|---|---|---|
| Caption | `___` | `___` | `___` | ___ |
| Body kecil | `___` | `___` | `___` | ___ |
| Body default | `___` | `___` | `___` | ___ |
| Body besar | `___` | `___` | `___` | ___ |
| Heading kecil (H4-H6) | `___` | `___` | `___` | ___ |
| Heading sedang (H2-H3) | `___` | `___` | `___` | ___ |
| Heading besar (H1) | `___` | `___` | `___` | ___ |
| Angka hero/KPI | `___` | `___` | `___` | ___ |

- Batas minimum ukuran body text: `___`px (jangan pernah di bawah ini)
- Letter-spacing khusus (mis. untuk label kapital kecil): `___`
- Angka pakai tabular-nums?: ___ (ya/tidak — penting untuk tabel data biar rata)

---

## 3. Spacing & Layout

- Base unit spacing (grid dasar, mis. 4px atau 8px): `___`
- Skala spacing yang dipakai (kelipatan base unit): `___` (mis. `4, 8, 12, 16, 24, 32, 48, 64`)
- Max-width container halaman: `___`
- Padding standar card: `___`
- Padding standar section/page: `___`
- Gap standar antar elemen dalam grup (mis. tombol berdekatan): `___`
- Gap standar antar card/section: `___`
- Grid kolom (kalau pakai grid system eksplisit): `___` kolom, gutter `___`

### Breakpoints Responsif
| Nama | Lebar minimum | Catatan |
|---|---|---|
| Mobile | `___` | ___ |
| Tablet | `___` | ___ |
| Desktop | `___` | ___ |
| Desktop lebar | `___` | ___ |

- Sidebar: perilaku di mobile (drawer overlay / hidden+toggle / bottom nav)?: `___`
- Tabel data: perilaku di mobile (scroll horizontal / card-stack / kolom tersembunyi)?: `___`

---

## 4. Radius & Border

- Radius kecil (input, chip kecil): `___`px
- Radius sedang (button, badge non-pill): `___`px
- Radius besar (card utama): `___`px
- Radius pill (button/badge/avatar bulat penuh): `___`px atau `9999px`
- Radius minimum yang diizinkan di seluruh produk: `___`px (jangan pernah di bawah ini, kecuali: ___)
- Ketebalan border default: `___`px
- Warna border default: `___`
- Border dipakai sebagai pemisah utama antar section, atau selalu shadow/spacing?: `___`

---

## 5. Shadow & Elevation

Isi tiap level elevation yang dipakai (hapus yang tidak relevan):

| Level | Dipakai untuk | box-shadow (value lengkap) |
|---|---|---|
| Flat/none | ___ | `none` |
| Rendah (card default) | ___ | `___` |
| Sedang (card hover/dropdown) | ___ | `___` |
| Tinggi (modal/popover) | ___ | `___` |
| Tertinggi (toast/tooltip) | ___ | `___` |

- Warna dasar shadow: `___` (mis. warm-toned `rgba(23,20,18,...)` vs netral `rgba(0,0,0,...)`)
- Shadow berubah di dark mode?: `___`
- z-index scale (urutan layer, isi tiap layer yang dipakai): base `___`, sticky header `___`, dropdown `___`, modal backdrop `___`, modal `___`, toast `___`

---

## 6. Ikonografi

- Library ikon: `___` (mis. `lucide-react`, `heroicons`) — SATU library saja, jangan campur
- Ukuran ikon standar: `___`px (kecil/sedang/besar kalau ada beberapa varian)
- Stroke width ikon (kalau outline-style): `___`
- Warna ikon default vs ikon di dalam elemen aksen (mis. di tombol primer): `___` / `___`
- Emoji boleh dipakai sebagai ikon fungsional?: `___` (rekomendasi: TIDAK — emoji hanya untuk konten/chat, bukan kontrol UI)
- Icon chip/container (kalau ikon dibungkus kotak berwarna): ukuran `___`px, radius `___`px, bg `___`

---

## 7. Komponen — State Interaktif

Untuk TIAP komponen interaktif di bawah, isi state yang relevan. Ini bagian yang
paling sering terlupa — isi selengkap mungkin supaya tidak ada tebakan.

### Button (primer, sekunder, ghost/tertiary, destructive — duplikasi tabel ini per varian)
- Default: bg `___`, text `___`, border `___`
- Hover: bg `___`, transisi `___`
- Active/pressed: bg `___`
- Focus (keyboard nav): outline/ring `___`
- Disabled: bg `___`, text `___`, opacity `___`, cursor `___`
- Loading (kalau ada state ini): `___`
- Ukuran (height, padding horizontal, font-size) per varian sm/md/lg: `___`

### Input/Form field
- Default: border `___`, bg `___`
- Focus: border `___`, ring/shadow `___`
- Error: border `___`, teks pesan error warna `___`
- Disabled: bg `___`, opacity `___`
- Placeholder: warna `___`
- Height standar: `___`

### Badge/Chip/Tag
- Radius: `___` (biasanya pill)
- Padding: `___`
- Font-size: `___`
- Varian warna: ikuti section 1.3/1.4 di atas

### Table/Data grid
- Header row: bg `___`, font-weight `___`, teks warna `___`
- Row hover: bg `___`
- Row selected: bg `___`
- Border antar baris: `___` (garis tipis / spacing tanpa garis / zebra-stripe)
- Sort indicator: ikon `___`, posisi `___`
- Kolom "No" wajib di paling kiri untuk tabel data?: `___`
- Empty state (tabel tanpa data): `___`

### Modal/Dialog
- Max-width: `___`
- Radius: `___`
- Posisi: center / drawer kanan / drawer bawah (mobile)?: `___`
- Animasi masuk/keluar: `___`

### Toast/Notifikasi
- Posisi di layar: `___`
- Durasi auto-dismiss: `___`
- Varian warna per tipe (success/error/info): ikuti section 1.3

### Toggle/Switch
- Ukuran track: `___` x `___`
- Warna track on/off: `___` / `___`
- Warna thumb: `___`
- **Perhatikan bug umum**: posisi thumb saat off (`left`/`translate-x`) harus eksplisit, jangan andalkan default — catat nilai persis: off `___`, on `___`

### Navigasi/Sidebar
- Item aktif: bg `___`, text `___`, indikator (garis kiri/dot/dsb) `___`
- Item hover: bg `___`
- Grouping/accordion (kalau ada): ikon expand `___`, animasi `___`

---

## 8. Motion & Transisi

- Durasi transisi standar (hover, fade, dst): `___`ms
- Durasi transisi elemen besar (modal, drawer, page transition): `___`ms
- Easing function default: `___` (mis. `ease-out`, `cubic-bezier(...)`)
- Animasi loading/skeleton: `___`
- Prefers-reduced-motion dihormati?: `___` (rekomendasi: ya, matikan animasi non-esensial)

---

## 9. Data Visualization (chart/grafik, kalau relevan)

- Warna seri data: mengikuti aksen tunggal (kepekatan berbeda) atau palet kategorikal terpisah?: `___`
- Warna grid/axis line: `___`
- Warna label axis: `___`
- Style default (bar/line/area — radius bar, ketebalan garis): `___`
- Tooltip chart: bg `___`, radius `___`, shadow `___`

---

## 10. Anti-Pattern / Larangan Eksplisit (isi berdasarkan proyek ini, tambahkan bebas)

- ___ (mis. "tidak ada warna aksen kedua di luar yang didefinisikan")
- ___ (mis. "tidak ada radius di bawah Xpx")
- ___ (mis. "tidak ada emoji sebagai kontrol UI")
- ___ (mis. "tidak ada border tebal sebagai pemisah utama, pakai shadow/spacing")
- ___

---

## 11. Sumber Kebenaran & Sinkronisasi

- File hasil isian template ini disimpan di mana (path pasti)?: `___`
- Ada file "rangkuman" terpisah yang dibaca tool lain (mis. CLAUDE.md, DESIGN.md ringkas di root repo)?: `___` — kalau ya, tulis ATURAN eksplisit "kalau ada konflik, file mana yang menang" di kedua file itu.
- Kalau ada perubahan warna/token demi alasan teknis (mis. WCAG contrast fix) yang dilakukan LANGSUNG di kode tanpa lewat dokumen ini dulu — WAJIB update dokumen ini di commit yang sama. (Ini pola yang pernah terlewat di proyek AbsenSI: token warna digelapkan untuk WCAG AA tapi dokumen sumber tidak ikut diperbarui — cek ulang berkala apakah nilai di sini masih cocok dengan `globals.css`/token file aktual.)

---

## Cara Pakai Template Ini

1. Salin file ini ke proyek baru, isi semua `___`.
2. Hapus baris yang benar-benar tidak relevan untuk proyek itu (jangan biarkan `___` kosong tak terjawab — kosong akan ditebak oleh Claude, dan tebakan tidak konsisten antar sesi).
3. Simpan sebagai file sumber-kebenaran desain proyek (mis. `06-Features/design-system/MASTER.md`).
4. Tambahkan pointer wajib-baca ke file itu di `CLAUDE.md` root repo proyek (section khusus, bukan cuma disebut sambil lalu) — supaya terbaca otomatis tiap sesi sebelum kerja UI.
5. Kalau proyek juga punya file rangkuman terpisah untuk tool lain (mis. `DESIGN.md` untuk Impeccable), pastikan rangkuman itu disinkronkan tiap kali nilai di sini berubah, dan keduanya menyatakan eksplisit siapa yang "menang" kalau beda.
