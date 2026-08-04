---
tags: [workflow, claude-code, meta]
updated: 2026-08-04
---

# Cara Claude Code Mengelola Pengetahuan — dan Cara Anda Bekerja Dengannya

> Dokumen ini GENERIK — tidak spesifik ke proyek AbsenSI. Ditulis 2026-08-04 setelah
> diskusi panjang soal miskonsepsi "vault Obsidian = memori otomatis Claude Code" dan
> "kapan pengetahuan Claude hilang". Baca ini sekali di proyek baru mana pun, lalu
> workflow Anda tidak perlu berubah dari proyek ke proyek.

---

## 1. Tiga lapis pengetahuan Claude Code — dan pengaruhnya

| Lapis | Contoh | Persisten? | Kapan dibaca |
|---|---|---|---|
| **Context window** (sesi aktif) | Percakapan yang sedang berjalan | TIDAK — hilang total begitu sesi ditutup/dibuka baru | Selalu, selama sesi masih hidup |
| **Memory** (`~/.claude/projects/.../memory/`) | Preferensi Anda, feedback yang dikoreksi, keputusan proyek yang berubah-ubah | YA, ditulis oleh Claude sendiri | `MEMORY.md` (index) otomatis tiap sesi baru; detail per topik dibaca kalau relevan |
| **File di disk** (CLAUDE.md, vault, kode) | Aturan wajib, spec desain, kode aktual | YA, Anda/Claude yang tulis manual | CLAUDE.md project: otomatis tiap sesi. Vault/dokumen lain: HANYA kalau Claude memanggil Read tool pada file itu |

**Miskonsepsi paling umum**: mengira vault Obsidian (atau dokumen mana pun di luar CLAUDE.md) otomatis "diketahui" Claude tanpa perlu dibaca. Ini SALAH — Claude tidak punya indexing/semantic-search otomatis atas isi vault. Ia baru tahu isi sebuah file setelah benar-benar memanggil Read tool pada file itu, sama seperti Anda baru tahu isi buku setelah membukanya, bukan hanya karena buku itu ada di rak Anda.

---

## 2. Kapan pengetahuan (context sesi) benar-benar hilang

**BUKAN karena waktu** (bukan ganti hari/minggu). Tiga pemicu nyata:

1. **Sesi ditutup / window baru dibuka** — context sebelumnya hilang total kecuali tersimpan di memory/file. Ini paling sering terjadi kalau Anda pakai beberapa terminal/tab paralel — masing-masing context terpisah sejak awal, bukan "ter-reset".
2. **Auto-compaction** — dalam 1 sesi yang SAMA (belum ditutup), kalau percakapan sudah sangat panjang, sistem otomatis meringkas bagian awal. Ini SELALU terlihat jelas di layar Anda (blok ringkasan eksplisit muncul) — tidak pernah terjadi diam-diam tanpa jejak.
3. **`/clear` atau sesi baru eksplisit** — nol pengetahuan tentang percakapan sebelumnya. Hanya CLAUDE.md dan `MEMORY.md` yang otomatis kembali terbaca.

**Implikasi praktis**: keputusan penting yang dibuat dalam diskusi panjang **harus ditulis ke file (memory/vault) sesegera mungkin setelah disepakati** — jangan menunggu sampai akhir sesi. Kalau compaction terjadi sebelum keputusan itu tersimpan ke file, detailnya bisa terpangkas di ringkasan otomatis.

---

## 3. CLAUDE.md vs vault/dokumen lain vs memory — kapan pakai yang mana

Prinsip pemilahan: **bukan soal mana yang "lebih penting", tapi seberapa sering informasi itu perlu ada di context.**

- **CLAUDE.md project** — taruh HANYA aturan yang (a) wajib dipatuhi TANPA KECUALI, dan (b) jarang berubah. Contoh: aturan database insert-only, role permission, stack teknis. File ini dibaca 100% di setiap sesi tanpa perlu diminta — jadi harus tetap KECIL (idealnya di bawah 200 baris). Kalau CLAUDE.md membengkak, setiap sesi baru langsung memotong budget context Anda sebelum percakapan dimulai.
- **Vault/dokumen desain** — spec panjang, histori keputusan, detail task. Tidak otomatis dibaca — CLAUDE.md WAJIB punya pointer tajam ke sana (nama file eksplisit, bukan "lihat dokumentasi") supaya Claude tahu kapan harus pergi membaca lebih detail.
- **Memory Claude** — preferensi kerja Anda dan pola yang Claude amati sendiri, BUKAN aturan sistem/kode. Contoh: "user selalu minta dry-run sebelum eksekusi", "user lebih suka jawaban singkat". `MEMORY.md` (index-nya) kecil dan otomatis terbaca; detail per topik dibaca sesuai relevansi.

