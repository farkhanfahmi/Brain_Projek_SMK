# T014 — Offline Buffer Kiosk (IndexedDB + Background Sync)

## Depends on
T011 (kiosk UI), T012 (endpoint /attendance/tap sudah ada)

## Objective
Tambahkan kemampuan offline pada kiosk: simpan tap ke IndexedDB kalau API tidak terjangkau, retry otomatis setiap 5 detik, dengan idempotency via client_uuid.

## Context
- **App:** `apps/kiosk`
- **ADR:** ADR-006, keputusan offline buffer di `absensi-gerbang.md`
- **Library:** `idb` (wrapper IndexedDB yang ringan)

## Spec Detail

### Install:
```
pnpm add idb nanoid --filter kiosk
```

### IndexedDB Schema:
```typescript
// DB name: 'absensi-kiosk-buffer'
// Store: 'pending-taps'
// Index: 'synced' (untuk query pending yang belum sync)

interface PendingTap {
  id: string;           // auto-increment atau nanoid
  uid: string;
  client_uuid: string;  // nanoid, generated saat tap terjadi
  timestamp: number;    // Date.now() saat tap — dipakai di body request
  kiosk_id: string;
  synced: boolean;      // false saat disimpan, true setelah berhasil sync
}
```

### Flow lengkap tap dengan offline support:

```typescript
async function processTap(uid: string) {
  const client_uuid = nanoid();
  const timestamp = Date.now();
  
  // Simpan ke IndexedDB dulu
  await db.add('pending-taps', {
    uid, client_uuid, timestamp, kiosk_id: KIOSK_ID, synced: false
  });
  
  // Coba kirim ke API
  try {
    const response = await fetch('/attendance/tap', {
      method: 'POST',
      body: JSON.stringify({ uid, client_uuid, kiosk_id: KIOSK_ID }),
      // Timestamp tidak perlu dikirim — server pakai server timestamp
      signal: AbortSignal.timeout(3000) // 3 detik timeout
    });
    const data = await response.json();
    
    // Tandai sebagai synced
    await markSynced(client_uuid);
    
    // Tampilkan feedback dari response API
    showFeedback(data);
    
  } catch (error) {
    // Offline / timeout — tampilkan feedback offline
    setOnlineStatus(false);
    showOfflineFeedback(); // "Tap tersimpan, menunggu koneksi..."
  }
}
```

### Background sync (setiap 5 detik):
```typescript
async function syncPendingTaps() {
  const pending = await db.getAllFromIndex('pending-taps', 'synced', false);
  
  for (const tap of pending) {
    try {
      await fetch('/attendance/tap', {
        method: 'POST',
        body: JSON.stringify({ uid: tap.uid, client_uuid: tap.client_uuid, kiosk_id: tap.kiosk_id }),
        signal: AbortSignal.timeout(3000)
      });
      await markSynced(tap.client_uuid);
    } catch {
      break; // Masih offline, stop retry sampai interval berikutnya
    }
  }
  
  // Update status indikator
  const stillPending = await db.countFromIndex('pending-taps', 'synced', false);
  setOnlineStatus(stillPending === 0);
}

// Jalankan setiap 5 detik
setInterval(syncPendingTaps, 5000);
```

### Indikator status di UI kiosk (update dari T011):
- `🟢 Online` — tidak ada pending tap
- `🟡 Sinkronisasi...` — ada pending tap, sedang mencoba sync
- `🔴 Offline (N tap pending)` — ada pending, API tidak terjangkau

## JANGAN
- ❌ JANGAN kirim `timestamp` dari kiosk sebagai sumber kebenaran waktu tap ke API — server selalu pakai server timestamp. `timestamp` lokal hanya disimpan di IndexedDB untuk referensi internal kiosk
- ❌ JANGAN buat retry yang aggressive (< 5 detik) — 5 detik sudah cukup untuk kasus network intermittent
- ❌ JANGAN hapus record `synced: true` dari IndexedDB — biarkan terakumulasi (ini log lokal, bukan concern utama). Kalau mau cleanup, hapus yang `synced: true` dan `timestamp` > 7 hari
- ❌ JANGAN blokir tap baru saat sedang retry sync — keduanya harus bisa jalan bersamaan

## Files
- **Buat:** `apps/kiosk/lib/db.ts` (IndexedDB setup via `idb`)
- **Buat:** `apps/kiosk/lib/sync.ts` (background sync logic)
- **Modifikasi:** `apps/kiosk/app/page.tsx` — integrasikan offline buffer ke flow tap yang sudah ada di T011

## Acceptance Criteria
- [ ] Matikan API (stop `apps/api`) → tap tetap bisa dilakukan, tersimpan di IndexedDB
- [ ] Nyalakan kembali API → dalam 5-10 detik, tap yang pending ter-sync otomatis
- [ ] Setelah sync berhasil, `tap_events` di database berisi tap yang dilakukan saat offline
- [ ] Tap yang sama tidak masuk 2x ke database (client_uuid idempotency bekerja)
- [ ] Indikator status berubah: offline (merah) → sync (kuning) → online (hijau) saat API pulih
