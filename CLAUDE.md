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
Google Sheets URL, into the parameter or the setup form; `sheetId()` reduces
either to the id, and anything outside `[A-Za-z0-9_-]` is stripped before it
reaches the gviz URL.

**The address is trimmed to the id in two places**, because the link is the
thing people forward. The setup form reduces the paste before it navigates, so
submitting a full Sheets URL still produces `?sheet=<id>` rather than a
percent-encoded mess; and a page opened with an untrimmed parameter — a link
made before this existed — rewrites its own address bar to the short form with
`replaceState`, keeping the hash. Neither reloads. A paste that reduces to
nothing gets a hint under the form instead of a navigation to `?sheet=`.

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

2. **Chains** — connected components of the spouse graph, **ordered by row**.
   Nothing else: not birth year, not the shape of the marriages.

   This used to walk the marriage graph from one end, which kept every couple
   side by side but meant the sheet had no say — the marriages decided, and
   two people who had never met could end up standing between a husband and
   a wife. Row order means the sheet always decides, including the case that
   motivated the change: a woman with three husbands listed above her comes
   out `[husband 1][husband 2][husband 3][wife]`, which no walk can produce.

   The price is that a bar between partners who are no longer adjacent has to
   dip under the row to reach across whoever the sheet put between them. That
   machinery already existed for exactly this, and a dip is a fair trade for
   an order somebody can predict and change.

3. **Order is priority, priority is the row, and the top generation goes
   first.** `p.row` is a person's position in the sheet — lower row, higher
   priority, further left — and it is the only thing consulted. Birth years
   decide nothing; `birth` is read only to print dates.

   Generation 1 is ordered by row outright. Generation *g* is ordered by row
   **only where that does not fight generation g-1**: the primary key is
   where your parents stand, and your own row breaks the tie. That "only
   where possible" clause is not a softening, it is what makes the rule
   satisfiable at all — see the note below.

   Two rules outrank the row, and both are what make a family tree readable:
   **partners stay side by side** (a chain is placed as a unit), and
   **children stay under their parents**. So a generation reads as families
   in row order, with people in row order inside each family — not as one
   flat sort of everyone in that generation.

   Measured against that spec, written out as an independent checker:
   **63 of 65 generations across 19 fixtures match exactly.** Both exceptions
   are cousin marriages, and they are exceptions because the rule does not
   say which parents win when a chain hangs from two families at once. The
   code uses whichever branch reached the chain first in `claim`'s walk. If
   that is ever pinned down, the natural completion is *the earlier-listed
   spouse's parents win* — the same lower-row-first principle, one level
   down. It affects 4 chains in the demo, 0 in any chart without cousin
   marriage.

   **Why the row cannot simply win.** Sorting every generation by its own row
   and also centring parents over children is not a hard problem, it is an
   impossible one whenever a sheet lists generations in disagreeing orders.
   Three families A, B, C, listed in that order in gen 1, but the children
   listed C's first:

       x(A) < x(B) < x(C)                                 row order, gen 1
       centre(C kids) < centre(A kids) < centre(B kids)    row order, gen 2

   Centring says each parent sits on their own children's centre; substitute
   and you get `x(C) < x(A) < x(B)`, contradicting the first line. This was
   built and measured: it produced an 890px bus across the chart and dead
   space between two siblings. Reverted.

   How often it bites, over 366 parent pairs: **zero irreconcilable pairs on
   every naturally-authored fixture** — the demo (31 pairs), the 70-person
   chart (72), the 107-person chart (142), and every hand-written shape.
   Conflicts appear only in sheets deliberately written inconsistently, and
   in the random generators, which emit children in arbitrary order and hit
   52% and 65%. And wherever a sheet is self-consistent, this code already
   puts every chain in exact row order. The two approaches only diverge where
   the strict one is infeasible anyway.

   A worthwhile future feature: detect those pairs and name them in the
   validator — "A's children are listed after C's, but A is listed first",
   with the real names in place of the letters — which turns an unfixable
   layout problem into a sheet the family can fix, after which row order is
   exact by construction.

