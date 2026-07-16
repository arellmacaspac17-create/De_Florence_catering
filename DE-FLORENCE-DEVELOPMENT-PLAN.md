# DE FLORENCE CATERING SERVICES
## Web-Based Catering Management and Online Reservation System with Event Scheduling
### Master Development Plan — v1.0 (July 2026)

---

## 1. PROJECT OVERVIEW

| Item | Detail |
|---|---|
| Client | De Florence Catering Services, San Fernando, Pampanga |
| Stack | PHP 8+, MySQL, HTML5, CSS3, JavaScript, Bootstrap 5 |
| Local Dev | XAMPP + Visual Studio Code |
| Deployment | Hostinger |
| Methodology | Agile Scrum (per Chapter III) |
| Quality Model | ISO/IEC 25010 (usability, reliability, security, performance) |

---

## 2. USER ROLES

| Role | Access |
|---|---|
| **Guest** | Public pages only (Home, Menu, Packages, Gallery, Contact) |
| **Customer** | Register/login, book reservations, pick menu items, upload proof of payment, view booking status, cancel/rebook, receive notifications |
| **Staff** | View/manage bookings, verify payments, update booking status, view calendar |
| **Owner (Admin)** | Everything Staff can do + manage menu/packages/prices, manage staff accounts, view reports & dashboard summaries |

---

## 3. BUSINESS RULES (HARD REQUIREMENTS)

These rules must be **enforced by code**, not just displayed as text:

1. **Maximum 3 reservations per day** — the calendar must block a date once 3 confirmed bookings exist.
2. **₱10,000 reservation fee** — non-refundable, required to lock the date.
3. **50% down payment** due 1 month before the event date.
4. **Full payment** due 1 week before the event date.
5. **Party tray orders:** minimum order of ₱20,000.
6. **Catering packages:** Basic ₱750 / Standard ₱900 / Premium ₱1,100 per pax, **minimum 100 pax**.
7. **No refunds.** Cancellation = rebook to a new date; payments already made carry over to the new booking or can be converted to party trays.
8. **No double booking** — same date cannot exceed the limit; system must prevent race conditions (two customers booking the last slot at the same time).
9. Events with **1,000+ pax** are flagged for special handling/owner approval.

---

## 4. SITEMAP

### A. Public Side (Guest)
```
/                      → Home (hero, tagline "Catering for all occasions...", highlights)
/menu.php              → Full menu (from Menu.jpg categories)
/packages.php          → Catering rates (Basic/Standard/Premium + inclusions)
/party-trays.php       → Party trays with Medium/Large prices
/stations.php          → Stations (Grazing, Street Food, Carving, etc.)
/gallery.php           → Event photos
/contact.php           → FB page, phone numbers, address, map
/login.php             → Login (all roles)
/register.php          → Customer registration
```

### B. Customer Side
```
/customer/dashboard.php        → My bookings overview + notifications
/customer/book.php             → Step-by-step booking wizard
/customer/menu-selection.php   → Pick dishes per package slot
/customer/payment.php          → View QR code, upload proof of payment
/customer/booking-details.php  → Status, payment timeline, receipts
/customer/rebook.php           → Cancel & rebook flow
/customer/profile.php          → Account settings
```

### C. Staff Side
```
/staff/dashboard.php           → Today's summary, pending verifications
/staff/calendar.php            → Month view, 3-slot indicator per day
/staff/bookings.php            → All bookings list + filters
/staff/verify-payments.php     → Pending proofs → approve/reject
/staff/booking-view.php        → Full booking detail + status updates
```

### D. Owner/Admin Side
```
/admin/dashboard.php           → KPIs: bookings, revenue, pending payments
/admin/manage-menu.php         → CRUD menu items
/admin/manage-packages.php     → CRUD packages & prices
/admin/manage-trays.php        → CRUD party trays
/admin/manage-staff.php        → CRUD staff accounts
/admin/reports.php             → Bookings/revenue reports (print/export)
/admin/settings.php            → QR code image upload, booking limits, fees
```

---

## 5. DATABASE DESIGN (ERD Summary)

### Tables

