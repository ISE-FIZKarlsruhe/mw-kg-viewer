# SPARQL Data Navigator (GitHub Pages edition)

A generic, LodView/SHMARQL-style resource browser for a SPARQL endpoint -
ported from the [server-backed version](../browser) to run **entirely in
the browser**, so it can be hosted as plain static files (GitHub Pages,
Netlify, Cloudflare Pages, or literally any web server) with no backend
process to run or maintain.

Configured for the NFDI-MatWerk knowledge graph
(`https://nfdi.fiz-karlsruhe.de/matwerk/sparql`), fixed in `config.json` - no
in-page switcher (see Known limitations). Same features, same UI as the
original; only *how it talks to the SPARQL endpoint* changed.

## The one real tradeoff: CORS

With no backend proxying queries, this browser's JavaScript (`sparql.js`)
talks **directly** to whatever SPARQL endpoint is configured - which only
works if that endpoint sends CORS headers (`Access-Control-Allow-Origin`).
Browsers enforce this; there is no way around it from a purely static site.

The good news: most public SPARQL endpoints meant for querying from the web
already send them - this includes the NFDI-MatWerk SHMARQL/Virtuoso endpoint
this app is configured against, as well as Wikidata, DBpedia, and most other
Virtuoso-backed triplestores (Virtuoso sends them by default). An endpoint
that doesn't will fail with a clear error naming CORS as the likely cause
(see `sparqlHttpFetch` in `sparql.js`) rather than something cryptic - but it
genuinely won't work, and there's no setting here that fixes that.

If you need to support an endpoint that truly has no CORS support, you need
a backend somewhere to proxy it - see the [server-backed version](../browser)
in this repo, which does exactly that.

## Run it locally

```bash
python3 server.py            # defaults to port 8000
```

Open **http://localhost:8000**. This is only for trying changes before
deploying - `server.py` just serves static files, nothing more (no API, no
proxying). Any other static file server works identically
(`python3 -m http.server`, `npx serve`, etc.).

## Deploy to GitHub Pages

1. Push this `rdf-browser/` folder's contents to a GitHub repo (as the repo
   root, or push the whole monorepo and set Pages' source folder to
   `rdf-browser/`).
2. Repo Settings -> Pages -> set the source to that branch/folder.
3. GitHub gives you a URL like `https://<you>.github.io/<repo>/`.

That's it - no build step, no server to deploy or keep running. Any other
static host (Netlify, Cloudflare Pages, S3 + CloudFront, ...) works the same
way: upload these files, done.

