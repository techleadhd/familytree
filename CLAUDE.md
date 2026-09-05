# Family tree visualizer

A single-file, static HTML chart of a four-generation family, reading live from
Google Sheets. No backend, no build step, no dependencies beyond a webfont.

## Files

| File | Role |
|---|---|
| `index.html` | The whole application. Open it directly or host it anywhere static. |
| `demo-family.csv` | An invented 49-person, five-generation family. Import it into a sheet to demo the chart, or to test against something other than the real family. Not read at runtime. |

The HTML embeds no family data at all — see Where the sheet id lives.

## Where the sheet id lives

**In the URL, not in the file.** `index.html?sheet=<id>` — or paste the whole
Google Sheets URL as the parameter, the id gets extracted. Anything outside
`[A-Za-z0-9_-]` is stripped before the id reaches the gviz URL.

With no parameter the page loads `DEMO_SHEET`: the invented family from
`demo-family.csv`, in a public sheet. An empty form explains nothing about what
this page is, and a chart does. The demo is not a fallback — a named sheet
always wins, and a sheet that fails to load still gets the setup card and the
error, never the demo quietly standing in for it. The footer says "This is a
made-up family" whenever the demo is what's on screen, next to a button that
opens the setup card pre-filled with the demo id, so the way to your own sheet
is one click from anywhere. That card is dismissible only when there is a chart
behind it; on the error path closing it would leave an empty page.

This is obscurity, not privacy, and the distinction matters:

- The deployed file names only the demo sheet and contains no family data, so a
  stranger who finds the URL gets an invented family, and a crawler indexes
  nothing real.
- The link you send to family **does** carry the id, so it lives in browser
  history, bookmarks and forwarded email.
- Anyone with that link can open the sheet directly and read every column,
  including any the chart doesn't render. Don't keep anything in the sheet
  you wouldn't show.

There is deliberately **no built-in family to fall back on**. If the sheet
can't be read you get the setup card with the error, not a stale tree
pretending to be the current one.

## Data source

One tab. The loader tries the tab named `PEOPLE_TAB` (`People`) and falls back
to whichever tab the sheet opens on, so the tab name doesn't really matter.
The sheet must be shared as "anyone with the link can view."

`name, sex, birth_year, death_year, parents, drive_photo, description`

**The name is the identifier**, so names have to be unique. There is no id
column, no generation column and no separate Unions tab.

**`parents` is one cell holding both names, comma separated** —
`Ada Rivera, Peter Rivera` — or a single name for a single parent. That
cell is also what creates the couple, so a couple with children needs nothing
of its own.

**A couple with no children between them is a nameless row**: `parents`
filled in, every other column empty. Without it they'd never be named
anywhere and wouldn't be drawn. (A row with neither a name nor parents is
flagged and skipped; a wholly empty row is ignored.)

**A name starting with `#` is a comment** and the whole row is skipped —
before validation, so a note never reads as a person with a missing surname,
and skipping the row rather than the cell leaves the other columns free for
the note to run into. This is where the sheet says things a sheet cannot say
in a column: where the source code lives, how the `parents` cell works, a
heading over the next block of rows. `demo-family.csv` opens with four of
them. It is also the escape hatch for temporarily retiring a row without
deleting it — put a `#` in front of the name and every `parents` cell that
mentioned that person will be flagged, which is the point.

Matching is forgiving on purpose, since a name gets typed many times: case and
runs of whitespace are ignored, and the two names in either order are the same
couple. These six column names are the only ones the loader reads; the older
`parent_union` / `blurb` / `given_name` + `surname` fallbacks are gone.

## The one design decision that matters

**A family tree with remarriage is not a tree. It's a DAG.** Once someone has
children with two partners, parent→child edges alone can't say who belongs with
whom. So unions are first-class nodes: a person points at a union, and the union
points at the children. This is the GEDCOM model. Half-siblings, single parents,
and second marriages all fall out of it without special cases.

The sheet hides this: there is no union column at all, just `parents` reading
like plain English. But the model underneath is still
person → union → children, and `buildModel` is where the flat rows become that.

Do not "simplify" this back into a parent-pointer tree. It will look like it
works until the first remarriage.

## How layout works

