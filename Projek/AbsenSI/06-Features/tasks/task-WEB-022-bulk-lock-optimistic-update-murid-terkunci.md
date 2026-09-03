# Task-WEB-022: Fix Bulk Lock Tidak Update Section "Murid Terkunci" (Data Stale)

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah audit kejanggalan Dashboard Piket + diskusi kritis dengan user (2026-09-03). Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.

**Task Terbuat:** 2026-09-03
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** low
**Alasan pemilihan:** Pola solusinya SUDAH ADA di file yang sama (`handleStudentLocked` untuk lock individual) — tinggal direplikasi ke handler bulk, bukan desain baru.

## 2. Konteks & Tujuan Utama

Audit Dashboard Piket (2026-09-03) menemukan `handlePerluDitinjauLockedBulk` (`apps/web/src/app/(piket)/piket/piket-board-view.tsx` baris ~387-398) — handler untuk tombol "Kunci Semua" di section "Perlu Ditinjau" — TIDAK memperbarui state lokal `terkunci` (yang menggerakkan section/badge "Murid Terkunci") setelah aksi bulk lock berhasil. Comment kode sendiri mengakui trade-off ini secara eksplisit.

Bandingkan dengan `handleStudentLocked` (baris ~417-425), handler lock INDIVIDUAL, yang sudah benar melakukan **optimistic update**: prepend siswa yang baru dikunci ke state `terkunci` lokal, sehingga section "Murid Terkunci" & badge count-nya langsung update TANPA perlu reload.

**Dampak nyata**: setelah piket klik "Kunci Semua" dari section "Perlu Ditinjau", section & badge "Murid Terkunci" tetap menunjukkan angka/daftar LAMA sampai piket reload manual halaman — piket bisa salah kira siswa belum terkunci padahal sudah, atau sebaliknya bingung kenapa jumlahnya tidak nambah setelah aksi yang baru saja berhasil.

**Depends on:** Tidak ada.

## 3. Langkah Eksekusi Detail

1. Baca `handleStudentLocked` (baris ~417-425) di `piket-board-view.tsx` — pahami PERSIS pola optimistic update yang dipakai (bagaimana data siswa yang dikunci di-mapping jadi bentuk yang cocok untuk di-prepend ke state `terkunci`, field apa saja yang perlu diisi — `lockedReason`, `lockedAt`, dll).

2. Ubah `handlePerluDitinjauLockedBulk` — setelah API call bulk lock sukses (response berisi daftar siswa yang berhasil dikunci, VERIFIKASI SAAT IMPLEMENTASI bentuk response endpoint bulk-lock yang dipakai, apakah sudah mengembalikan detail siswa yang terkunci atau cuma count/id) — REPLIKASI pola mapping yang sama dari `handleStudentLocked`, prepend SEMUA siswa yang berhasil dikunci (bukan cuma 1) ke state `terkunci` lokal.

3. **Kalau response endpoint bulk-lock TIDAK mengembalikan detail lengkap yang dibutuhkan** (mis. hanya `{success: true, count: N}` tanpa data siswa) — dua opsi, PILIH salah satu saat implementasi berdasar mana yang lebih murah:
   - (a) ubah response backend bulk-lock supaya mengembalikan detail siswa yang terkunci (array `{id, nama, kelas, lockedReason, lockedAt}`) — REPLIKASI bentuk data yang sudah dipakai `handleStudentLocked` untuk 1 siswa.
   - (b) kalau ubah backend dianggap scope terlalu besar untuk fix kecil ini, ambil data siswa yang di-bulk-lock dari state `perluDitinjau`/`belumKembali` YANG SUDAH ADA di frontend SEBELUM aksi (piket mengklik "Kunci Semua" dari daftar itu, jadi datanya sudah ada di client) — bentuk objek `terkunci` baru dari situ + tambahkan field `lockedReason`/`lockedAt` hasil dari response API (minimal timestamp+alasan).
   Rekomendasi: opsi (b) lebih murah (tidak sentuh backend), VERIFIKASI SAAT IMPLEMENTASI apakah data yang tersedia di client sudah cukup lengkap untuk direpresentasikan di tabel "Murid Terkunci" tanpa field hilang.

4. Pastikan badge count "Murid Terkunci" (kalau ada, cek `SummaryCard`/badge di atas board) ikut re-render otomatis dari state `terkunci` yang sudah diupdate (biasanya otomatis kalau badge derive dari `terkunci.length`, VERIFIKASI SAAT IMPLEMENTASI).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/web/src/app/(piket)/piket/piket-board-view.tsx` — `handlePerluDitinjauLockedBulk`
- **Modifikasi (kalau pilih opsi (a) di langkah 3):** endpoint bulk-lock backend terkait di `attendance.service.ts`/`attendance.controller.ts` — HANYA kalau opsi (b) benar-benar tidak memungkinkan.
- **Jangan sentuh:** `handleStudentLocked` (referensi pola, tidak diubah), section "Perlu Ditinjau" sendiri (hanya callback setelah sukses yang diubah).

**Dilarang dilakukan:**
- Jangan ubah endpoint bulk-lock backend kecuali opsi (b) di langkah 3 terbukti tidak cukup — prioritaskan solusi frontend-only dulu.

**Skenario kegagalan yang WAJIB ditangani:**
- Kondisi: bulk-lock PARTIAL SUCCESS (sebagian siswa berhasil dikunci, sebagian gagal — mis. sudah terkunci duluan oleh proses lain) → Perilaku benar: HANYA siswa yang benar-benar berhasil dikunci yang di-prepend ke `terkunci`, siswa yang gagal TIDAK ikut ditambahkan (VERIFIKASI response API membedakan sukses/gagal per siswa).
- Kondisi: bulk-lock dari section "Perlu Ditinjau" mengunci siswa yang KEBETULAN juga baru saja di-fetch ulang lewat refresh manual/polling lain → Perilaku benar: tidak ada duplikat di state `terkunci` (cek `id` sebelum prepend, skip kalau sudah ada).

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Setelah klik "Kunci Semua" di section "Perlu Ditinjau" berhasil, section & badge "Murid Terkunci" LANGSUNG update tanpa perlu reload halaman.
- [ ] Data siswa yang ditampilkan di "Murid Terkunci" hasil bulk lock lengkap (nama, kelas, alasan, waktu dikunci) — tidak ada field kosong/undefined yang mencolok.
- [ ] Partial success ditangani benar (hanya yang sukses masuk state baru).
- [ ] Build + typecheck bersih.

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 100 baris perubahan)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency: tidak ada
