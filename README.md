# Cycling allowed (OSM)

**Demo:** <https://bertt.github.io/cyclingallowed/>

Created by Bert Temme.

Notes on the QGIS Cloud map
<https://qgiscloud.com/PeeWee32/OSM_Fiets_toegankelijk_WMS/> ("OSM Fiets toegankelijk").

## Which WFS has the cycling accessibility?

The OWS endpoint of the project is:

```
https://wms.qgiscloud.com/PeeWee32/OSM_Fiets_toegankelijk_WMS/
```

WFS feature types (LineString, one OSM way each):

| Feature type | Content |
|---|---|
| **`fietstoegankelijkheid`** | Full classification of OSM ways in the attribute `toegankelijk` |
| `fietstoegankelijkheid_simpel` | Same schema, simplified (presumably fewer classes; style only differs in the map) |
| `path_track` | `highway=path/track` ways |

Attributes: `way_id`, `highway`, `tags` (JSON with the raw OSM tags), `name`, `bicycle_access`,
`toegankelijk`, plus HTML link fields (`osm_way`, `osm_note`, `osm_location`, `pewu`, `gsv`,
`mapillary`, `josm_remote`).

Examples:

```
# capabilities
https://wms.qgiscloud.com/PeeWee32/OSM_Fiets_toegankelijk_WMS/?SERVICE=WFS&REQUEST=GetCapabilities
# features as GeoJSON
https://wms.qgiscloud.com/PeeWee32/OSM_Fiets_toegankelijk_WMS/?SERVICE=WFS&VERSION=1.1.0&REQUEST=GetFeature&TYPENAME=fietstoegankelijkheid&MAXFEATURES=100&OUTPUTFORMAT=GeoJSON
```

## Classes of `toegankelijk`

Counts from a 30,000-feature sample (not representative of all of NL, but indicative):

| Value | Meaning | Count |
|---|---|---|
| `ja impliciet` | Allowed by default road type | 13260 |
| `discutabel` | Questionable / unclear | 7594 |
| `nee impliciet` | Not allowed by default road type | 3450 |
| `nee expliciet` | Forbidden by explicit tags | 3273 |
| `ja expliciet` | Allowed by explicit tags | 1830 |
| `ja heen impliciet` / `ja heen expliciet` | Allowed in one direction (oneway) | 272 / 170 |
| `voorwaardelijk impliciet/expliciet` | Conditional (customers, destination) | 111 / 28 |
| `ja afstappen expliciet` | Allowed, but dismount | 12 |

"impliciet" = derived only from the `highway` value; "expliciet" = decided by an access tag.

## Inferred OSM rules (reverse-engineered from the data, not from the source)

The underlying SQL/QGIS expression is not published through the service, so these rules are
inferred from the tag/class combinations in the sample. They are the standard OSM
access-tag logic (the original map covers the Netherlands; the web viewer works worldwide):

1. **Explicit tags win.** Checked first, on `bicycle`, then `vehicle`, then `access`
   (and `foot`/`motor_vehicle` less clearly):
   - `bicycle=no`, `bicycle=use_sidepath` (also `foot=use_sidepath`), `vehicle=no`,
     `access=no`, `access=private` → **nee expliciet**
   - `bicycle=yes` / `designated` (e.g. on `footway`, `pedestrian`, `path`) → **ja expliciet**
   - `bicycle=dismount` → **ja afstappen expliciet**
   - `access=customers|destination|delivery` etc. → **voorwaardelijk**
2. **`highway=cycleway`** → **ja expliciet** (a cycleway is explicitly for bicycles).
3. **Oneway handling:** `oneway=yes` without `oneway:bicycle=no` → **ja heen …** (one direction only).
   `oneway=yes` + `oneway:bicycle=no` (contraflow cycling allowed) → **ja expliciet**
   (this explains many `residential` ways classed `ja expliciet`).
4. **Implicit by `highway` value** when no relevant access tag is present:
   - Yes: `residential`, `unclassified`, `tertiary`, `secondary`, `living_street`
     (and presumably `primary` unless tagged) → **ja impliciet**
   - No: `footway`, `steps`, `pedestrian`, `platform`, `motorway(_link)`, `trunk(_link)`
     → **nee impliciet**
   - Questionable: `path`, `track`, `service` without any bicycle/access tag
     → **discutabel** (they are not default-allowed for bicycles, but often passable;
     `foot=yes` or `access=permissive` on them does not change this)
5. `cycleway=*` tags on the road itself (lane etc.) do not change the class; they just
   confirm `ja impliciet`.

