# T017 — Socket.IO Setup & Attendance Gateway

## Depends on
T012 (BullMQ attendance.recorded event sudah di-dispatch)

## Objective
Setup Socket.IO di API dan buat gateway yang push update realtime ke Dashboard TV dan Dashboard Piket saat ada tap baru.

## Context
- **App:** `apps/api` (server) + `apps/web` dan `apps/kiosk` (client)
- **ADR:** ADR-006 (event-driven)
- **Channels:** `attendance:today` (agregat untuk TV), `attendance:kampus:{id}` (per-siswa untuk piket)

## Spec Detail

### Install:
```
pnpm add @nestjs/websockets @nestjs/platform-socket.io socket.io --filter api
pnpm add socket.io-client --filter web
```

### `AttendanceGateway` (`apps/api`):

```typescript
@WebSocketGateway({ cors: { origin: '*' } })
export class AttendanceGateway {
  
  @WebSocketServer()
  server: Server;
  
  // Dipanggil dari AttendanceProcessor saat ada event attendance.recorded
  async broadcastTapEvent(payload: AttendanceRecordedPayload) {
    // 1. Ambil agregat terbaru hari ini
    const todayStats = await this.attendanceService.getTodayStats();
    
    // 2. Push ke channel TV (agregat)
    this.server.emit('attendance:today', todayStats);
    
    // 3. Push ke channel piket per kampus (detail per siswa)
    const kampusId = payload.student?.kelas?.kampus_id;
    if (kampusId) {
      this.server.to(`kampus:${kampusId}`).emit(`attendance:kampus:${kampusId}`, {
        student_id: payload.student_id,
        nama: payload.student.nama,
        status: payload.status,
        waktu_masuk: payload.waktu_masuk,
        waktu_pulang: payload.waktu_pulang,
      });
    }
  }
  
  // Client join room kampus
  @SubscribeMessage('join:kampus')
  handleJoinKampus(client: Socket, kampus_id: string) {
    client.join(`kampus:${kampus_id}`);
  }
}
```

### Auth WebSocket:
- Client kirim JWT di handshake: `{ auth: { token: accessToken } }`
- Gateway validasi token di `handleConnection()` — disconnect kalau invalid
- Untuk TV (kepsek): JWT long-lived, tidak ada issue
- Untuk Dashboard Piket: JWT standard 15 menit, client harus handle reconnect + re-auth saat token expire

### Payload `attendance:today`:
```json
{
  "tanggal": "2026-07-03",
  "total_siswa": 2500,
  "hadir": 2350,
  "terlambat": 45,
  "belum_hadir": 105
}
```

### Payload `attendance:kampus:{id}` (per update):
```json
{
  "student_id": "xxx",
  "nama": "Budi Santoso",
  "kelas": "XI-RPL-1",
  "status": "hadir",
  "waktu_masuk": "07:32:00",
  "waktu_pulang": null
}
```

### Client setup di `apps/web`:
```typescript
// Hook useSocket() di packages/ui atau apps/web/lib/socket.ts
// Singleton socket connection
// Auto-reconnect
// Handle token refresh + re-auth saat expired
```

## JANGAN
- ❌ JANGAN kirim JWT sebagai query parameter di URL WebSocket (`ws://...?token=xxx`) — JWT di URL masuk ke access log server, tidak aman
- ❌ JANGAN buat channel per-siswa — terlalu banyak channel. Channel per-kampus sudah cukup, frontend filter per siswa
- ❌ JANGAN query database langsung di gateway saat ada event — delegate ke service
- ❌ JANGAN setup CORS `origin: '*'` untuk production — ini hanya untuk development. Catat sebagai TODO untuk production config

## Files
- **Buat:** `apps/api/src/attendance/attendance.gateway.ts`
- **Modifikasi:** `apps/api/src/attendance/processors/attendance.processor.ts` — panggil gateway saat ada event
- **Modifikasi:** `apps/api/src/attendance/attendance.module.ts` — export gateway
- **Buat:** `apps/web/lib/socket.ts` — singleton Socket.IO client + hook

## Acceptance Criteria
- [ ] Tap kartu → dalam < 1 detik, client yang subscribe `attendance:today` menerima update
- [ ] Client Dashboard Piket join room `kampus:1` → hanya menerima event siswa Kampus 1
- [ ] WebSocket connection dengan token invalid → server disconnect client
- [ ] Reconnect setelah server restart → client reconnect otomatis dalam beberapa detik

## Handoff ke T018 & T023
T018 (Dashboard TV) akan subscribe ke `attendance:today`. T023 (Dashboard Piket) akan join room `kampus:{id}` dan subscribe ke `attendance:kampus:{id}`.
