# T167 — API: Fallback Byte-Reversal UID Saat Tap Gagal (Beda Algoritma Antar-Reader RFID)

## Depends on
Tidak ada dependency teknis. Independen, PRIORITAS TINGGI (akar masalah "kartu terdaftar tapi ditolak" yang berdampak banyak siswa/guru, ditemukan+diverifikasi matematis 2026-08-13).

## Objective
Kalau tap kartu GAGAL karena UID tidak ditemukan (`rejected_unknown`) — SEBELUM menyatakan kartu benar-benar tidak terdaftar, sistem WAJIB mencoba SEKALI LAGI dengan UID hasil **transformasi byte-reversal** — untuk menangani kasus kartu yang didaftarkan lewat 1 model reader RFID (byte order A) tapi di-tap sehari-hari lewat reader BERBEDA (byte order terbalik, model B) — TANPA perlu admin mendaftar ulang ribuan kartu yang sudah beredar.

## Context — Akar Masalah Ditemukan + DIVERIFIKASI MATEMATIS dengan Data Nyata (2026-08-13)

User (pemilik sistem) menemukan: 2 model reader RFID yang dipakai sekolah ini (1 di meja admin untuk registrasi, 1+ di kiosk gerbang untuk tap harian) **membaca UID kartu fisik yang SAMA dengan byte order TERBALIK** — bukan kerusakan hardware (diagnosis T165 sebelumnya soal "scanner rusak" TERNYATA salah kaprah — SEBENARNYA kedua reader itu SEHAT, cuma beda ALGORITMA/urutan byte output-nya).

**BUKTI NYATA (diberikan user, DIVERIFIKASI dengan perhitungan matematis presisi, BUKAN dugaan)**:
- Kartu fisik yang SAMA, di-tap di **Reader Admin**: UID desimal `0793501459` (hex `2F4BDF13`).
- Kartu fisik YANG SAMA, di-tap di **Reader Kiosk** (gerbang): UID desimal `0333400879` (hex `13DF4B2F`).
- Verifikasi: `2F4BDF13` dibalik byte-per-byte (`2F 4B DF 13` → `13 DF 4B 2F`) = **PERSIS** `13DF4B2F`. **Transformasi terbukti benar 100% secara matematis** (dihitung ulang dan dikonfirmasi, bukan asumsi).

**Algoritma transformasi yang TERVERIFIKASI BENAR** (PENTING: ini BUKAN operasi "balik urutan karakter string" — itu SALAH secara matematis dan TIDAK BOLEH diimplementasikan seperti itu):
```
1. Parse string UID desimal (yang tersimpan di Card.uid, misal "0793501459") → integer (793501459).
2. Konversi integer itu → 4 byte BIG-ENDIAN (struct.pack('>I', n) di Python, setara Buffer.writeUInt32BE di Node.js).
3. Balik URUTAN 4 BYTE itu (byte pertama jadi terakhir, dst — BUKAN bit, BUKAN karakter string).
4. Parse 4 byte hasil pembalikan itu KEMBALI jadi integer (big-endian).
5. Stringify integer hasil itu, PAD dengan leading-zero ke 10 digit (lebar tetap, konsisten format UID yang sudah ada di sistem — UID maksimum 32-bit unsigned adalah 4294967295, 10 digit).
```
**Contoh konkret TERVERIFIKASI** (dihitung ulang, hasilnya PERSIS cocok): `reverse_uid_bytes("0793501459")` menghasilkan `"0333400879"` — SAMA PERSIS dengan UID yang dibaca reader kiosk untuk kartu fisik yang sama.

## Spec Detail

### 1. Backend — helper fungsi transformasi (implementasi PERSIS sesuai algoritma terverifikasi di atas)

Buat fungsi baru, misal `reverseUidByteOrder(uid: string): string | null` di lokasi yang sesuai (`apps/api/src/attendance/` atau `apps/api/src/common/`, PUTUSKAN saat implementasi lokasi paling logis — kemungkinan `apps/api/src/common/` kalau nanti dipakai lebih dari 1 tempat):

