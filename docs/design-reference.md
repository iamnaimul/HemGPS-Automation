# হেমনগর সরকারি প্রাথমিক বিদ্যালয় — School Data Suite
## Reference & Rebuild Guide · Version 1.1.1-pro

> **Purpose of this file:** one place that explains what the three HTML apps are, how they store and format data, how they look, how backup/restore works, and how to rebuild any of them from zero.
> **Source of truth:** the three HTML files. Everything below was read from their code. Where a detail was *inferred from a function name only* it is marked **(by name)** — verify in the source before relying on it.

---

## Table of contents

1. [Goal & scope](#1-goal--scope)
2. [File map & external assets](#2-file-map--external-assets)
3. [Technology rules](#3-technology-rules)
4. [Architecture shared by all three apps](#4-architecture-shared-by-all-three-apps)
5. [Design system (colors, type, components, responsive)](#5-design-system)
6. [Data-format conventions (digits, dates, phone, email, NID, address, money)](#6-data-format-conventions)
7. [App 1 — school.html](#7-app-1--schoolhtml)
8. [App 2 — teachers.html](#8-app-2--teachershtml)
9. [App 3 — students.html](#9-app-3--studentshtml)
10. [Unified backup / restore](#10-unified-backup--restore)
11. [Storage map (every localStorage key)](#11-storage-map)
12. [Excel export](#12-excel-export)
13. [Function index per file](#13-function-index-per-file)
14. [Rebuild-from-scratch playbook](#14-rebuild-from-scratch-playbook)
15. [Reusable code snippets](#15-reusable-code-snippets)
16. [QA checklist](#16-qa-checklist)
17. [Coding guidelines, gotchas, to-do](#17-coding-guidelines-gotchas-to-do)
18. [AI prompt template for rebuilding](#18-ai-prompt-template-for-rebuilding)

---

## 1. Goal & scope

**Goal:** a free, offline, no-server data store for one primary school, usable by non-technical staff on a phone, tablet or PC, entirely in **Bengali**.

| App | File | What it stores | Main user actions |
|---|---|---|---|
| School info | `school.html` | One school profile: identity, facilities, classes, land, buildings, rooms, wash blocks, water, sanitation, furniture, ICT, committees, activities | View dashboard cards, edit a section in a modal, add/remove list items, print, export Excel |
| Teacher DB | `teachers.html` | A list of teacher records (professional, personal, contact, addresses, education, training, emergency, note) | Add / edit / view / delete, search, filter, sort (seniority default), print profile, export Excel |
| Student DB | `students.html` | A list of student records (class info, personal, family, address, health, emergency, scholarship/scouting, final result) | Add / edit / view / delete, search, multi-rule sort/filter, bulk select, bulk promotion, export Excel (choose fields), open result page |
| Backup (all three) | built into each | One combined JSON of all three datasets | Backup, Restore (pick sections), Recovery Backup |

**Non-goals:** no login, no server, no sync, no encryption, no photo storage, no multi-user editing.

**Owner/metadata (as written in the files):** author Naimul Hassan · `iam.naimul@gmail.com` · school name used everywhere: `হেমনগর সরকারি প্রাথমিক বিদ্যালয়`.

---

## 2. File map & external assets

```
/ (same folder, same origin)
├── index.html        landing page that links the apps   (NOT part of the three files; referenced in project notes)
├── school.html       school info app          v1.1.1-pro
├── teachers.html     teacher DB app           v1.1.1-pro
├── students.html     student DB app           v1.1.1-pro
├── result.html       result page opened by students.html (separate file, not covered here)
└── img/
    └── SCHOOL-LOGO-round.png   round logo; used as favicon and header logo (header hides it via onerror if missing)
```

* **Same origin is mandatory** for combined backup: all pages must be served from one site/app (WebView, local server, hosting). Opening them as separate `file://` pages works in Chrome (shared storage) but **not** in Firefox (storage is per file).
* `students.html → result.html` hand-off: the student list is written to `hemnagar_result_snapshot` before `result.html` opens in a new tab.
* **SheetJS** is loaded lazily from `https://cdn.sheetjs.com/xlsx-0.20.2/package/dist/xlsx.full.min.js` the first time Excel export is used (internet needed once per session; everything else is offline).

---

## 3. Technology rules

* **Each app = one self-contained `.html` file** (HTML + CSS + JS inline). No build step, no framework, no npm.
* Vanilla ES2015+ JavaScript, `localStorage` for data, `Blob` + `<a download>` for files, `FileReader` for restore.
* Line endings in the shipped files are **CRLF** — keep them when patching with scripts (don't mix LF).
* Language `bn`; all UI text Bengali; internal keys English camelCase.
* No external fonts/CSS/JS except SheetJS (lazy). No analytics, no network calls.
* `<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">` (needed for notch/safe-area support).

---

## 4. Architecture shared by all three apps

### 4.1 Script order inside each file
1. `<style>` — base theme → page-specific CSS → **UNIFIED DESIGN SYSTEM v1.1.1-pro** block (always last, identical in all three).
2. `<body>` — `.container` → `.dashboard-header` → page content → `.app-footer` (backup bar + credit) → modals → `<div id="toast" class="toast">`.
3. `<script>` #1 — **UNIFIED BACKUP v1.1.1-pro** module (identical in all three, defines `window.UnifiedBackup`).
4. `<script>` #2 — the page's own logic (constants → helpers → Store → load → render → events → `init()`).

### 4.2 Standard pieces (same idea in every file)
| Piece | Behaviour |
|---|---|
| `Store` wrapper | Probes `localStorage`; if blocked, falls back to an in-memory object and exposes `Store.available`. Methods `get/set/del`, quota errors are caught (`set` returns `false`). |
| `$()` | `document.getElementById` shortcut. |
| `toBn / toEn` | Convert digits English ⇄ Bengali (see §6). |
| `clean(v)` | `String(v ?? "").trim()`. |
| `esc / escapeHtml` | Escape text before inserting into `innerHTML`. Always use it. |
| `showToast(msg, type)` | Sets `#toast` class to `toast show ok|err`, auto-hides (~3 s). |
| `debounce` | Used for search and draft autosave. |
| Persist-after-every-change | Every add/edit/delete calls the save function immediately (so backup can read straight from `localStorage`). |
| Snapshot-and-rollback | Before saving: `snapshot = list.slice()`; if save fails (quota) → restore snapshot and show error. |
| Event delegation (teachers) | One click handler reads `data-action` (`toggle-form`, `backup`, `restore`, `recover`, `view-edit`, `close-view`, `print`, …). |
| Inline handlers (school/students) | `onclick="downloadJSONBackup()"` etc. on buttons. |

### 4.3 Footer block (identical markup in all three)
```
.app-footer
 ├─ .backup-bar
 │   ├─ .backup-info   (.title with icon, .meta#backupMeta)
 │   └─ .backup-actions  [JSON ব্যাকআপ (primary)] [রিস্টোর] [Recovery Backup]  + hidden <input type=file id=restoreFileInput accept=".json,application/json">
 └─ .footer-credit  "School|Teacher|Student data store by <strong>Naimul Hassan</strong> v1.1.1-pro"
```

---

## 5. Design system

The design block lives at the end of `<style>` in every file (marker comment `UNIFIED DESIGN SYSTEM v1.1.1-pro`). **If you change a token, change it in all three files.**

### 5.1 Color tokens
| Token | Value | Use |
|---|---|---|
| `--bg` | `#f4f6f5` | page background base |
| `--card` | `#ffffff` | cards, bars, modals |
| `--surface` | `#f7f9f8` | inputs, soft panels |
| `--text` | `#11201b` | main text |
| `--text-soft` | `#43544d` | secondary text |
| `--muted` | `#7b8a84` | hints, meta lines |
| `--border` / `--border-strong` | `#e3e9e6` / `#cfd9d4` | hairlines / inputs |
| `--primary` | `#0b6b57` | brand green (buttons, links, accents) |
| `--primary-hover` | `#085646` | hover |
| `--primary-soft` / `--primary-soft-2` | `#e6f3ee` / `#cde7dd` | tinted backgrounds / selected |
| `--accent` / `--gold` | `#e0a526` | gold highlight |
| `--gold-soft` | `#fdf3da` | warning tint |
| `--ink` | `#0b2a22` | very dark green (toast bg `#0b2a22`) |
| `--success` / `--danger` / `--warning` | `#10b981` / `#ef4444` / `#f59e0b` | status |
| `--excel` | `#1d7a4a` | Excel button |
| `--grad-brand` | `135deg, #082f27 → #0b6b57 58% → #14906f` | header gradient |
| `--grad-soft` | `135deg, #eef8f3 → #fcf6e6` | soft section background |
| `--grad-gold` | `135deg, #ecb53a → #e0a526` | gold pill buttons |

### 5.2 Shape, shadow, ring
`--radius:16px` · `--radius-sm:11px` · header radius `24px` (20 ≤780px, 16 ≤420px) · backup bar `18px` · buttons **pill** (`999px`).
`--shadow-sm: 0 1px 2px rgba(11,42,34,.06)` · `--shadow` (cards) · `--shadow-md: 0 14px 34px rgba(11,107,87,.18)` · `--shadow-lg: 0 30px 80px rgba(8,30,24,.35)` · focus ring `--ring: 0 0 0 4px rgba(11,107,87,.16)`; keyboard focus outline `3px solid rgba(11,107,87,.38)`.

### 5.3 Typography
* `--font-body`: `"BCC Purno", "Noto Sans Bengali", "Hind Siliguri", Nikosh, NikoshBAN, "Segoe UI", Arial, sans-serif`
* `--font-head`: same stack starting with `"BCC Purno Black"`; headings weight 900.
* Base size `15.5px`, line-height `1.6`; steps: ≥1600px → 16px, ≥2200px → 17px, ≤780px → 15px, ≤420px → 14.5px.
* Page title `h2` 28px (25 ≤1024, 21 ≤780, 19 ≤420); subtitle `h3` 14px.

### 5.4 Layout & breakpoints
| Range | `--pad-x` / top / bottom | Notes |
|---|---|---|
| ≥ 2200px | 40 / 24 / 40 | `--page-max` 1920px |
| ≥ 1600px | 28 / 24 / 40 | `--page-max` 1640px |
| default | 20 / 24 / 40 | `--page-max` 1500px, `.container` centered |
| ≤ 1024px (tablet) | 16 / 18 / 32 | header padding 24×26 |
| ≤ 780px (phone) | 10 / 12 / 30 | header stacks, logo 52px, backup bar stacks, buttons ⅓ width, toast full-width bottom |
| ≤ 560px | — | backup buttons ½ width; primary full width |
| ≤ 420px | 6 / 8 / 24 | logo 44px, buttons full width |
| `pointer: coarse` | — | backup buttons min-height 44px; text inputs 16px (stops iOS zoom) |

Body padding always uses `max(var(--pad-x), env(safe-area-inset-*))` so notches/home bars are respected.

### 5.5 Components
* **Header** `.dashboard-header` — green gradient card, `.brand` (round logo 72px with white ring) + `.dashboard-title` (h2 school name, h3 page name) + action buttons on the right (teachers: «নতুন শিক্ষক যোগ করুন»; school: Print, Excel).
* **Backup bar** — white card, 4px green left border, title with shield/save icon, meta line (muted 13px), three pill buttons.
* **`.btn-backup-sm`** — `primary` = solid green + soft green shadow; `secondary` = white, 1.5px border, hover tinted green.
* **Toast** — fixed bottom-right (desktop) / bottom full-width (≤780px), dark green `#0b2a22`, 5px left border, `.ok` green / `.err` red variants.
* **Restore dialog (`#ubModal`)** — blurred dark overlay, 24px-radius white box (max 540px), rows with checkboxes (tinted green when selected), gold warning box, pill buttons; becomes a **bottom sheet** ≤560px; Escape closes it.
* **Modals in apps** — `.modal-overlay` + `.modal-content` (teacher view modal has print header `.print-school-head`).
* **Forms** — sectioned (numbered titles), 2–3 column grid collapsing to 1 on phones; invalid inputs get class `invalid`.

### 5.6 Print
`@media print` hides `.app-footer` and interactive chrome; teachers' view modal and school dashboard have dedicated print layouts (school-name header block `.print-school-head`: `.ps-name`, `.ps-title`).

---

## 6. Data-format conventions

### 6.1 Digits
* UI always **shows Bengali digits** (০১২৩৪৫৬৭৮৯) and accepts either Bengali or English input.
* `toBn(n)` English→Bengali; `toEn(v)` Bengali→English; `digitsOnly(v)` = `toEn` then strip non-digits.
* Live input masks: `digits` fields convert to Bengali as you type; `decimal` keeps one `.`; `date` fields allow only digits and `/`.

### 6.2 What is **stored** (important difference!)
| Data | `teachers.html` stores | `students.html` stores | `school.html` stores |
|---|---|---|---|
| Numeric ids (PIN, NID, mobile, ward, salary) | **English digits** (`digitsOnly`) | **Bengali digits** (`toBn(...)`) | text as typed |
| Dates | `DD/MM/YYYY` **English digits** | `DD/MM/YYYY` **Bengali digits** | text |
| Booleans (checkbox) | n/a | `true/false` (`hasGuardian`, `customEmergency`) | n/a |
| Yes/No flags | n/a | `"হ্যাঁ"` / `"না"` (diseases) | select value text |

So when moving data between apps (e.g. the result page) normalise with `toEn()`.

### 6.3 Dates
* **Format: `DD/MM/YYYY`** everywhere (display, input, storage).
* `normalizeDateString` accepts: `d/m/yyyy`, `dd-mm-yyyy`, `dd.mm.yyyy`, spaces, `yyyy-mm-dd`, 8 digits `ddmmyyyy`, 6 digits `ddmmyy` (→ `20yy`); pads to two digits; rejects month >12 / day >31.
* `isRealDate` checks the calendar (rejects 31/02 etc.). Date of birth in the future is rejected (teacher DOB, student/parent DOB).
* **Date in words** (`dateToBengaliWords`): `dobBnWords` auto-filled read-only, e.g. ordinal day + month name + year in words (`১ → পহেলা`, `2 → দোসরা`, `3 → তেসরা`, `4 → চৌঠা`, `5 → পাঁচই`…, months জানুয়ারি…ডিসেম্বর, year via `yearToBengaliWords`: 17xx → সতেরো শত, 18xx → আঠারো শত, 19xx → উনিশ শত, 20xx → দুই হাজার …).
* Sorting dates: `parseDateForSort` → number `yyyymmdd`; invalid → `Infinity` (goes last).

### 6.4 Date-time and file stamps
| Use | Format |
|---|---|
| Info line (backup/restore) | `dd/mm/yyyy hh:mm` in Bengali digits (e.g. `২০/১০/২০২৬ ০৫:৪১`) |
| `formatDateTimeDDMMYYYY` | `dd/mm/yyyy, hh:mm` (24-hour, zero-padded) |
| Stored timestamps | ISO 8601 (`new Date().toISOString()`) in `exportDate`, last-backup, last-restore |
| Backup file name | `hemnagar_full_backup_DD-MM-YYYY_HH-MM.json` |
| Excel file names | `teachers_list_DD-MM-YYYY.xlsx`, `school_data_DD-MM-YYYY.xlsx`, students similar date stamp (`dateStamp()` = `DD-MM-YYYY`) |
| Draft expiry (teachers) | `Date.now()` ms; draft discarded after **7 days** |

### 6.5 Phone, email, ID numbers
| Field | Rule | Message idea |
|---|---|---|
| Mobile (`mobile`, `secondMobile`, `emergencyMobile`, father/mother/guardian mobile) | digits only, **11 digits**, regex `^01[3-9]\d{8}$` | «সঠিক মোবাইল নম্বর দিন (১১ ডিজিট, 01XXXXXXXXX)!» |
| Email | `^[^\s@]+@[^\s@]+\.[^\s@]+$` (students require TLD ≥2 chars); stored as typed (not converted to Bengali) | «সঠিক ই-মেইল ঠিকানা দিন!» |
| NID (teacher, parents, guardian) | **10, 13 or 17 digits** `^(\d{10}\|\d{13}\|\d{17})$`; max length 17 | «এনআইডি নম্বর ১০, ১৩ বা ১৭ ডিজিটের হতে হবে!» |
| Birth registration (`birthRegNo`) | exactly **17 digits** | — |
| IPEMIS PIN (teachers) | digits, required, **unique** | duplicate message names the existing teacher |
| Ward number | digits, max 3 | — |

### 6.6 Address (Bangladesh, 6 parts)
Order everywhere: **গ্রাম → পোস্ট অফিস → ইউনিয়ন/পৌরসভা → ওয়ার্ড নম্বর → উপজেলা → জেলা**.
* Teachers: `present*` and `permanent*` field groups (`Village, PostOffice, Union, Ward, Upazila, District`); `composeAddress(rec, prefix)` joins non-empty parts with `", "`. Checkbox **«স্থায়ী ঠিকানা বর্তমান ঠিকানার অনুরূপ»** (`sameAddressCheck`) copies present → permanent live and collapses the permanent block (`ADDR_PAIRS`).
* Students: `current*` and `permanent*` with the same six parts.
* **Defaults** used when adding a new person: village/post office/union `হেমনগর`, upazila `গোপালপুর`, district `টাঙ্গাইল`; students' nationality `বাংলাদেশি`.
* School identity section: `division, district, upazila, union, cluster, addressLine`.

### 6.7 Names, money, other
* Bengali name = free text; **English names are forced UPPERCASE** (`nameEn`, `fatherNameEn`, …).
* Salary: store digits only; display `toBn(groupBdDigits(n)) + "/="` → `২২,১০০/=` (Indian-style grouping: last 3 digits, then pairs). Excel exports the grouped number without `/=`.
* Blood groups list: `A/B/AB/O` × `Positive/Negative` (English spelling).
* Height in cm, weight in kg (digits).

### 6.8 Personal-info handling rules
* Required fields are marked `*`; validation stops at the first error, focuses the field and shows a toast.
* Personal data (NID, birth reg, mobile, email, address, health) is stored **only in the browser's localStorage and in backup files you download** — treat backup JSON files as confidential (not encrypted).
* Never log personal data to the console; never send it over the network.
* Health fields are optional; disability/disease detail panels show only when status is «আছে».

---

## 7. App 1 — school.html

**Storage key:** `hemnagar_school_v1` → one **object** (`schoolData`).

### 7.1 Data object
```js
schoolData = {
  // flat fields by id (text/select/date), e.g. schoolNameLocal, schoolName, schoolCode, schoolType, division, district, upazila, union, cluster, addressLine, academicYear, ... playground, boundary, ... tidalWave, cyclone, ... amountOfLand, ledgerBsNumber, dagNumber, ...
  classes: [],        // array of class names (see list below)
  buildings: [], rooms: [], washblocks: [], waters: [], sanitations: [], furnitures: [], ictEquipments: []   // arrays of objects
}
```
On load: `Object.assign(schoolData, parsed)` (defaults survive missing keys).

### 7.2 Sections (18) — id → title
`identity ১. বিদ্যালয়ের নাম ও ঠিকানা` · `basic ২. মৌলিক তথ্য` · `facilities ৩. সুযোগ-সুবিধা ও উপকরণ` · `classes ৪. পাঠদানকৃত শ্রেণি` · `geo ৫. জলবায়ু ও দুর্যোগ` · `land ৬. ভূমির তথ্য` · `buildings ৭. ভবনের তথ্য` · `rooms ৮. কক্ষের তথ্য` · `washblock ৯. ওয়াশব্লক` · `water ১০. পানীয় জল` · `sanitation ১১. স্যানিটেশন` · `furniture ১২. আসবাবপত্র` · `ict ১৩. তথ্য প্রযুক্তি ও মাল্টিমিডিয়া` · `ictEquipment ১৪. আইসিটি সরঞ্জাম` · `smc ১৫. এসএমসি` · `pta ১৬. সামাজিক উদ্বুদ্ধকরণ সভা ও পিটিএ` · `cultural ১৭. খেলাধুলা ও সাংস্কৃতিক কর্মকাণ্ড` · `specialNeeds ১৮. বিশেষ চাহিদাসম্পন্ন শিক্ষার্থীর সুবিধা`

Section definition shape:
```js
{ id, title, icon /* inline SVG path markup */, fields:[{id,label,type:"text|select|date",options?,required?,full?}] }
// or { id, title, icon, special:"classes|buildings|rooms|washblock|water|sanitation|furniture|ictEquipment" }
```
Select options are mostly `আছে/নাই`, `হ্যাঁ/না`, `বিদ্যমান/বিদ্যমান নয়`, `হয়/হয় না`.

### 7.2.1 Classes (checkbox chips)
`প্রাক-প্রাথমিক ৪+`, `প্রাক-প্রাথমিক ৫+`, `প্রথম শ্রেণি` … `অষ্টম শ্রেণি` (10 items). Stored as the selected names array.

### 7.2.2 List-item field sets (dynamic sections)
| List key | Fields (id) |
|---|---|
| `buildings` | buildingNo, ownershipType (নিজস্ব/ভাড়া…), numberOfCurrentFloors, yearOfConstruction, numberOfFoundationFloors, numberOfRooms, condition, constructionType, sourceOfFund, ramp |
| `rooms` | buildingNo, roomName, usagePurpose, electricityInClassroom, roomLength, roomWidth, constructionType, yearOfConstruction, condition |
| `washblocks` | yearOfConstruction, constructionUnderProject (পিইডিপি-১…৪…), washblockUserType (বালক/বালিকা/শিক্ষক…), condition, washblockInUse, withElectricity, cleaningMaterial, withWaterSource, withSoap |
| `waters` | safeWaterSource (টিউবওয়েল/সাপ্লাইওয়াটার/ডিপটিউবওয়েল/কুয়া…), condition, arsenicFree, sourceOfFund, ecolaiTested, salinityOrIron, yearOfEstablishment |
| `sanitations` | toiletUserType, condition, cleaningMaterial, withSoap, withWaterSource, withElectricity, constructionProject |
| `furnitures` | furnitureType, numberOfUsableFurnitures, numberOfRepairableFurnitures, numberOfUnsuitableFurnitures |
| `ictEquipments` | productName (কম্পিউটার/ল্যাপটপ/প্রিন্টার/প্রজেক্টর/মডেম…), serialNo, brand, model, yearOfReceiptOfGoods, sourceOfProductAcquisition, condition, commentsOnIctEquipment |

Common `condition` options: `নতুন/ভালো`, `ব্যবহারযোগ্য`, `মেরামতযোগ্য`, `নির্মাণাধীন`, …

### 7.3 UI behaviour
* **Dashboard of section cards** with a sticky **table of contents** (TOC) and **scroll-spy** highlighting the active section; first paint is fast, cards are built in `requestAnimationFrame`.
* Each card shows its fields read-only; **Edit** opens a modal editor for that section (fields editor, classes chip editor, or dynamic list editor with add/remove items).
* Copy-to-clipboard buttons on values; **Print** (`printDashboard`) and **Excel** buttons in the header.
* Backup/Restore/Recovery bar in the footer (§10).

### 7.4 Actions list
`openEdit(sectionId)` · `saveEdit()` (validate required, write to `schoolData`, persist, re-render that section) · `addDynItem / removeModalDynItem / updateModalDynItem` · `exportExcel()` · `printDashboard()` · `downloadJSONBackup / handleRestoreFile / recoverPrevious` (delegates to `UnifiedBackup`).

---

## 8. App 2 — teachers.html

**Storage keys:** data `hemnagar_teachers_v1` (array) with a mirror `hemnagar_teachers_v1__bak`; draft `hemnagar_teachers_draft_v1`; sort mode `hemnagar_teachers_sortmode`; legacy read-only fallbacks `hemnagar_teachers`, `teachers_hemnagar`.

### 8.1 Record (object in the array)
`FIELD_ORDER` fields + two dynamic arrays:

| Group | Field id | Label | Kind | Req |
|---|---|---|---|---|
| ১. পেশাগত | `ipemisPin` | IPEMIS পিন নম্বর | digits, unique | ✔ |
| | `designation` | পদবি | select: প্রধান শিক্ষক / ভারপ্রাপ্ত প্রধান শিক্ষক / সহকারি শিক্ষক | ✔ |
| | `firstJoiningDate` | প্রথম যোগদান তারিখ | date | ✔ |
| | `currentSchoolJoiningDate` | বর্তমান বিদ্যালয়ে যোগদান তারিখ | date | ✔ |
| | `currentSalary` | বর্তমান মূল বেতন (টাকায়) | digits → `২২,১০০/=` | |
| | `currentGrade` | বর্তমান গ্রেড | ৮ম … ১৩তম গ্রেড | |
| | `prlDate` | পিআরএল তারিখ | date | |
| ২. ব্যক্তিগত | `nameBn` | শিক্ষকের নাম | text | ✔ |
| | `nid` | এনআইডি নম্বর | digits 10/13/17, unique | ✔ |
| | `dob` | জন্ম তারিখ | date, not future | ✔ |
| | `dobBnWords` | জন্ম তারিখ (কথায়) | readonly auto | |
| | `maritalStatus` | বৈবাহিক অবস্থা | অবিবাহিত / বিবাহিত / অন্যান্য | |
| | `religion` | ধর্ম | ইসলাম / হিন্দু / বৌদ্ধ / খ্রিস্টান / অন্যান্য | |
| | `bloodGroup` | রক্তের গ্রুপ | 8 groups | |
| | `gender` | লিঙ্গ | পুরুষ / মহিলা | ✔ |
| ৩. যোগাযোগ | `mobile` | মোবাইল নম্বর | 11 digits | ✔ |
| | `secondMobile` | দ্বিতীয় মোবাইল | 11 digits | |
| | `email` | ই-মেইল | email | |
| বর্তমান ঠিকানা | `presentVillage/PostOffice/Union/Ward/Upazila/District` | §6.6 | text/digits | |
| স্থায়ী ঠিকানা | `permanentVillage/PostOffice/Union/Ward/Upazila/District` | §6.6 | text/digits | |
| ৪. শিক্ষাগত যোগ্যতা | `education[]` | dynamic rows | | |
| ৫. প্রশিক্ষণ | `trainings[]` | dynamic rows | | |
| ৬. জরুরি যোগাযোগ | `emergencyName`, `emergencyRelation`, `emergencyMobile` | | | |
| ৭. অন্যান্য | `specialNote` | textarea | | |

**Dynamic rows**
* `education[]` → `certName` (এস.এস.সি, এইচ.এস.সি, স্নাতক (সম্মান), স্নাতক (পাস), মাস্টার্স, বিবিএ, এমবিএ, অন্যান্য — combo), `group` (বিজ্ঞান, ব্যবসায় শিক্ষা, মানবিক, রাষ্ট্রবিজ্ঞান, হিসাববিজ্ঞান, বিএসএস, ব্যবস্থাপনা, অন্যান্য), `board` (ঢাকা, জাতীয় বিশ্ববিদ্যালয়, অন্যান্য), `passYear` (year), `result` (GPA/division text).
* `trainings[]` → `name` (সাব-ক্লাস্টার, কারিকুলাম ডিসেমিনেশন, ইনডাকশন, প্রাক-প্রাথমিক, বিষয় ভিত্তিক (সমাজ/গণিত/বাংলা/ইংরেজি), ডি.পি.এড., সঙ্গীত, শারীরিক শিক্ষা, আইসিটি ইন এডুকেশন, সেন্ড, বি.এড., সি-ইন-এড, অন্যান্য), `institution`, `startDate`, `endDate` (dates), `result`.
* Combo = dropdown with a free-typing option.

### 8.2 Validation order (`validateForm`)
1. Required fields (`*`). 2. Date fields are real dates (`DD/MM/YYYY`). 3. `PATTERNS` (NID, mobiles, email). 4. Training/education dates real. 5. DOB not in the future. 6. **Unique IPEMIS PIN and NID** across other records (message names the owner).

### 8.3 Sorting (default = seniority)
`compareSeniority`: designation rank (প্রধান শিক্ষক 0 → ভারপ্রাপ্ত 1 → সহকারি 2) → earlier `firstJoiningDate` → earlier `dob` → name (`localeCompare(..., "bn")`).
Other modes (`SORT_MODES`): name · first-join asc/desc · school-join asc/desc · dob asc/desc · gender (পুরুষ→মহিলা). Chosen mode persists in `hemnagar_teachers_sortmode`.
**Search:** space-separated terms, all must match (AND) across every field, case-insensitive, digit-agnostic (`toEn`). **Filters:** designation, gender.

### 8.4 Behaviours
* **Draft autosave** (`saveDraft/loadDraft/clearDraft`, debounced, only for *new* records, discarded after 7 days).
* **Add/Edit form** is a collapsible section toggled by the header button; editing uses hidden `editIndex` (-1 = new).
* **View modal** with avatar, meta tags, grouped sections, sub-tables for education/training, **Print**, **Edit**, **Close**.
* **Delete** with confirmation (`deleteTeacher`).
* `saveData()` writes both data and `__bak` mirror; returns `false` on quota error (caller rolls back).
* `loadTeachers()` tries data key → mirror → legacy keys; every record passes `normalizeTeacherRecord` (clean strings, digits English, dates normalized, `dobBnWords` recomputed, dynamic rows normalized).
* Cross-tab: a `storage` listener refreshes when another tab changes data.

---

## 9. App 3 — students.html

**Storage keys:** data `hemnagar_students_v20` (array); `hemnagar_auto_backup_before_restore` (old per-app recovery, read-only fallback); `hemnagar_last_backup`; `hemnagar_result_snapshot` (hand-off to `result.html`).

### 9.1 Record fields (`FORM_SCHEMA` id : type)
Types: `text`, `digits` (Bengali digits), `decimal`, `upper` (uppercase), `date`, `raw` (multi-line), `yn` (checkbox → `হ্যাঁ/না`), `bool`.

| Group | Fields |
|---|---|
| শিক্ষা | `academicYear`(digits), `class`, `roll`(digits), `section` (প্রযোজ্য নয়/ক/খ/গ) |
| পাঠচক্র সমাপ্তি (only when class = পাঠচক্রের সমাপ্তি) | `finalAvg`(decimal), `finalGrade`, `finalPosition` — cleared automatically otherwise |
| সাধারণ | `nameBn`, `nameEn`(upper), `dob`, `dobBnWords`(auto), `birthRegNo`(17 digits), `gender` (ছাত্র/ছাত্রী), `religion`, `nationality` |
| পিতা | `fatherNameBn/En`, `fatherNid`, `fatherDob`, `fatherMobile`, `fatherEmail`, `fatherOccupation` |
| মাতা | `motherNameBn/En`, `motherNid`, `motherDob`, `motherMobile`, `motherEmail`, `motherOccupation` |
| অভিভাবক | `guardianNameBn/En`, `guardianNid`, `guardianDob`, `guardianMobile`, `guardianEmail`, `guardianOccupation`, `guardianRelation`; flag `hasGuardian` (checkbox «অভিভাবক প্রযোজ্য») |
| পরিবার | `siblingsCount` |
| ঠিকানা | `current*` and `permanent*` × (Village, PostOffice, Union, Ward, Upazila, District) |
| স্বাস্থ্য | `bloodGroup`, `height`(cm), `weight`(kg), `vision`/`hearing` (স্বাভাবিক/অস্বাভাবিক), `hasDisability` (আছে/নেই) + `disability` (≤150 chars), `hasDisease` + `diseaseThalassemia`, `diseaseIncurable`, `diseaseAllergy`, `diseaseEpilepsy` (each হ্যাঁ/না) |
| জরুরি যোগাযোগ | `emergencyName`, `emergencyRelation`, `emergencyMobile`; flag `customEmergency` |
| অন্যান্য | `hasScholarship` (সুবিধাভোগী নয়/সুবিধাভোগী), `hasScouting` (না/হ্যাঁ), `specialNote` |

**Classes (`CLASS_LIST`)**: `প্রাক প্রাথমিক ৪+`, `প্রাক প্রাথমিক ৫+`, `১ম (প্রথম)`, `২য় (দ্বিতীয়)`, `৩য় (তৃতীয়)`, `৪র্থ (চতুর্থ)`, `৫ম (পঞ্চম)`, plus terminal value `FINAL_CLASS = "পাঠচক্রের সমাপ্তি"`. Order map `CLASS_ORDER` (final = 99) is used for sorting.

### 9.2 Smart defaults when saving (`collectFormData`)
* If **no guardian** → guardian fields are copied from the **mother** and relation = «মাতা».
* If **not custom emergency** → emergency = father's name, relation «পিতা», father's mobile.
* If disability status ≠ আছে → `disability = ""`. If disease status ≠ আছে → all four disease flags = «না».
* `dobBnWords` recomputed from `dob`.

### 9.3 Validation (`handleFormSubmit`)
1. Required (`*`): `nameBn, class, roll, academicYear, gender, religion, fatherNameBn, motherNameBn`.
2. Format map (`VALIDATION_MAP`): `birthRegNo`→17 digits; `father/mother/guardianNid`→10/13/17; four mobile fields→`01[3-9]xxxxxxxx`; three emails→email.
3. Dates real & not future (`validateDates`: student, father, mother, guardian DOB).
4. **Duplicate roll check** (`findDuplicateRoll`): same `academicYear + class + section + roll` (ignored for final class) → asks «তবুও সংরক্ষণ করবেন?».
5. Save with snapshot/rollback.

### 9.4 List features
* **Search** across all fields; **filters**: year, class, section, gender, religion, blood group, … (`refreshFilterOptions` keeps options in sync).
* **Multi-rule sorting**: add several sort rules (field + direction); numeric fields (`academicYear, roll, birthYear, birthMonth`) compare numerically, others as Bengali text; class uses `CLASS_ORDER`.
* **Selection**: row checkboxes, select-all, «select visible», clear; selection drives bulk actions.
* **View modal** (print, edit, promote shortcuts) · **Edit** · **Delete**.
* **Promotion** (`openPromoteModal` → per-student rows + bulk bar `applyBulkPromote` → `confirmPromotion`): change year/class/roll/section for selected students; moving into «পাঠচক্রের সমাপ্তি» also takes final average/grade/position and shows a 🎓 count; incomplete rows are reported before updating the rest.
* **Excel export modal** (`openExcelExportModal`): fields grouped into 7 groups (শিক্ষা · সাধারণ · পারিবারিক · স্বাস্থ্য · জরুরি যোগাযোগ · অন্যান্য · পাঠচক্রের সমাপ্তি), group/field checkboxes with counts, scope selection, summary.
* **Result file**: `openResultFile()` stores `{timestamp,totalRecords,students,source:"students.html"}` in `hemnagar_result_snapshot` then opens `result.html`.
* Disability/disease panels (`syncDisabilityPanel`, `syncDiseasePanel`) expand only on «আছে» and show a counter for the 150-char limit.

---

## 10. Unified backup / restore

Embedded **identically** in all three files as `window.UnifiedBackup` (header comment `UNIFIED BACKUP v1.1.1-pro`). It reads/writes `localStorage` directly, so all three datasets are handled from any page.

### 10.1 JSON file format
```json
{
  "format": "hemnagar-unified-backup",
  "formatVersion": 1,
  "appVersion": "1.1.1-pro",
  "appName": "হেমনগর সরকারি প্রাথমিক বিদ্যালয়",
  "exportDate": "2026-10-20T05:41:00.000Z",
  "counts": { "school": 12, "teachers": 9, "students": 240 },
  "data": {
    "school":   { "...": "object (schoolData)" },
    "teachers": [ { "...": "teacher records" } ],
    "students": [ { "...": "student records" } ]
  }
}
```
Sections with no data are omitted. **Never rename `format`** — restore recognises files by it.

### 10.2 Sections registry (inside the module)
| id | Label | Kind | Keys written on restore | Record validity test |
|---|---|---|---|---|
| `school` | বিদ্যালয় তথ্য | object | `hemnagar_school_v1` | non-empty plain object |
| `teachers` | শিক্ষক তথ্য | array | `hemnagar_teachers_v1` **and** `hemnagar_teachers_v1__bak` | object with `nameBn` or `ipemisPin` |
| `students` | শিক্ষার্থী তথ্য | array | `hemnagar_students_v20` | object with `nameBn` or `roll` |

### 10.3 Backup flow
1. Read current data of all three sections from storage.
2. If all empty → toast error, no file.
3. Build payload, download `hemnagar_full_backup_DD-MM-YYYY_HH-MM.json`.
4. Store ISO time in `hemnagar_unified_last_backup` (+ legacy per-app keys `hemnagar_last_backup`, `hemnagar_teachers_last_backup` so old displays stay correct).
5. Toast: «সম্পূর্ণ ব্যাকআপ ডাউনলোড হয়েছে (বিদ্যালয় তথ্য + শিক্ষক তথ্য + শিক্ষার্থী তথ্য)». The page then refreshes its info line (`updateBackupMeta`).

### 10.4 Restore flow
1. User picks a `.json` file → `UnifiedBackup.handleFile(event, hostPage)`.
2. Parse → `detect()`:
   * unified file → sections present in `data`;
   * **legacy single-app files** (`school-data-backup`, `teacher-data-backup`, `student-data-backup`) → that one section;
   * file with no `format` → interpreted as the **host page's** data (bare array / object).
   * Invalid JSON or nothing valid → error toast, no dialog.
3. Dialog: one row per section — checkbox (default ticked), counts «ব্যাকআপে / বর্তমানে», file date and app version; sections missing in the file are greyed out; **students** row has a mode select: *সম্পূর্ণ প্রতিস্থাপন* or *মার্জ* (merge key = `academicYear|class|section|roll|nameBn`; same key skipped, new appended).
4. On **«রিস্টোর করুন»**:
   * save **Recovery Backup** of *all* current data to `hemnagar_unified_prerestore` (if storage is full, ask whether to continue);
   * write chosen sections; **all-or-nothing** — if any write fails, every key written so far is rolled back and an error is shown;
   * set `hemnagar_unified_last_restore`;
   * toast + `location.reload()` after ~1.1 s so the page reloads the restored data.
5. **Recovery Backup** button re-opens the same dialog fed from `hemnagar_unified_prerestore` (falls back to the old per-app recovery keys). Restoring from it swaps current/previous (so it can be undone).

### 10.5 Info line text (`UnifiedBackup.metaText(host, count, unit)`)
`সর্বশেষ ব্যাকআপ: <dd/mm/yyyy hh:mm> • সর্বশেষ রিস্টোর: <…> • মোট: <n> <unit>`
Placeholders: «এখনো নেওয়া হয়নি» / «এখনো করা হয়নি». Units: teachers `জন`, students `রেকর্ড`, school shows no total. Students adds « • (এই ব্রাউজারে স্থায়ী স্টোরেজ বন্ধ)» when storage is blocked.

### 10.6 Public API
`UnifiedBackup.backup() → boolean` · `.handleFile(event, host)` · `.recover(host)` · `.metaText(host,count,unit)` · `.VERSION` · `.FORMAT`.
Each page wires: `downloadJSONBackup()`, `handleRestoreFile(event)`, `recoverPrevious()` → these three calls.

### 10.7 Rules & limits
* Keep the module byte-identical across files; bump `APP_VERSION` in all three.
* Other open tabs of these apps can overwrite restored data later — close them first (dialog warns).
* `localStorage` ≈ 5 MB per origin; very large datasets may fail to restore (rollback protects old data).
* Backup files are **plain JSON**; store them safely (no encryption).
* Take a backup before restore, before promotion, and at least weekly.

---

## 11. Storage map

| Key | Owner | Content |
|---|---|---|
| `hemnagar_school_v1` | school | `schoolData` object |
| `hemnagar_teachers_v1` | teachers | array of teachers |
| `hemnagar_teachers_v1__bak` | teachers | mirror of the array (fallback) |
| `hemnagar_teachers_draft_v1` | teachers | `{at, data}` unsaved new-teacher form (7-day TTL) |
| `hemnagar_teachers_sortmode` | teachers | selected sort mode |
| `hemnagar_teachers_v1__prerestore` | teachers (legacy) | old per-app recovery |
| `hemnagar_teachers_last_backup` | teachers | ISO time of last backup |
| `hemnagar_teachers`, `teachers_hemnagar` | teachers (legacy) | old data keys, read-only fallback |
| `hemnagar_students_v20` | students | array of students |
| `hemnagar_auto_backup_before_restore` | students (legacy) | old per-app recovery |
| `hemnagar_last_backup` | students | ISO time of last backup |
| `hemnagar_result_snapshot` | students → result.html | `{timestamp,totalRecords,students,source}` |
| `hemnagar_unified_prerestore` | unified | combined Recovery Backup payload |
| `hemnagar_unified_last_backup` | unified | ISO time |
| `hemnagar_unified_last_restore` | unified | ISO time |

Version suffixes (`_v1`, `_v20`) are **schema versions** — if you change a record shape incompatibly, bump the suffix *and* update the unified module's `SECTIONS[].keys`.

---

## 12. Excel export

* SheetJS loaded on demand (`loadXLSX()` → promise, cached; error toast if offline).
* **teachers:** one sheet «শিক্ষক তথ্য»; columns `ক্রম` + every `FIELD_ORDER` field (labels from `FIELD_LABELS`) + `শিক্ষাগত যোগ্যতা` + `প্রশিক্ষণ` (dynamic rows flattened to text by `dynToText`); exports the **currently filtered & sorted** list; auto column widths, autofilter on; salary written as grouped digits.
* **students:** modal to choose fields (groups §9.4); exports according to chosen fields/scope.
* **school:** sheet «সারসংক্ষেপ» (columns বিভাগ / ক্ষেত্র / মান; classes joined by `, `) + one sheet per non-empty list: ভবন, কক্ষ, ওয়াশব্লক, পানীয় জল, স্যানিটেশন, আসবাবপত্র, আইসিটি সরঞ্জাম.
* Empty data → alert/toast «এক্সপোর্ট করার জন্য কোনো ডেটা নেই!».

---

## 13. Function index per file

*(Names exactly as in code; purposes by name/behaviour.)*

**Shared in all three:** `loadXLSX · toBn · clean · showToast · downloadBlob/dateStamp · updateBackupMeta · downloadJSONBackup · handleRestoreFile · recoverPrevious`

**school.html:** `displayBn · escapeHtml · escapeJs · safeParse · copyToClipboard · buildToc · scrollToSection · updateTocActive · renderDashboard · rerenderSection · renderSection · renderFieldsBody · renderInfoItem · renderDynBody · renderClassesBody · findSection · openEdit · closeEdit · saveEdit · renderFieldsEditor · renderClassesEditor · renderModalDynList · updateModalDynItem · addDynItem · removeModalDynItem · exportExcel · printDashboard · bindScrollSpy · bindModals · init`

**teachers.html:** formatting — `toEn · digitsOnly · yearToBengaliWords · dateToBengaliWords · normalizeDateString · parseDateForSort · formatDateTimeDDMMYYYY · groupBdDigits · formatSalaryBn · formatSalaryInputBn · maskDigits · maskDate · isRealDate` · records — `blankDynRow · normalizeDynRow · normalizeTeacherRecord · composeAddress · saveData · parseTeacherArray · loadTeachers` · form — `optionsHTML · fieldHTML · dynSectionHTML · buildForm · dynFieldHTML · renderDyn · syncDyn · addDynRow · removeDynRow · collectFormRaw · fillForm · resetForm · applySameAddress · updateAddrSummary · openNewForm · closeForm · toggleForm · saveDraft · loadDraft · clearDraft · failValidation · validateForm · handleSubmit` · list — `compareSeniority · compareByMode · getFiltered · clearSearch · designationClass · rowHTML · renderTable` · view — `viewItemHTML · addressItemHTML · subTableHTML · sectionHeaderHTML · viewTeacher · closeViewModal · viewEditFromModal · editTeacher · deleteTeacher` · misc — `copyButtonHTML · copyText · debounce · esc · dynToText · exportExcel · handleFormInput · handleFormChange · handleClick · fillSelect · populateSortSelect · bindEvents · init`

**students.html:** helpers — `toEn · safeParse · escapeHtmlAttr · escapeForJsAttr · debounce · isQuota · bnDigitsOnlyInput · bnDecimalInput · bnDateInput · bnDigitsInput · normalizeDateField · dateToInputFormat · yearToBengaliWords · dateToBengaliWords · readField · writeField · validateInput · attachValidation · attachInputFilters · createCopyButton · copyText` · form — `populateClassSelect · onClassChange · onBulkClassChange · getDisabilityStatus · getDiseaseStatus · updateDisabilityCounter · syncDisabilityPanel · setDisabilityStatus · syncDiseasePanel · setDiseaseStatus · currentAcademicYearBn · toggleForm · resetForm · updateEmergencyPreview · updateDobWords · attachFormEvents · updateAddrSummary · collectFormData · fillForm · isRealDate · validateDates · findDuplicateRoll · handleFormSubmit` · storage — `saveToLocalStorage · isStudentLike · extractStudents · loadStorage` · list — `getSortValue · getUniqueValues · hasAnyDisease · createSortRuleElement · updateFilterValueOptions · addSortRule · updateSortRuleNumbers · getSortRules · getFilteredSortedStudents · clearSearch · clearAllFilters · toggleStudentSelection · toggleSelectAll · updateSelectAllCheckbox · updateSelectionUI · selectVisible · clearSelection · renderTable · refreshFilterOptions` · view — `viewStudent · closeViewModal · printViewModal · viewEditFromModal · viewPromoteFromModal · editStudent · deleteStudent` · promotion — `openPromoteModal · renderPromoteList · updatePromoteField · applyBulkPromote · confirmPromotion · closePromoteModal` · export — `openExcelExportModal · buildExportGroups · expandAllGroups · updateGroupState · updateExportSummary · toggleAllExportFields · updateExportScopeInfo · updateScopeSelection · closeExportModal · confirmExcelExport · openResultFile` · boot — `bindToolbar · bindModals · init`

---

## 14. Rebuild-from-scratch playbook

Follow this order; each step ends with something testable.

1. **Skeleton** — one `.html`, `lang="bn"`, `<meta charset>`, viewport (§3), title «<পেজ নাম> | হেমনগর সরকারি প্রাথমিক বিদ্যালয়», favicon `img/SCHOOL-LOGO-round.png`.
2. **Design tokens** — paste the `:root` token set and the base `body`/`.container` rules (§5). Add header, cards, buttons, form grid, modal, toast.
3. **Helpers** — `$`, `toBn/toEn/digitsOnly/clean/esc`, `Store`, `showToast`, `debounce`, date helpers (§15).
4. **Data model** — decide storage key + schema version; define constants (option lists), the field schema (id, label, kind, required, options, maxlength, default).
5. **Form builder** — generate inputs from the schema (so labels/validation/export all share one source). Add input masks by kind (digits/date/decimal/upper).
6. **Validation** — required → format patterns → real dates → business rules (duplicates, future dates). Fail fast with `failValidation(msg, el)`: toast + focus + `invalid` class.
7. **CRUD** — add/edit via hidden `editIndex`; delete with `confirm`; always snapshot → mutate → `save()` → on failure roll back; then re-render.
8. **List UI** — table/cards, search (AND terms), filters, sorting (single or rule-based), selection if bulk actions are needed, empty state.
9. **View modal + print** — read-only profile, print CSS (hide chrome, show school header block).
10. **Draft autosave** (long forms) — debounce, key `*_draft_v1`, TTL 7 days, clear on save.
11. **Excel export** — lazy `loadXLSX()`, `aoa_to_sheet`, column widths, autofilter, date-stamped file name.
12. **Backup bar + `UnifiedBackup` module** — paste the module unchanged; add the three delegating functions and `updateBackupMeta()`; add the footer markup (§4.3). Add the section to the module's `SECTIONS` registry if it is a *new* app/dataset.
13. **Unified design block** — append last in `<style>` so it overrides page CSS.
14. **Responsive pass** — test 360 / 390 / 768 / 1024 / 1440 / 1920 / 2560 px widths, portrait + landscape; check no horizontal page scroll; tables scroll inside their own wrapper.
15. **Version & credit** — footer credit text, `APP_VERSION`, header comment with author/date.
16. **QA** — run §16.

**Adding a new dataset (e.g. `staff.html`) to the combined backup:** pick key `hemnagar_staff_v1`; add an entry to `SECTIONS` in the module (id, label, kind, keys, valid test, legacy format name, lastKeys) **in all three existing files + the new one**; bump `APP_VERSION`; old backups keep working because missing sections are simply skipped.

---

## 15. Reusable code snippets

### 15.1 Digits & strings
```js
const BN = ["০","১","২","৩","৪","৫","৬","৭","৮","৯"];
const toBn = n => String(n).replace(/\d/g, d => BN[+d]);
const toEn = v => v == null ? "" : String(v).replace(/[০-৯]/g, m => BN.indexOf(m));
const digitsOnly = v => toEn(v).replace(/[^0-9]/g, "");
const clean = v => String(v == null ? "" : v).trim();
const esc = s => String(s == null ? "" : s).replace(/&/g,"&amp;").replace(/</g,"&lt;").replace(/>/g,"&gt;").replace(/"/g,"&quot;");
```

### 15.2 Store with memory fallback
```js
const Store = (() => {
  let available = true; const mem = Object.create(null);
  try { localStorage.setItem("__p","1"); localStorage.removeItem("__p"); } catch (e) { available = false; }
  return {
    get available() { return available; },
    get(k) { if (k in mem) return mem[k]; if (!available) return null; try { return localStorage.getItem(k); } catch (e) { return null; } },
    set(k, v) { if (!available) { mem[k] = v; return true; } try { localStorage.setItem(k, v); return true; } catch (e) { return false; } },
    del(k) { delete mem[k]; try { localStorage.removeItem(k); } catch (e) {} }
  };
})();
```

### 15.3 Date normalise + real-date check (DD/MM/YYYY)
```js
function normalizeDate(value) {
  let v = toEn(value).trim(); if (!v) return "";
  v = v.replace(/[.\-\s]+/g, "/");
  const p = v.split("/").filter(Boolean); let d, m, y;
  if (p.length === 3) { if (p[0].length === 4) [y, m, d] = p; else [d, m, y] = p; }
  else if (p.length === 1) {
    const g = p[0].replace(/\D/g, "");
    if (g.length === 8) { d = g.slice(0,2); m = g.slice(2,4); y = g.slice(4); }
    else if (g.length === 6) { d = g.slice(0,2); m = g.slice(2,4); y = "20" + g.slice(4); }
    else return "";
  } else return "";
  d = String(d).padStart(2,"0"); m = String(m).padStart(2,"0"); if (String(y).length === 2) y = "20" + y;
  if (!/^\d{2}$/.test(d) || !/^\d{2}$/.test(m) || !/^\d{4}$/.test(String(y))) return "";
  if (+d < 1 || +d > 31 || +m < 1 || +m > 12) return "";
  return d + "/" + m + "/" + y;
}
function isRealDate(n) {
  const m = /^(\d{2})\/(\d{2})\/(\d{4})$/.exec(n || ""); if (!m) return false;
  const d = +m[1], mo = +m[2], y = +m[3], dt = new Date(y, mo - 1, d);
  return dt.getFullYear() === y && dt.getMonth() === mo - 1 && dt.getDate() === d;
}
```

### 15.4 Validators
```js
const VALID = {
  mobile: v => /^01[3-9]\d{8}$/.test(v),
  nid: v => /^(\d{10}|\d{13}|\d{17})$/.test(v),
  birthReg: v => /^\d{17}$/.test(v),
  email: v => /^[^\s@]+@[^\s@]+\.[^\s@]{2,}$/.test(v)
};
// always test on digitsOnly()/toEn() value, not on the Bengali display value
```

### 15.5 Money (Bangladeshi grouping)
```js
const groupBdDigits = s => { s = String(s); if (s.length <= 3) return s; const last3 = s.slice(-3), rest = s.slice(0,-3);
  return rest.replace(/\B(?=(\d{2})+(?!\d))/g, ",") + "," + last3; };
const formatSalaryBn = s => { const e = digitsOnly(s).replace(/^0+(?=\d)/, ""); return e ? toBn(groupBdDigits(e)) + "/=" : ""; };
```

### 15.6 Seniority comparator
```js
const RANK = { "প্রধান শিক্ষক":0, "ভারপ্রাপ্ত প্রধান শিক্ষক":1, "সহকারি শিক্ষক":2 };
const key = d => { const n = normalizeDate(d); if (!n) return Infinity; const p = n.split("/"); return +p[2]*10000 + +p[1]*100 + +p[0]; };
const bySeniority = (a, b) => (RANK[a.designation] ?? 99) - (RANK[b.designation] ?? 99)
  || key(a.firstJoiningDate) - key(b.firstJoiningDate) || key(a.dob) - key(b.dob)
  || clean(a.nameBn).localeCompare(clean(b.nameBn), "bn");
```

### 15.7 Save with rollback
```js
function saveAll() { return Store.set(KEY, JSON.stringify(list)); }
const snapshot = list.slice();
isEdit ? list[idx] = rec : list.push(rec);
if (!saveAll()) { list = snapshot; showToast("সংরক্ষণ করা যায়নি (স্টোরেজ পূর্ণ)", "err"); return; }
```

### 15.8 Download a file
```js
function downloadBlob(blob, name) {
  const url = URL.createObjectURL(blob), a = document.createElement("a");
  a.href = url; a.download = name; document.body.appendChild(a); a.click(); a.remove();
  setTimeout(() => URL.revokeObjectURL(url), 1500);
}
```

### 15.9 Backup bar HTML skeleton
```html
<div class="app-footer">
  <div class="backup-bar">
    <div class="backup-info">
      <div class="title"><svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"></svg> ডেটা ব্যাকআপ ও সুরক্ষা</div>
      <div class="meta" id="backupMeta">সর্বশেষ ব্যাকআপ: এখনো নেওয়া হয়নি • সর্বশেষ রিস্টোর: এখনো করা হয়নি</div>
    </div>
    <div class="backup-actions">
      <button class="btn-backup-sm primary" type="button" onclick="downloadJSONBackup()">JSON ব্যাকআপ</button>
      <input type="file" id="restoreFileInput" accept=".json,application/json" hidden onchange="handleRestoreFile(event)">
      <button class="btn-backup-sm secondary" type="button" onclick="document.getElementById('restoreFileInput').click()">রিস্টোর</button>
      <button class="btn-backup-sm secondary" type="button" onclick="recoverPrevious()" title="রিস্টোরের আগের অবস্থা ফিরিয়ে আনুন">Recovery Backup</button>
    </div>
  </div>
  <div class="footer-credit">Teacher data store by <strong>Naimul Hassan</strong> v1.1.1-pro</div>
</div>
```

---

## 16. QA checklist

**Data**
- [ ] Add, edit, delete one record in each app; reload page — data persists.
- [ ] Required-field, mobile, NID, birth-reg, email, date, future-DOB errors appear and focus the field.
- [ ] Duplicate IPEMIS/NID (teachers) and duplicate roll (students) are caught.
- [ ] Bengali and English digit input both accepted; stored form matches §6.2.
- [ ] Sorting (teachers seniority; students multi-rule), search, filters behave.
- [ ] Draft (teachers) survives a reload; clears after save.

**Backup / restore**
- [ ] Backup from each page produces **one** file containing all three sections with correct counts and `appVersion`.
- [ ] Restore into an empty browser from each page restores all three sections.
- [ ] Untick a section → it stays unchanged; students merge adds only new keys.
- [ ] Recovery Backup returns the pre-restore state.
- [ ] Old single-app backup files still restore.
- [ ] Invalid JSON / wrong file → error toast, no dialog.
- [ ] Info line shows backup and restore times on all three pages.

**UI**
- [ ] Same backup bar, buttons, footer, toast, dialog look on all three pages.
- [ ] No horizontal page scroll at 360 px; header stacks; buttons full-width ≤420 px.
- [ ] Tablet (768/1024) and desktop (1440/1920/2560) layouts use space sensibly.
- [ ] Print: chrome hidden, school header shown.
- [ ] Keyboard: Tab order, visible focus, Escape closes the restore dialog.
- [ ] Missing `img/SCHOOL-LOGO-round.png` does not break the header.

**Release**
- [ ] Version string `1.1.1-pro` (or new) updated in footer credit, module `APP_VERSION`, header comment.
- [ ] CRLF line endings preserved; the three embedded modules identical.

---

## 17. Coding guidelines, gotchas, to-do

### Guidelines
1. **One schema drives everything** — form, validation, view, export, search all read the same field list.
2. **Escape on output** (`esc`) for every user value placed in HTML.
3. **Persist immediately**, roll back on failure, never leave memory and storage out of sync.
4. **Normalise on save, format on display** (teachers: English digits stored; students: Bengali digits stored — don't mix within an app).
5. Keep UI text Bengali; keep keys English; keep messages short and actionable.
6. Never block the UI on Excel loading; show a toast on failure.
7. Treat any change to the shared module, design block, or token as a **three-file change**.
8. Never store or print personal data in logs; backups stay on the user's device.
9. After editing shipped files with scripts, re-run syntax checks on every inline `<script>` and parse the `<style>` block.

### Known gotchas
* Combined backup needs **same origin** (see §2).
* Restoring while another tab is open can be overwritten by the stale tab.
* `file://` in Firefox isolates storage per file.
* Teacher vs student digit storage differs (§6.2) — convert before sharing data.
* `students.html` previously had a per-app restore modal; it was removed — all restores go through the unified dialog.
* The restore reloads the page; unsaved form input on that page is lost.
* If storage is blocked (private mode), data lives only in memory and backup will report "no data".

### To-do / ideas (not implemented)
- Optional password/encryption for backup files.
- Auto-reminder when last backup is older than N days.
- Share one `UnifiedBackup`/design file via a build step to avoid triple maintenance.
- IndexedDB for larger datasets (photos).
- Photo field (compressed, stored separately).
- Staff / attendance / fees datasets using the same registry pattern (§14).
- Import from Excel (round-trip of the export).

---

## 18. AI prompt template for rebuilding

Paste this together with this file when asking an AI to regenerate or extend an app:

```
You are rebuilding "<teachers|students|school>.html" for হেমনগর সরকারি প্রাথমিক বিদ্যালয়.
Follow instructions.md exactly: single self-contained HTML file, vanilla JS, Bengali UI,
localStorage key <key>, schema from §<7|8|9>, formats from §6, design tokens + responsive
rules from §5, shared UnifiedBackup module and backup bar from §10/§4.3, version 1.1.1-pro.
Deliver the complete file (CRLF), then list what you tested.
Do not add frameworks, external fonts, or network calls (SheetJS lazy load is the only exception).
```

---

*End of guide · Suite version 1.1.1-pro · generated from the shipped source files.*
