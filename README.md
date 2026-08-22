# ODOO Dayflow — Human Resource Management System

A fully client-side, single-file Human Resource Management System (HRMS) built as a hackathon prototype. The entire application — routing, state management, UI rendering, and data persistence — lives inside one HTML file (`hackthon.html`, ~4,743 lines) with no build step, no backend, and no external database.

---

## 1. Overview

**Dayflow** simulates a complete corporate HR platform covering attendance, leave management, expense reimbursement, payroll, team directory, task tracking, analytics, and system auditing — with two distinct experiences depending on whether the logged-in user is an **Admin (HR)** or a **Staff Employee**.

- **Type:** Single Page Application (SPA)
- **Architecture:** Vanilla JavaScript, hand-rolled virtual "router" + string-template rendering (no React/Vue/Angular)
- **Persistence:** Browser `localStorage` (no server, no real database)
- **Currency/Locale:** Indian Rupees (₹), Bengaluru-based fictional company
- **Simulated system clock:** Hardcoded to `2026-08-22T09:33:11` for consistent demo data

---

## 2. Tech Stack

| Layer | Technology | Delivery |
|---|---|---|
| Styling | Tailwind CSS (JIT via CDN) | `cdn.tailwindcss.com` |
| Fonts | Google Fonts — Inter & Outfit | `fonts.googleapis.com` |
| Icons | Lucide Icons | `unpkg.com/lucide@latest` |
| Charts | Chart.js | `cdn.jsdelivr.net/npm/chart.js` |
| PDF Export | jsPDF + jsPDF-AutoTable | `cdn.jsdelivr.net/npm/jspdf` |
| Email (simulated) | EmailJS | `cdn.jsdelivr.net/npm/@emailjs/browser` |
| State/Logic | Vanilla JS (`window.app` global object) | Inline `<script>` |
| Storage | `localStorage` (`df_*` keys) | Browser-native |

**No frameworks, no npm, no bundler** — everything runs directly in the browser from a single HTML file, making it fully portable and easy to demo (just double-click and open, or host as a static file).

---

## 3. Core Architecture

### 3.1 The `window.app` Object
Everything is centralized in one global object, `window.app`, which acts as:
- A **state container** (`app.state`) holding users, tasks, leaves, attendance, expenses, payroll, notifications, announcements, holidays, and audit logs.
- A **router** (`app.navigate(route)`) that swaps `currentRoute` and re-renders the relevant view.
- A **render engine** — each "page" is a function (`renderDashboard`, `renderAttendance`, `renderLeaves`, etc.) that returns an HTML string, injected into the DOM.
- An **action layer** — functions like `applyLeave()`, `processMonthlyPayroll()`, `login()`, `verifyCode()` mutate state and re-render.

There is no virtual DOM diffing — views are re-rendered wholesale as raw HTML strings on every state change (typical of a rapid hackathon build, not production-grade).

### 3.2 Routing
A simple `switch` statement drives view selection based on `state.currentRoute`:

```
dashboard | attendance | leaves | expenses | profile | team | payroll | settings | analytics | tasks | audit
```

### 3.3 Persistence Layer
The `persist()` method writes selected state slices to `localStorage` under these keys:
`df_users`, `df_leaves`, `df_attendance`, `df_expenses`, `df_announcements`, `df_auditLogs`, `df_companySettings`, `df_notifs_prefs`, `df_tasks`.

Note: `payrollRuns`, `notificationsFeed`, and `holidays` are **not persisted** — they reset to seed data (`INITIAL_*` constants) on every page reload.

---

## 4. Authentication System

A simulated (client-side only, insecure-by-design) auth flow:

- **Sign In** — matches plaintext email/password against the in-memory `users` array. There is a CAPTCHA-style check before login/signup submission.
- **Sign Up** — collects Employee ID, name, email, role, and password; validates:
  - Email uniqueness
  - Employee ID uniqueness
  - Password strength (length ≥ 8, uppercase, number, special character) with a live strength meter
  - Password confirmation match