```ts
function reverseUidByteOrder(uid: string): string | null {
  // WAJIB validasi input adalah string desimal murni dalam rentang 32-bit unsigned
  // SEBELUM parse — UID non-numerik (misal "TESTUID000", data uji yang SUDAH ADA
  // di database) TIDAK BOLEH diproses, harus return null (skip fallback untuk kasus ini).
  if (!/^\d+$/.test(uid)) return null;
  const n = BigInt(uid); // pakai BigInt untuk hindari masalah presisi angka besar di JS
  if (n < 0n || n > 0xffffffffn) return null; // di luar rentang 32-bit unsigned, bukan UID valid

  const buf = Buffer.alloc(4);
  buf.writeUInt32BE(Number(n), 0); // 4 byte big-endian
  const reversed = Buffer.from(buf).reverse(); // balik urutan BYTE (bukan bit/karakter)
  const reversedInt = reversed.readUInt32BE(0);

  return String(reversedInt).padStart(10, "0"); // pad 10 digit, konsisten format UID existing
}
```
**VERIFIKASI IMPLEMENTASI INI DENGAN DATA NYATA SEBELUM LANJUT**: `reverseUidByteOrder("0793501459")` HARUS menghasilkan PERSIS `"0333400879"` — kalau tidak cocok, ADA KESALAHAN di implementasi (kemungkinan besar endianness terbalik atau padding salah), JANGAN lanjut ke integrasi sebelum tes unit ini lulus 100%.

### 2. Integrasi ke `tap()` — titik PERSIS yang perlu diubah

`apps/api/src/attendance/attendance.service.ts`, method `tap()`, di blok pencarian kartu (sekitar baris ~120-131):
```ts
let card = await this.prisma.card.findUnique({ where: { uid: dto.uid }, include: {...} });

// FALLBACK BARU — kalau UID asli tidak ketemu, coba versi byte-reversal SEBELUM menyerah.
if (!card) {
  const reversedUid = reverseUidByteOrder(dto.uid);
  if (reversedUid) {
    card = await this.prisma.card.findUnique({ where: { uid: reversedUid }, include: {...} });
  }
}

if (!card) {
  await this.logTapEvent(dto, kioskId, TapResult.rejected_unknown, null, period);
  return { result: TapResult.rejected_unknown, message: "Kartu tidak terdaftar" };
}
```
- **QUERY KEDUA hanya dijalankan KALAU query pertama gagal** (`card === null`) — TIDAK ADA overhead performa untuk kasus normal (mayoritas tap, UID langsung cocok).
- **KEPUTUSAN WAJIB DIPUTUSKAN**: `tap_events.uid` (kolom forensik, insert-only) dan `isDebounced(uid)` (baris ~748-751) — dicatat pakai UID **ASLI dari reader** (mentah, `dto.uid`, sebelum reversal) ATAU UID **hasil-reversal yang berhasil match**? REKOMENDASI: catat UID ASLI (`dto.uid`) di `tap_events` untuk keperluan FORENSIK/AUDIT (supaya kalau nanti perlu investigasi lagi, jelas persis apa yang dikirim reader saat itu) — TAPI `AttendanceRecord`/proses selanjutnya HARUS pakai `card` yang SUDAH DITEMUKAN (baik dari UID asli maupun hasil reversal, keduanya merujuk ke `Card` yang SAMA di database). Debounce (`isDebounced`) SEBAIKNYA tetap pakai `dto.uid` MENTAH (bukan hasil reversal) — supaya debounce tetap konsisten per-STRING-YANG-DITERIMA, bukan per-kartu-fisik (edge case: kalau reader A dan reader B kebetulan tap kartu yang SAMA dalam window 30 detik dari 2 device BERBEDA, debounce TIDAK PERLU menganggapnya duplikat karena secara teknis itu 2 STRING UID berbeda yang diterima — TAPI INI KEPUTUSAN YANG BISA DIDISKUSIKAN, evaluasi saat implementasi apakah masuk akal atau perlu direvisi).

### 3. TIDAK PERLU migrasi/backfill data — kartu yang SUDAH TERDAFTAR TIDAK PERLU diubah UID-nya

Ini KEUNGGULAN UTAMA pendekatan fallback dibanding "standarisasi ulang semua UID" — kartu yang SUDAH terdaftar (baik dengan UID dari Reader A maupun Reader B, SUDAH TERLANJUR TERCAMPUR di database SAAT INI) **TETAP BISA DIPAKAI TAP dari REDER MANAPUN** setelah fix ini, TANPA perlu identifikasi/migrasi data satu-satu. Fallback ini WORK UNTUK KEDUA ARAH (UID tersimpan dari Reader A, di-tap dari Reader B ATAU SEBALIKNYA — transformasi byte-reversal BERSIFAT SIMETRIS, membalik dua kali kembali ke asal).

