# Instructions — How to use School Automation

Written for everyday users. No programming knowledge is needed.

---

## Step 1 — Unzip

Right-click the zip file and choose **Extract All** (or use 7-Zip / WinRAR).
Put the `School_Automation` folder somewhere permanent, such as `Documents`.

> Do not move the files out of their folders. Do not open pages from inside the zip file;
> always extract first.

## Step 2 — Install the fonts (one time only)

The pages use these fonts for correct Bangla text and printing. Open the `fonts/` folder, then:

- **Windows:** select all font files, right-click, choose **Install**.
- **macOS:** double-click each font, then click **Install Font**.

| Font file | Used for |
|---|---|
| `BCC Purno Regular.ttf`, `BCC Purno Black.ttf` | Main Bangla text and headings in printouts |
| `Nikosh.ttf`, `NikoshBAN.ttf` | Backup Bangla fonts |
| `digital-7.ttf`, `digital-7 (mono).ttf` | Digital clock style on the home page |

Restart your browser after installing.

## Step 3 — Open the program

Open the `app` folder and double-click **`index.html`**. This is the home portal.
From there, click the tool you need. You can also bookmark it in your browser.

## Step 4 — The tools

| Page | What it does |
|---|---|
| `students.html` | Add, edit, search and export student information |
| `teachers.html` | Add, edit, search and export teacher information |
| `school.html` | School information dashboard |
| `result.html` | Prepare result sheets for printing |
| `certification.html` | Certificates |
| `certificationWithPhoto.html` | Certificates with a photo |
| `seat-sticker.html` | Seat stickers for exams |
| `valuation-form.html` | Assessment (valuation) record form |
| `mr-front.html` | MR front page |
| `mr-teachers.html` | Teacher attendance statement |
| `leave.html` | Casual leave application |
| `pion.html` | Daily attendance and bill form for the office assistant |
| `pion-blank.html` | Blank form (opened from `pion.html`) |

## Step 5 — Backups (very important)

Your data is stored **inside your browser**, on this computer. It is not in the folders.
That means clearing browser data, changing browser, or changing computer can lose it.

- Use the **Backup / Export** button inside the apps regularly (for example every week).
- Save the downloaded `.json` file into the `backups/` folder.
- To restore, open the same app and use **Restore / Import** and choose the `.json` file.

The `backups/` folder already contains earlier backups:

| File | Contents |
|---|---|
| `hemnagar_full_backup_20-10-2026_11-13.json` | Full backup (most recent) |
| `school_backup_12-10-2026.json` | School data only |
| `students_backup_12-10-2026_13-01.json` | Students only |
| `teachers_backup_12-10-2026_12-53.json` | Teachers only |

Tip: keep a second copy of `backups/` on a USB drive or cloud storage.

## Step 6 — Printing

- Use Chrome or Edge for best results.
- In the print window, set **Margins** as the page suggests and enable **Background graphics**
  if colors or backgrounds are missing.
- Make sure the fonts from Step 2 are installed before printing.

---

## Moving to a new computer or a new folder

Browsers can store data separately for each folder location. If you move the project to a
different folder or computer and your data seems missing, do not worry:

1. Open the app in the new location.
2. Restore from your latest backup file (Step 5).

## Troubleshooting

| Problem | Solution |
|---|---|
| Logo or images missing | The `app/img/` folder must sit next to the HTML pages. Do not move it. |
| Bangla text looks wrong | Install the fonts (Step 2) and restart the browser. |
| Data disappeared | Restore from a backup in `backups/` (Step 5). |
| A page does not open from the portal | Open the project from the extracted folder, not from inside the zip. |
| Print layout is off | Use Chrome/Edge, enable Background graphics, check the paper size. |

## For developers

Technical notes, design rules and the rebuild guide are in `docs/design-reference.md`.
The code is plain HTML, CSS and JavaScript with no build step.
