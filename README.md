# ftree — Family Tree Viewer

A family tree that reads straight from a Google Sheet. One HTML file, no backend,
no build step, no database — you edit the sheet, the chart follows.

As featured in [this video](https://www.youtube.com/watch?v=sqA5RL_FUuo) ·
**[Live demo](https://techleadhd.github.io/ftree/)**

<img width="3338" height="1688" alt="The chart: five generations, with one person selected and their ancestry drawn in bold" src="https://github.com/user-attachments/assets/77748077-8768-4a1f-9b9a-7812ca42ef6a" />

<img width="1914" height="1090" alt="The same family as rows in a Google Sheet" src="https://github.com/user-attachments/assets/872da11a-17a2-4f2e-ba75-79fd0c138fba" />

*The whole database: one tab, one row per person.*

## Get started

1. **Make the sheet.** Import [`demo-family.csv`](demo-family.csv) into a new
   Google Sheet, then replace the invented family with your own.
2. **Share it.** *Share → General access → Anyone with the link → Viewer.*
   The page reads the sheet as an anonymous visitor; without this it sees nothing.
   Any linked photos also need to be shared.
4. **Open the chart.** `index.html?sheet=<paste the sheet link or its id>` —
   or open `index.html` with no parameter and paste the link into the box.

That URL is the thing you send to family. Host `index.html` anywhere static —
Cloudflare Pages, GitHub Pages, an S3 bucket — or just open it off your disk;
it works from `file://` too.

## The sheet

| Column | Holds |
|---|---|
| `name` | The person. **This is their id**, so names have to be unique. |
| `sex` | `M`, `F`, or blank — decides the box colour and words like *niece*. |
| `birth_year`, `death_year` | Years. Blank is fine. |
| `parents` | **Both parents in one cell**, comma separated: `Ada Rivera, Peter Rivera`. One name for a single parent. |
| `drive_photo` | A Google Drive link, a bare Drive file id, or any image URL. |
| `description` | A line or two about them, shown in the box and the panel. |

Four rules worth knowing:

- **`parents` is what builds the tree.** It creates the couple as well as the
  parent-child link, so a couple with children needs no row of their own.
- **A couple with no children between them is a nameless row** — fill in
  `parents`, leave every other column empty, and they get drawn as a couple.
- **A name starting with `#` is a comment.** The whole row is skipped, so you
  can leave notes for whoever edits the sheet next.
- **Matching is forgiving.** Case and extra spaces are ignored, and the two
  names in a `parents` cell work in either order.

Anything the sheet gets wrong — a parent nobody has a row for, a duplicated
name, three names in a `parents` cell — is reported in the page itself rather
than quietly dropping a branch.

## What it does

- **Remarriage, half-siblings and single parents** are first-class, not special
  cases; the model underneath is person → union → children.
- **Click anyone** for their photo, dates, story and relations, with their
  ancestry and siblings drawn in bold and everyone else dimmed.
- **"How are we related?"** names the connection between any two people —
  *second cousin once removed*, *your partner's brother* — and stays on so you
  can walk down a row of relatives.
- **Two levels of detail**, chosen by zoom: the full box with photo and story up
  close, name and dates when you pull back.
- **Permalinks.** The URL carries whoever is selected (`#ada-rivera`), so you can
  send someone straight to their grandmother.
- **Touch.** Pinch to zoom, drag to pan, and the detail panel becomes a sheet on
  a phone.

## A word on privacy

A link-shared sheet is readable by anyone who has the URL, and so is every photo
in a link-shared Drive folder. The chart's URL carries the sheet id, so it lives
in browser history and forwarded email.

The deployed file itself names no family: with no `?sheet=` parameter it loads
the invented demo family, so a stranger who finds the page learns nothing.
