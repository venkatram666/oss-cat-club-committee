# Club & Committee Management Portal

A web portal for managing the clubs and committees of a college: members, meetings and minutes, attendance, budgets and expenses, recruitment drives, events, announcements and reports.

**Stack:** PHP 8.2+ (server-rendered pages, sessions) · MongoDB · Bootstrap 5 · vanilla JavaScript · Chart.js
**No** MySQL, no JavaScript framework, no Node.js. Bootstrap, Bootstrap Icons and Chart.js are bundled in `public/assets/vendor`, so the portal also works without internet.

---

## Contents

1. [Features](#1-features)
2. [Roles and permissions](#2-roles-and-permissions)
3. [Project structure](#3-project-structure)
4. [Installation on Windows (step by step)](#4-installation-on-windows-step-by-step)
5. [Running the tests](#5-running-the-tests)
6. [MongoDB transactions and replica sets](#6-mongodb-transactions-and-replica-sets)
7. [Troubleshooting](#7-troubleshooting)
8. [Deploying to a production server](#8-deploying-to-a-production-server)
9. [Security features](#9-security-features)
10. [Feature checklist and known limitations](#10-feature-checklist-and-known-limitations)

More detail: [`docs/DATABASE.md`](docs/DATABASE.md) (collections, relationships, indexes) and [`docs/TESTING.md`](docs/TESTING.md) (manual test plan).

---

## 1. Features

| Module | What it does |
|---|---|
| **Authentication** | Login / logout, `password_hash()` + `password_verify()`, session id regenerated on login, idle timeout, login rate limiting (5 failures per e-mail / 20 per IP in 15 min), optional student self-registration, forgot-password by e-mail (only when SMTP is configured), administrator created only from the command line. |
| **Dashboard** | Clubs, active members, upcoming meetings/events, pending applications and approvals, income/expenses this month, 6-month income vs expense chart, attendance chart, budget utilization by club, upcoming deadlines, recent activity. Every number comes from MongoDB `count` / `aggregate`. Different content per role. |
| **Clubs** | Create / edit / activate / deactivate (soft delete), logo upload, category, establishment date, contact details, officer assignment, club-specific custom member fields, search + filters + pagination, club registration requests approved by the admin. |
| **Memberships** | Applications with custom fields, approve / reject, statuses (pending, active, inactive, suspended, former, rejected), several clubs per student, full membership history, add member by e-mail, searchable table, CSV export. |
| **Meetings & minutes** | Scheduling, agenda, location, organizer, status, minutes (discussion, decisions), action items with responsible member and deadline, draft → final (locked) minutes, document uploads, audit history. |
| **Attendance** | Meetings and events, Present / Absent / Late / Excused, “mark all” helper, percentage per member and per club, individual reports, duplicate prevention (unique index), corrections written to the audit log. |
| **Budget & finance** | Annual and event budgets with categories, income, expenses, reimbursements (with receipts), funding requests, approval workflows, no self-approval, receipt/invoice uploads, per-category spending, utilization chart. Money is stored as integer paise. |
| **Recruitment** | Drives with open/close dates, eligibility, positions, custom questions; applications, shortlisting, interview scheduling, scores and private evaluation notes, selection automatically creates the membership, statistics. |
| **Events** | Create/edit, capacity (atomic seat reservation – never overbooked), registration deadline, members-only or open-to-all, registration/cancellation, event attendance. |
| **Announcements & notifications** | Club or portal-wide announcements, officers-only audience, publish and expiry dates (scheduled announcements), in-app notification bell, optional e-mail via SMTP. |
| **Reports** | Clubs, memberships, attendance, meetings, action items, budgets, expenses, recruitment, events — each with club/date/status/text filters, pagination, CSV export and print layout. |
| **Administration** | Users (create, edit, reset password, activate/deactivate), settings (portal name, categories, page size, registration on/off, e-mail on/off), audit log viewer. |

## 2. Roles and permissions

There are two kinds of roles:

* **Portal role** (stored in `users.role`): `admin` or `user`.
* **Club role** (stored in `memberships.role`, per club): `president`, `secretary`, `treasurer`, `member`.
  One student can be President of one club and a normal Member of another.

| Permission | President | Secretary | Treasurer | Member |
|---|:-:|:-:|:-:|:-:|
| View club, meetings, events | ✔ | ✔ | ✔ | ✔ |
| Edit club details, custom fields | ✔ | | | |
| View members list | ✔ | ✔ | ✔ | |
| Approve applications, change member status, assign secretary/treasurer | ✔ | | | |
| Schedule meetings, minutes, action items, documents | ✔ | ✔ | | |
| Mark / correct attendance | ✔ | ✔ | | |
| View finance | ✔ | | ✔ | |
| Create budgets, record income/expenses, funding requests | | | ✔ | |
| Approve expenses, reimbursements, event budgets | ✔ | | | |
| Request reimbursement | ✔ | ✔ | ✔ | ✔ |
| Recruitment, events, announcements | ✔ | ✔ | | |
| Reports | ✔ | ✔ | ✔ (finance & member reports) | |

The **Administrator** can do everything in every club, plus users, settings, audit logs, club approval, annual budgets and funding requests.

Rules enforced on the server:
* Every controller action calls `Authz::authorize(permission, club_id)` **using the club of the record from the database**, so changing an id in the URL or a hidden form field never gives access to another club (returns 403/404).
* Nobody can approve their own financial request or their own budget (checked twice: in PHP and in the MongoDB update filter).
* Permissions only come from **active** memberships of **active** clubs.

## 3. Project structure

```
club-committee-portal/
├── public/                 <- the ONLY folder the web server exposes
│   ├── index.php           front controller (all pages: index.php?r=route)
│   ├── login.php, dashboard.php   short URLs
│   └── assets/ css, js, images, vendor (Bootstrap, Icons, Chart.js)
├── app/
│   ├── bootstrap.php       loads Composer, .env, error handling
│   ├── Controllers/        one controller per module
│   ├── Models/             one class per MongoDB collection
│   ├── Services/           Database, Authz, FinanceService, AttendanceService,
│   │                       ReportService, UploadService, Mailer, Notifier, AuditLogger, RateLimiter
│   ├── Middleware/         Auth (sessions), Csrf
│   ├── Validators/         Validator (server-side validation rules)
│   ├── Helpers/            functions.php (e(), url(), money() ...), Env, Money
│   ├── Core/               Router, View, HttpException
│   └── Views/              PHP templates (layouts/, partials/, one folder per module)
├── config/                 app.php, database.php (read values from .env)
├── routes/web.php          list of all routes and their access level
├── includes/               (note only – shared templates are in app/Views)
├── storage/
│   ├── logs/               app-YYYY-MM-DD.log, php-errors.log
│   └── private_uploads/    logos, documents, receipts (served only through PHP)
├── scripts/                create_indexes.php, create_admin.php, seed.php, send_scheduled.php
├── tests/                  run_tests.php, http_smoke_test.php, http_workflow_test.php
├── docs/                   DATABASE.md, TESTING.md
├── vendor/                 Composer packages (already included)
├── .env.example, .gitignore, composer.json, README.md
```

## 4. Installation on Windows (step by step)

> Commands are for **Command Prompt** (cmd) opened in the project folder, e.g. `C:\projects\club-committee-portal`.

### Step 1 – Install PHP 8.2 or newer

**Option A – plain PHP (recommended for development)**
1. Download *PHP 8.2/8.3 VS16 x64 **Thread Safe*** zip from <https://windows.php.net/download/>.
2. Extract to `C:\php`.
3. Copy `C:\php\php.ini-development` to `C:\php\php.ini`.
4. Add `C:\php` to the Windows **PATH** (Start → “Edit the system environment variables” → Environment Variables → Path → New).
5. Open a new cmd window: `php -v`

**Option B – XAMPP** (includes PHP 8.2 and Apache): install XAMPP 8.2 from <https://www.apachefriends.org>. PHP is in `C:\xampp\php`; add that folder to PATH. MongoDB is installed separately (step 2).

### Step 2 – Install MongoDB

**Option A – MongoDB Community Server (local)**
1. Download the MSI from <https://www.mongodb.com/try/download/community>.
2. Choose *Complete* and keep **“Install MongoD as a Service”** ticked. (MongoDB Compass is optional but useful for looking at the data.)
3. The server now runs at `mongodb://127.0.0.1:27017`.

**Option B – MongoDB Atlas (cloud, free tier)**
1. Create a free cluster at <https://www.mongodb.com/atlas>.
2. *Database Access*: create a database user + password.
3. *Network Access*: add your current IP address.
4. *Connect → Drivers*: copy the `mongodb+srv://...` connection string (you will paste it into `.env`).
5. Atlas needs the PHP `openssl` extension (step 4).

### Step 3 – Install Composer

Download and run **Composer-Setup.exe** from <https://getcomposer.org/download/>. It finds `php.exe` automatically. Check with `composer -V`.

### Step 4 – Enable the MongoDB PHP extension

1. Check your PHP build: `php -i | findstr /C:"Thread Safety" /C:"Architecture" /C:"PHP Version"`
2. Download the matching DLL zip from <https://github.com/mongodb/mongo-php-driver/releases> (version **1.20 or newer**; e.g. `php_mongodb-1.20.1-8.2-ts-vs16-x64.zip` for PHP 8.2 Thread Safe x64; `nts` = Non Thread Safe).
3. Copy `php_mongodb.dll` into the `ext` folder (`C:\php\ext` or `C:\xampp\php\ext`).
4. Open `php.ini` and add / uncomment these lines:
   ```ini
   extension_dir = "ext"
   extension=mongodb
   extension=fileinfo
   extension=openssl
   extension=curl
   extension=mbstring
   ; optional, bigger uploads
   upload_max_filesize = 8M
   post_max_size = 10M
   ```
5. Check: `php -m | findstr mongodb` → prints `mongodb`.

> With XAMPP restart Apache after changing `php.ini` (XAMPP Control Panel → Stop/Start).

### Step 5 – Install the Composer dependencies

The `vendor/` folder is already included, so this step is only needed if you delete it or want to update:

```cmd
composer install
```

This installs `mongodb/mongodb` (the official MongoDB library) and creates the autoloader.

### Step 6 – Configure the environment

```cmd
copy .env.example .env
notepad .env
```

Change at least:

```ini
MONGO_URI=mongodb://127.0.0.1:27017        ; or your Atlas mongodb+srv://... string
MONGO_DB=club_portal
APP_URL=http://localhost:8000
APP_TIMEZONE=Asia/Kolkata
```

SMTP settings are optional (needed for forgot-password e-mails and e-mail notifications). For Gmail use `SMTP_HOST=smtp.gmail.com`, `SMTP_PORT=587`, `SMTP_ENCRYPTION=tls` and an **App Password**.

### Step 7 – Create collections, validation rules and indexes

```cmd
php scripts/create_indexes.php
```

It is safe to run again any time.

### Step 8 – Create the first administrator (securely)

```cmd
php scripts/create_admin.php
```

You type the name, e-mail and password interactively (the password is never written in any file). There is **no** web page for creating administrators.

### Step 9 (optional) – Load sample data

```cmd
php scripts/seed.php
```

Enter a password for all demo accounts (e.g. `Demo12345`). It creates 12 students, 4 clubs, meetings with minutes, attendance, budgets, transactions, a recruitment drive, events and announcements. Demo logins:

| E-mail | Role |
|---|---|
| aarav@demo.example.edu | Coding Club President |
| diya@demo.example.edu | Coding Club Secretary |
| kabir@demo.example.edu | Coding Club Treasurer |
| sneha@demo.example.edu | Member of Coding & Music clubs |
| meera@demo.example.edu | Music Club President |
| ananya@demo.example.edu | Student without a club (requested the Literary Society) |

`php scripts/seed.php --fresh` deletes all portal data (administrator accounts are kept) and loads it again. The seeder refuses to run when `APP_ENV=production`.

### Step 10 – Start the application

**PHP built-in server (easiest):**

```cmd
php -S localhost:8000 -t public
```

Open <http://localhost:8000> and log in.

**XAMPP / Apache:** point the document root at the `public` folder (never at the project root). Add to `C:\xampp\apache\conf\extra\httpd-vhosts.conf`:

```apache
<VirtualHost *:8080>
    DocumentRoot "C:/projects/club-committee-portal/public"
    <Directory "C:/projects/club-committee-portal/public">
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

Add `Listen 8080` to `httpd.conf`, restart Apache, open <http://localhost:8080> and set `APP_URL=http://localhost:8080` in `.env`.

### Optional – scheduled announcements

Scheduled announcements are sent automatically when someone opens the Announcements page. To send them exactly on time, create a Windows **Task Scheduler** task that runs every 15 minutes:

```cmd
php C:\projects\club-committee-portal\scripts\send_scheduled.php
```

## 5. Running the tests

| Test | Command | What it checks |
|---|---|---|
| Unit + database tests | `php tests/run_tests.php` (or `composer test`) | Money maths, validation, permissions matrix, club isolation, unique indexes (no duplicate membership/attendance/registration/application), finance formulas, no self-approval, double-approval protection, event capacity, upload checks. Uses and **empties** the `MONGO_DB_TEST` database. |
| HTTP smoke test | `php tests/http_smoke_test.php --password=Demo12345 --admin=YOUR_ADMIN_EMAIL --admin-password=YOUR_ADMIN_PASSWORD` | Logs in as every role, crawls every reachable page (no PHP errors), checks 403s for other clubs' records, CSRF, file download permissions, minutes locking, duplicate attendance, membership approval, CSV export, logout. |
| HTTP workflow test | `php tests/http_workflow_test.php --password=Demo12345 --admin=... --admin-password=...` | Submits the real forms of every module: clubs, users, members, budgets, income/expense/approvals and balance maths, recruitment to selection, event capacity, attendance corrections, announcements, profile, settings, registration, login lockout. |

For the HTTP tests: run `php scripts/seed.php --fresh --password=Demo12345` first, keep `php -S localhost:8000 -t public` running in a second window, and enable the `curl` extension. Run the smoke test and the workflow test each on freshly seeded data (the workflow test changes some passwords and settings).

**Results of the last run while building the project:** unit/DB tests 25/25 passed, HTTP smoke test 62/62 passed (≈330 pages crawled as six different users with no errors), HTTP workflow test 73/73 passed. See [Notes on the test environment](#notes-on-the-test-environment) below.

A manual test plan for the parts that need a person (e-mail sending, printing, mobile layout, etc.) is in [`docs/TESTING.md`](docs/TESTING.md).

## 6. MongoDB transactions and replica sets

Approving or rejecting a financial request / budget changes the record **and** writes an audit-log entry. On a **replica set** (MongoDB Atlas always is one) these two writes run inside a real **multi-document transaction** (`Database::transaction()`), so they succeed or fail together.

A default local Windows installation is a **standalone** server, which does not support transactions. The portal detects this automatically and still stays consistent because:

* every approval uses a **conditional update** – `status: 'pending'` and `requested_by ≠ me` are part of the update filter – so a request can never be approved twice or approved by its requester, even if two people click at the same time;
* event seats are reserved with one atomic `findOneAndUpdate` (`seats_left > 0`), so capacity can never be exceeded;
* unique indexes block duplicate memberships, attendance rows, registrations and applications.

To get real transactions locally, turn your local server into a one-node replica set:

1. Edit `C:\Program Files\MongoDB\Server\<version>\bin\mongod.cfg` and add:
   ```yaml
   replication:
     replSetName: rs0
   ```
2. Restart the *MongoDB* service (services.msc).
3. In `mongosh` run: `rs.initiate()`
4. Use `MONGO_URI=mongodb://127.0.0.1:27017/?replicaSet=rs0` in `.env`.

*Admin → Settings → System information* shows whether transactions are available.

## 7. Troubleshooting

| Problem | Fix |
|---|---|
| `The MongoDB PHP extension is not enabled` | `php_mongodb.dll` missing or wrong build (TS/NTS, x64, PHP version). Check `php --ini` to see which php.ini is used and that it contains `extension=mongodb`. |
| `Class "MongoDB\Driver\Manager" not found` | Same as above – the extension is not loaded. |
| `Composer dependencies are missing` | Run `composer install` in the project folder. |
| `Cannot connect to MongoDB` / page says “database is not available” | Start the MongoDB service (`services.msc` → MongoDB Server → Start). For Atlas: check the user/password in `MONGO_URI` (URL-encode special characters such as `@` → `%40`) and that your IP is in *Network Access*. |
| `No suitable servers found` with Atlas | Enable `extension=openssl`; check your firewall allows port 27017. |
| “Your form expired” (419) | The session expired or cookies are blocked. Refresh the page and try again. |
| Logo / receipt upload fails | Enable `extension=fileinfo`; increase `upload_max_filesize` / `post_max_size`; check that `storage/private_uploads` is writable. |
| Login says “Too many failed attempts” | Wait 15 minutes (or delete documents from the `login_attempts` collection in Compass). |
| Forgot password says “contact the administrator” | SMTP is not configured. The admin can set a new password from *Users → Edit*. |
| Blank page / “unexpected error” | Read `storage/logs/app-YYYY-MM-DD.log` and `storage/logs/php-errors.log`. Set `APP_DEBUG=true` temporarily on your own computer to see the message. |
| Times are wrong | Set `APP_TIMEZONE` in `.env` (dates are stored in UTC and shown in this zone). |
| Port 8000 in use | Use another port: `php -S localhost:8080 -t public` and update `APP_URL`. |

## 8. Deploying to a production server

1. Use **HTTPS** only (e.g. Let's Encrypt). Set `SESSION_SECURE_COOKIE=true`.
2. Set `APP_ENV=production` and `APP_DEBUG=false`; set `APP_URL` to the real https address.
3. The web server's document root must be the **`public/`** folder. `app/`, `config/`, `storage/`, `vendor/`, `.env` must not be reachable (they also contain deny-all `.htaccess` files as a safety net).
4. Run `composer install --no-dev --optimize-autoloader`.
5. Run `php scripts/create_indexes.php` and `php scripts/create_admin.php` on the server. **Do not** run the seeder.
6. MongoDB: enable authentication, create a user with access only to the portal database, never expose port 27017 to the internet (or use Atlas with IP access list). Prefer a replica set so transactions are used.
7. Give the web-server user write access only to `storage/`.
8. In `php.ini`: `display_errors=Off`, `log_errors=On`, `session.cookie_httponly=1`, `expose_php=Off`.
9. Back up the database regularly: `mongodump --uri="..." --out=backup-YYYYMMDD` and back up `storage/private_uploads`.
10. Schedule `scripts/send_scheduled.php` (cron / Task Scheduler) every 15 minutes.

## 9. Security features

* Passwords: `password_hash()` (bcrypt) / `password_verify()`, automatic rehash, minimum 8 characters with letters and numbers; timing-safe login (dummy hash for unknown e-mails).
* Sessions: HttpOnly + SameSite cookies, strict mode, id regenerated on login and password change, idle timeout, user status re-checked on every request.
* CSRF token on every POST form (checked centrally in the router).
* Output escaped with `e()` (htmlspecialchars) everywhere; strict Content-Security-Policy (no inline scripts), `X-Frame-Options`, `nosniff`.
* MongoDB injection: ids validated with `oid()` (24-hex only), user input read only as strings (`input()` ignores arrays), explicit filters, whitelisted sort fields, regex searches escaped with `preg_quote`.
* Role + club level authorization on every view/edit/delete/export/download (see section 2).
* Uploads: extension whitelist + real MIME check with `finfo`, size limit, random file names, stored outside `public/`, downloaded only through `FileController` after a permission check.
* Login rate limiting and account lock-out window, generic error messages (no account enumeration).
* Audit log of logins, approvals, corrections, exports, downloads, changes.
* Secrets only in `.env`; errors logged to `storage/logs`, users see a friendly message without stack traces.
* CSV exports protected against formula injection.

## 10. Feature checklist and known limitations

**Completed:** all phases 1–8 — project structure, MongoDB connectivity and error handling; authentication and role permissions; dashboard, clubs, users and memberships; meetings, minutes, action items and attendance; budgets, transactions, approvals and financial reports; recruitment, events, registrations, announcements and notifications; analytics, CSV exports, responsive UI and accessibility (skip link, labels, keyboard focus styles, ARIA on charts); security checks, automated tests and documentation.

**Known limitations**
* Forgot-password and e-mail notifications need SMTP settings; without them, notifications stay in-app and the administrator resets passwords.
* Scheduled announcements are dispatched when the Announcements page is opened, or by the optional scheduled script.
* Real multi-document transactions need a replica set (see section 6); on a standalone server the conditional-update safeguards are used instead.
* The bundled `create_admin.php` cannot hide the typed password on Windows cmd (it can on Linux/macOS).

### Notes on the test environment

The project was built and tested on PHP 8.3 with MongoDB PHP extension 1.20.1 and the official `mongodb/mongodb` 1.20 library. The build machine could not download the MongoDB server itself, so the database tests ran against **FerretDB 1.24**, an open-source server that speaks the MongoDB protocol. Because FerretDB does not implement `$jsonSchema` validation (`collMod`) or TTL indexes, those two features were skipped there with a warning by `create_indexes.php`; they are standard MongoDB features and are created normally on MongoDB Community Server / Atlas. Please run `php tests/run_tests.php` once on your own MongoDB installation as well.