4. **Horizontal (x)** — `claim`, `measure`, `place`, then `settle`.

   `claim` walks down from the root chains building a spanning tree, and
   records as a `crossLink` every child it could not take because another
   branch reached them first. `measure` sizes the tree bottom-up, `place`
   positions it top-down, `settle` repairs what a tree cannot express.

   **Each union is centred on its own children — not the chain on all of
   them.** Those are the same thing only when a chain is one couple with one
   marriage. Give someone a second marriage and there are two drops but still
   one chain centre; make a chain three people and the drop sits between two
   of them, nowhere near the middle. Both used to draw as a long sideways
   reach from a drop to a brood centred on something else. So `measure` gives
   each node a width **plus an `anchor`** — how far into that width the
   chain's own left edge sits — because a brood centred on a drop near one
   end of a long chain hangs out past it, and a plain bounding box from the
   chain's left edge cannot say so.

   A brood is measured to **the children's own boxes**, never to their
   subtrees or their spouses': the bus meets the children, and a child who
   married drags a spouse into the subtree that is no part of what the drop
   aims at.

   **`fitGroups`** handles one person's several marriages, whose broods want
   overlapping space. It slides them apart by the least that separates them —
   pool adjacent violators, exact, one pass. This is the "where possible"
   clause of rule 2, and the residuals it leaves are it working: Maria's two
   broods are wider than the 182px between her drops, so they spread the
   minimum and the difference has to go somewhere. Three marriages give three
   drops 91px apart needing 196px, so the outer two move ±105.

   **`settle`** is one pass upward, once every box has an x. A family tree is
   not a tree: a child who marries into another family is placed once, in
   whichever branch reached them first, so the *other* set of parents is left
   centred on whichever children it happened to keep. Grace and Karl have
   three; Nina and Owen married Whitfields and sit in the Whitfield block, so
   the tree pass centred them over Sten alone, 1,286px from the middle of the
   three. So each generation is re-fitted to where its unions' children
   actually ended up — all of them, wherever they were placed.

   Upward, because that leaves the row below already final: fitting a row
   never moves anything it was measured against, so one pass is the whole of
   it and nothing iterates. A chain moves as a unit, so partners stay side by
   side. A chain with no children asks to stay where it is and is given almost
   no weight — with an equal vote a childless in-law becomes an anchor, and
   the fit splits the difference between it and a neighbour reaching for three
   children of their own. Its gap rule is `SIB_GAP` for every pair, not the
   wider `GROUP_GAP` that `measure` gives a stranger: the separation between
   families is already in the subtree widths, and asking for it twice only
   lets this pass shove apart a row it was meant to leave alone. Its one hard
   constraint is that boxes must not touch.

   **Where two connectors want the same space, one of them gets it.** Two
   drops that cannot both be straight used to take half the error each, which
   is the worse of both worlds: two lines each bent a little read as two
   mistakes, where one straight line and one frank detour reads as a single
   line being routed round something. So the error is concentrated rather than
   shared. `reach` scores a union by how many people its connector touches —
   partners plus children — and the highest score takes the position outright.
   This happens at two levels. `pick` settles it inside one chain, where a
   person's several marriages give one chain several targets: the biggest
   union wins and the rest bend. `straightenRuns` settles it between chains,
   after the fit has pressed a group of neighbours together at minimum
   separation: the run is already rigid, so it slides as a unit until its
   most-connected member sits on its own target, clamped by the free space on
   either side.

   Concentrating is free when the shift can be taken in full — for two
   competitors it is exactly a wash, and for three, zeroing the *middle* one is
   the least total bend there is while zeroing an end is the most, which is why
   `middling` takes the median and never a leftmost. A run clamped by its
   neighbours takes only part of the shift, though, and a partial move can end
   worse than the even split it replaced: everybody bent and still nobody
   straight. So each run is measured both ways and the better kept.

   Two exclusions. A chain with no children has nothing to be straight about
   and never wins. And a chain that straddles two generations — marry someone
   a generation below you and the pair is one rigid unit with a foot in each
   row — is fitted with the higher row while its lower member is a box in the
   row beneath, a row already fitted and treated from here on as fixed. Slide
   such a run and that box moves too, invalidating the child positions the
   targets were measured from; on one test chart the couple below chased its
   own child leftwards and finished further out than it started. Runs
   containing one are left where the fit put them.

   Layout is a single pass end to end — 0-3ms on every fixture, nothing
   iterates, and it runs once, from `render()`, after the sheet loads.

