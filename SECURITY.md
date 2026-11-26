# Pawsly: security notes for multi-branch + venue-owner access

`supabase-setup.sql` was not in the zip, so **I could not check your database rules (RLS)**.
The browser code can hide buttons, but only the database can actually stop someone.
Please run the checks in section 2 before giving a venue owner a login.

## 1. What changed in the code (index.html)
- Staff sessions sign out after 30 minutes of inactivity (shared tablets at branches).
- The public sign-in page no longer shows setup hints or the "use a different database" button.
- Connection diagnostics on the public page only appear with `#debug` in the URL.
- CSV exports are protected against spreadsheet formula injection.
- Branch limit (`MAX_BRANCHES = 5`) in the HQ console. This is a convenience, not a security control.
- Phone numbers were already masked on dashboards (`phone_mask`). Keep it that way.

## 2. Check these in Supabase (SQL editor, read-only)
```sql
-- RLS must be ON for every table
select tablename, rowsecurity from pg_tables where schemaname='public';

-- Read every policy and confirm the rules below
select tablename, policyname, cmd, roles, qual, with_check
from pg_policies where schemaname='public' order by tablename, cmd;

-- Who can read the customer phone column?
select grantee, privilege_type from information_schema.column_privileges
where table_schema='public' and table_name='orders' and column_name='phone';
```

**Rules the policies must enforce (the app UI already assumes them):**
1. Role `venue`: can SELECT only orders where `venue_code` = their own `staff.venue_code`. No other branch.
2. Role `venue`: **no UPDATE or DELETE** on `orders` or `venues`. Marking payments received, settling,
   changing revenue share and deleting branches must be HQ-only.
3. Role `venue`: must **not** be able to read `orders.phone`. The app never requests it, but a venue login can call the API
   directly. Revoke column SELECT on `phone` for `authenticated`, or keep phones in a separate table that only the
   WhatsApp worker (service role) can read.
4. Role `hq`: full access. Nobody else.
5. `staff` table: users can read only their own row; nobody can insert or edit it except you in the dashboard.

## 3. Biggest item: `place_order` must not trust the browser
The page sends `subtotal`, `total`, `rate` and `commission` to the database. Anyone can call the API directly with
different numbers (for example Rp 1 total, or a 90% commission). Inside the `place_order` function:
- Recompute prices from a server-side price table (or at least reject totals that do not match).
- Take `rate` and `commission` from `venues.rate`, not from the payload.
- Reject inactive branches, and reject a payment method that the branch does not offer
  (Paws = cashier only). The branch rule in the page (`BRANCH_PAYMENTS`) is for convenience only.
- Rate-limit it (Cloudflare in front of the site is the easy way).
- Confirm it accepts `payment = 'cash'` / `collect = 'venue'` (new "Pay at cashier" orders use these).

## 4. Accounts for the venue owner
- Turn **off** public sign-ups (Authentication > Providers > Email > disable "Allow new users to sign up"). Create accounts yourself.
- One login per person, never shared. Add them in `staff` with role `venue` and their branch code.
- Turn on MFA for your own HQ account, and enable leaked-password protection.
- Never put the `service_role` / `sb_secret_` key in `config.js` or any web page. Only the publishable key belongs there.
- Remove a person's `staff` row the day they leave.
- Customer name and WhatsApp number are personal data (Indonesia's PDP law applies). Agree in writing with the venue
  owner what they may see (sales totals: yes; phone numbers: no) and tell customers what the number is used for.
- Consider an audit log (who changed a rate, who marked something settled) before the venue owner gets write-adjacent screens.

## 5. Quick test before go-live
1. Sign in as a test `venue` user for Paws. Confirm you see only Paws orders, no other branch.
2. While signed in as that user, try (in the browser console or an API tool) to PATCH an order's `settled`,
   change `venues.rate`, or select `phone`. **All three must be refused.**
3. As a customer (not signed in), try to read `orders` and `staff`. Both must return nothing.