### Evidence (class vs. tags in sample)

- `discutabel`: `service` 4316, `path` 1521+79, `track` 1472 — almost always with no bicycle tag.
- `nee expliciet`: `service`/`track`/`path` with `access=private` (683/260/48),
  `tertiary`/`secondary`/`unclassified` with `bicycle=use_sidepath` (310/154/131),
  `primary` with `bicycle=no`.
- `ja expliciet`: `cycleway` (757+), `footway`/`pedestrian`/`path` with `bicycle=yes`,
  `residential` with `oneway:bicycle=no`.
- `ja impliciet`: `residential` 8309, `unclassified` 3390, `tertiary` 633, `living_street` 467.
- `nee impliciet`: `footway` 2614, `steps` 184, `pedestrian` 151, motorway/trunk.
- `voorwaardelijk impliciet`: `service` with `access=customers|destination`.

### Caveats

- Rules are inferred; exact precedence (e.g. `bicycle` vs `access`) and the full list of
  `highway` values are unconfirmed. Ask the author (PeeWee32) or inspect the QGIS project
  for the exact expression.
- `bicycle_access` is empty in sampled rows; its semantic is unknown.
- Sample was the first 30,000 features returned by the server, not a random sample.

## Evaluation: gaps in the classification

1. **Precedence of tags** is not documented; `bicycle=*` should override `vehicle=*` and `access=*`
   (otherwise `access=private` + `bicycle=yes` wrongly becomes "no").
2. **`primary` / `trunk`:** in NL `primary` is usually cyclable but often with a mandatory separate
   cycle path (`use_sidepath`); some `trunk` ways are accessible. A class "yes, but use cycle path"
   is missing.
3. **`discutabel` is too coarse** (7594 of 30000 features):
   - `service=driveway/parking_aisle` is mostly "no"/conditional; `service=alley` mostly "yes".
   - `track` with `tracktype=grade1/2` or `surface=asphalt` is cyclable; `grade4/5`, sand or grass is not.
   - `path` + paved surface + `foot=yes` and no `bicycle=no` is probably "yes".
4. **Apparently unused tags:** `bicycle=dismount|permissive|destination`, `bicycle:conditional`,
   `access:conditional`.
5. **Direction:** oneway is handled only globally. Missing: `cycleway=opposite*`,
   `oneway:bicycle=yes` on two-way roads, `bicycle:forward` / `bicycle:backward`, and a separate
   class for contraflow ("yes, both directions").
6. **Crossings:** `highway=footway` + `footway=crossing` + `bicycle=yes` is not treated specially.
7. **Unseen `highway` values:** `bridleway`, `corridor`, `proposed`, `construction`, `bus_guideway`.
8. **`bicycle_access`** is empty in the sample, so it is unused.
9. **Sidepaths:** `sidewalk:*:bicycle=yes` and `cycleway:*=track` are not recognised as cycling facilities.

## Proposed improvements

- Split the result into separate fields: `access` (`yes`, `no`, `conditional`, `dismount`, `unknown`),
  `source` (`explicit`, `implicit`) and `direction` (`both`, `forward`, `backward`).
- Document a fixed precedence, e.g. `bicycle:direction` → `bicycle` → `vehicle` → `access` → `highway` default.
- Refine `discutabel` using `surface`, `tracktype`, `smoothness`, `foot=yes|designated`
  into "probably yes" and "probably no".
- Add a quality layer: ways without `surface`, or with conflicting tags (e.g. `bicycle=no` + `bicycle=designated`).
- Validate against a reference, e.g. compare with BRouter or OSRM bicycle routing.
- Reimplement the rules in a script on the full dataset and measure deviations from the original.

## Styling (from the WMS SLD)

Retrieved with `GetStyles` (`SERVICE=WMS&VERSION=1.3.0&REQUEST=GetStyles&LAYERS=fietstoegankelijkheid`,
likewise for `fietstoegankelijkheid_simpel`). All rules filter on `toegankelijk = '<class>'`
and are visible up to scale 1:100,001. A ready-made Mapbox/MapLibre layer definition is in
[`mapbox-style.json`](mapbox-style.json) (needs a sprite image `arrow`, preferably SDF).

