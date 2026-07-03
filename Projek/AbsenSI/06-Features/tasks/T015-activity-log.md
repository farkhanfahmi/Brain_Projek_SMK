# T015 — Activity Log (Audit Trail Aksi User)

## Depends on
T003 (auth harus ada — activity_log butuh actor_id dari JWT)

## Objective
Implementasi interceptor yang mencatat setiap aksi user yang login ke tabel activity_log, beserta state sebelum dan sesudah perubahan.

## Context
- **App:** `apps/api`
- **Tables:** `activity_log` (insert-only)
- **ADR:** ADR-020

## Spec Detail

### NestJS Interceptor:

```typescript
@Injectable()
export class ActivityLogInterceptor implements NestInterceptor {
  // Hanya log request yang punya req.user (sudah authenticated)
  // Hanya log mutating methods: POST, PATCH, PUT, DELETE
  // GET tidak dilog
}
```

### Aksi yang WAJIB dilog di Fase 1:

| Resource | Aksi | action string |
|---|---|---|
| cards | create | `card.create` |
| cards | revoke | `card.revoke` |
| cards | replace | `card.replace` |
| students | lock | `student.lock` |
| students | unlock | `student.unlock` |
| permits | create | `permit.create` |
| permits | confirm_kembali | `permit.confirm_kembali` |
| permits | set_pulang | `permit.set_pulang` |
| attendance_records | manual_pulang | `attendance.manual_pulang` |
| attendance_records | konfirmasi_izin_pulang | `attendance.confirm_izin_pulang` |
| users | create | `user.create` |
| users | update | `user.update` |
| users | reset_password | `user.reset_password` |
| school_holidays | create | `holiday.create` |
| school_holidays | delete | `holiday.delete` |

### snapshot_before & snapshot_after:
- Untuk POST: `snapshot_before: null`, `snapshot_after: { record yang baru dibuat }`
- Untuk PATCH: `snapshot_before: { record sebelum diubah }`, `snapshot_after: { record setelah diubah }`
- Untuk DELETE: `snapshot_before: { record sebelum dihapus }`, `snapshot_after: null`
- Simpan sebagai JSON, hilangkan field sensitif (`password_hash`)

### Implementasi via decorator (lebih eksplisit dari interceptor global):
```typescript
// Di controller:
@Post()
@LogActivity('card.create')
async createCard(@Body() dto) { ... }
```

Decorator `@LogActivity(action)` yang menggunakan interceptor untuk capture before/after.

### API read-only:
`GET /activity-log` (akses: `super_admin`)
- Filter: `actor_id`, `action`, `target_type`, `from`, `to`
- Pagination
- Include: nama actor

### Admin UI:
`/admin/audit/activity` — tabel sederhana:
- Waktu | Dilakukan oleh | Aksi | Target | Detail (expand)

## JANGAN
- ❌ JANGAN log aksi GET (read) — hanya mutating actions
- ❌ JANGAN log tap events di sini — tap sudah di-log di `tap_events`, bukan `activity_log`
- ❌ JANGAN simpan `password_hash` di snapshot — filter field sensitif sebelum simpan ke JSON
- ❌ JANGAN buat endpoint DELETE atau PATCH untuk `activity_log`
- ❌ JANGAN gunakan global interceptor yang log semua request — terlalu noisy. Gunakan decorator `@LogActivity()` per endpoint yang diperlukan

## Files
- **Buat:** `apps/api/src/common/interceptors/activity-log.interceptor.ts`
- **Buat:** `apps/api/src/common/decorators/log-activity.decorator.ts`
- **Buat:** `apps/api/src/activity-log/activity-log.module.ts`
- **Buat:** `apps/api/src/activity-log/activity-log.controller.ts` (GET endpoint saja)
- **Buat:** `apps/web/app/(admin)/audit/activity/page.tsx`

## Acceptance Criteria
- [ ] Buat kartu baru → 1 baris muncul di `activity_log` dengan `action: card.create`
- [ ] `snapshot_after` berisi data kartu yang baru dibuat (tanpa field sensitif)
- [ ] Request GET ke endpoint manapun → tidak menghasilkan baris di `activity_log`
- [ ] `GET /activity-log?action=card.create` return hanya log pembuatan kartu
- [ ] `password_hash` tidak pernah muncul di kolom `snapshot_before` atau `snapshot_after`