## Edge Cases
- UID yang secara KEBETULAN, setelah di-reversal, MATCH ke kartu MILIK ORANG LAIN (bukan kartu fisik yang sebenarnya di-tap) — SECARA MATEMATIS SANGAT KECIL kemungkinannya (perlu 2 UID 32-bit acak yang PERSIS jadi reversal satu sama lain DAN keduanya terdaftar sebagai kartu berbeda — praktis mustahil untuk skala ribuan kartu), TAPI TIDAK BISA 100% dieliminasi secara teori — TIDAK PERLU penanganan khusus untuk risiko yang secara praktis dapat diabaikan ini, KECUALI user secara eksplisit minta pengamanan tambahan (misal logging khusus setiap kali fallback ini BERHASIL match, untuk audit trail bisa ditinjau manual kalau ada kejanggalan — PERTIMBANGKAN menambahkan log ini, TIDAK WAJIB tapi berguna untuk transparansi).
- UID yang bukan 10-digit angka murni (misal UID kartu jenis lain, atau data uji seperti `"TESTUID000"` yang SUDAH ADA di database) — fungsi `reverseUidByteOrder()` WAJIB return `null` untuk kasus ini (SUDAH dicakup validasi regex di poin 1), TIDAK BOLEH crash/exception.
- Kartu yang SUDAH dicabut/nonaktif (`status: inactive`) — fallback INI TIDAK MENGUBAH LOGIC status kartu SAMA SEKALI, kalau UID hasil reversal ditemukan tapi kartunya `inactive`, hasil tap TETAP `rejected_inactive` (BUKAN `rejected_unknown`) — SESUAI perilaku normal yang sudah benar untuk kartu nonaktif, task ini TIDAK mengubah itu.

## Files
- **Buat:** fungsi helper `reverseUidByteOrder()` (lokasi diputuskan saat implementasi) + unit test WAJIB yang memverifikasi transformasi `"0793501459"` → `"0333400879"` PERSIS (data nyata terverifikasi dari user).
- **Modifikasi:** `apps/api/src/attendance/attendance.service.ts` (`tap()`, tambah query fallback).
- **Jangan sentuh:** logic validasi status kartu (`active`/`inactive`/lock), logic debounce/duplicate — TIDAK diubah, task ini MURNI menambah 1 percobaan lookup TAMBAHAN sebelum menyerah, bukan mengubah aturan bisnis lain.

## Acceptance Criteria
- [x] Unit test: `reverseUidByteOrder("0793501459")` menghasilkan PERSIS `"0333400879"` (data nyata terverifikasi) — WAJIB LULUS sebelum lanjut. LULUS (juga diverifikasi independen via node -e SEBELUM implementasi ditulis).
- [x] Kartu yang terdaftar dengan UID dari Reader A, di-tap dari Reader B (byte order terbalik) → BERHASIL diterima (`accepted`), BUKAN `rejected_unknown` — verified live dengan kartu produksi nyata (card id 360, uid `0794307971`, tap via `2200786991` → accepted).
- [x] Kartu yang MEMANG tidak terdaftar (UID asli DAN UID hasil-reversal keduanya tidak ada di database) → TETAP `rejected_unknown` seperti biasa — verified live.
- [x] Kartu yang statusnya `inactive` — tap (baik via UID asli maupun hasil reversal yang match) TETAP menghasilkan `rejected_inactive`, BUKAN `rejected_unknown` — verified live (card id 437 di-flip inactive sementara, tap via uid reversal → rejected_inactive, DIREVERT setelah tes).
- [x] Performa: TIDAK ADA query tambahan untuk tap yang BERHASIL di percobaan PERTAMA — fallback dalam blok `if (!card)`, hanya jalan kalau lookup pertama `null`.
- [x] `TapEvent.uid` mencatat UID YANG BENAR-BENAR DIKIRIM reader (mentah) — TIDAK diubah sama sekali (`logTapEvent` sudah pakai `dto.uid` sejak awal, tidak perlu perubahan), verified live (`tap_events.uid = 2200786991`, `card_id = 360`, keduanya benar).
- [x] Build + type-check `apps/api` hijau. Test suite existing lulus 100% (273/273, 265 lama + 8 baru).