| Symbol | Name (`toegankelijk`) | Class definition (inferred) | Color full / simple | Width |
|---|---|---|---|---|
| ![](legend/ja_expliciet.svg) | `ja expliciet` | `highway=cycleway`, `bicycle=yes/designated`, or `oneway:bicycle=no` contraflow | `#238823` / `#238823` | 6 |
| ![](legend/ja_impliciet.svg) | `ja impliciet` | `residential`, `unclassified`, `tertiary`, `secondary`, `living_street`, no restricting tags | `#82d941` / `#238823` | 6 |
| ![](legend/ja_heen_expliciet.svg) | `ja heen expliciet` | Explicitly cyclable, oneway: only in way direction | `#238823` / `#238823` | 4 + arrow |
| ![](legend/ja_heen_impliciet.svg) | `ja heen impliciet` | Implicitly cyclable, `oneway=yes` (no `oneway:bicycle=no`) | `#82d941` / `#238823` | 4 + arrow |
| ![](legend/ja_terug_expliciet.svg) | `ja terug expliciet` | Explicitly cyclable only against way direction (e.g. `oneway=-1`); not in sample | `#238823` / `#238823` | 4 + arrow |
| ![](legend/ja_terug_impliciet.svg) | `ja terug impliciet` | Implicitly cyclable only against way direction; not in sample | `#82d941` / `#238823` | 4 + arrow |
| ![](legend/ja_afstappen_expliciet.svg) | `ja afstappen expliciet` | `bicycle=dismount` | `#1029f2` | 6, dashed `1 2` |
| ![](legend/voorwaardelijk_expliciet.svg) | `voorwaardelijk expliciet` | Conditional via explicit tag (e.g. `bicycle:conditional`) | `#008e00` | 6, dashed `1 2` |
| ![](legend/voorwaardelijk_impliciet.svg) | `voorwaardelijk impliciet` | `access=customers/destination` on default-allowed road | `#82d941` / `#008e00` | 6, dashed `1 2` |
| ![](legend/nee_expliciet.svg) | `nee expliciet` | `bicycle=no/use_sidepath`, `vehicle=no`, `access=no/private` | `#e31a1c` | 6 |
| ![](legend/nee_impliciet.svg) | `nee impliciet` | `footway`, `steps`, `pedestrian`, `platform`, `motorway`, `trunk` | `#ec7e31` / `#e31a1c` | 6 |
| ![](legend/discutabel.svg) | `discutabel` | `path`, `track`, `service` without bicycle/access tag | `#7c8aa5` | 6 |
| ![](legend/onbekend.svg) | `onbekend` | Unknown / unclassifiable (thin line with arrow); not in sample | `#b3a16a` | 2 + arrow |

Unnamed fallback rules (no filter, i.e. any other value): thin black line (2) with arrow, thin teal
`#14a8b5` line (2) with arrow, and an empty rule.

### Direction arrows

- Oneway classes (`ja heen *`, `ja terug *`, `onbekend`, fallback) draw an arrow at the **last point** of the
  way (SLD `placement=lastPoint`), a filled triangle (size 18) in the line color, and a small
  tick at the first point (`firstPoint`, `ja heen`).
- `ja heen *`: arrow rotated 90° → points along the digitised direction.
  `ja terug *`: arrow rotated 270° → points against it.
- In Mapbox/MapLibre use a `symbol` layer with `symbol-placement: line` and `icon-rotate: 0` (heen)
  or `180` (terug). There is no "last point only" placement; repeat arrows with `symbol-spacing`,
  or precompute end points as a separate point source with a bearing property.
- The `simpel` layer uses one green (`#238823`) for all "ja" classes and one red (`#e31a1c`) for all "nee" classes.

## Web viewer (GitHub Pages)

[`index.html`](index.html) is a standalone MapLibre GL JS viewer: the OpenFreeMap **Positron** basemap
(`https://tiles.openfreemap.org/styles/positron`) with the cycling classification drawn on top of the
OpenFreeMap vector tiles (`transportation` layer) using the colors, widths, dashes and direction arrows
from the QGIS Cloud style above. No WFS or server is needed. Enable GitHub Pages (branch `main`, root) to publish it.

