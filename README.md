# School Automation — Hemnagar Government Primary School

A set of offline web pages for running everyday school paperwork: student and teacher
databases, results, certificates, seat stickers, attendance forms, leave applications and more.
Everything runs inside your web browser. No installation and no server are needed.

**Version:** 1.1.1 pro

---

## Quick start

1. Unzip this folder anywhere on your computer (for example `Documents`).
2. Install the fonts in the `fonts/` folder (see `INSTRUCTIONS.md`, Step 2).
3. Open `app/index.html` in Chrome, Edge or Firefox.
4. Use the portal page to open any tool.

For the full guide, read **[INSTRUCTIONS.md](INSTRUCTIONS.md)**.

---

## Folder structure

```
School_Automation/
├── README.md              <- this file (overview)
├── INSTRUCTIONS.md        <- step-by-step guide for users
│
├── app/                   <- THE PROGRAM. Open app/index.html
│   ├── index.html         <- home portal (links to every tool)
│   ├── students.html      <- student information database
│   ├── teachers.html      <- teacher information database
│   ├── school.html        <- school information dashboard
│   ├── result.html        <- result sheet maker
│   ├── certification.html            <- certificate
│   ├── certificationWithPhoto.html   <- certificate with photo
│   ├── seat-sticker.html  <- exam seat stickers
│   ├── valuation-form.html <- assessment (valuation) record form
│   ├── mr-front.html      <- MR front page
│   ├── mr-teachers.html   <- teacher attendance statement (MR)
│   ├── leave.html         <- casual leave application
│   ├── pion.html          <- daily attendance & bill form (office assistant)
│   ├── pion-blank.html    <- blank version of the form above (opened from pion.html)
│   └── img/               <- logos and certificate backgrounds used by the pages
│
├── fonts/                 <- fonts to install on the computer (Bangla + digital clock)
├── backups/               <- saved data backups (.json files)
└── docs/
    └── design-reference.md <- technical/design reference for developers
```

> **Important:** keep the `app/` folder exactly as it is. The pages find each other and
> the `img/` folder by relative paths, so the HTML files and `img/` must stay together.

---

## What changed from the original archive

| Change | Details |
|---|---|
| Code | **Not changed.** All 14 HTML pages are byte-for-byte identical to the originals. |
| Organized | Pages + images grouped in `app/`; fonts, backups and docs each in their own folder. |
| Removed (old copies) | `certification copy.html`, `certificationWithPhoto - Copy.html` (older versions, never linked). |
| Removed (unused images) | `SCHOOL-LOGO-round bw.png`, `SCHOOL-LOGO-round.jpg`, `SCHOOL-LOGO-round.svg`, `SCHOOL-LOGO-round_bw.png` (no page uses them). |
| Renamed folders | `Fonts` -> `fonts`, `_Backups` -> `backups`, `Design Instructions/instructions.md` -> `docs/design-reference.md`. |
| Added | `README.md`, `INSTRUCTIONS.md`. |

---

## Requirements

- A modern browser: Google Chrome, Microsoft Edge or Firefox (latest versions recommended).
- Windows, macOS, Linux, or any system that can open HTML files.
- Internet is not required for normal use.
