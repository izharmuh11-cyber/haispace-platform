# Runtime Readiness Score

This checklist acts as the final gatekeeper before Swift Runtime development begins. It validates that the backend (Mission Control v1.0.0-rc1) fully supports the required capabilities.

### 1. API Readiness
- ✅ **Capabilities Endpoint**: `GET /v1/runtime/capabilities` implemented.
- ✅ **Clock Sync Endpoint**: `GET /v1/runtime/time` implemented.
- ✅ **Heartbeat Endpoint**: `PATCH /v1/devices/{id}/descriptor` implemented.
- ✅ **Manifest Endpoint**: `GET /v1/runtime/manifest` implemented.

### 2. Authentication Readiness
- ✅ **Device Registration**: iPad can register itself via `POST /v1/devices` and receive an API Key.
- ✅ **API Key Protection**: All mission-critical runtime endpoints are protected by `BoothApiKeyGuard` (not Operator Sessions).
- ✅ **Key Revocation**: Administrators can revoke a device via Mission Control.

### 3. Session Pipeline Readiness
- ✅ **Session Upload**: `POST /v1/sessions` accepts valid schema.
- ✅ **Domain Events Upload**: `POST /v1/session-events` accepts multi-status arrays.
- ✅ **Audit Events Upload**: `POST /v1/audit-events` accepts hardware logs.
- ⚠️ **Photo Upload**: CDN/S3 bucket integration is mocked in boilerplate, needs physical implementation on production.

### 4. Offline Readiness
- ✅ **Idempotency**: `POST` endpoints use upsert logic to handle duplicate offline queue retries.
- ✅ **Timestamp Reliability**: Backend respects the `timestamp` field from the client rather than generating it purely on the server.

### 5. Observability Readiness
- ✅ **Operations Center Dashboard**: Mission Control can visualize live Booth heartbeats.
- ✅ **Runtime Status Table**: Hardware telemetry is exposed to operators.

### 6. Deployment Readiness
- ✅ **Database Schema**: Frozen (`schema.prisma`).
- ✅ **Containerization**: VPS deployment uses Docker Compose without caching issues.
- ✅ **Platform Tag**: `v1.0.0-rc1` stamped.

**Total Score**: 95% (Ready for Swift Development)