5. **Connectors** — four primitives, all axis-aligned: a marriage bar between
   partners, a drop line from its midpoint, a horizontal sibling bus, and short
   stubs down to each child. Everything a row hangs below itself lives on a
   **ladder of lanes** — `LANE0` (+22) and every `LANE_STEP` (24) below it — and
   `laneFor()` gives a bus the first lane where nothing else is in its way,
   `LANE_GAP` being the clearance two buses need to count as clear of each
   other. Height is assigned by conflict, not by kind: a lone couple's bus sits
   at +22, and a second bus in that row only moves down if it would actually
   run into the first.

   This is what stops one person's several marriages from reading as one
   marriage. Their unions' buses all start life at the same height, overlapping
   in x because they share a partner, and at one height two overlapping buses
   are one line — a child then appears to hang off a bus belonging to somebody
   else's marriage, which is how a chart ends up showing two fathers for the
   same person. A dipped bar takes a lane too, and its children take one below
   it (`firstLane`); a bus with a far-flung child is simply wide, so it
   conflicts with everything and lands near the bottom of the ladder on its
   own.

   **Partners on different rows always dip, and the dip hangs below the lower
   of the two.** An aunt who married a man a generation below her is a real
   thing, and `between()` only ever looked at one partner's own row, so such a
   couple was drawn as though side by side: a flat bar at the upper row's
   height, and a stub down to their own child running straight through the
   lower partner's box. `rowY` is the *lower* row, so everything the union
   hangs below itself clears both of them, and the dip carries a top per leg
   (`t1`/`t2`) — each starting at that partner's own box bottom, the same
   height when they share a row and a longer reach down for whichever sits
   higher. `paintLineage` uses the same per-leg top, so the bold overlay
   follows the line underneath it.

   **Verticals are drawn first and horizontals last**, so every horizontal can
   arc over every vertical it crosses — not just the long lane runs, which is
   all that used to hop. A dipped bar's legs crossing a sibling bus, a lane
   bus's drop crossing an ordinary one: unmarked, each of those reads as a
   junction, and a junction is how somebody ends up in the wrong family. A run
   never hops its own drop or stubs, because `hops()` only arcs verticals whose
   span straddles the line, and those touch it at an endpoint.

   `GEN_BASE-BOX_BASE = 154` is the gap under a row, which fits `LANE_MAX`+1
   lanes and still leaves the lowest one a stub's worth of room; past that a bus
   shares the last lane rather than colliding with the next generation. The
   first lane used to sit at +5 and read as an underline on the boxes rather
   than as a connector — the spacing is what makes them legible as separate
   things. **An only child within `snapReach` of the drop
   gets one straight line instead of all four.** Their x comes from packing the
   whole sibling group — which can include the children of a partner's other
   union — so it can land a handful of pixels off the parent's own centre, and
   the bus then degenerates into a stair-step that reads as a rendering bug
   rather than a junction. Nobody can see a line leaving a 200px box a few
   pixels off centre; everybody can see the kink. `snapReach` is `SNAP` (38, a
   quarter box) for a single parent, whose "bar" is the whole box bottom, but
   only half the gap between the boxes for a couple, because their drop has to
   stay on the marriage bar — snap further and the line appears to come out of
   one partner alone, which says something false about who the parents are.

   The same rounding shows up with several children: a couple's children are
   centred as a group, so one of them regularly lands two or three pixels off
   the drop, and the bus between them becomes a step in what looks like one
   line. `STUB_SNAP` (8) puts such a stub on the drop instead. Both fixes are
   the same trade: a few pixels off the centre of a 200px box cannot be seen,
   and a three-pixel dogleg in a connector can — people read it as a bug,
   because it is one. Neither moves a box; they only move where a line is
   drawn.

   **A row is one-dimensional, so a third marriage cannot sit next to the
   person.** Only two partners can flank someone; any others end up separated by
   whoever the chain put in between, and their bar used to run at bar height
   straight through that person's box — with the children's drop, taken from
   the midpoint of the span, going through it too. `between()` detects it and
   the bar **dips under the row**: down from each partner's box bottom
   (`DIP_IN` inside the inner edge), along a lane it asks `laneFor()` for like
   any other run, hopping any stub it passes, and back up into the other
   partner. This union's children then hang from a bus one lane below that
   (`firstLane`), so the bar and its own children never share a height. The dip
   carries a top per leg (`t1`/`t2`), each starting at that partner's own box
   bottom — the same height when they share a row, a longer reach down when
   they don't.

   Under, not over. The first version went over the top and was wrong: every
   line above a row arrives from the parents, so a horizontal up there reads as
   descent — it looked like the couple had a parent in common. Below the row is
   where connections between people in the same row belong. Dips are drawn in
   the second pass, with the cross-links, because that is when every vertical
   they might have to hop is known.

   Cross-links (below) are the fifth primitive and the only one that isn't
   local. This is why the layout is hand-rolled rather than
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
`buildModel`, and only then does `setMetrics` add `PHOTO_BAND` to `BOX_H` and
`GEN_H`. An empty photo column renders exactly the chart that existed before
the column did.

