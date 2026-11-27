# Pawsly access guide: usernames, temporary passwords, links

## The three account types
| Type | Who | Sees | Database role |
|---|---|---|---|
| **Admin** | You | All branches, reports, settlement, branches, QR codes | `hq` |
| **Branch** | Branch manager / venue owner | Their own branch: revenue, their share, order list, QR banner, CSV | `venue` |
| **Staff** | Cashier | Their own branch, **today's orders only**, what to collect at the cashier. No money-share, payout or QR banner | `staff` |

People sign in with a **username** (no email). Behind the scenes the app turns `paws.kasir1` into
`paws.kasir1@staff.pawsly.id`. That address never receives mail; it only exists so Supabase has an id to store.
Suggested names: `admin.dika`, `paws.manager`, `paws.kasir1`, `hq.kasir1` (lower case, no spaces).
Anyone with an older real-email login can still type the full email.

## Links
| Who | Link |
|---|---|
| Admin, Branch, Staff (all the same page; the login decides what they see) | `https://order.pawsly.id/#hq` or `https://order.pawsly.id/#partner` |
| Customers | the printed QR code (`https://order.pawsly.id/?v=BRANCH_CODE`) |

## A. Create an account (about 2 minutes)
1. Supabase > **Authentication** > **Users** > **Add user** > **Create new user**.
2. **Email:** `username@staff.pawsly.id` (for example `paws.kasir1@staff.pawsly.id`).
3. **Password:** your temporary password. **Tick "Auto Confirm User"** so no email is needed. Click Create.
4. Give them access: add one row to the `staff` table (step B).
5. Message them the username, the temporary password and the link.

## B. Give the account its access (SQL editor)
I have not seen your `staff` table, so check its columns first:
```sql
select column_name, data_type from information_schema.columns
where table_schema='public' and table_name='staff';

select conname, pg_get_constraintdef(oid) from pg_constraint where conrelid='public.staff'::regclass;
```
- If the role check only allows `hq` and `venue`, allow the new `staff` role (use the constraint name from the second query):
```sql
alter table public.staff drop constraint staff_role_check;   -- use your real constraint name
alter table public.staff add  constraint staff_role_check check (role in ('hq','venue','staff'));
```
- Add the row. This template assumes the table links to the user with a `user_id` column; change that name if yours differs:
```sql
insert into public.staff (user_id, role, venue_code)
select id, 'staff',  'PAWS_CODE' from auth.users where email = 'paws.kasir1@staff.pawsly.id';

-- branch manager example:
insert into public.staff (user_id, role, venue_code)
select id, 'venue',  'PAWS_CODE' from auth.users where email = 'paws.manager@staff.pawsly.id';

-- admin example (no branch):
insert into public.staff (user_id, role, venue_code)
select id, 'hq', null from auth.users where email = 'admin.dika@staff.pawsly.id';
```
- Your `orders` read policy must also let role `staff` read their own branch's orders (see `SECURITY.md`, section 2).
  If the policy only names `venue`, add `staff`.

Send me the output of the first query and your current `orders` policies and I will write exact SQL for your tables.

## C. Reset a forgotten password (no email involved)
```sql
update auth.users
set encrypted_password = crypt('NEW-TEMP-PASSWORD', gen_salt('bf'))
where email = 'paws.kasir1@staff.pawsly.id';
```
Then send the new temporary password. They can change it again with the **Password** button.

## D. Remove someone
Delete their row in `staff`, then delete the user under Authentication > Users. Do both the same day they leave.

## E. What staff do
1. Open the link, enter username and temporary password.
2. Tap **Password** (top right) > current password > new password (8+ characters) > Save. Optional but recommended.
3. Keep the page open. New orders appear within 20 seconds. Signs out after 30 minutes of no use.

## F. One-time Supabase settings
- Authentication > Providers > Email: turn **off** "Allow new users to sign up".
- Because you create users with Auto Confirm, no emails are sent. The "forgot password by email" feature will not work for username accounts; use step C.
