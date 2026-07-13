# Investigation Report: Property Revenue Dashboard Issues

## 1. Data Inaccuracy for Client A (Sunset Properties)
**Issue:** Revenue numbers on the dashboard don't match internal records for March.
**Root Causes:**
- **Missing Timezone Handling:** The system currently doesn't account for property timezones when aggregating data by month. A reservation on "2024-02-29 23:30:00 UTC" for a property in Paris (UTC+1) is locally "2024-03-01 00:30:00", meaning it should count towards March revenue, not February.
- **Incorrect Revenue Function & No Date Filtering:** The `/dashboard/summary` API currently returns all-time total revenue for a property by calling `calculate_total_revenue()`. The `calculate_monthly_revenue()` function exists but is just a placeholder and is never used. Therefore, clients see all-time revenue instead of just the target month (March).

## 2. Privacy/Data Leakage for Client B (Ocean Rentals)
**Issue:** Refreshing the page occasionally shows another company's revenue numbers.
**Root Cause:**
- **Cache Key Collision:** In `backend/app/services/cache.py`, the Redis cache key is generated as `f"revenue:{property_id}"`. It completely omits the `tenant_id`. Because multiple tenants might use identical property IDs (e.g., "prop-001" exists for both Sunset Properties and Ocean Rentals), the cached result for one tenant gets served to the other if they query the same property ID.

## 3. Financial Inaccuracy (Slightly Off Totals)
**Issue:** Finance team noticed revenue totals are slightly off by a few cents.
**Root Cause:**
- **Floating Point Precision Loss:** The database stores `total_amount` as `NUMERIC(10, 3)`. The backend retrieves this as a string, casts it to a Python `float` in `dashboard.py` (`total_revenue_float = float(revenue_data['total'])`), and sends it to the frontend. The frontend (`RevenueSummary.tsx`) tries to round it with `Math.round(data.total_revenue * 100) / 100`. JavaScript's floating-point arithmetic can lose precision on multiplication (e.g., `Math.round(1.005 * 100) / 100` evaluates to `1`, not `1.01`). The rounding to two decimal places (cents) should be performed precisely on the backend using `Decimal` before being cast to a float.