1. **Generation (y)** — an integer per person, always inferred; there is no
   column to override it. Relaxed to a fixpoint: children sit one below the
   lower of their parents, and people who married in with no parents of their
   own get pinned to their spouse's generation. Only married-ins get pinned, never blood relatives,
   so a cousin marriage can't distort a lineage.

2. **Chains** — connected components of the spouse graph, ordered by walking
   from the endpoint whose union appears first in the sheet. (This used to key
   off marriage years, which no longer exist; sheet order is the replacement,
   which means reordering rows is how you nudge a chain.) Produces
   `Susan — Henry — Patricia` rather than an arbitrary order.

3. **Horizontal (x)** — three passes. `claim` walks down from the root chains
   building a tree and recording which chain owns which child. `measure` computes
   subtree widths bottom-up. `place` assigns x top-down, centering each couple
   over the span of their children. Children before parents is what makes couples
   land centered above their kids.

4. **Connectors** — four primitives, all axis-aligned: a marriage bar between
   partners, a drop line from its midpoint, a horizontal sibling bus, and short
   stubs down to each child. Cross-links (below) are the fifth primitive and
   the only one that isn't local. This is why the layout is hand-rolled rather than
   dagre or elk — generic graph layouters produce diagonal splines and will
   separate a married couple to reduce edge crossings.

## Photos

`drive_photo` takes whatever a family member manages to paste: any of Drive's
link shapes (`/file/d/<id>/view`, `?id=<id>`, `/uc?export=view&id=<id>`), a bare
Drive file id, a URL on any other host, or a filename sitting next to this
page. The column is named for the workflow, not the limit: a URL on any other host
works too. But nothing downsizes those, so a relative pasting a 4MB original
from elsewhere would ship all of it to every viewer — hence the name.

`photoURL()` pulls the id out of the Drive shapes and rewrites them to
`lh3.googleusercontent.com/d/<id>=w640`, so the page pulls thumbnails rather
than 4MB phone originals and Google does the downscaling. A bare id is told
apart from a filename by having no dot or slash in it.

`PHOTO_PX` is one size for both the chart avatar and the panel, so the two
share a cache entry and each face is fetched once. It is sized for the panel
on a 2x display, which leaves both chart avatars over-supplied — deliberately,
because the avatar is the thing people zoom into, and because one URL for every
size is what keeps a face to a single fetch.

The `lh3` host is undocumented. It is the one part of this that Google could
break without warning; the supported alternative is Drive's `thumbnailLink`
field, which needs an API key and a `files.list` call the page doesn't make.
If photos stop loading, that is the first thing to check.

**The photo band costs nothing until it is used.** `hasPhotos` is set in
`buildModel`, and only then does `setMode` add `PHOTO_BAND` to `BOX_H` and
`GEN_H`. An empty photo column renders exactly the chart that existed before
the column did.

**Both detail levels show faces**, at their own radius: `PHOTO_R` is a property
of the mode (32 full, 18 compact), and `PHOTO_BAND` is derived from whichever
is current, so a compact box grows 42px rather than 70. What makes a zoomed-out
box unreadable is ten lines of description, not a face — and a face is usually
the fastest way to find someone from across the chart, which is exactly what
zoomed-out is for. `.compact .initials` shrinks the text in an empty circle to
match, and the baseline offset follows the type size.

People without a photo get their initials in a circle, so a row with one
photo in it doesn't look broken. `PHOTO_BAND` is derived from `PHOTO_R`, so
resizing the face is one number per mode; the only gap under the circle is the
name's own leading. Both are drawn through a single
`clipPathUnits="objectBoundingBox"` circle, which crops any image to a circle
regardless of its size or position — one clip path for the whole chart.

Drive photos of living relatives in a link-shared folder are world-readable
to anyone with the URL, same as the birth years. Still unresolved.

## Detail levels

The chart renders at one of two sizes, chosen by zoom (`DETAIL` in the config
block). Zoomed out, a ten-line description per box is an unreadable smear, so
boxes collapse to a name and a date range; past `ZOOM_IN` they expand again.
The gap between `ZOOM_OUT` and `ZOOM_IN` is hysteresis — without it the mode
flaps while the wheel is turning.

Only heights differ between the modes. x positions depend on `BOX_W` alone,
so switching re-renders but nothing moves sideways, and the view is pinned by
whatever sits in the middle of the viewport. `DROP` is derived from `BOX_H`
rather than tuned by hand, so a new mode only needs its three numbers.

