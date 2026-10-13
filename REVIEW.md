# REVIEW.md

Defects found in the starter `app/main.py`.

| # | Where | What is wrong | How you'd notice it | How I fixed it |
|---|---|---|---|---|
| 1 | Line 22, `MongoClient(...)` | Client is not `tz_aware=True`, so every datetime read from MongoDB comes back without a time zone and can't be compared safely with IST-aware values | Print `tzinfo` of a `punch_in` read from the DB: it is `None` | (Phase 3) |
| 2 | Lines 33-37, `compute_late_minutes` | Three problems: it floors to whole minutes before the `> 10` check, so 09:40:01 returns 0 instead of 10; it builds the shift start from `punch_in` in whatever time zone it carries instead of IST; it ignores microseconds and overnight shifts | `compute_late_minutes(09:40:01 IST, "09:30")` returns 0, spec says 10 | (Phase 3) |
| 3 | Lines 40-41, `compute_work_hours` | `round()` uses round-half-to-even on a float, not half-up, and the inputs are not truncated to whole seconds | 09:00:00 to 17:07:30 (8.125 h) returns 8.12, spec says 8.13 | (Phase 3) |
| 4 | Lines 44-47, `compute_overtime` | No 30-minute minimum; the shift end is always on the record date, so overnight shifts are wrong; it attaches IST to the end time but `punch_out` can be naive, which would raise a TypeError | 18:45 punch-out on an 18:30 shift returns 15, spec says 0 | (Phase 3) |
| 5 | Lines 53-60, `EmployeeIn` | Every field is a bare `str`: no `EMP\d{4,6}` pattern, email check, length limits, `HH:MM` format check or valid date for `joined_on` | POST an employee with `emp_code: "abc"`, `email: "x"`, `shift_start: "99:99"`: it is accepted | (Phase 3) |
| 6 | Lines 63-66, `PunchInIn` | `punched_at` accepts any integer (seconds, negatives, 0) and `status` accepts any string | `punched_at: 1791779400` (seconds) is accepted | (Phase 3) |
| 7 | Lines 72-74, `/health` | Always returns ok and never touches MongoDB | Stop the Mongo container: `/health` still returns 200 | (Phase 3) |
| 8 | Lines 79-83, `create_employee` | Check-then-insert race: two simultaneous requests can both pass the `find_one` and both insert; no unique index on `emp_code` | Reasoning from the code; two parallel POSTs create two employees | (Phase 3) |
| 9 | Lines 82 and 85, `create_employee` | `created_at` uses `datetime.now()` (local, naive time, not UTC), and it is returned as a datetime instead of epoch milliseconds | The response shows a non-epoch `created_at` | (Phase 3) |
| 10 | Line 89, `list_employees` | `page` and `page_size` are not validated: 0, negative or huge values are accepted | `GET /employees?page=0` and `page_size=100000` are accepted | (Phase 3) |
| 11 | Line 93, `list_employees` | `skip = page * page_size`, so page 1 skips the first page of results; it should be `(page - 1) * page_size` | `?page=1&page_size=2` starts at the third employee | (Phase 3) |
| 12 | Line 94, `list_employees` | `total` counts the whole collection and ignores the department filter | `?department=Sales` reports a total of all employees | (Phase 3) |
| 13 | Line 95, `list_employees` | No sort, so MongoDB can return employees in any order and pages can repeat or skip rows | Reasoning from the code; there should be a deterministic sort by `emp_code` | (Phase 3) |
| 14 | Lines 101 and 113, `punch_in` | Unknown employee: `emp` is `None` and `emp["shift_start"]` raises a TypeError, so the client gets a 500 instead of a 404 | `emp_code: "EMP9999"` returns 500 | (Phase 3) |
| 15 | Lines 102-103 and 110, `punch_in` | `fromtimestamp` and `now()` use the machine's local time and make a naive datetime, which PyMongo then stores as if it were UTC; the date is not taken in IST and ignores the overnight-shift rule; `if body.punched_at` treats 0 as "not given" and nothing validates the range | Compare the stored `punch_in` with the request time: it's shifted by the local offset | (Phase 3) |
| 16 | Lines 104-118, `punch_in` | Same check-then-insert race as row 8, with no unique index on `(emp_code, date)` | Two parallel punch-ins for the same day can both succeed | (Phase 3) |
| 17 | Line 109, `punch_in` | `status` is stored without validation, so `ABSENT`, `LEAVE` or `banana` can be punched in | POST with `status: "ABSENT"` returns 201 | (Phase 3) |
| 18 | Lines 119-121, `punch_in` | Response leaks an `id` and returns datetimes instead of epoch milliseconds | Any successful punch-in response | (Phase 3) |
| 19 | Lines 144-147, `list_attendance` | Loads every matching record into memory and sorts in Python instead of using MongoDB sort, skip and limit; there is no tie-break on `emp_code` | Reasoning from the code; slow and memory heavy at 100k records | (Phase 3) |
| 20 | Lines 125-143 and 149, `list_attendance` | No validation of dates, `date_from > date_to`, status, `page` or `page_size`; `id` is exposed; times are not epoch ms; `history` and `half_day` are not defaulted when missing | `GET /attendance` shows `id` in every item | (Phase 3) |
| 21 | Whole file | No indexes are created at startup, so every query would scan the whole collection | No `create_index` call anywhere | (Phase 3) |

## Looked at and decided not a defect

- `load_dotenv()` (line 20) does not override variables already set in the real environment by default, which is what the spec wants.
- `IST = timezone(timedelta(hours=5, minutes=30))` (line 25) is the correct offset.
- ISO `YYYY-MM-DD` strings compare and sort correctly as text, so the `$gte` and `$lte` range on `date` (lines 136-141) is fine.
- The fallback defaults on lines 22-23 only apply when no env value is set; this is acceptable convenience for local development.