- **Email Verification** — generates a random 6-digit code, "sends" it via EmailJS (or logs it to console if EmailJS keys aren't configured), and requires entry of that code (or the universal bypass code `123456`) to activate the account.
- **Demo credentials autofill** — `fillDemoCredentials('admin' | 'employee')` pre-fills known demo logins:
  - Admin: `admin@dayflow.com` / `password`
  - Employee: `marcus@dayflow.com` / `password`
- **Logout** clears the current session and impersonation state.

⚠️ **Security note:** Passwords are stored and compared in plaintext in client-side JS/localStorage. This is appropriate only for a demo/prototype — never for production.

---

## 5. Role-Based Views (Admin vs. Employee)

Nearly every page checks `isAdmin = currentUser.role === 'admin' && !impersonatedUserId` and renders a different UI/dataset accordingly:

| Module | Employee View | Admin View |
|---|---|---|
| **Dashboard** | Personal stats, quick actions (profile, punch time, file expense) | Org-wide stats, pending approvals queue, team activity |
| **Attendance** | Own punch history | Master ledger of all employees + manual punch entry |
| **Leaves** | Apply for leave, view own quota/status | Approve/reject requests with admin comments |
| **Expenses** | File claims, view own reimbursement status | Audit all claims, bulk-disburse approved ones |
| **Payroll** | View own payslips (itemized, downloadable PDF) | CTC master register, run monthly payroll for all staff |
| **Team** | View directory/org chart | Add members, manage org chart |
| **Settings** | Personal preferences | Company-wide policy settings (office hours, grace period, etc.) |
| **Analytics** | — | Charts: headcount, attrition/growth trends (Chart.js) |
| **Audit Logs** | — | Full system action log (logins, approvals, edits) with PDF export |

### Admin "Impersonation" (View as Employee)
Admins can select any employee from a dropdown to **view the app as that employee** (`impersonatedUserId`). This swaps the effective "active user" everywhere without logging out, useful for support/demo purposes. A banner (`Viewing {name}`) indicates impersonation is active.

---

## 6. Feature-by-Feature Breakdown

### 6.1 Dashboard
Role-aware landing page with quick-action buttons, summary cards, and (for admins) a live feed of pending leave/expense approvals.

### 6.2 Attendance
- Employees see their punch/clock history.
- Admins get a **Manual Attendance Punch** modal to log entries on behalf of staff.
- Filterable by status, search query, and date (`attendanceFilterStatus`, `attendanceSearchQuery`, `attendanceDateFilter`).
- Company-wide rules configurable in Settings: office start time, grace period (minutes), half-day cutoff, lunch break duration.
- Exportable to PDF via jsPDF-AutoTable.

### 6.3 Leaves
- Employees apply for time off via a modal (`openApplyLeaveModal`) — captures leave type and duration.
- Admins review/approve/reject with an optional comment field (`review-comment` textarea).
- Leave balances tracked per type (e.g., Paid Leave) with approved-days aggregation.
- PDF export of leave records.

### 6.4 Expenses
- Employees file claims by category (travel, meals, internet, office supplies, etc.) in ₹.
- Admins audit, approve/reject, and **bulk-reimburse** approved claims (`bulkReimburseExpenses`).
- Filterable by status/category/search.
- PDF export of the expense ledger.

### 6.5 Payroll
- Admins can **Run Monthly Payroll** (`processMonthlyPayroll`) to generate payslips for all staff based on CTC, PF, and TDS deductions.
- Employees view historical payslips and can export any month as a PDF (`exportPayslipPDF`).
- Salary structure fields include gross CTC, deductions, and net pay (INR formatted via `formatINR`).

### 6.6 Team Directory & Org Chart
- Searchable/filterable employee directory (by name, department).
- Visual org chart rendering (`renderOrgNode`) rooted at the admin/HR head.
- Admins can add new team members (`openAddMemberModal`).

### 6.7 Tasks
A lightweight task board/list (`renderTasks`, `INITIAL_TASKS`) with a "New Task" creation modal for assigning and tracking work items.

### 6.8 Analytics (Admin only)
Chart.js-powered dashboards, including:
- Headcount chart (`initCharts`)
- Customer/organization growth trend (`initCustomerGrowthChart`)

### 6.9 Announcements
Company-wide announcement board; admins can post new announcements (`openPostAnnouncementModal`); visible to all users on the dashboard/sidebar.

### 6.10 Notifications
- **Toast notifications** — ephemeral popups (`notify()`) for success/error/info feedback, auto-dismiss after 4 seconds.
- **Persistent notification feed** — a bell-icon dropdown log of events (`addNotificationFeed`), with per-category preferences (leave updates, expense updates, payroll alerts, task alerts) configurable in Settings.

### 6.11 Audit Logs (Admin only)
Every significant action (login, approval, edit) is recorded via `addAuditLog(action, details, user)` with a timestamp, and displayed in a searchable, PDF-exportable table.

### 6.12 Command Palette
A keyboard-driven quick-navigation overlay (`cmd-input`, `filterCommands`) — likely triggered by a keyboard shortcut — lets users search commands, pages, and employees, similar to Spotlight/Cmd+K patterns in modern SaaS apps (e.g. Linear, Notion).

### 6.13 Theming
Light/dark mode toggle (`toggleTheme`) using Tailwind's `class`-based dark mode strategy; preference is applied to `<html class="dark">`.

### 6.14 PDF Export Engine
Three dedicated export functions built on jsPDF + AutoTable:
- `exportTablePDF(title, type)` — generic ledger exporter (attendance, leaves, expenses, audit logs)
- `exportPayslipPDF(userName, month)` — formatted individual payslip document

### 6.15 Simulated Email Notifications
`sendEmailNotification()` wraps EmailJS. Since the demo ships with placeholder credentials (`YOUR_SERVICE_ID`, etc.), it gracefully **no-ops and logs to console** instead of actually sending email — meaning verification codes are surfaced via toast notification instead, keeping the demo fully functional offline.

---

## 7. Data Model (Seed Data)

All seed/mock data is defined as top-level constants before the app initializes:

| Constant | Purpose |
|---|---|
| `INITIAL_USERS` | Employee/admin records (name, email, password, role, department, designation, salary, join date, avatar, etc.) |
| `INITIAL_TASKS` | Seed task board items |
| `INITIAL_LEAVES` | Sample leave applications |
| `INITIAL_ATTENDANCE` | Sample punch records |
| `INITIAL_EXPENSES` | Sample expense claims |
| `INITIAL_PAYROLL_RUNS` | Sample payroll run history |
| `INITIAL_NOTIFICATIONS` | Seed notification feed |
| `INITIAL_ANNOUNCEMENTS` | Seed company announcements |
| `INITIAL_HOLIDAYS` | Company holiday calendar |
| `INITIAL_AUDIT_LOGS` | Seed audit trail |

On load, the app checks `localStorage` first and falls back to these constants — so a fresh browser/incognito session always resets to a clean demo state.

---

## 8. Known Limitations (as a prototype)

- **No real backend/database** — all data lives in `localStorage`, scoped to a single browser; nothing syncs across devices or users.
- **Plaintext password storage** — not suitable for production without a real auth/identity provider.
- **No real email delivery** unless valid EmailJS credentials are supplied (currently placeholders).
- **`payrollRuns`, `notificationsFeed`, `holidays` are not persisted** — they reset on refresh, which may look like a bug in a demo (e.g., a freshly run payroll disappearing after reload).
- **Full-string re-rendering** on every state change — fine for a demo, but not optimized for larger datasets or production scale.
- **Hardcoded system date** (`2026-08-22`) — useful for consistent demo screenshots, but will need to become dynamic (`new Date()`) for real-world use.
- **No input sanitization visible** for HTML injected via template strings — potential XSS risk if this were exposed to untrusted multi-tenant input in production.

---

## 9. How to Run

No installation required:

1. Download `hackthon.html`.
2. Open it directly in any modern browser (Chrome/Edge/Firefox), or serve it via any static file server.
3. Log in with a demo account:
   - **Admin:** `admin@dayflow.com` / `password`
   - **Employee:** `marcus@dayflow.com` / `password`
4. To reset all demo data, clear the site's `localStorage` (DevTools → Application → Local Storage) or open in a fresh Incognito window.

**Optional — enabling real email verification:**
Replace the placeholder constants near the top of the script with real [EmailJS](https://www.emailjs.com/) credentials:
```js
const EMAILJS_SERVICE_ID = 'YOUR_SERVICE_ID';
const EMAILJS_TEMPLATE_ID = 'YOUR_TEMPLATE_ID';
const EMAILJS_PUBLIC_KEY = 'YOUR_PUBLIC_KEY';
```

---

## 10. Suggested Next Steps (if evolving beyond a hackathon demo)

- Replace `localStorage` with a real backend (e.g., Node/Express + PostgreSQL, or Firebase/Supabase) and proper session-based or JWT authentication.
- Hash passwords server-side; never store/compare plaintext credentials client-side.
- Persist `payrollRuns`, `notificationsFeed`, and `holidays` consistently, or move all state to server-side storage.
- Replace the current large-string re-render pattern with a component framework (React/Vue) for maintainability and performance as the codebase grows.
- Add role-based API authorization (not just UI-level `isAdmin` checks) so permissions can't be bypassed by manipulating client state.
- Make the system clock dynamic (`new Date()`) instead of hardcoded.
- Sanitize/escape any user-supplied text injected into HTML templates to prevent XSS.

---

*This README was generated from a direct code analysis of `hackthon.html` — a single-file demo of "ODOO Dayflow," an HR Management System prototype covering authentication, attendance, leave, expenses, payroll, team management, tasks, analytics, and audit logging, built with vanilla JS, Tailwind CSS, Chart.js, and jsPDF.*

