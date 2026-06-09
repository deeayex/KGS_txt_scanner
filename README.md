# Lease Production Tools — Guide

**American Warrior, Inc.**

A simple tool for exploring our oil-lease data. It opens in a web browser, there's
nothing to install, and the data file you load never leaves your computer.

The app has two tabs: **Scanner** (search and filter the data) and **Map** (see the
leases on a map of Kansas).

---

## Getting started

1. **Open** `lease_scanner.html` — double-click it and it opens in your browser
   (Chrome, Edge, Firefox, or Safari).
2. **Load your data** — drag the lease file onto the box, or click to browse for it.
   This is the `.txt` file of lease production data. When you get a newer file later,
   just load it the same way.
3. The tool reads the file and shows a quick summary (records, leases, months). Two
   tabs appear: **Scanner** and **Map**.

---

## The Scanner tab

Set what you're looking for, then press **Scan**:

- **Match rule** — search a **date range** (a From month through a To month), or a
  single **exact month**. For "from a month up to now," leave the To box on the
  newest month.
- **Minimum production** — the lowest number of barrels you want to see (0 = all).
- **Operator** (optional) — start typing a company name and pick it from the list
  that appears.
- **County** (optional) — start typing a county and pick it from the list. Leave it
  blank for all counties.

**Reading the results:**

- Each matching line is shown in a table.
- **Click any column heading to sort** — names sort A–Z, numbers sort high-to-low
  (click again to reverse).
- **Download matches (CSV)** saves the full results as a spreadsheet (it includes a
  few extra columns that aren't shown on screen, and follows whatever sort you set).
- The table shows up to the first 1,000 matches for speed; the CSV always has them
  all.

---

## The Map tab

Every lease is a dot on a map of Kansas. **Bigger, darker-orange dots produce more**;
small pale dots produce little. The legend in the corner explains the sizes.

- **Click any dot** for that lease's details, including a **production-per-day**
  figure and a link to its official KGS page.
- **All leases / Scan results** — switch between every lease and just the ones from
  your last scan. After you run a scan it automatically focuses on those results, so
  you can scan to narrow things down, then see exactly those leases on the map.
- **American Warrior wells (past year)** — our own active wells appear as blue **"A"**
  markers that stay on the map no matter what else you're viewing, so you can see
  which of our wells sit near big producers. Use the checkbox to hide or show them.
- If a lease can't be placed (no coordinates in the data), a small note under the map
  lists which ones.

**Note:** the Scanner works without internet, but the **Map's background needs an
internet connection** to load the map of Kansas. Your data still stays on your
computer either way.

---

## Sharing it

`lease_scanner.html` is one self-contained file — the logo, the code, everything is
inside it. Email it or put it on a shared drive, and it works the same for anyone who
opens it. Each person loads their own copy of the data file.