Re-rendering throws away the person groups, and with them every class that
carries state — selection, dimming, the comparison target, search hits. That
is why `render()` ends in `repaint()`, which puts all of them back. Anything
new that lives in a class on `.person` has to be added there or it will
vanish the first time someone zooms. The same goes for anything keyed to
coordinates: `linkGeom` and the lit-lineage layer are rebuilt by every render,
and `repaint()` redraws the bold run from them.

## The selected lineage

Selecting someone dims everyone off their line — `lineage` is the line up, the
line down, and the row they stand in (siblings and half-siblings, whose
partners and children stay dim). Siblings hang off a parent union the up-walk
passes straight through, so they have to be added explicitly. On top
of that, their **ancestry and their siblings** are drawn in bold — pine
connectors from the person up to every ancestor and sideways to each sibling,
and a heavier pine border on those boxes (`.kin`, 1.75 against the selection's
marine 2.75, so the person you clicked still leads). Descendants stay undimmed but
unbolded: the bold reads up and across, which keeps a large family from
becoming a solid green mesh. A half-sibling's edge carries `bar:false`, so the
run to the shared parent is bold but the parent's other marriage bar is not.

The bold is a **separate overlay**, not a class on the existing rules, because
the run for one child is not the same as the segments under it: its own stub,
the stretch of sibling bus back to the drop, then the drop and the marriage
bar. Lighting the whole bus would claim every sibling hanging off the rest of
it — siblings are lit one stub at a time, by the same edge machinery. So `render()` records each union's connector coordinates in `linkGeom`
(bar, barY, midX, busY, and per-child stub x) as it draws, and `paintLineage()`
retraces `M stubX,boxTop V busY H midX V barY` into `litLayer` — a group
between the rules and the boxes, so bold passes behind a box and never over its
text. A cross-link is the exception: it is already one path for one child, so
it gets a `.lit` class in place, hop and all.

`ancestry()` guards against revisiting a person, since a cousin marriage can
rejoin a branch and would otherwise walk the shared ancestors twice.

## Permalinks

The URL hash is the selected person's name, slugified (`#nathan-smith`).
Selecting sets it, opening a link selects and centres that person at zoom 1,
and `hashchange` handles the back button. Links are readable and survive the
sheet being re-ordered; only renaming someone breaks them.

`history.replaceState` throws on some `file://` origins, so `setHash` falls
back to assigning `location.hash`. That fallback fires `hashchange`, which is
harmless only because `openHash` ignores a hash that already matches the
selection — don't remove that guard.

## Relationships

`kinship(a, b)` names what B is to A. It finds the nearest common ancestor,
preferring the closest and then the least lopsided, and reads the term off
the two distances: (0, n) descendant, (n, 0) ancestor, (1, 1) sibling — full
or half depending on whether the parent union matches — (n, 1) aunt/uncle,
(1, n) nephew/niece, anything else a cousin with degree and removal. Terms
follow `sex` where it is known and stay neutral where it isn't.

Married-ins have no blood path to most of the tree, so `kinship` falls back
to one step through a marriage: `your first cousin's partner`, `your
partner's brother`. **One step only** — two patriarchs whose children married
each other come back as no relation, which is correct-ish and much better
than a search that wanders the whole graph.

All of this sits above the `/* ---- render ---- */` marker and is DOM-free,
so it can be tested in Node (see Testing).

The picker in front of it is a **toggle, not a one-shot**. "How are we
related?" arms comparison mode and stays armed: each click on the chart
replaces `compareId` and re-answers from the selected person, so you can walk
a row of cousins without reaching back for the button. It disarms on a second
press, on Escape, and whenever the panel closes — including a click into
empty space, which dismisses the panel outright rather than only disarming the
picker; Escape is the exit that keeps the selection. `picking` has exactly one
writer, `setPicking()`, which also carries the button's `aria-pressed` state —
the pressed styling is driven off that attribute, so state and chrome cannot
drift. A second click on the same person always means undo, armed or not: on
the selected person it dismisses the panel and clears the selection, on the
person being compared it drops the answer and waits for the next pick. Both
ends of a comparison get the same pine outline — the compare target used to be
brick red, which claimed a second meaning the panel's sentence already carried.

## Known limitations