**Idea and credits:** based on the QGIS Cloud map
[OSMNL_Highway by PeeWee32](https://qgiscloud.com/PeeWee32/OSMNL_Highway/?l=Info%20%2Ctoegankelijkheid_fiets&bl=mapnik&t=OSMNL_Highway&e=554693%2C6809530%2C558757%2C6811622).
Map data © OpenStreetMap contributors, tiles and style by OpenFreeMap.

**Adaptations compared with the original:**

- The classification is **recomputed client-side** with MapLibre expressions from the few attributes
  OpenFreeMap tiles carry: `class`, `subclass`, `bicycle`, `access`, `oneway`, `brunnel`.
  It is an approximation of the `toegankelijk` attribute, not the original SQL.
- Not available in the tiles, so not handled: `oneway:bicycle=no` (contraflow, shown as "ja heen"
  instead of "ja expliciet"), `vehicle`, `bicycle:conditional`, `access=customers/destination`
  (no "voorwaardelijk" classes), `tracktype`, `cycleway:*`, `bicycle:forward/backward`, `way_id`.
  The `onbekend` class is not used.
- `access=private` and `access=no` cannot be told apart (OpenFreeMap maps both to `no`).
- Direction arrows are repeated along the line (`symbol-placement: line`, dark triangle) instead of a single
  arrow at the last point; `oneway=-1` rotates the arrow 180° ("ja terug").
- Tunnels are hidden; lines are drawn from zoom 10, arrows from zoom 13; widths scale with zoom.
- Layers are inserted below the first label layer of Positron so street names stay readable.
- Clicking a way shows its tile attributes.
- A more accurate result needs custom vector tiles that keep all OSM tags (e.g. Planetiler/tilemaker → PMTiles).

### Rules used in the viewer

Evaluated top to bottom, first match wins. Attributes are those of the OpenFreeMap `transportation` layer
(roughly the OSM tags `highway`, `bicycle`, `access`, `oneway`, `tunnel`). Only ways with
`class` in `motorway, trunk, primary, secondary, tertiary, minor, service, track, path` are drawn
(`minor` = residential/unclassified/living_street/road); tunnels are hidden.

| # | Condition | Result |
|---|---|---|
| 1 | `bicycle` = `no` or `use_sidepath`, or `access` = `no` / `private` | `nee expliciet` |
| 2 | `bicycle` = `dismount` | `ja afstappen expliciet` |
| 3 | `bicycle` = `yes` / `designated` / `permissive`, or `subclass` = `cycleway` | `ja expliciet`; with `oneway=1` → `ja heen expliciet`, `oneway=-1` → `ja terug expliciet` |
| 4 | `class` = `motorway` / `trunk`, or `class` = `path` with `subclass` = `footway`, `pedestrian`, `steps`, `platform`, `corridor` | `nee impliciet` |
| 5 | `class` = `primary`, `secondary`, `tertiary`, `minor` | `ja impliciet`; with `oneway=1` → `ja heen impliciet`, `oneway=-1` → `ja terug impliciet` |
| 6 | everything else (`service`, `track`, `path`/`bridleway` subclass `path`) | `discutabel` |

Notes: rule 1 beats rule 3, so `bicycle=no` + `bicycle=yes` conflicts cannot occur, but `access=private`
beats `bicycle=yes`. Because `oneway:bicycle=no` is not in the tiles, contraflow-allowed oneways show as
"heen". In the sampled tile data `bicycle`/`foot` = `use_sidepath` is common on `secondary`/`tertiary`/`minor`
roads, which therefore become `nee expliciet` (red).

## Better approach: build your own vector tiles

The OpenFreeMap tiles only carry a fixed, small set of attributes, so the viewer can only approximate the
classification. Generating custom vector tiles that keep the needed OSM tags gives an exact and extensible result.

1. **Source:** an OSM extract, e.g. `netherlands-latest.osm.pbf` from Geofabrik (any country or the planet also works).
2. **Classification at tile-build time:** use Planetiler (Java) or Tilemaker (Lua) with a custom profile that applies
   the full tag logic with a fixed precedence: `bicycle:forward/backward` → `bicycle` → `vehicle` → `access` →
   `highway` default. Write per way: `toegankelijk`, `richting` (`both` / `forward` / `backward`) and source fields.
   Tags that become usable: `oneway:bicycle`, `cycleway=opposite*`, `bicycle:conditional`, `access:conditional`,
   `tracktype`, `surface`, `smoothness`, `foot=yes`, `service=*`, `sidewalk:*:bicycle`, `cycleway:*`.
3. **Output:** one `.pmtiles` file with only the road layer (expected: hundreds of MB, too big for GitHub Pages).
   Host it on GitHub Releases, Cloudflare R2 or S3 with HTTP range-request support.
4. **Viewer:** add the file as a `pmtiles://` source in MapLibre (with the `pmtiles` protocol plugin), keep the
   OpenFreeMap Positron basemap. The style expressions in `index.html` shrink to
   `['get','toegankelijk']` and `['get','richting']`.
5. **Maintenance:** a scheduled GitHub Action downloads the extract, builds the tiles and publishes them
   (weekly or monthly).

Benefits: exact classification including contraflow and conditional access, no WFS dependency, fast and scalable
delivery, and the rules live in one documented profile instead of a client-side expression.

