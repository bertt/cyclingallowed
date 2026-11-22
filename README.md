# Cycling allowed (OSM, Netherlands)

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
access-tag logic (Dutch defaults):

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
