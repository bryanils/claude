---
name: date-handling
description: Handle dates correctly in this codebase. Use when working with dates, timestamps, date ranges, date formatting, dayjs, or any code involving time. Enforces string-based date system.
allowed-tools: Read, Edit, Write, Glob, Grep
---

# Date Handling Guide

## CRITICAL RULES - NEVER VIOLATE

1. **ALL DATES ARE STRINGS** - Never use `Date` objects in application code
2. **USE `toDBTimestamp()` FOR WRITING** - From `~/utils/dateUtils`
3. **USE `dayjs` FOR PARSING** - From `~/utils/dateUtils`
4. **MariaDB FORMAT:** `"YYYY-MM-DD HH:mm:ss"` - This is what the database uses
5. **DO NOT USE `new Date()` OR `.toISOString()`** - Except WhenIWork API (see below)

## Type Definitions

```typescript
// All date fields are strings
interface Employee {
  hireDate: string | null;        // "YYYY-MM-DD"
  terminationDate: string | null; // "YYYY-MM-DD"
  createdAt: string;              // "YYYY-MM-DD HH:mm:ss"
  updatedAt: string;              // "YYYY-MM-DD HH:mm:ss"
}

// Date range type
interface DateRange {
  startDate: string;  // "YYYY-MM-DD"
  endDate: string;    // "YYYY-MM-DD"
}
```

### Two Formats

| Format | Example | Use For |
|--------|---------|---------|
| **DateString** | `"2025-01-15"` | Report periods, working days, hire dates |
| **DateTimeString** | `"2025-01-15 14:30:00"` | Timestamps, loan approval times, timeclock events |

## Schema Definition (Drizzle)

**CRITICAL:** Always use `mode: "string"` for ALL date/time columns:

```typescript
import { date, timestamp, datetime } from "drizzle-orm/mysql-core";
import { toDBTimestamp } from "~/utils/dateUtils";

// For calendar dates - returns "YYYY-MM-DD"
snapshotDate: date("snapshot_date", { mode: "string" }),

// For timestamps - returns "YYYY-MM-DD HH:mm:ss"
createdAt: timestamp("created_at", { mode: "string" }),
approvedAt: datetime("approved_at", { mode: "string" }),

// For $onUpdate callbacks - use toDBTimestamp(), NOT new Date()
updatedAt: timestamp("updated_at", { mode: "string" })
  .$onUpdate(() => toDBTimestamp()),
```

**Why `mode: "string"`?** Without it, Drizzle returns JavaScript `Date` objects which cause timezone bugs.

## Core Utilities (from ~/utils/dateUtils)

### Writing to Database

```typescript
import { toDBTimestamp } from "~/utils/dateUtils";

// Current timestamp for DB
const now = toDBTimestamp(); // "2024-01-15 14:30:00"

// Convert existing date string
const dbDate = toDBTimestamp("2024-01-15"); // "2024-01-15 00:00:00"
```

### Getting Current Date

```typescript
import { getTodayDate, getYesterdayDate } from "~/utils/dateUtils";

const today = getTodayDate();         // "2024-01-15"
const yesterday = getYesterdayDate(); // "2024-01-14"
```

### Parsing Dates

```typescript
import { dayjs } from "~/utils/dateUtils";

// Parse string to dayjs object (for calculations only)
const date = dayjs("2024-01-15");
const date2 = dayjs("2024-01-15 14:30:00");

// Check validity
if (!date.isValid()) {
  console.error("Invalid date");
}

// ALWAYS convert back to string for storage/passing
const dateStr = date.format("YYYY-MM-DD");
```

### Formatting for Display

```typescript
import { formatDate } from "~/utils/dateUtils";

// For UI display
const display = formatDate("2024-01-15", "M/D/YYYY");     // "1/15/2024"
const full = formatDate("2024-01-15", "MMMM D, YYYY");    // "January 15, 2024"
const withTime = formatDate("2024-01-15 14:30:00", "M/D/YYYY h:mm A"); // "1/15/2024 2:30 PM"
```

### Date Calculations

```typescript
import { dayjs } from "~/utils/dateUtils";

// Month boundaries
const monthStart = dayjs().startOf("month").format("YYYY-MM-DD");
const monthEnd = dayjs().endOf("month").format("YYYY-MM-DD");

// Add/subtract
const nextWeek = dayjs().add(7, "day").format("YYYY-MM-DD");
const lastMonth = dayjs().subtract(1, "month").format("YYYY-MM-DD");

// Difference
const days = dayjs("2024-01-20").diff(dayjs("2024-01-15"), "day"); // 5
```

### Date Comparisons

```typescript
import { dayjs } from "~/utils/dateUtils";

const date1 = dayjs("2024-01-15");
const date2 = dayjs("2024-01-20");

// Comparisons
date1.isBefore(date2);  // true
date1.isAfter(date2);   // false
date1.isSame(date2);    // false

// Range check (requires isBetween plugin - already loaded)
date1.isBetween("2024-01-01", "2024-01-31"); // true
```

## SQL Date Handling

### In Raw SQL Queries

