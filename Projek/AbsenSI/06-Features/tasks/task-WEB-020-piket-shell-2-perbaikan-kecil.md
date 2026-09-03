# Task-WEB-020: Piket Shell — 2 Perbaikan Kecil (Accordion Sidebar Dead Toggle + Default Duty Fail-Open)

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah audit kejanggalan Dashboard Piket + diskusi kritis dengan user (2026-09-03). Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.
> Digabung 1 file karena masing-masing perbaikan sangat kecil (1-3 baris) dan berada di area shell/layout piket yang sama — pola sama seperti T254 (3 perbaikan kecil digabung 1 task).

**Task Terbuat:** 2026-09-03
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** low
**Alasan pemilihan:** 2 fix kecil, masing-masing perubahan logic 1-3 baris di file berbeda, tidak butuh riset — murni koreksi bug yang sudah terverifikasi jelas root cause-nya saat audit.

## 2. Konteks & Tujuan Utama

Audit Dashboard Piket (2026-09-03) menemukan 2 kejanggalan kecil independen di layer shell/layout piket:

**A. Sidebar accordion "Perizinan" — tombol collapse/expand mati total saat berada di dalam grupnya**
`apps/web/src/app/(piket)/piket-sidebar.tsx` baris ~81-82:
```js
const groupHasActiveItem = group.items.some((item) => isItemActive(pathname, item.href));
const [expanded, setExpanded] = useState(groupHasActiveItem);
const isOpen = expanded || groupHasActiveItem;
```
Selama piket berada di salah satu dari 4 halaman grup "Perizinan" (Perizinan Keluar/Permintaan Izin/Input Izin/Riwayat Izin), `groupHasActiveItem` selalu `true`, sehingga `isOpen` SELALU `true` tidak peduli state `expanded` — klik tombol toggle (baris 138: `onClick={() => setExpanded((open) => !open)}`) mengubah `expanded` tapi TIDAK berefek visual apa pun (chevron tidak berubah, grup tidak bisa ditutup). Dead interaction yang membingungkan.

**B. Default context `usePiketOnDuty()` fail-open (default `true`), seharusnya fail-closed**
`apps/web/src/app/(piket)/piket-duty-context.tsx` baris ~5:
```js
const PiketDutyContext = createContext<boolean>(true);
```
Context ini menentukan apakah UI menampilkan aksi tulis (lock siswa, approve izin, dll) sebagai aktif. Default `true` berarti kalau ada consumer `usePiketOnDuty()` yang ter-render DI LUAR `PiketDutyProvider` (skenario: refactor masa depan, komponen dipindah lokasi, dsb), UI akan DIAM-DIAM mengasumsikan "sedang bertugas" (mengizinkan aksi tulis tampil aktif) — bukan fail-safe. Untuk gate UI yang menentukan tampil/tidaknya aksi mutasi, default seharusnya restriktif (`false`).

**Depends on:** Tidak ada — 2 fix independen, digabung 1 file karena lokasi sama.

## 3. Langkah Eksekusi Detail

### A. Fix Accordion Sidebar (`apps/web/src/app/(piket)/piket-sidebar.tsx`)

1. Ubah logic `isOpen` supaya klik toggle eksplisit oleh user MENANG atas auto-expand awal. Pendekatan: pisahkan "auto-expand saat pertama kali halaman ini aktif" dari "state buka/tutup manual setelahnya".
   ```js
   const groupHasActiveItem = group.items.some((item) => isItemActive(pathname, item.href));
   const [expanded, setExpanded] = useState(groupHasActiveItem);
   const isOpen = expanded; // HAPUS OR groupHasActiveItem — expanded sudah diinisialisasi dari situ, biarkan user override
   ```
   **VERIFIKASI SAAT IMPLEMENTASI**: pastikan behavior "auto-expand saat navigasi PERTAMA KALI ke halaman di grup itu dari grup lain" tetap terjadi (mis. via `useEffect` yang re-set `expanded` ke `true` HANYA saat `groupHasActiveItem` berubah dari `false` ke `true`, bukan tiap render) — supaya tetap nyaman dipakai (grup otomatis terbuka saat dikunjungi), TAPI setelah itu piket bisa menutupnya secara manual tanpa auto-force-open lagi selama masih di halaman yang sama.

2. Cek chevron/indikator visual expand-collapse (biasanya ada icon rotate berdasar `isOpen`) — pastikan sinkron dengan state baru.

### B. Fix Default Duty Context (`apps/web/src/app/(piket)/piket-duty-context.tsx`)

3. Ubah default `createContext<boolean>(true)` → `createContext<boolean>(false)`.

4. **VERIFIKASI SAAT IMPLEMENTASI**: pastikan `PiketDutyProvider` benar-benar membungkus SEMUA halaman/komponen yang memanggil `usePiketOnDuty()` (cek `layout.tsx` — provider sudah dipasang di root shell `(piket)`). Kalau ternyata ADA consumer yang render di luar provider (mis. via halaman yang keluar dari shell piket seperti temuan Direktori Murid di task-WEB-024), pastikan perubahan default ke `false` tidak tiba-tiba mematikan aksi yang sebelumnya (secara tidak sengaja) berfungsi karena fail-open — kalau ditemukan kasus begini saat verifikasi, LAPORKAN sebagai catatan di bagian Implementasi (jangan diam-diam dibiarkan tombol jadi disabled tanpa penjelasan).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/web/src/app/(piket)/piket-sidebar.tsx` — logic `isOpen`/`expanded`
- **Modifikasi:** `apps/web/src/app/(piket)/piket-duty-context.tsx` — default context
- **Jangan sentuh:** Struktur data `GROUPS`/menu items, `PiketDutyProvider` value assignment (provider tetap kirim value asli dari API, hanya DEFAULT saat tanpa provider yang berubah)

**Dilarang dilakukan:**
- Jangan redesign accordion jadi komponen baru — ini fix logic state, bukan refactor visual.

**Skenario kegagalan yang WAJIB ditangani:**
- Kondisi: piket navigasi pertama kali ke halaman "Permintaan Izin" dari halaman lain (mis. dari Dashboard) → Perilaku benar: grup "Perizinan" otomatis terbuka (auto-expand tetap jalan).
- Kondisi: piket sudah di halaman "Permintaan Izin", grup Perizinan otomatis terbuka, piket klik toggle untuk menutup → Perilaku benar: grup TERTUTUP secara visual (chevron berubah, item-item grup hilang dari tampilan), meski masih berada di halaman aktif grup itu.
- Kondisi: consumer `usePiketOnDuty()` di luar provider (edge case masa depan) → Perilaku benar: default `false` (fail-closed, aksi tulis tersembunyi/disabled), bukan diam-diam aktif.

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Grup "Perizinan" di sidebar bisa ditutup manual oleh piket meski sedang berada di salah satu halaman grup itu, chevron berubah sesuai state.
- [ ] Auto-expand saat navigasi pertama kali ke grup itu tetap berfungsi (tidak regresi UX yang sudah baik).
- [ ] `createContext<boolean>(false)` — default fail-closed dikonfirmasi lewat test/verifikasi manual (consumer di luar provider tidak lagi menampilkan aksi tulis sebagai aktif).
- [ ] Tidak ada regresi pada halaman piket lain yang memakai `usePiketOnDuty()` normal di dalam provider (perilaku sama seperti sebelumnya, karena value asli tetap dikirim provider).

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 50 baris perubahan total)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency: tidak ada