People without a photo get their initials in a circle, so a row with one
photo in it doesn't look broken, and the initials are sized off `PHOTO_R`
rather than pinned to a number of their own. Both are drawn through a single
`clipPathUnits="objectBoundingBox"` circle, which crops any image to a circle
regardless of its size or position — one clip path for the whole chart.

**The margin is the constant and the radius is the leftover**, not the other
way round: `PHOTO_PAD` (32) is the inset on all three sides and
`PHOTO_R = BOX_W/2 - PHOTO_PAD` is what the box has left. Sizing the face first
and hoping the margins agreed is how it once ended up with 32px at the sides
and 6px at the top — even-looking in isolation, and obviously wrong the moment
anybody looked at the whole card. Text keeps a much tighter `SIDE_PAD` (8): a
face wants air around it, a sentence wants the width.

Drive photos of living relatives in a link-shared folder are world-readable
to anyone with the URL, same as the birth years. Still unresolved.

## One box, one size

There used to be two: a full box, and a shorter one with no description and a
smaller face that the zoom switched to below k=0.62, with a hysteresis band up
to 0.72 so it wouldn't flap while the wheel turned. **It is gone**, and the
reasoning is worth keeping because the idea will come back.

The premise was right — ten lines of description are an unreadable smear from
far away — but the smear costs nothing, and the switch cost plenty. It
re-rendered the whole chart mid-gesture, it changed every height under whatever
was being pinched, framed or centred (so `fit`, `fitTo` and the pinch handler
each carried a second pass or a re-anchor to survive it), and it made the box
you were looking at not the box you had zoomed towards. Removing it took out
`DETAIL`, `setMode`, `wantMode`, `maybeSwitch`, `ZOOM_OUT`/`ZOOM_IN`, the
`.compact` CSS, and a two-pass loop in three functions; zooming now never
re-renders at all.

What is left is `setMetrics()`, which computes `BOX_H`, `GEN_H` and `DROP` from
`BOX_BASE`/`GEN_BASE` plus the photo band. Two things vary, and both are
settled once, when the rows arrive: whether the sheet has photos, and how tall
the boxes have to be.

**`BOX_BASE` is measured from the family, not fixed.** It was 176px for years —
room for `DESC_LINES` (10) whether or not a single person had written ten, so a
family whose notes run to a line each paid ~110px of empty box apiece and the
chart read as a grid of tall blank cards. `boxRows(p, lines)` now walks one
person's card and returns both the baselines and the height it needs; the
tallest answer over everyone becomes `BOX_BASE`. Ten lines still get ten if
somebody writes them.

`boxRows` also writes the rhythm as **gaps between the pieces** rather than
offsets from the top (`NAME_Y`, then `YEAR_GAP`, `RULE_GAP`, `DESC_GAP`,
`DESC_STEP`, `FOOT`). With offsets, every piece knew where the years line sat
whether or not the person had one, so a card with no dates carried the empty
slot anyway — a name, a hole, then the description, which reads as a gap
somebody forgot rather than as a date nobody knows. Gaps let what is missing
take up no room. `NAME_Y` stays fixed so names still line up across a row;
only what hangs below it moves.

If a low-zoom view ever does need simplifying, do it in CSS off a class on the
`<svg>` — fading `.desc` out costs no re-render and moves nothing, which is the
part that made the old system annoying.

Re-rendering (a reload, a resize, a fresh sheet) throws away the person groups, and with them every class that
carries state — selection, dimming, the comparison target, search hits. That
is why `render()` ends in `repaint()`, which puts all of them back. Anything
new that lives in a class on `.person` has to be added there or it will
vanish the first time the chart is rebuilt. The same goes for anything keyed to
coordinates: `linkGeom` and the lit-lineage layer are rebuilt by every render,
and `repaint()` redraws the bold run from them.