- **Commas are structural.** `parents` splits on commas, so a name containing
  one (`John Smith, Jr.`) will be read as two people. In the CSV seed the cell
  itself has to be quoted; in the sheet it's just a cell and needs nothing.
- **Names as identifiers.** Renaming someone in the sheet orphans every
  `parents` cell that names them, and two cousins genuinely called the same
  thing can't both exist. This was the deliberate trade for a sheet a family
  member will actually keep up to date. A hidden id column is the escape hatch
  if it ever bites.
- **Cross-links.** When two branches reconnect, whichever branch reaches a
  person first claims them, and the other has to draw a long connector to
  where they actually sit. Currently two of these, one of them 2,232px long.
  Inherent to strict generational banding — the alternative is duplicating
  people, which is worse: one box per person is what makes selection, dimming
  and the relationship finder unambiguous. `sort_order` to pin horizontal
  order by hand would shorten them, and is still the planned mitigation.

  They used to be dashed, which was the wrong fix for the wrong problem. The
  actual damage was that they ran at `barY + DROP` — exactly the height of
  every sibling bus in that generation — so a long line lay *on top of* two
  short ones and read as one continuous bus, quietly adopting someone into
  the wrong family. Now they get `CROSS_LANE` (88px under the box, below both
  the +5 and +56 bus heights, still 36px clear of the next generation),
  staggered by `CROSS_STEP` per link per row so two of them can't overlap
  each other either. With the collinearity gone they are drawn solid and hop
  over the verticals they cross: `hops()` arcs over any vertical whose y-range
  spans the lane. Every real junction in this chart is a T, so a crossing was
  already distinguishable; the hop makes it unambiguous.

  `verticals` accumulates as the local passes draw, and cross-links push their
  own segments as they go, so a second cross-link hops the first. Only the
  horizontal run is hopped — the short descents are left plain. The footer
  key that explained the old dashed line is gone: a hopped line that runs
  half the chart explains itself.
- **Description wrapping is estimated, not measured.** SVG text doesn't wrap,
  so `wrap()` breaks by character count (`DESC_CHARS = 27`, `DESC_LINES = 10`,
  and `BOX_H` is sized to hold that many lines). Long unbroken words
  can run slightly wide. A real fix measures with `getComputedTextLength()`.
- **Width.** Generation 3 is ~2,300px, and fit-to-screen is limited by width,
  not height — which is why compact mode leaves so much empty space above and
  below. A focus mode showing two generations either side of a selected person
  would still beat panning, and doesn't exist yet.
- **Privacy.** Link-sharing makes birth years and full names of living people
  world-readable to anyone with the URL. Unresolved. Options: a `public` flag
  column suppressing years for the living, or hosting behind Cloudflare Access.

## Gotchas — do not undo these

- **No `setPointerCapture` on the SVG.** Capturing swallows the click before it
  reaches a person group, which made selection intermittently fail. Pan runs on
  window-level pointer listeners with a 5px movement threshold instead.
- **`composedPath()`, not `target.closest()`, in the click-off handler.**
  Selecting from a panel relation link rebuilds `#pRel`, so by the time the
  click bubbles to `document` the clicked `<a>` is detached and `closest()`
  reports it as outside the panel — which cleared the selection the link had
  just made. The event path is fixed at dispatch, so walking it still sees the
  panel.

- **`.person:focus{outline:none}`, and `clearSelection()` blurs.** A clicked
  person group keeps DOM focus, and the browser's own focus ring for it is a
  blue rectangle that outlived the selection — still drawn after the panel
  closed, and stacking up on every pick in comparison mode. The box stroke
  under `:focus-visible` is the focus cue instead, so keyboard users keep one
  and the mouse never draws a ring; the blur in `clearSelection` is what stops
  the ring coming back the moment the window regains focus.

- **JSONP, not `fetch`.** Script tags ignore CORS, so the page works opened from
  `file://` as well as hosted. Switching to `fetch` reintroduces a null-origin
  CORS failure on double-click that looks exactly like a permissions problem.
  The tradeoff is executing whatever Google returns — acceptable for this
  endpoint, but don't reuse the mechanism for other data sources.
- **Google returns a DataTable, not CSV.** `toRows()` flattens it. Numeric cells
  arrive as real numbers and empty cells as `null`; both are handled.

