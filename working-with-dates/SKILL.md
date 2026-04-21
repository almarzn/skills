---
name: working-with-dates
description: Use when working with dates, times, durations, or time-related data on the frontend — parsing ISO strings, formatting for display, timers, or any temporal calculations
---

# Working with Dates (Temporal API)

## Overview

**ALWAYS use the Temporal API. NEVER use `Date`, `new Date()`, or `Date.now()`.**

The `@js-temporal/polyfill` is installed and `Temporal` is available globally via `src/types/temporal.d.ts`.

## Quick Reference

| Task | Use | Example |
|------|-----|---------|
| Parse ISO timestamp from API | `Temporal.PlainDateTime.from()` | Strip `Z` suffix first |
| Parse date-only string | `Temporal.PlainDate.from()` | `"2025-03-15"` |
| Get current instant | `Temporal.Now.instant()` | For timestamps |
| Get current date/time | `Temporal.Now.zonedDateTimeISO()` | Local timezone |
| Duration from API string | `Temporal.Duration.from()` | PT30M, PT1H30M |
| Format duration | `dur.round().toLocaleString()` | Round first, then format |
| Format for display | `.toLocaleString()` or manual | `pdt.toLocaleString()` |
| Compare two dates | `PlainDate.compare()` / `.equals()` | Returns -1/0/1 |
| Add/subtract time | `.add()` / `.subtract()` | `pdt.add({ days: 7 })` |
| Elapsed time since | `Temporal.Now.instant().since(past)` | Returns Duration |

## Parsing ISO Strings from API

API returns ISO 8601 strings (e.g. `"2025-03-15T14:30:00Z"`). Strip the `Z` before parsing as `PlainDateTime`:

```typescript
// ❌ BAD — never use Date
const d = new Date(iso);
d.getMonth(); // 0-indexed, error-prone

// ✅ GOOD — Temporal
const pdt = Temporal.PlainDateTime.from(iso.endsWith('Z') ? iso.slice(0, -1) : iso);
pdt.month;  // 3 (March) — 1-indexed
pdt.day;    // 15
pdt.year;   // 2025
```

For date-only strings (`"2025-03-15"`), use `PlainDate.from()` directly — no `Z` stripping needed.

## Formatting for Display

Use `toLocaleString()` for standard formatting, or manual formatting for custom layouts:

```typescript
// Standard locale formatting
pdt.toLocaleString();                          // locale default
pdt.toLocaleString('en-GB', { dateStyle: 'medium' });  // "15 Mar 2025"
pdt.toLocaleString('en-US', { dateStyle: 'medium', timeStyle: 'short' });

// Manual formatting (for custom layouts like "Mar 15, 2025")
const monthNames = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'];
`${monthNames[pdt.month - 1]} ${pdt.day}, ${pdt.year}`;
```

## Durations (Timer Attachments)

`TimerAttachment.duration` comes as ISO 8601 duration strings (e.g. `"PT30M"`, `"PT1H30M"`).

```typescript
const dur = Temporal.Duration.from('PT1H30M15S');

// Round to remove fractional/nano parts, then format with locale
dur.round({ smallestUnit: 'seconds' }).toLocaleString();
// "1 hour, 30 minutes, 15 seconds" (locale-dependent)

// Round to minutes
dur.round({ smallestUnit: 'minutes' }).toLocaleString();
// "1 hour, 30 minutes"

// Access components directly
dur.hours;    // 1
dur.minutes;  // 30
```

## Current Time, Elapsed, Arithmetic

```typescript
const now = Temporal.Now.instant();                    // epoch instant
const zdt = Temporal.Now.zonedDateTimeISO();           // local zoned datetime
const pd = zdt.toPlainDate();                          // date only

// Elapsed time (timers)
const start = Temporal.Now.instant();
const elapsed = Temporal.Now.instant().since(start);   // Duration

// Arithmetic
pd.add({ days: 7 });                                   // "2025-03-22"
pd.subtract({ months: 1 });                            // "2025-02-15"
pd.until(Temporal.PlainDate.from('2025-04-01'));       // Duration: { days: 17 }
```

## Common Mistakes

- `new Date()` / `Date.parse()` / `Date.now()` — use Temporal equivalents
- 0-indexed month confusion — Temporal months are 1-indexed (Jan = 1)
- Parsing `"Z"`-suffixed ISO strings directly — strip `Z` before `PlainDateTime.from()`
- Using `setTimeout` delays as Duration values — Duration is for data, not scheduling

## Where to Put Utilities

Shared date/duration helpers go in `src/app/shared/utils/date.ts`, exported from `src/app/shared/utils/index.ts`.
