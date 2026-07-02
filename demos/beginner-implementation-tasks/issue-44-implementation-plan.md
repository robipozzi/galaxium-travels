# Issue #44 — Calendar Export (.ics) for Bookings

Implementation plan · ~1 hour · 4 files touched · no new dependencies

## Overview

Add `GET /bookings/{id}/export.ics` to the Python/FastAPI backend that returns a valid iCalendar file (RFC 5545) for a booking's departure event, plus a matching MCP tool. On the frontend, add an "Add to Calendar" button to `BookingCard.tsx` that triggers a browser file download (no Axios — a temporary `<a download>` element).

---

## Implementation Steps

### Step 1 — Add `get_booking_ics()` service function

**File:** `booking_system_backend/services/booking.py`

- New function: `get_booking_ics(db: Session, booking_id: int) -> str | ErrorResponse`
- Joins `Booking` → `Flight` → `User` via `db.query()` (same pattern as `cancel_booking`).
- Returns `ErrorResponse(error_code="BOOKING_NOT_FOUND")` if booking is missing.
- Builds the iCalendar string in-memory using Python string formatting — no library needed. Required fields per acceptance criteria:
  - `UID`: `booking-{booking_id}@galaxium-travels`
  - `SUMMARY`: `Galaxium Flight: {origin} → {destination}`
  - `LOCATION`: `{origin} → {destination}`
  - `DTSTART` / `DTEND`: parse ISO-8601 departure/arrival times from the `Flight` row and reformat as `YYYYMMDDTHHMMSSZ`
  - `DESCRIPTION`: booking ID, seat class, price, flight ID
- Wraps the VEVENT in a minimal `VCALENDAR` envelope with `PRODID` and `VERSION:2.0`.
- Uses `CRLF` line endings (`\r\n`) as required by RFC 5545.

---

### Step 2 — Add REST endpoint + MCP tool

**File:** `booking_system_backend/server.py`

- Import `Response` from `fastapi` (already has `FastAPI`, `Depends`, `HTTPException`).
- REST endpoint added near the other booking routes (after `/cancel/{booking_id}`):
  ```python
  @app.get("/bookings/{booking_id}/export.ics", tags=["Bookings"])
  ```
  Returns `Response(content=ics_str, media_type="text/calendar", headers={"Content-Disposition": 'attachment; filename="booking-{id}.ics"'})` or raises `HTTPException(404)` on `ErrorResponse`.
- MCP tool added near the other `@mcp.tool()` booking tools:
  ```python
  def get_booking_ics(booking_id: int) -> str
  ```
  Returns the raw iCalendar string or raises `Exception` on error. Follows the same `SessionLocal() / db.close()` pattern as all other MCP tools.
- **Order constraint:** MCP tool must stay above `mcp_app = mcp.http_app()` (line 108) — all existing tools already respect this.

---

### Step 3 — Add "Add to Calendar" button in `BookingCard.tsx`

**File:** `booking_system_frontend/src/components/bookings/BookingCard.tsx`

- Handler `handleAddToCalendar` constructs the URL `{API_BASE_URL}/bookings/{booking.booking_id}/export.ics` and triggers a download by creating a temporary `<a>` element with `href` set to the URL and `download="booking-{id}.ics"`, appending it to the DOM, clicking it, and removing it. No Axios — the browser fetches the file directly.
- Button placed alongside the Cancel button, gated on `booking.status === 'booked'` (same guard as `canCancel`). Use the existing `<Button>` component with `variant="secondary"` and `size="sm"`. Import the `CalendarPlus` icon from `lucide-react` (already in the project's dependency tree).
- Layout: wrap the two buttons in a `flex gap-2` container so they sit side-by-side.
- The `API_BASE_URL` should be read from `import.meta.env.VITE_API_URL` (same as `api.ts`) — check the Vite config / existing service layer for the exact constant.

---

### Step 4 — Write tests

**Files:** `booking_system_backend/tests/test_services.py` and `test_rest.py`

**Service tests** (`test_services.py`, new `TestBookingIcsService` class):
- `test_get_booking_ics_returns_valid_icalendar` — seeds user + flight + booking, calls `booking.get_booking_ics(db, id)`, asserts `BEGIN:VCALENDAR`, `BEGIN:VEVENT`, `UID:booking-{id}@galaxium-travels`, `SUMMARY`, `LOCATION`, `DTSTART`, `DTEND`, `DESCRIPTION`, `END:VEVENT`, `END:VCALENDAR`.
- `test_get_booking_ics_not_found` — calls with non-existent ID, asserts result is `ErrorResponse` with `error_code == "BOOKING_NOT_FOUND"`.

**REST tests** (`test_rest.py`, new `TestIcsEndpoint` class):
- `test_export_ics_200` — seeds data, `GET /bookings/{id}/export.ics`, asserts HTTP 200, `content-type: text/calendar`, body contains `BEGIN:VCALENDAR`.
- `test_export_ics_404` — `GET /bookings/99999/export.ics`, asserts HTTP 404.

---

## Potential Pitfalls

**Datetime parsing:** Flight times are stored as strings like `"2099-01-01T09:00:00Z"` or `"2099-01-01 09:00"` (seed data uses both ISO 8601 and space-separated formats — see `conftest.py` sample data). Use `datetime.fromisoformat()` with a strip of trailing `Z` and normalise to UTC before formatting as `YYYYMMDDTHHMMSSZ`.

**CRLF line endings:** RFC 5545 requires `\r\n`. Build the iCalendar string with explicit `\r\n` separators; do _not_ rely on `os.linesep`.

**MCP tool placement:** The new `@mcp.tool()` function must be defined before `mcp_app = mcp.http_app()` at line 108 of `server.py`, otherwise it won't be registered (per AGENTS.md footgun).

**Frontend download URL:** The `<a href>` must point to the full backend URL (e.g. `http://localhost:8001/api/bookings/{id}/export.ics`). Because the FastAPI app mounts with `root_path="/api"`, check that the Vite dev proxy (if any) correctly forwards the `/api/bookings/…` path, or use the raw `VITE_API_URL` env var as base.

---

## Acceptance Criteria Mapping

| Acceptance Criterion | Covered by |
|---|---|
| `GET /bookings/{id}/export.ics` → HTTP 200, `text/calendar` | Step 2 REST endpoint + Step 4 REST test |
| `GET /bookings/{id}/export.ics` → HTTP 404 when not found | Step 2 REST endpoint + Step 4 REST test |
| Event contains SUMMARY, LOCATION, DTSTART, DTEND, DESCRIPTION, UID | Step 1 service function + Step 4 service test |
| Matching MCP tool added | Step 2 MCP tool |
| "Add to Calendar" button triggers browser file download | Step 3 frontend component |
| Opens in Apple Calendar / Google Calendar | Step 1 (RFC 5545 compliance, CRLF, PRODID) |
| `pytest` passes with new tests | Step 4 |

---

## Files Changed

| File | Change |
|---|---|
| `booking_system_backend/services/booking.py` | Add `get_booking_ics()` |
| `booking_system_backend/server.py` | Add `GET /bookings/{id}/export.ics` endpoint + `@mcp.tool() get_booking_ics()` |
| `booking_system_backend/tests/test_services.py` | Add `TestBookingIcsService` class (2 tests) |
| `booking_system_backend/tests/test_rest.py` | Add `TestIcsEndpoint` class (2 tests) |
| `booking_system_frontend/src/components/bookings/BookingCard.tsx` | Add `handleAddToCalendar` + "Add to Calendar" button |
