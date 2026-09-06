# ftree — Family Tree Viewer

A family tree that reads straight from a Google Sheet. One HTML file — no
backend, no build step, no database. You edit the sheet, the chart follows.

As featured in [this video](https://www.youtube.com/watch?v=sqA5RL_FUuo) ·
**[Live demo](https://techleadhd.github.io/ftree/)**

<img width="3338" height="1688" alt="The chart: five generations, with one person selected and their ancestry drawn in bold" src="https://github.com/user-attachments/assets/77748077-8768-4a1f-9b9a-7812ca42ef6a" />

<img width="1914" height="1090" alt="The same family as rows in a Google Sheet" src="https://github.com/user-attachments/assets/872da11a-17a2-4f2e-ba75-79fd0c138fba" />

*The whole database: one tab, one row per person.*

## Get started

1. **Make the sheet.** Import [`demo-family.csv`](demo-family.csv) into a new
   Google Sheet, then replace the invented family with your own.
2. **Share it.** *Share → General access → Anyone with the link → Viewer.*
   The page reads it as a stranger would; without this it sees nothing. Photos
   need sharing too.
3. **Open the chart.** `index.html?sheet=<the sheet link>` — or open
   `index.html` with no parameter and paste the link into the box.

That URL is the thing you send to family. Host `index.html` anywhere static, or
just open it off your disk — it works from `file://` too.

## The sheet

One row per person. First row is the header.

| Column | Holds |
|---|---|
| `name` | The person, and their id — so names must be unique. |
| `sex` | `M`, `F`, or blank. Sets the box colour and words like *niece*. |
| `birth_year`, `death_year` | Years, or blank. |
| `parents` | **Both parents in one cell**, comma separated: `Ada Rivera, Peter Rivera`. One name for a single parent. |
| `drive_photo` | A Drive link, a Drive file id, or any image URL. |
| `description` | A line or two, shown on the card. |

- **`parents` builds the tree.** It makes the couple as well as the
  parent-child link, so a couple with children needs no row of their own.
- **A childless couple is a nameless row** — fill in `parents`, leave the rest
  blank.
- **A row whose name starts with `#` is a comment**, and is skipped.
- **Matching is forgiving** about case, spacing, and which parent you put first.

Mistakes — a parent with no row, a duplicate name, three names in one
`parents` cell — are reported on the page, not silently dropped.

## Ordering

**Left to right follows top to bottom.** Whoever is higher in the sheet stands
further left. That's the whole rule, and it's the only one — `birth_year` is
shown but never used to sort, so a missing or wrong year can't move anybody.

To move someone left, move their row up. It works the same for siblings, for
a person's several marriages, and for whole generations. List three husbands
above the wife and you get `[husband 1] [husband 2] [husband 3] [wife]`.

Two things outrank it: partners stay side by side, and parents stay centered
over their children. You'll only notice if the sheet contradicts itself — say
generation 1 runs Ada, Bea, Cara, but Cara's children are listed before Ada's.
Then keep the families in the same relative order all the way down, and the
order you wrote is the order you get.

## What it does

- **Remarriage, half-siblings and single parents** are first-class, not special
  cases.
- **Click anyone** for their photo, dates, story and relations, with their
  ancestry and siblings picked out in bold.
- **"How are we related?"** names the connection between any two people —
  *second cousin once removed*, *your partner's brother*.
- **Permalinks.** The URL carries whoever is selected (`#ada-rivera`), so you
  can send someone straight to their grandmother.
- **Touch.** Pinch to zoom, drag to pan, flick to send the chart coasting. On a
  phone the card is a sheet you can push down out of the way — it parks at the
  bottom with the name still showing.

## A word on privacy

A link-shared sheet is readable by anyone with the URL, and so is every photo in
a link-shared Drive folder. The chart's URL carries the sheet id, so it lives in
browser history and forwarded email.

The file itself names no family: with no `?sheet=` it loads the invented demo,
so a stranger who finds the page learns nothing.