## Text that measures itself

SVG doesn't wrap, so `wrap()` has to decide the line breaks. It used to do it
by counting characters against a `DESC_CHARS` constant, and a count is not a
width. The constant had to be re-tuned by hand every time the type or the box
changed — six times in one afternoon of design tweaks — and it had to be
pessimistic enough for the widest line it might ever meet, so every ordinary
line stopped well short of the room it was allowed. The nominal margin was
14px and the margin you actually saw was nearer 30.

It measures now. `charWidth()` keeps a `Map` of per-character advances taken
off one hidden `<text class="desc">`, so a hundred-person family costs about
sixty measurements rather than one per line, and `textWidth()` sums them.
Chinese, which no character count can approximate, comes out right for free.
The ruler lives in its own off-screen `<svg>` on the body rather than in
`#stage`, because `render()` empties `#stage`, and `getComputedTextLength()`
on a detached node returns 0 — hence the `isConnected` check in `rulerNode()`.

Two consequences worth knowing:

- **The ellipsis has a width.** Truncation used to append `…` *after* the fit
  check, so the one truncated line was reliably the widest thing in the box —
  194px in 184px of room. It now drops characters until the line plus the
  ellipsis fits.
- **Layout depends on the font, so the font has to be in first.** Box heights
  come from measured text, so a first layout against the fallback would size
  every box wrong. `load()` awaits `fontsIn`, which races
  `document.fonts.ready` against a 1.5s timeout — a font host that never
  answers should cost a moment, not the chart.

Names get the same treatment from the other end. A name is one line and cannot
wrap without costing every box in the chart a line, so `fitNames()` measures
each one and shrinks the ones that overrun to a `NAME_MIN` floor. English name
plus the Chinese one in brackets runs the full width at any size worth reading,
and shrinking that one name is quieter than truncating it — people search for
the half that truncation would hide. Every width is read before any font-size
is written, so the browser lays out once instead of once per person: 0.4ms
across 107 people.

## The selected lineage

Selecting someone dims everyone off their line — `lineage` is **every ancestor,
the row they stand in** (siblings and half-siblings, whose partners and children
stay dim) **and one step down**: partners and children. Siblings hang off a
parent union the up-walk passes straight through, so they have to be added
explicitly. The walk down used to be recursive, and from anyone old enough to
have grandchildren it lit half the chart — a set that large says nothing about
the person you clicked. Up stays unlimited, because a line of ancestors is
finite and is the thing people came to see. On top
of that, **everyone the selection keeps is drawn in bold** — pine connectors to
every ancestor, sibling, partner and child, and a heavier pine
border on those boxes (`.kin`, pine 1.75 against the selection's azure 4, so
the person you clicked still leads). Bolding and dimming are the same set by
construction: both come from `lineage()`, so a box can never be bright with a
thin line into it, which is what the first version did — it bolded ancestors
only, and a highlighted spouse or child hung off a hairline.

`kinRun()` turns that set into edges: for every union with a kept partner, the
line to each kept child is bold, and the **bar** is bold only when *both*
partners are kept. That distinction is the whole trick — a sibling's husband or
a parent's second wife is dimmed, so their marriage bar stays thin while the
line down to the child still lights up.

When only one partner is kept, though, the bar is still bold **from the drop to
that partner's own box** (`halves`) — half of the T, not the whole bar. Without
it the bold run from a half-sibling climbed to the marriage bar and stopped in
mid-air, a box's width short of the parent it was connecting to, and the whole
path read as broken. The same applies when that bar had to dip under the row:
`dipSpan` is kept for exactly this, so the overlay can light the leg on the kept
parent's side and the stretch of lane back to the drop, hops and all. A bold run
should always end on a person or on another bold run, never at a junction.

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

## Search

Typing in the find box marks the hits (`.hit-match`) **and frames them**:
`fitTo()` takes the smallest rectangle holding every match and centres it, so a
name three screens to the left comes to you rather than being highlighted where
you can't see it. One hit lands at k=1; a common surname zooms out until they
all fit.

Two rules keep it from being twitchy: the view moves only when the **set** of
hits changes — typing the rest of a name that already matched one person leaves
it alone — and a query that matches nobody, or an emptied box, doesn't move it
at all. Yanking the view somewhere on the way to a name nobody has is worse
than staying put.