Because GitHub Pages *project* sites serve from a subpath
(`you.github.io/repo/`, not the domain root), every asset reference in this
app is a **relative** path (`style.css`, not `/style.css`) - keep it that
way if you edit `index.html`, or it'll 404 under a subpath while still
working fine locally at the root (where the distinction doesn't show up).

## Configuration

Identical to the server-backed version: everything deployment-specific -
endpoint, title, which graphs/types to hide, label-language preference, an
optional Impressum link - lives in `config.json`, fetched once at startup
(see `loadConfig()` in `sparql.js`). This repo's actual `config.json`:

```json
{
  "title": "NFDI-MatWerk Knowledge Graph",
  "endpoint": "https://nfdi.fiz-karlsruhe.de/matwerk/sparql",
  "namespaces_file": "data/namespaces.matwerk.json",
  "graph_short_prefix": "https://nfdi.fiz-karlsruhe.de/matwerk/",
  "excluded_graphs": ["http://www.openlinksw.com/schemas/virtrdf#", "..."],
  "label_lang_preference": ["en", null, "de"],
  "part_of_predicate": "http://purl.obolibrary.org/obo/BFO_0000050",
  "has_part_predicate": "http://purl.obolibrary.org/obo/BFO_0000051",
  "impressum_url": "https://www.fiz-karlsruhe.de/en/impressum",
  "impressum_label": "Imprint"
}
```

- `namespaces_file` points at the prefix map used to render curies.
  `data/namespaces.json` (the code's built-in default) only has common
  RDF/OWL/Dublin Core prefixes; `data/namespaces.matwerk.json` adds this
  dataset's own vocabulary (BFO/OBO, NFDIcore, MWO, PMD) - swap in your own
  file for a different dataset, rather than editing either in place.
- `graph_short_prefix` / `excluded_graphs` only make sense for a triplestore
  that groups its real data under one namespace and ships its own system
  graphs (true of this Virtuoso deployment) - leave `excluded_graphs` at its
  default and `graph_short_prefix: ""` for an endpoint without that shape.
- `excluded_types` - `rdf:type` values never shown on a resource page
  (defaults to `owl:Thing`/`owl:NamedIndividual` - universally noisy on any
  OWL-reasoned endpoint).
- `part_of_predicate` - the IRI of a "part of" relation (a part points to its
  whole), used to show a breadcrumb trail above a resource's title. Empty
  (the default) disables the feature - there's no universal RDF/OWL "part
  of" to default to. This deployment uses BFO's `part of`
  (`obo:BFO_0000050`), which this dataset's curriculum items are structured
  with.
- `has_part_predicate` - the inverse relation (BFO's `has part`,
  `obo:BFO_0000051` here). Not used for the breadcrumb walk itself; when
  both this and `part_of_predicate` are set, a resource's Properties list
  always sorts them immediately next to each other, rather than wherever
  they'd otherwise land alphabetically among everything else - they're
  inverses, so seeing them together makes a resource's part-whole
  relationships easier to read at a glance. Both are configurable, not
  hardcoded, since which vocabulary term a dataset actually uses for these
  can change.
- `impressum_url` left empty hides the Impressum link in the footer
  entirely.
- `endpoint` is the only SPARQL endpoint this deployment ever queries -
  there's no in-page switcher (a deliberate choice; see Known limitations).
  Fork this folder and edit `config.json` to point at a different one.

There are no env var overrides here (there's no process to set them on) -
for a different deployment, fork this folder and edit `config.json`
directly, the same way you'd edit any other static asset.

## Features

Mostly identical to the server-backed version (same UI, same data, just
fetched client-side now) - with one addition specific to this fork:

- **"Part of" breadcrumb**: when `part_of_predicate` is configured, a
  resource page shows a breadcrumb trail above its title - the chain of
  ancestors (root first) reached by walking that predicate upward one hop at
  a time, e.g. `Mathematik 1/2 › Raum und Form › Geometrische Abbildungen
  erkennen und darstellen` leading into the current item's own title. Hidden
  entirely for a resource with no such ancestors. A resource can have more
  than one "part of" parent in this data (it isn't strictly a tree - the
  same item can sit under more than one curriculum variant); since a
  breadcrumb is inherently one path, one parent is picked deterministically
  at each level rather than branching (see `fetchPartOfChain` in
  `sparql.js`, including how it avoids "skipping" a real intermediate level
  when the endpoint has materialized the transitive closure of the relation
  as direct triples). Separately, when `has_part_predicate` is also
  configured, "has part" and "part of" always sort as neighbors in a
  resource's Properties list (see Configuration below).
- **Property values sort alphabetically**: within any one predicate's list
  (both "Properties" and "Referenced by"), values are ordered by whatever
  text is actually shown for them (label, falling back to curie or the bare
  IRI; a literal's own text) - so e.g. a `has part` list of curriculum
  sub-items reads alphabetically rather than in whatever order the endpoint
  happened to return them, without needing a separate section for it.
- **"Load more" instead of a hard wall**: outgoing/incoming triples are
  still capped per request (1500 rows, raised in graph-scoped queries since
  one triple can appear once per graph - see Known limitations), but hitting
  it now offers a "Load more" button that re-fetches the same resource with
  the cap raised, rather than just a dead-end note. Resets back to the
  default on the next navigation.
- **`/` focuses search** from anywhere on the page (skipped while already
  typing in a field), the same convention as GitHub/Slack.
- **Labels are cached for the page's lifetime**: the same predicates,
  classes, and frequently-referenced resources turn up on page after page:
  once a label (or a confirmed absence of one) is resolved, it's never
  re-queried for the rest of the session.
- **Search-as-you-type**: the search box queries `rdfs:label`/`skos:prefLabel`
  live and shows a dropdown of matches (label + curie); arrow keys + Enter to
  pick one, or paste a full IRI and press Enter to jump straight to it.
- **Shareable/bookmarkable URLs**: every resource view is addressable as
  `?iri=<url-encoded-IRI>` - copy the URL bar, or use the "Copy link" button
  on a resource page. Browser back/forward works as expected.
- **Full resource detail view**: for any IRI, shows its `rdf:type`s as a
  stacked, indented list when some of them are related by `rdfs:subClassOf`
  - most general first, each more specific type indented a little further
  than its ancestor, with the most specific one (last in the list) in bold.
  Also shows every outgoing property grouped by predicate ("Properties"),
  and everything that points *to* it grouped by predicate ("Referenced by")
  - both directions, not just outgoing triples like most LOD browsers.
- **Labels everywhere**: classes, properties, and instance values are all
  rendered with their `rdfs:label`/`skos:prefLabel` when one exists (falling
  back to a prefixed curie, then the bare IRI) - resolved in batched queries,
  not one request per value, with configurable language preference.
- **Every resource reference is a link, including its curie.**
- **Named-graph provenance and filtering**: a "Graph" selector in the header
  shows exactly which graph(s) assert each value, and can filter the whole
  view down to just one or more graphs - part of the shareable URL too.
- **Smart blank node handling**: expanded one level inline instead of being
  a dead end (blank nodes still can't be navigated to directly - see Known
  limitations).
- **Manchester syntax for complex `rdf:type`/`rdfs:subClassOf` expressions**:
  an `owl:Restriction` or boolean combinator (`intersectionOf`/`unionOf`/
  `complementOf`/`oneOf`) is rendered as proper Manchester syntax instead of
  a flat property dump, via a bounded-depth closure fetch reconstructed and
  interpreted recursively in `sparql.js`.
- **"Open in SPARQL editor"** button on every resource page.
- **Light, flat, color-coded theme** with full light/dark mode support.
- **Helpful empty state when a graph filter hides everything.**

See the [server-backed version's README](../browser/README.md) for the full
prose on each of these - the behavior is identical, only ported.

## Known limitations

Everything in the [server-backed version's "Known limitations"](../browser/README.md#known-limitations)
applies equally here (type-ordering only sees direct `subClassOf` edges,
blank nodes aren't independently navigable, etc.) **except**:

- The 1500-row cap is no longer a hard wall in this fork - a "Load more"
  button raises it for that one view (see Features above). The row count
  itself, and why it's higher than a plain deduplicated query, are unchanged.
- The two that were specific to having a backend and a user-facing endpoint
  switcher - neither applies here:
- ~~Endpoint switcher has no allowlist~~ - moot: there's no server here to
  make requests on your behalf, and no switcher either. This deployment
  queries exactly one endpoint (`config.json`'s `endpoint`), fixed at deploy
  time - fork the folder and edit the config for a different one. If you
  want visitors to be able to point the browser at an arbitrary endpoint at
  runtime, see the [server-backed version](../browser), which has that
  feature (and needs a backend for it, for the CORS reasons above).
- ~~Querying always goes through this server~~ - replaced by the CORS
  constraint described above, which is the real tradeoff of removing that
  server.

## Architecture

Fully static: `index.html` + `style.css` + two JS modules, no build step, no
framework.

- **`sparql.js`** - the whole "backend," ported from `server.py`: the SPARQL
  1.1 Protocol client (POST, falling back to GET on HTTP 405 - some
  endpoints, like SHMARQL, only implement GET), config/namespace loading,
  label resolution, blank-node/Manchester-syntax rendering, the type-hierarchy
  sort, and the top-level `fetchResource`/`suggest`/`listGraphs`/
  `graphExamples` queries. Deliberately close to `server.py`'s structure and
  function names so the two stay easy to compare; verified to produce
  byte-for-byte-identical results against real endpoint data during
  development.
- **`app.js`** - the UI layer: rendering, search, the graph filter, URL
  routing. Unchanged from the server-backed version except for the four call
  sites that used to `fetch("/api/...")`, which now call `sparql.js`
  functions directly.
- **`server.py`** - local development only (serves static files, nothing
  else); not used in production - any static host serves this app just as
  well.

### Tests

```bash
node --test tests/
```

`tests/sparql.test.mjs` - offline unit/regression tests for `sparql.js`'s
internal logic (exposed only to tests via `sparql.js`'s `_internal` export,
not part of the app's real API surface). No live endpoint needed; `fetch` is
replaced with a small in-memory mock per test. Covers the parts of this file
that are easy to get subtly wrong and have actually gone wrong once already
during development:

- Manchester-syntax rendering (`renderClassExpr`/`flattenRdfList`) - every
  restriction/combinator shape, plus the "no data" vs. "depth exceeded" vs.
  "incomplete restriction" distinctions.
- The "part of" breadcrumb's transitive-shortcut handling
  (`fetchPartOfChain`) - reproduces the exact failure mode found during
  development (an endpoint that materializes the transitive closure as
  direct triples, so a single hop can return a distant ancestor alongside
  the true immediate parent) and asserts it's filtered out correctly. This
  test is deliberately written so the *wrong* answer would sort
  alphabetically first - it was run against a temporarily-reverted version
  of the fix to confirm it actually fails when the logic regresses, not
  just when the code throws.
- Type-hierarchy ordering, predicate grouping/graph-merging/alphabetical
  value ordering, and the label
  cache (including that a confirmed "no label" is cached too, not just hits).