**users**
- user_id (PK), full_name, email (unique), phone, password_hash, role ENUM('customer','staff','admin'), created_at, status

**packages**
- package_id (PK), name, price_per_pax, min_pax, description, is_active

**package_slots** (defines what each package includes, e.g., Premium = Soup 1, Salad 1, Beef 1...)
- slot_id (PK), package_id (FK), category, quantity

**menu_items**
- item_id (PK), category ENUM('appetizer','soup','salad','chicken','beef','pork','fish','vegetable','pasta','dessert','beverage','station'), name, description, is_active

**party_trays**
- tray_id (PK), category, name, price_medium, price_large, is_active

**bookings**
- booking_id (PK), user_id (FK), event_date, event_type, venue_address, pax_count, package_id (FK), status ENUM('pending','reserved','confirmed','completed','cancelled','rebooked'), total_amount, notes, created_at
- INDEX on event_date (critical for the 3-per-day check)

**booking_menu** (customer's chosen dishes)
- id (PK), booking_id (FK), item_id (FK), slot_category

**booking_trays** (if ordering trays)
- id (PK), booking_id (FK), tray_id (FK), size ENUM('medium','large'), qty, price

**payments**
- payment_id (PK), booking_id (FK), payment_type ENUM('reservation_fee','down_payment','full_payment','tray_payment'), amount, proof_image, status ENUM('pending','verified','rejected'), verified_by (FK users), submitted_at, verified_at, remarks

**notifications**
- notif_id (PK), user_id (FK), booking_id (FK), message, type, is_read, created_at

**settings**
- setting_key (PK), setting_value  — stores: qr_image_path, max_bookings_per_day, reservation_fee, tray_minimum

**activity_logs** (nice-to-have for the paper's security section)
- log_id (PK), user_id, action, timestamp

---

## 6. THE BOOKING FLOW (Core Logic)

```
STEP 1  Customer picks event date on calendar
        → System checks: bookings on that date < 3? Date at least X days away?
STEP 2  Event details: type, venue, pax (min 100 for packages)
STEP 3  Choose package (Basic/Standard/Premium) OR party trays (min ₱20,000)
STEP 4  Menu selection — pick items per slot (e.g., Premium: 1 soup, 1 salad,
        1 beef, 1 pork, 1 chicken, 1 fish, 1 veg, 1 pasta, 2 desserts, 1 beverage)
STEP 5  Review summary + total computation
STEP 6  Booking saved as PENDING → payment page shows the owner's QR code
STEP 7  Customer pays via QR (GCash/bank), uploads screenshot as proof
STEP 8  Staff verifies proof → booking becomes RESERVED (date slot consumed)
STEP 9  System tracks deadlines:
        - 1 month before → notify: 50% down payment due
        - 1 week before  → notify: full payment due
STEP 10 After event date passes + fully paid → COMPLETED
```

**Cancellation/Rebooking flow:** Customer requests rebook → picks new available date → payments carry over → old booking marked REBOOKED (frees the slot), new booking created linked to old payments.

---

## 7. SPRINT PLAN (Agile Scrum — matches Chapter III)

### Sprint 0 — Setup & Design (Week 1)
- Finalize ERD, create database in phpMyAdmin
- Project folder structure, Git repo
- Base template: header/footer/navbar, Bootstrap theme using brand colors (red #B22B27, green, orange, yellow from the logo)
- Deliverable: empty but navigable public site skeleton

### Sprint 1 — Public Site + Auth (Week 2)
- Home, Menu, Packages, Party Trays, Stations, Gallery, Contact pages (data pulled from DB)
- Registration + Login with password hashing (password_hash/password_verify)
- Role-based redirects, session management
- Deliverable: guest can browse; users can register/login

### Sprint 2 — Reservation & Scheduling Module (Weeks 3–4)
- Availability calendar (3-slot logic, colored indicators)
- Booking wizard Steps 1–6
- Menu selection per package slots
- Double-booking prevention (transaction + row locking on date check)
- Deliverable: customer can create a pending booking end-to-end

### Sprint 3 — Payment Module (Week 5)
- Admin uploads QR image in settings
- Customer payment page: shows QR + amount due + upload proof
- Staff verification queue: approve/reject with remarks
- Payment timeline logic (reservation fee → 50% → full)
- Deliverable: full payment lifecycle working

### Sprint 4 — Notifications + Cancel/Rebook (Week 6)
- In-system notifications (bell icon) for: booking status, payment verified/rejected, payment deadlines, event reminders
- Cancel & rebook flow with payment carry-over
- Deliverable: policy-compliant rebooking + alerts

### Sprint 5 — Dashboards & Reports (Week 7)
- Staff dashboard + calendar view
- Admin dashboard: KPIs (monthly bookings, revenue, pending payments)
- Reports page with date filters + print/export
- Deliverable: management tools complete

### Sprint 6 — Testing, Security Hardening, Deployment (Week 8)
- Unit + system testing checklist
- Security pass: prepared statements everywhere, input validation, file upload restrictions (images only, size limit, renamed files), session hardening, role guards on every page
- ISO/IEC 25010 evaluation instrument (survey) preparation
- Deploy to Hostinger, final review with owner/staff/customers
- Deliverable: live system + evaluation data for Chapter IV

---

## 8. FOLDER STRUCTURE

```
deflorence/
├── index.php
├── menu.php, packages.php, party-trays.php, stations.php, gallery.php, contact.php
├── login.php, register.php, logout.php
├── config/
│   └── db.php              (PDO connection)
├── includes/
│   ├── header.php, footer.php, navbar.php
│   ├── auth-check.php      (role guard)
│   └── functions.php       (shared helpers: slot check, totals, notify)
├── customer/  ...
├── staff/     ...
├── admin/     ...
├── assets/
│   ├── css/style.css
│   ├── js/main.js
│   └── img/                (logo, gallery photos)
└── uploads/
    ├── proofs/             (payment screenshots)
    └── qr/                 (owner's QR code)
```

---

## 9. SECURITY CHECKLIST (for Chapter III compliance)

- [ ] All queries via PDO prepared statements (SQL injection)
- [ ] password_hash() / password_verify() — never plain text
- [ ] Every protected page starts with auth-check + role check
- [ ] Upload validation: JPG/PNG only, max 5MB, randomized filename
- [ ] htmlspecialchars() on all echoed user input (XSS)
- [ ] CSRF tokens on forms (bonus points in defense)
- [ ] HTTPS on Hostinger (free SSL)

---

## 10. CONSISTENCY RULES (so all 4 members produce uniform output)

1. **Naming:** snake_case for DB and PHP variables, kebab-case for files.
2. **All prices** stored as DECIMAL(10,2); display with `number_format($x, 2)` and ₱ symbol.
3. **All dates** stored as DATE/DATETIME; displayed as "F j, Y" (e.g., July 16, 2026).
4. **Status badge colors:** pending=warning, reserved=info, confirmed=primary, completed=success, cancelled/rejected=danger, rebooked=secondary.
5. **One shared functions.php** — no duplicating logic like slot checking.
6. **Brand palette:** Red #B22B27 (primary), Green #2E9E63, Orange #E2711D, Yellow #F0B429, off-white backgrounds — matching the De Florence logo.
7. Every page uses the same header/footer includes. No standalone styling per page.

---

## 11. WHAT MAPS TO YOUR PAPER

| Paper Section | System Feature |
|---|---|
| Objective 1: monitor bookings & enforce limits | Sprint 2 (calendar, 3/day rule) |
| Objective 2: QR payment + verification + rebooking | Sprints 3–4 |
| Objective 3: ISO/IEC 25010 evaluation | Sprint 6 survey |
| Objective 4: efficiency vs manual process | Reports + Chapter IV comparison |
| ERD, Flowcharts, Use Case (Appendices) | Sprint 0 outputs |

---

## 12. BUILD ORDER — WHERE WE START

Next immediate steps, in order:
1. **Database SQL script** (all tables + seed data from the flyers: packages, menu items, trays)
2. **Base template + config** (db.php, header/footer, brand styling)
3. **Public pages** (they're easy wins and use the seed data)
4. Then proceed sprint by sprint above.