`fitTo()` frames into what the panel leaves of the viewport — width on a
desktop, height under the sheet on a phone.

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
  where they actually sit. Not every one of them deserves a lane, though:
  `inlineCross()` promotes a cross-link back onto the ordinary sibling bus when
  the child sits one generation below its parents — the row the bus already
  serves — and the run from the drop to its stub passes over nobody else's box
  in either row. That covers the common shape where two children of the same
  couple marry each other, or a placeholder ancestor ("China") stands above both
  spouses: the layout can only place their shared chain once, so the second
  child comes back remote, but drawn it is just a sibling on a slightly longer
  bus. The lane is for runs that pass over other families, where a long line at
  bus height reads as somebody else's bus. Currently two of these, one of them 2,232px long.
  Inherent to strict generational banding — the alternative is duplicating
  people, which is worse: one box per person is what makes selection, dimming
  and the relationship finder unambiguous. `sort_order` was once the planned
  mitigation; it is closed, not pending — the row *is* the handle now, and a
  second ordering column would give the sheet two ways to say one thing.

  They used to be dashed, which was the wrong fix for the wrong problem. The
  actual damage was that they ran at `barY + DROP` — exactly the height of
  every sibling bus in that generation — so a long line lay *on top of* two
  short ones and read as one continuous bus, quietly adopting someone into
  the wrong family. They have no lane of their own any more: a union with a
  far-flung child asks `laneFor()` like everything else, and because its bus
  is wide it conflicts with every short one and lands further down the ladder
  by itself. That is the same mechanism that keeps one person's several
  marriages apart, rather than a second set of constants doing the same job.
  With the collinearity gone they are drawn solid and hop
  over the verticals they cross: `hops()` arcs over any vertical whose y-range
  spans the lane. Every real junction in this chart is a T, so a crossing was
  already distinguishable; the hop makes it unambiguous.

  **A far-flung child takes the whole bus down with them.** There used to be
  two lines: the sibling bus at its usual +5 for the children the layout could
  place together, and a separate cross-link running the length of the chart in
  a lane below for the one it couldn't — two long rails, joined at one end,
  each hopping the other's stubs. Siblings drawn on two different lines is
  exactly backwards. Now a union with any remote child puts its **entire** bus
  in the lane and hangs every child off that one straight run, near ones and
  far. Same primitives as any other union — drop, bus, stubs — only lower, and
  the run hops the verticals it passes under, which is the one thing an
  ordinary bus never has to do.

  It does not get a special height any more — it asks `laneFor()` like every
  other bus, and because it is wide it conflicts with everything and lands on a
  lane of its own. The point stands: a long bus at a height another bus already
  occupies lies on top of it, and their children appear to hang off yours.

  These buses are drawn in the second pass, after every vertical is known, so
  they can hop. `lastVerticals` keeps that list for `paintLineage()`, so the
  bold overlay arcs over exactly what the line beneath it arcs over.

  `verticals` accumulates as the local passes draw, and lane buses push their
  own segments as they go, so a second lane bus hops the first. Only the
  horizontal run is hopped — the short descents are left plain. The footer
  key that explained the old dashed line is gone: a hopped line that runs
  half the chart explains itself.
- **Long unbroken words still run wide.** A single word longer than the box
  becomes its own line and overflows it, same as it always did. Everything
  else about the wrap is now measured rather than counted (see below).
- **Photos are fetched at `PHOTO_PX` (640) for a 136px avatar.** One size
  serves the chart and the panel so each face is fetched once, and 640 is what
  the panel wants on a 2x desktop display. The chart pays for it: 60 photos is
  ~94MB of decoded bitmap, 107 is ~167MB, and an old tablet re-samples every
  one of them per frame while you pinch. 384 would cut that by 64%, 320 by 75%,
  and on a phone or tablet the panel photo is 128px anyway — so the large size
  is only ever earning its keep on a desktop.
- **Width.** Generation 3 is ~2,300px, and fit-to-screen is limited by width,
  not height, so a fitted chart leaves empty space above and below and the
  boxes are still small. A focus mode showing two generations either side of a
  selected person would beat panning, and doesn't exist yet.
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

