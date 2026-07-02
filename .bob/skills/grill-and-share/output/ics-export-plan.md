# ICS Calendar Export — Plan

**Status:** Draft · Reviewed

## Summary

The ICS export feature adds a `GET /bookings/{booking_id}/export.ics` REST endpoint and a matching MCP tool, generating RFC 5545 VCALENDAR strings with all timestamps emitted as UTC (trailing `Z`). The "Add to Calendar" button in the frontend reuses the existing `canCancel` guard for simplicity and consistency. The new REST endpoint must be registered *before* `GET /bookings/{user_id}` in `server.py` to guarantee correct path resolution, as explicit ordering is safer than relying on path-depth matching alone.

---

## Key Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Timezone handling in VEVENT | Treat all times as UTC — append `Z` to `DTSTART`/`DTEND` | Simplest approach; consistent with demo's fake interplanetary data where a "real" timezone is meaningless. Avoids VTIMEZONE component. |
| Frontend button visibility guard | Reuse `canCancel` exactly — no separate variable | Keeps the export and cancel buttons in sync; `canCancel` is already the correct guard for active (`booked`) bookings. |
| Route registration order | Register `/bookings/{booking_id}/export.ics` BEFORE `/bookings/{user_id}` | FastAPI matches routes in registration order; the more-specific (3-segment) path must be registered first to prevent the 2-segment wildcard swallowing requests. |

---

## Open Items

- **PRODNAME field in UID:** The plan doesn't specify a `PRODID` string — use `-//Galaxium Travels//Booking {id}//EN` as a placeholder.
- **Arrival time for DTEND:** `arrival_time` is available on the `Flight` model but the plan's VEVENT field list doesn't explicitly confirm it's used for `DTEND` — confirm during implementation (Sub-Task 1, step 5).
- **Frontend Vite proxy:** The `handleExportIcs` URL uses `/api/bookings/...` — confirm the Vite dev proxy (`vite.config.ts`) rewrites `/api` to `http://localhost:8001`. If not, the download will 404 in dev.
- **E2E test coverage:** Sub-Task 3 covers unit and REST tests but not an e2e test. Low risk for this feature; acceptable for a demo.

---

## Next Steps

1. **Sub-Task 1** — Add `generate_ics(db, booking_id)` to [`booking_system_backend/services/booking.py`](booking_system_backend/services/booking.py); emit UTC timestamps with `Z` suffix.
2. **Sub-Task 2** — Add the MCP tool `export_booking_ics` and REST endpoint to [`booking_system_backend/server.py`](booking_system_backend/server.py); register the new GET route *before* `GET /bookings/{user_id}`.
3. **Sub-Task 3** — Add `TestIcsExport` service tests and `TestIcsExportEndpoint` REST tests; run `pytest` to confirm all green.
4. **Sub-Task 4** — Add "Add to Calendar" button to [`booking_system_frontend/src/components/bookings/BookingCard.tsx`](booking_system_frontend/src/components/bookings/BookingCard.tsx); gate on `canCancel`; use `<a download>` pattern (no Axios); run `npm run lint`.
5. Verify Vite proxy config rewrites `/api` correctly before testing the frontend download in dev mode.