```typescript
// CORRECT - Use DATE_FORMAT to get strings
const rows = await lms`
  SELECT
    loan_id,
    DATE_FORMAT(approve_date, '%Y-%m-%d') as approve_date,
    DATE_FORMAT(created_at, '%Y-%m-%d %H:%i:%s') as created_at
  FROM loan
  WHERE approve_date BETWEEN ${startDate} AND ${endDate}
`;

// WRONG - Returns Date objects with timezone issues
const rows = await lms`SELECT approve_date FROM loan`;
```

### In Drizzle Queries

```typescript
import { sql } from "drizzle-orm";
import { toDBTimestamp } from "~/utils/dateUtils";

// Writing
await db.insert(snapshots).values({
  snapshotDate: "2024-01-15",           // Date only
  createdAt: toDBTimestamp(),            // Full timestamp
  updatedAt: toDBTimestamp(),
});

// Reading - dates come back as strings from schema
const snapshot = await db.query.snapshots.findFirst({
  where: eq(snapshots.snapshotDate, "2024-01-15"),
});
// snapshot.createdAt is already a string
```

## Common Patterns

### Function Parameters

```typescript
// CORRECT - dates as strings
function getLoansInRange(startDate: string, endDate: string): Promise<Loan[]> {
  // ...
}

// WRONG - dates as Date objects
function getLoansInRange(startDate: Date, endDate: Date): Promise<Loan[]> {
  // ...
}
```

### API Input Validation (Zod)

```typescript
import { z } from "zod";

// Date range schema
const dateRangeSchema = z.object({
  startDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/),
  endDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/),
});

// Optional date
const optionalDateSchema = z.string().nullable().optional();
```

### Checking Termination

```typescript
import { isAfterTermination } from "~/utils/employeeUtils";

// Check if a date is after employee's termination
if (isAfterTermination(dateStr, employee.terminationDate)) {
  // Employee was terminated before this date
}
```

### Business Day Calculations

```typescript
import {
  isWeekday,
  generateWeekdays,
  getNthBusinessDayOfMonth
} from "~/utils/dateUtils";

// Check if weekday
const isWorkday = isWeekday("2024-01-15"); // true (Monday)

// Generate weekdays in range
const workdays = generateWeekdays("2024-01-01", "2024-01-31");
// ["2024-01-02", "2024-01-03", ...] (excludes weekends)

// Get nth business day
const fifthBusinessDay = getNthBusinessDayOfMonth(2024, 1, 5);
```

## FORBIDDEN Patterns

### Never Use `new Date()`

```typescript
// WRONG
const now = new Date();
const dateStr = new Date().toISOString();
const parsed = new Date("2024-01-15");

// CORRECT
import { toDBTimestamp, dayjs } from "~/utils/dateUtils";
const now = toDBTimestamp();
const parsed = dayjs("2024-01-15");
```

### Never Store Date Objects

```typescript
// WRONG
interface Record {
  createdAt: Date;
}

// CORRECT
interface Record {
  createdAt: string;
}
```

### Never Use `.toISOString()`

```typescript
// WRONG
const isoDate = someDate.toISOString();

// CORRECT
const dbDate = toDBTimestamp(someDate);
const formatted = dayjs(someDate).format("YYYY-MM-DD");
```

## Exceptions

### WhenIWork API

The WhenIWork external API requires ISO format with Z suffix. This is the ONLY place where ISO format is acceptable:

```typescript
import { parseISOAsLocalTime } from "~/utils/dateUtils";

// WhenIWork returns ISO strings - parse them stripping timezone
const localTime = parseISOAsLocalTime(wiwEvent.start_time);
```

### Internal Cache (storage.ts)

The cache system uses timestamps (numbers) and Date objects for filesystem metadata. Acceptable because it's cache metadata, not database storage.

### Better Auth Tables

Better Auth tables use `mode: "string"` but may need special handling if auth breaks.

## Month Boundary Helpers

```typescript
import { getMonthBoundaries } from "~/utils/dateUtils";

const { startDate, endDate } = getMonthBoundaries(2024, 1);
// startDate: "2024-01-01"
// endDate: "2024-01-31"
```

## Quick Reference

| Task | Function | Import |
|------|----------|--------|
| Current timestamp for DB | `toDBTimestamp()` | `~/utils/dateUtils` |
| Today's date | `getTodayDate()` | `~/utils/dateUtils` |
| Yesterday's date | `getYesterdayDate()` | `~/utils/dateUtils` |
| Parse date string | `dayjs(str)` | `~/utils/dateUtils` |
| Format for display | `formatDate(str, format)` | `~/utils/dateUtils` |
| Format for DB | `dayjs(str).format("YYYY-MM-DD")` | `~/utils/dateUtils` |
| Check if weekday | `isWeekday(str)` | `~/utils/dateUtils` |
| Month boundaries | `getMonthBoundaries(year, month)` | `~/utils/dateUtils` |
| After termination check | `isAfterTermination(date, termDate)` | `~/utils/employeeUtils` |

## Reference Files

- `src/utils/dateUtils.ts` - All date utilities
- `src/utils/employeeUtils.ts` - Employee date helpers
- `src/types/date.ts` - Date type definitions
- `docs/DATE_SYSTEM.md` - Full documentation