## Validasi Claudian
- [x] **WAJIB tes unit fungsi `reverseUidByteOrder()` dengan DATA NYATA** — dilakukan 2x: sekali via `node -e` MURNI SEBELUM menulis file implementasi (memverifikasi algoritma itu sendiri benar), sekali lagi via Jest SETELAH file dibuat (memverifikasi implementasi = algoritma yang sudah divalidasi).
- [x] **JANGAN implementasikan sebagai "balik urutan karakter string"** — TIDAK, implementasi PERSIS ikuti algoritma desimal→BigInt→4-byte-big-endian→Buffer.reverse()→parse ulang→pad 10 digit, sama persis kode di spec.
- [x] BigInt/Buffer dipakai sesuai contoh — TIDAK ADA operasi bitwise langsung pada `number`.
- [x] Keputusan `tap_events.uid`/debounce — TIDAK PERLU diubah sama sekali: `logTapEvent()` dan `isDebounced()` SUDAH pakai `dto.uid` mentah sejak sebelum task ini (dikonfirmasi baca kode existing), variabel `card` (bukan `dto.uid`) yang diganti jadi `let` untuk menampung hasil query kedua — TIDAK ada keputusan baru yang perlu diambil, perilaku forensik existing otomatis sudah benar.

## Status Eksekusi (2026-08-13)

**Selesai.** Helper + integrasi + test + verifikasi live dengan kartu produksi NYATA (bukan data sintetis), semua hijau.

**File baru**:
- `apps/api/src/common/uid-byte-reversal.ts` — `reverseUidByteOrder(uid: string): string | null`, implementasi PERSIS sesuai algoritma di spec (regex digit-murni → BigInt → `Buffer.writeUInt32BE` → `Buffer.reverse()` → `readUInt32BE` → pad 10 digit).
- `apps/api/src/common/uid-byte-reversal.spec.ts` — 8 test: data nyata terverifikasi (kasus WAJIB), simetri 2x-aplikasi, UID non-numerik → null, string kosong → null, di luar rentang 32-bit → null, UID negatif → null, leading-zero diproses benar, UID maksimum 32-bit tidak exception.

**Integrasi (`attendance.service.ts`, `tap()`)**:
- `card` diubah dari `const` jadi `let` (satu-satunya perubahan struktural) + `cardInclude` diekstrak jadi konstanta (dipakai 2x, query pertama dan fallback, hindari duplikasi objek include).
- Query fallback HANYA jalan di dalam blok `if (!card)` — hasil `reverseUidByteOrder(dto.uid)` di-null-check dulu sebelum query kedua (kalau UID bukan digit murni/di luar rentang, fallback di-skip, langsung `rejected_unknown` seperti sebelumnya).
- `logTapEvent`/`isDebounced` TIDAK disentuh — sudah pakai `dto.uid` mentah sejak awal (dikonfirmasi baca kode), memenuhi keputusan forensik spec tanpa perlu perubahan tambahan.

**Verifikasi live end-to-end** (dev DB port 3307, production tidak disentuh, dengan KARTU PRODUKSI NYATA yang sudah ter-sync ke dev — bukan data buatan):
1. Card id 360 (uid `0794307971`, siswa "ANGGI ASTARI AMANDA PUTRI", aktif) — tap via `POST /attendance/tap` dengan uid `2200786991` (hasil reversal, dihitung independen) → `{"result":"accepted", "name":"ANGGI ASTARI AMANDA PUTRI", ...}`. Fallback BERHASIL menemukan kartu yang benar.
2. `tap_events` row untuk tap itu: `uid=2200786991` (mentah, PERSIS yang dikirim), `card_id=360` (resolusi BENAR ke kartu asli) — forensik akurat, tidak tercampur.
3. UID `1111111111` (reversalnya `3342154306`, keduanya dikonfirmasi TIDAK ADA di database) → tap tetap `rejected_unknown` — regresi nol untuk kartu benar-benar tidak terdaftar.
4. Card id 437 (uid `3537746904`) DI-FLIP SEMENTARA jadi `status='inactive'` → tap via uid reversal (`3636190674`) → `{"result":"rejected_inactive", "message":"Kartu tidak aktif"}` — fallback menemukan kartu, TAPI status inactive tetap dihormati, TIDAK jatuh ke `rejected_unknown`. Card 437 DIKEMBALIKAN ke `active` setelah tes.
5. Semua data uji (3 baris `tap_events`, 1 baris `attendance_records`) dihapus setelah verifikasi. `allowed_ip` kiosk test dikembalikan ke `10.10.10.103`.
6. `tsc --noEmit` bersih. Jest 273/273 pass (265 existing + 8 baru untuk `reverseUidByteOrder`), TIDAK ADA regresi di test existing manapun.