## Testing

There's no test suite. The layout is verified headlessly by extracting the
script, cutting it at `/* ---- render ---- */` (everything above that line is
DOM-free), and requiring it in Node with `demo-family.csv` fed through `parseCSV`.
The context needs `URLSearchParams` and a stub `location.search`, since the
config block reads the sheet id at load. For browser checks, replace the
trailing `load();` with a `buildModel(parseCSV(...))` call on the seed file —
there is no sheet to reach from a test. Worth re-running after any layout change:

```js
// checks: every person placed, no box overlaps within a generation,
// every couple adjacent, names fit their boxes, validator fires on bad refs
```

The in-app validator surfaces parents cells naming someone with no row of
their own, duplicate names, rows with neither a name nor parents, and cells
listing more than two parents. Comment rows are removed before any of that
runs, so they can hold anything at all. It's the highest-value part
of the app — a family-edited sheet's failure mode is a silently vanishing branch.

## Phones

`touch-action:none` on the SVG is what makes panning work, and it also means the
browser hands us every gesture instead of scrolling or zooming the page with it.
So **pinch is implemented here** — without it a phone has no way to zoom at all,
which is exactly how it used to be. `pointers` holds every finger down on the
chart: one is a drag, a second turns it into a pinch anchored on the world point
under the midpoint (so the same two fingers scale and pan at once), and lifting
back to one finger resumes the drag rather than jumping. A pinch spends `moved`,
so the lift at the end can't be read as a tap on whoever is underneath.

Three more things a phone needs:

- **`fit()` floors at k=0.3** on a narrow screen. The whole chart at once is a
  five-pixel smear; a fit that fits everything and shows nothing is worse than
  one that stops at the smallest size worth looking at and lets you pan.
- **`startView()`** opens on the eldest, leftmost person at k=0.8 rather than
  on the fitted whole, because the middle of the top row is usually empty air
  between two couples. A permalink still wins — `openHash()` runs after it.
- **The panel becomes a bottom sheet** at 52vh, and `keepVisible()` lifts a
  newly selected person into the strip above it, but only when they are
  actually covered: a chart that jumps on every tap is worse than one that
  occasionally has to. 52vh is not arbitrary — the strip left over still has to
  hold a full-detail box. `focusOn()` centres in that same strip.

The breakpoint is **860px in two places** — the media query and `NARROW()` —
and they have to stay in step, because the script measures the sheet's height to
keep someone clear of it and would measure a side panel that isn't there.

Resize is not a refit unless the width actually changed: a phone fires resize
every time the address bar slides away, and refitting there throws away wherever
the reader had panned to.

## Visual direction

Cool linen ground (`#EDEEE9`), deep green-black ink, sage connector rules,
Newsreader as the single type family. Deliberately not the cream-and-terracotta
default. Sex fills are muted (`#DAE6F0` / `#F2DEE5`) because saturated blue and
pink fight the background.

One accent does not belong to that palette: `--marine` (`#12557F`) outlines the
selected person and the person being compared. Everything else the selection
draws — the bold run up to the ancestors and siblings, their borders — is pine,
and a pine box sitting inside a pine run at nearly the same weight is
impossible to pick out. Marine is the answer to "which one did I click?", so it
belongs to those two boxes only. Generation bands are tinted but unlabeled.

## The demo family

`demo-family.csv` is invented, and it is shaped to exercise the parts of this
app that a tidy family would not: five generations; two Whitfield siblings
marrying two Lindqvist siblings, which is what produces the cross-links; Maria's
second marriage, so Hank is a half-brother; Paloma raising Mira alone, so a
`parents` cell with a single name; and Ivy and Dean, a childless couple who
exist only as a nameless row. Four generations of surnames change as the women
marry, which is the case that makes name-matching worth testing.

Import it into a sheet to demo the chart, or point a headless run at it (see
Testing) rather than at the real family.

## Open threads

1. `sort_order` column to pin horizontal order and shorten the cross-links.
2. Focus mode — click a person, show two generations up and down.
3. Photos. A `photo` column of URLs or filenames; needs somewhere to host
   them. See the note in Open threads below.
3. Half-sibling cue on the chart itself; currently only visible in the panel.
4. Measured text wrapping instead of character estimation.
5. Decide the privacy posture before circulating the URL.