- **`touch-action:none` lives on the `body`, not on `#stage`.** Safari ignores
  `touch-action` on SVG elements, so the declaration sat for a long time on the
  one element in the page that could not carry it, and the chart's own
  background stayed pinchable while everything else behaved. On iOS a pinch out
  past 1:1 hands the tab to the app switcher, and page zoom is a state the
  chart's own zoom cannot undo. Ancestors count toward the effective value, so
  saying it once on the body covers the chart, the panel and the frame;
  `#panelBody` opts back into a vertical drag with `pan-y`, which works because
  the intersection with an ancestor stops at whatever is actually being
  scrolled. Safari runs its own pinch alongside all this, hence the
  `gesturestart`/`gesturechange` guard on the document.

- **The svg's rectangle is cached (`svgRect`), not asked for per move.** It was
  read inside the pinch branch of `pointermove`, once per event.
  `getBoundingClientRect()` is a synchronous style-and-layout flush, and it
  landed immediately after `apply()` had invalidated the whole chart, so the
  browser re-laid-out every box, line and glyph before it could answer: 0.01ms
  for the attribute write, 4.6ms for the write plus the question, on a
  60-person chart. That is most of a frame on a desktop and all of one on an
  older tablet — and it is why zooming dragged while panning did not, since the
  drag path never asked. Nothing in a gesture moves the svg. Invalidated on
  resize, orientationchange, scroll and visualViewport changes; if something
  new can move or resize the chart, it has to call `forgetRect()`.

- **`user-select:none` on `#stage` only.** A pan that wandered into the panel
  selected everything between mousedown and mouseup — 2,029 characters in the
  test. Scoping it to the chart is what leaves the panel's own text selectable,
  which is where anyone would actually want to copy from.

- **`#panelBody.scrollTop = 0` when the panel opens.** The scroll lives on the
  body and survives the panel closing, so without it a long entry read halfway
  down leaves the next person opening mid-sentence. At open time rather than on
  close, which also covers navigating between people by a relation link without
  the panel ever shutting.

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
- **The panel becomes a bottom sheet** at exactly `50dvh`, and `keepVisible()`
  lifts a newly selected person into the strip above it, but only when they are
  actually covered: a chart that jumps on every tap is worse than one that
  occasionally has to. Half is not arbitrary — the strip left over still has to
  hold a full-detail box. `focusOn()` centres in that same strip.
- **The frame gives the chart everything it isn't using.** `100dvh` rather than
  `100%`, so the page grows as the address bar slides away; the header is one
  row on a phone instead of two; the pan/zoom tip is hidden, since a phone
  teaches itself in one gesture, and the footer disappears entirely once that
  leaves nothing in it but an empty bar. Between them that is ~70px of chart on
  a 390x844 screen.
- **The panel scrolls in `#panelBody`, not in the panel.** The close button is
  positioned against the panel, so with the scroll one level in it stays where
  it was put instead of leaving with the content — and it carries the card's
  own background, or names slide through the ×.

### The sheet has three states

Up, parked at the bottom showing only the name, and gone. Reading a card and
then wanting the chart back used to mean closing it and losing the selection
with it; parked, the name follows you and the detail is one swipe away.

`sheetH()` is the single answer to "how much room is the sheet taking" —
`keepVisible`, `fitTo` and `focusOn` all ask it, so a parked sheet costs the
chart 54px instead of half the screen and none of them has to know which state
it is in. `setSheet('closed')` goes through `clearSelection()` rather than just
hiding the sheet, so the chart never keeps a bold lineage for a card nobody can
see.

Two entry points, one state machine (`beginSheet` / `moveSheet` / `endSheet`):
the grabber, and the scroll itself. **Read to the top of the card and keep
pulling and the sheet takes the gesture over**, following the finger down — one
continuous motion from reading to putting away. The takeover has to be claimed
on the first move that means it, because `touchmove` stops being cancelable
once the browser has committed to a scroll; at `scrollTop` 0 pulling down there
is nothing to scroll, which is why `#panelBody` also carries
`overscroll-behavior:contain` — a rubber-band bounce would get there first.

Release decides, not the crossing: past halfway it parks, above it springs
back, and a flick beats both. Deciding live would snap the sheet away
mid-drag and take the gesture out of your hands — "drag down, change your mind,
drag back up" is the thing that has to keep working. A new selection always
comes up in full: parking is how you get the chart back, but picking somebody
is asking to read about them.