**Tidak ada informasi yang "dikorbankan"** dengan skema ini — semuanya tetap ada, cuma beda KAPAN dibaca. Analoginya: CLAUDE.md itu buku pegangan wajib di meja kerja, vault itu rak referensi di belakang — isinya sama lengkap, cuma Anda tidak bawa seluruh rak tiap kali kerja.

---

## 4. Menjembatani beberapa sesi paralel (misal: 1 sesi diskusi + N sesi eksekusi)

Kalau Anda membuka beberapa sesi Claude Code sekaligus (misal 1 untuk diskusi/perencanaan, lainnya untuk domain kerja terpisah seperti frontend/backend) — sesi-sesi itu **context-nya terisolasi total satu sama lain**. Tidak ada mekanisme otomatis yang membuat sesi B tahu apa yang diputuskan di sesi A.

Pola yang bekerja:
1. Sesi diskusi jadi tempat keputusan besar dibuat DAN ditulis ke file (memory + dokumen proyek) **segera setelah disepakati**, bukan ditunda.
2. Sesi kerja lain, sebelum mulai, membaca `MEMORY.md` (otomatis) + dokumen status proyek (misal `STATUS.md` di masing-masing proyek) untuk tahu apa yang baru diputuskan.
3. Kalau proyek punya tool knowledge-graph atas kode (misal skill semacam "graphify"), query itu dulu untuk pertanyaan arsitektur sebelum grep manual — jauh lebih hemat token daripada baca file mentah satu-satu.

---

## 5. Struktur dokumen proyek yang disarankan (hasil evaluasi 2026-08-04)

Per proyek, idealnya ada:

- **`CLAUDE.md`** (di root repo kode) — minimal, aturan wajib + pointer.
- **1 file `STATUS.md` hidup** — satu-satunya tempat cek "apa yang sudah selesai, apa yang belum". BUKAN banyak file status/backlog/task-tracker terpisah yang mudah tidak-sinkron satu sama lain.
- **Task individual tetap 1 file per task** (granularitas per-task memang pantas terpisah) — tapi begitu selesai, cukup ditandai selesai di STATUS.md, detail implementasi lengkapnya boleh tetap di file task itu (tidak perlu diulang di STATUS.md).
- **Folder `_archive/`** — dokumen yang sudah usang/digantikan dipindah ke sini, TIDAK dihapus. Ini menjaga histori tanpa mengotori dokumen aktif yang dibaca berulang.
- **Tidak pakai wikilink `[[...]]`** kalau vault-nya di Obsidian tapi dibaca juga oleh Claude Code — wikilink butuh format persis (termasuk spasi/suffix nama file) yang mudah salah ketik dan sering menyebabkan broken-link massal. Referensi antar file cukup pakai path teks biasa (`lihat 11-Decisions.md`) — Claude membaca ini dengan grep/Read biasa, tidak butuh parser khusus.

---

## 6. Kesepakatan operasional yang berlaku (kecuali diubah eksplisit)

- Claude BOLEH menulis memory secara proaktif/agresif tanpa minta konfirmasi dulu setiap kali menangkap preferensi/koreksi dari user — bukan menunggu diminta.
- Klaude BOLEH mengelola/restrukturisasi vault secara penuh (termasuk ubah struktur folder) demi kemudahan dirinya membaca, TANPA perlu mengoptimalkan untuk keterbacaan manusia — user akan bertanya langsung ke Claude kalau butuh memahami isi vault, bukan membaca file mentah sendiri.
- Sebelum restrukturisasi besar apa pun ke vault/dokumen penting, buat backup/commit dulu (kalau ada git) sebagai jaring pengaman reversible.
- Keputusan arsitektur besar (bukan sekadar preferensi gaya kerja) TIDAK langsung ditulis sebagai aturan wajib permanen tanpa didiskusikan detailnya dulu — beda dari preferensi kerja yang bisa langsung disimpan sebagai memory.