One CSS trap: the parked rule has to be `aside.open.peek`, because
`aside.open{transform:none}` sits later in the sheet at equal specificity and
would otherwise win — leaving the class on and the sheet exactly where it was.

### A flick keeps going

`throwChart()` glides the chart on after a lift. The interesting part is not
the decay, it is reading the velocity, and **the last move is a trap**. Two
pointermove events can arrive a fraction of a millisecond apart — a coalesced
pair, a device reporting faster than it paints — and one pixel over a fraction
of a millisecond is an enormous instantaneous velocity.

Smoothing does not save you. The first version averaged instantaneous `dx/dt`
with a time-weighted blend, and a single spike was enough: replaying synthetic
streams through it, a steady 0.3px/ms drag launched at **2.52px/ms** when the
last four samples arrived in one frame, and at **1.95px/ms** off a single 1px
jump 0.2ms before the lift. Six to eight times what the hand did, which is
exactly what it felt like — and only on gentle drags, because a real flick
already has a velocity big enough to hide the spike.

`tailVelocity()` measures distance over elapsed time across the last
`TRACK_MS` (80) of the drag instead. Whatever the samples inside the window
look like, first-to-last over first-to-last is what the finger actually did —
arithmetically incapable of exceeding it. `MAX_FLICK` (2.2px/ms) caps a genuine
hard throw, and a window shorter than 12ms is not a throw at all. A finger that
settles before lifting now reads about 0.06px/ms on its own, below `FLICK_MIN`,
so the old "did the finger rest" special case is gone.

`FRICTION` is per frame at 60Hz and scaled by the real frame time, so the glide
covers the same ground on an old tablet as on a fast desktop. It started at
.94, which is roughly what a scrolling list wants: a hard flick coasted 540px
over a second and a quarter. A list is a rail and overshooting costs nothing,
but a chart is a place — you throw it towards somebody, and a glide still
running after you have arrived has to be caught and dragged back. `.84` is
204px in .4s. Anything that places the chart deliberately calls `stopGlide()`
first, and `prefers-reduced-motion` skips it entirely.

Pinch has no momentum on purpose. A flick is a pan gesture; continuing to scale
after the fingers leave reads as the chart getting away from you.

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

Two accents do not belong to that palette. `--azure` (`#1478D4`) at 4px
outlines the selected person; `--marine` (`#12557F`) at 2.25 outlines the
person being compared. Everything else the selection draws — the bold run up
to the ancestors and siblings, their borders — is pine, and a pine box sitting
inside a pine run at nearly the same weight is impossible to pick out. Azure is
the answer to "which one did I click?", so it belongs to that one box, and it
is brighter and heavier than anything else on the chart on purpose: at the zoom
where a whole family fits on screen a box is a thumbnail and a 2.75px stroke a
hairline, which is what the first version of this was. Generation bands are
tinted but unlabeled.

Connectors are `--wire` (`#6E7669`), a step darker than the `--rule` used for
box strokes, the avatar ring and the card divider. A border is furniture and
can recede; the lines are the argument the chart is making, and at the zoom
where a whole family fits they were the first thing to dissolve into the
ground.

`HOP` is 7. At 4 it was a technically-correct dimple nobody could see, which is
the same as not drawing it — the whole point is to say "these two do not meet".
Reading it costs a little of the run's straightness, which is the trade. Two
crossings closer together than a bump is wide share one, because overlapping
arcs draw a knot, and a knot reads worse than the plain line the hop was there
to clarify.

A card has three weights, not two. The name is `--ink` at 20px, the note under
it is `--note` (`#3F4841`) at 18px, and the years and other furniture are
`--muted` at 14px. `--note` exists because the note is the only thing on a card
anyone reads twice, and it was sharing a colour with the footer and the panel's
definition terms — darkening `--muted` to fix the card would have dragged all
of those with it.

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

1. Focus mode — click a person, show two generations up and down.
2. Half-sibling cue on the chart itself; currently only visible in the panel.
3. Decide the privacy posture before circulating the URL.
4. `SNAP` (38) was set as a quarter of a 152px box and did not move when the
   box went to 200. It is a fifth now. Probably wants to be 50, but it changes
   where connectors snap, so it needs measuring rather than assuming.

`sort_order` is closed rather than pending: row order is the handle, and a
second ordering column would only give the sheet two ways to say one thing.
