# Data format

All of Nubi's campus data is one versioned SQLite file: the **nubi-campus** release. Any SQLite reader works
(the `sqlite3` shell, Python's `sqlite3`, DuckDB's SQLite extension, QGIS, DBeaver). The file uses only standard
SQLite features: `STRICT` tables, declared foreign keys, JSON text columns and an `rtree` virtual table.

This page is the schema manual of **release 1.1** (schema 1.1.0). Every release is listed in
[CHANGELOG.md](CHANGELOG.md). The release is currently read through the map and the assistant; it is not offered as a
download.

Contents: [The release](#the-release) · [Naming rules](#naming-rules) · [Coordinates and levels](#coordinates-and-levels) ·
[Ids](#ids) · [Provenance](#provenance) · [Tables](#tables) · [Main tables, column by column](#main-tables-column-by-column) ·
[Vocabularies](#vocabularies) · [Example queries](#example-queries) · [Versioning](#versioning)

## The release

| | |
|---|---|
| Name, version | `nubi-campus` release `1.1`, schema `1.1.0` |
| File | `nubi-campus.sqlite` (about 14 MB, 4 MB compressed) and `manifest.json` |
| Contents | 42 tables and 2 views: the campus, its buildings and the building register, levels and their uses, entrances, the underground, facilities and their hours, the academic calendar, the map's features, paths, areas, trees, the city around, the walking graph and the audit of the map |
| Determinism | the same input gives the same file byte for byte (rows in a fixed order, no clock); `manifest.json` also holds a SHA-256 per table over the rows' values, independent of the SQLite version |
| Self-describing | the schema is inside the file: `schema_tables`, `schema_columns` (type, unit, meaning in English and Korean, vocabulary, foreign key) and `vocab` (every enumerated value with its Korean label) |
| Built from | `release_info` holds the commit the release was built from and that commit's date |

## Naming rules

1. Table and column names are snake_case English, in full words. Each table has one grain: one row per building
   piece, per level, per graph edge, and so on.
2. A physical quantity ends with its unit: `_m` (metres), `_m2` (square metres), `_deg` (degrees), `_pct`
   (percent), `_krw` (won), `_kcal`, `_dm` (decimetres). Dates end with `_on` (`YYYY-MM-DD`), moments with `_at`
   (ISO 8601, Seoul time).
3. Counts start with `n_`; booleans start with `is_` / `has_` and hold 0 or 1.
4. A column holding several values is JSON text with a plural name (`aliases`, `conflicts`, `source_ids`). Shapes
   and renderer specifications end with `_json` (`outline_json`, `line_json`, `style_json`). Read them with
   `json_each`, `json_extract` and `json_array_length`.
5. Columns marked **derived** (centres, bounding boxes, `lat`/`lon`, areas, lengths, counts, flag booleans,
   `grade_pct`) are computed from the others when the release is built, for convenient SQL.

## Coordinates and levels

- `x_m` (east) and `y_m` (north) are metres in a local frame whose origin is `venue.origin_lat` /
  `venue.origin_lon`, the same frame as the map. `lat` and `lon` are WGS84 degrees.
- Heights `z_m` are metres above the terrain grid's base (`terrain`); the terrain is the Copernicus DEM GLO-30,
  smoothed and adjusted where the map grades the ground.
- Every row with a shape has its bounding box: `x_min_m`, `y_min_m`, `x_max_m`, `y_max_m`.
- Levels are **IMDF ordinals** everywhere: `0` = 1층 (ground floor), `1` = 2층, `-1` = 지하 1층 (B1).
  `levels.short_name` is the usual label: `1F`, `3F`, `B1`, rooftop `R1`.

## Ids

Ids are explicit text, and joins name them (`building_id`, `level_id`, `facility_id`, `unit_id`, `register_pk`).

| Thing | Id | Examples |
|---|---|---|
| building piece | its OpenStreetMap way or relation, or its GIS id; a piece that GIS split off gets `g/i…`; Nubi's own models a name | `w/163525865` (중앙도서관), `r/3383545` (과학원), `g/i5923` (a piece of 과학관), `baekyang` (백양누리) |
| level | `<building_id>/<short name>` | `w/163525865/3F`, `w/163525865/B1`, `baekyang/B2` |
| unit, section | `<building_id>/<model id>` | `baekyang/B1/item0` |
| opening (entrance) | `<building_id>#<n>` | `g/100194711#0` |
| facility | a slug | `accessible-toilet-library` |
| map feature | `<zone>/<id>` | `amphitheater/cr_bus_bays` |
| path, area, tree, city block, graph node and edge | a position within the release (`path:12`) | valid within one release only |

Buildings are split into pieces where the surveyed outlines split them. To count buildings use the view
`named_buildings` (one row per named building: its largest piece, with `n_pieces` and `total_footprint_m2`). The
view `places` lists every named thing with a position across all tables.

## Provenance

- Every row with data has `source_ids`: a JSON list of ids in `sources` (title, kind, attribution, URL, licence,
  the day it was read). Kinds are `dataset` (a public dataset), `page` (a web page) and `model` (a model or value made
  for Nubi).
- `is_est` = 1 marks a value that is Nubi's estimate rather than a source's: an entrance placed where no source gives
  one, a modelled room, a position inferred from a floor plan.
- `facility_sources` says which source gave which fact of a facility (`used_for`: position, hours …).
- Building heights follow a fixed order: the register's height, then the register's storeys times a storey height
  for the building's use, then OpenStreetMap's height or levels, then other published sources, then a default. The
  choice is kept in `buildings.height_source`.

## Tables

The indoor tables follow **IMDF** (the Indoor Mapping Data Format, an OGC community standard): venue → building →
level → unit / opening / amenity / occupant, with sections inside a level and relationships (stairs, escalators,
elevators, ramps) between levels. Tables with 0 rows are defined and waiting for data: lecture rooms will go into
`units` (category `lecture`, `room_number`, `n_seats`), departments and offices into `occupants`.

### Release and schema

| Table | Rows | Key | What |
|---|---|---|---|
| `release_info` | 22 | key | release facts: name, version, commit, dates, bundle constants |
| `sources` | 23 | source_id | sources: each dataset or page, its URL, licence and the day it was read |
| `schema_tables` | 42 | table_name | this schema's tables: meaning, key, IMDF feature type |
| `schema_columns` | 552 | table_name, column_name | this schema: every column's name, type, unit, meaning, vocabulary, foreign key |
| `vocab` | 189 | vocab, value | vocabularies: the values an enumerated column may take |

### Campus and ground

| Table | Rows | Key | What |
|---|---|---|---|
| `venue` | 1 | venue_id | the venue (IMDF venue): the campus, its origin, boundary and start |
| `terrain` | 1 | terrain_id | ground: the terrain grid (decimetres above base) |
| `zones` | 15 | zone_id | zones: the campus' detail zones and their boxes |

### Buildings

| Table | Rows | Key | What |
|---|---|---|---|
| `buildings` | 353 | building_id | buildings (IMDF building): one row per drawn building piece |
| `building_parts` | 607 | part_id | building masses and parts |
| `facades` | 156 | facade_id | facades: the renderer's facade table |
| `registers` | 128 | register_pk | building register title records (표제부) |
| `register_floors` | 827 | register_pk, row_index | building register floor rows (층별개요) |

### Indoor (IMDF)

| Table | Rows | Key | What |
|---|---|---|---|
| `levels` | 670 | level_id | levels (IMDF level): one row per storey of a building |
| `level_uses` | 765 | level_id, ord | uses of a level (the register rows of that floor) |
| `sections` | 8 | section_id | sections of a level (IMDF section) |
| `units` | 23 | unit_id | rooms and spaces (IMDF unit); lecture rooms to come |
| `openings` | 329 | opening_id | entrances (IMDF opening): doors in a building's walls |
| `connections` | 16 | connection_id | links between levels and the surface |
| `occupants` | 0 | occupant_id | occupants: departments, offices, institutes (no data yet) |

### Underground

| Table | Rows | Key | What |
|---|---|---|---|
| `underground_ways` | 92 | way_id | underground ways (OSM) |
| `underground_points` | 46 | point_id | underground points (OSM) |

### Facilities and time

| Table | Rows | Key | What |
|---|---|---|---|
| `facilities` | 191 | facility_id | facilities (IMDF amenity / occupant) |
| `facility_hours` | 104 | facility_id, ord | facility hours, one row per season |
| `facility_sources` | 277 | facility_id, ord | facility sources |
| `calendar_periods` | 5 | period_id | academic calendar: terms and exam periods |
| `holidays` | 10 | day_on | public holidays |
| `menus` | 0 | menu_item_id | cafeteria menus by day, meal and corner (no data yet) |
| `events` | 0 | event_id | events (no data yet) |

### Map features

| Table | Rows | Key | What |
|---|---|---|---|
| `features` | 1,078 | feature_id | map features by zone |
| `paths` | 4,084 | path_id | paths: footways, roads, steps (OSM) |
| `areas` | 486 | area_id | areas: grass, woods, plazas, pitches, water (OSM) |
| `context_areas` | 13 | context_area_id | the context layer: forests and clearings around the campus, outside the zones |
| `trees` | 2,541 | tree_id | trees (OSM) |
| `city_buildings` | 9,959 | city_building_id | city blocks around the campus |

### Walking graph

| Table | Rows | Key | What |
|---|---|---|---|
| `route_nodes` | 14,965 | node_id | walking graph nodes (ids valid within a release) |
| `route_edges` | 16,518 | edge_id | walking graph edges |
| `route_way_names` | 193 | way_name_id | way names of the graph |
| `route_doors` | 747 | building_id, node_id | each building's door nodes |
| `route_landmarks` | 138 | landmark_id | landmarks for directions |

### Checks and index

| Table | Rows | Key | What |
|---|---|---|---|
| `audit` | 539 | zone_id, cell | per-cell audit findings of the zones |
| `spatial_items` | 50,694 | geo_id | items of the R-tree spatial_index |

## Main tables, column by column

The columns of the tables most queries start from. Every other table is described the same way inside the file
(`SELECT * FROM schema_columns WHERE table_name = '…'`).

### `venue`

The venue (IMDF venue): the campus, its origin, boundary and start.

| Column | Type | Meaning |
|---|---|---|
| `venue_id` | text, not null | venue id |
| `name` | text, not null | name |
| `osm_id` | text | OSM element |
| `origin_lat` | real, not null | latitude of the local origin |
| `origin_lon` | real, not null | longitude of the local origin |
| `level_height_m` | real, not null | default storey height |
| `n_default_levels` | integer, not null | default storeys when unknown |
| `start_x_m` | real, not null | walk start x |
| `start_y_m` | real, not null | walk start y |
| `start_heading_deg` | real, not null | start heading |
| `start_name` | text, not null | start name |
| `x_min_m` | real | bounding box west x (derived) |
| `y_min_m` | real | bounding box south y (derived) |
| `x_max_m` | real | bounding box east x (derived) |
| `y_max_m` | real | bounding box north y (derived) |
| `boundary_json` | json, not null | campus boundary ring [x, y, ...] local metres |
| `source_ids` | json, not null | source ids of the row (list, sources.source_id) |

### `buildings`

Buildings (IMDF building): one row per drawn building piece.

| Column | Type | Meaning |
|---|---|---|
| `building_id` | text, not null | building id: OSM w/ r/, GIS g/, or ours |
| `name` | text | name shown on the map |
| `name_en` | text | English name |
| `building_number` | text | building number (derived) |
| `layer` | text, not null | surface building or underground structure (vocabulary `layer`) |
| `split_of` | text | the OSM footprint this GIS piece was split from → `buildings.building_id` |
| `is_from_context` | boolean, not null | a university building added from the surrounding OSM answer, recognised by its register name |
| `is_hidden` | boolean, not null | not drawn (a feature draws it) (derived) |
| `height_m` | real | height (none for an underground structure) |
| `min_height_m` | real | height of the underside above ground |
| `tower_height_m` | real | tower height |
| `storey_height_m` | real | storey height (checked) |
| `n_floors` | integer | storeys used by the map |
| `height_source` | text | where the height comes from (vocabulary `height_source`) |
| `geometry_source` | text, not null | outline source (vocabulary `geometry_source`) |
| `gis_id` | text | GIS building id |
| `material` | text | material class (vocabulary `material`) |
| `roof_shape` | text | roof shape (vocabulary `roof_shape`) |
| `facade_id` | integer | facade (facades) → `facades.facade_id` |
| `facade_sides_json` | json | facade per side [all, north, east, south, west] |
| `top_facade_json` | json | top storey facade [h, facade_id, wall colour] |
| `facade_spans_json` | json | wall spans with their own facade |
| `holes_json` | json | courtyard rings |
| `style_json` | json | look (style; masses and parts are in building_parts) |
| `ref` | text | web reference for height or storeys |
| `note` | text | note |
| `register_pk` | text | building register record → `registers.register_pk` |
| `register_name` | text | name in the register |
| `register_floors_above` | integer | storeys above ground (register) |
| `register_floors_below` | integer | storeys below ground (register) |
| `register_structure` | text | structure (register) |
| `register_roof` | text | roof (register) |
| `approved_year` | integer | year of the use approval (completion) |
| `register_basis` | text | which register value set the height |
| `register_from` | text | building the register facts were lent from |
| `zone_id` | text | zone → `zones.zone_id` |
| `footprint_area_m2` | real | footprint area (derived) |
| `n_entrances` | integer, not null | entrances (derived) |
| `ground_z_m` | real | ground floor height above the terrain base |
| `basement_outlines_json` | json | basement footprint rings |
| `x_m` | real | local x, metres east of the venue origin (derived) |
| `y_m` | real | local y, metres north of the venue origin (derived) |
| `lat` | real | latitude, WGS84 degrees (derived) |
| `lon` | real | longitude, WGS84 degrees (derived) |
| `x_min_m` | real | bounding box west x (derived) |
| `y_min_m` | real | bounding box south y (derived) |
| `x_max_m` | real | bounding box east x (derived) |
| `y_max_m` | real | bounding box north y (derived) |
| `outline_json` | json | footprint ring [x, y, ...] |
| `source_ids` | json, not null | source ids of the row (list, sources.source_id) |

### `levels`

Levels (IMDF level): one row per storey of a building.

| Column | Type | Meaning |
|---|---|---|
| `level_id` | text, not null | level id |
| `building_id` | text, not null | building → `buildings.building_id` |
| `ordinal` | integer, not null | ordinal (IMDF: 0 = ground floor) |
| `short_name` | text, not null | short name |
| `name` | text, not null | name |
| `is_underground` | boolean, not null | below ground (derived) |
| `z_m` | real | floor height above the terrain base (estimate) |
| `height_m` | real | storey height (estimate) |
| `area_m2` | real | area (sum of the register rows) |
| `use_kind` | text | main use class (vocabulary `use_kind`) |
| `register_ordinal` | integer | the register's ordinal for it |
| `about` | text | description |
| `source_ids` | json, not null | source ids of the row (list, sources.source_id) |

### `units`

Rooms and spaces (IMDF unit); lecture rooms to come.

| Column | Type | Meaning |
|---|---|---|
| `unit_id` | text, not null | unit id |
| `building_id` | text, not null | building → `buildings.building_id` |
| `level_id` | text | level → `levels.level_id` |
| `ord` | integer, not null | order |
| `local_id` | text | id in the model |
| `name` | text, not null | name |
| `category` | text, not null | use class (vocabulary `use_kind`) |
| `room_number` | text | room number |
| `n_seats` | integer | seats |
| `has_position` | boolean, not null | position known (else only its level) |
| `z_m` | real | floor height |
| `floor_offset_m` | real | offset from the level's floor |
| `area_m2` | real | area (derived) |
| `about` | text | description |
| `facility_id` | text | facility in this unit → `facilities.facility_id` |
| `osm_id` | text | OSM element |
| `x_m` | real | local x, metres east of the venue origin (derived) |
| `y_m` | real | local y, metres north of the venue origin (derived) |
| `lat` | real | latitude, WGS84 degrees (derived) |
| `lon` | real | longitude, WGS84 degrees (derived) |
| `x_min_m` | real | bounding box west x (derived) |
| `y_min_m` | real | bounding box south y (derived) |
| `x_max_m` | real | bounding box east x (derived) |
| `y_max_m` | real | bounding box north y (derived) |
| `outline_json` | json | ring |
| `source_ids` | json, not null | source ids of the row (list, sources.source_id) |
| `is_est` | boolean, not null | estimated by us rather than given by a source |

### `openings`

Entrances (IMDF opening): doors in a building's walls.

| Column | Type | Meaning |
|---|---|---|
| `opening_id` | text, not null | opening id |
| `building_id` | text, not null | building → `buildings.building_id` |
| `level_id` | text | level (null: an outside door at ground level) → `levels.level_id` |
| `ord` | integer, not null | order (first is the main entrance) |
| `kind` | text, not null | door kind (vocabulary `opening_kind`) |
| `source` | text, not null | where the position comes from (vocabulary `opening_source`) |
| `width_m` | real, not null | width (derived) |
| `height_m` | real, not null | height |
| `canopy_m` | real, not null | canopy depth (0: none) |
| `n_steps` | integer, not null | steps in front |
| `is_step_free` | boolean, not null | no steps (derived) |
| `x1_m` | real, not null | door end 1 x |
| `y1_m` | real, not null | door end 1 y |
| `x2_m` | real, not null | door end 2 x |
| `y2_m` | real, not null | door end 2 y |
| `x_m` | real | local x, metres east of the venue origin (derived) |
| `y_m` | real | local y, metres north of the venue origin (derived) |
| `lat` | real | latitude, WGS84 degrees (derived) |
| `lon` | real | longitude, WGS84 degrees (derived) |
| `x_min_m` | real | bounding box west x (derived) |
| `y_min_m` | real | bounding box south y (derived) |
| `x_max_m` | real | bounding box east x (derived) |
| `y_max_m` | real | bounding box north y (derived) |
| `source_ids` | json, not null | source ids of the row (list, sources.source_id) |
| `is_est` | boolean, not null | estimated by us rather than given by a source |

### `facilities`

Facilities (IMDF amenity / occupant).

| Column | Type | Meaning |
|---|---|---|
| `facility_id` | text, not null | facility id |
| `ord` | integer, not null | order |
| `name` | text, not null | name |
| `category` | text, not null | category (vocabulary `facility_category`) |
| `building_id` | text | building → `buildings.building_id` |
| `building_name` | text | building name as shown |
| `is_building_inferred` | boolean, not null | building found from the position |
| `floor` | text | floor, as the source says |
| `ordinal` | integer | level ordinal read from floor (derived) |
| `room` | text | room |
| `place_source` | text, not null | where the position comes from (vocabulary `place_source`) |
| `osm_id` | text | OSM element |
| `phone` | text | phone |
| `url` | text | URL |
| `access` | text | who may use it |
| `note` | text | note |
| `aliases` | json | other names (list) |
| `conflicts` | json | where sources disagree (list) |
| `confidence` | text, not null | confidence (vocabulary `confidence`) |
| `hours_raw` | text | hours in the source's words |
| `has_hours` | boolean, not null | has published hours (derived) |
| `x_m` | real | local x, metres east of the venue origin (derived) |
| `y_m` | real | local y, metres north of the venue origin (derived) |
| `lat` | real | latitude, WGS84 degrees (derived) |
| `lon` | real | longitude, WGS84 degrees (derived) |
| `x_min_m` | real | bounding box west x (derived) |
| `y_min_m` | real | bounding box south y (derived) |
| `x_max_m` | real | bounding box east x (derived) |
| `y_max_m` | real | bounding box north y (derived) |
| `source_ids` | json, not null | source ids of the row (list, sources.source_id) |

### `facility_hours`

Facility hours, one row per season.

| Column | Type | Meaning |
|---|---|---|
| `facility_id` | text, not null | facility → `facilities.facility_id` |
| `ord` | integer, not null | order |
| `season` | text, not null | season (vocabulary `season`) |
| `rule` | text, not null | rule (OSM opening_hours syntax) |
| `description_ko` | text, not null | rule in Korean |

### `registers`

Building register title records (표제부).

| Column | Type | Meaning |
|---|---|---|
| `register_pk` | text, not null | register key |
| `name` | text | building name |
| `dong_name` | text | dong name |
| `address` | text | lot address |
| `road_address` | text | road address |
| `is_annex` | boolean, not null | an annex |
| `main_purpose` | text | main use |
| `purpose_detail` | text | use in detail |
| `structure` | text | structure |
| `structure_detail` | text | structure in detail |
| `roof` | text | roof |
| `height_m` | real | height |
| `n_floors_above` | integer | storeys above ground |
| `n_floors_below` | integer | storeys below ground |
| `building_area_m2` | real | building area |
| `total_floor_area_m2` | real | total floor area |
| `n_elevators` | integer | elevators |
| `n_parking_indoor` | integer | indoor parking spaces |
| `n_parking_outdoor` | integer | outdoor parking spaces |
| `permit_on` | date | permit date |
| `construction_on` | date | construction start |
| `approved_on` | date | use approval date |
| `has_seismic_design` | boolean | seismic design applied |
| `source_ids` | json, not null | source ids of the row (list, sources.source_id) |

### `route_edges`

Walking graph edges.

| Column | Type | Meaning |
|---|---|---|
| `edge_id` | integer, not null | edge index |
| `node_a` | integer, not null | from node → `route_nodes.node_id` |
| `node_b` | integer, not null | to node → `route_nodes.node_id` |
| `length_m` | real, not null | 3D length |
| `climb_up_m` | real, not null | climb a to b |
| `climb_down_m` | real, not null | descent a to b |
| `grade_pct` | real, not null | mean grade, max(up, down) / length x 100 (derived) |
| `n_steps` | integer, not null | steps |
| `flags` | integer, not null | bit flags (vocabulary `route_flag`) |
| `is_steps` | boolean, not null | steps (derived) |
| `is_road` | boolean, not null | a street without sidewalks (derived) |
| `is_indoor` | boolean, not null | indoors (derived) |
| `is_covered` | boolean, not null | covered (derived) |
| `is_lit` | boolean, not null | lit (derived) |
| `is_est` | boolean, not null | an estimated link (derived) |
| `is_crossing` | boolean, not null | crossing (derived) |
| `is_bridge` | boolean, not null | bridge (derived) |
| `is_underground` | boolean, not null | underground (derived) |
| `is_unpaved` | boolean, not null | unpaved (derived) |
| `is_lift` | boolean, not null | elevator or short steep ramp (derived) |
| `way_name_id` | integer | way name → `route_way_names.way_name_id` |
| `width_m` | real, not null | width |
| `x_min_m` | real | bounding box west x (derived) |
| `y_min_m` | real | bounding box south y (derived) |
| `x_max_m` | real | bounding box east x (derived) |
| `y_max_m` | real | bounding box north y (derived) |

### `sources`

Sources: each dataset or page, its URL, licence and the day it was read.

| Column | Type | Meaning |
|---|---|---|
| `source_id` | text, not null | source id |
| `title` | text, not null | title |
| `kind` | text, not null | kind (vocabulary `source_kind`) |
| `attribution` | text | attribution to show |
| `url` | text | URL |
| `licence` | text | licence |
| `licence_url` | text | licence URL |
| `fetched_on` | date | day it was fetched or read |
| `raw_copy` | text | kept copy in the repo |
| `extra_json` | json | other values, as the bundle has them |

## Vocabularies

Enumerated columns take values from `vocab`; each value has a Korean label. The most used:

- `facility_category`: `cafeteria` (학생식당), `restaurant` (식당), `cafe` (카페), `convenience` (편의점), `store` (매장), `bookstore` (서점), `vending` (자판기), `print` (복사·인쇄), `library` (도서관·자료실), `reading_room` (열람실), `lounge` (휴게·라운지), `nursing` (수유실), `health` (건강센터), `aed` (AED), `bank` (은행), `atm` (ATM), `post` (우체국), `accessible` (장애인 편의), `parking` (주차장), `shuttle` (셔틀 정류장), `bus_stop` (버스 정류장), `sports` (체육시설), `smoking` (흡연구역), `toilet` (화장실), `water` (식수대), `bike` (자전거)
- `use_kind`: `parking` (주차장), `medical` (의료), `food` (식당·카페), `shop` (상점·편의), `library` (도서관), `hall` (공연·강당), `lecture` (강의실), `research` (연구·실험), `dorm` (기숙), `plant` (기계·전기실), `storage` (창고), `office` (사무·로비), `corridor` (통로), `road` (도로·교통광장), `other` (대학 시설)
- `opening_kind`: `door` (문), `glass` (유리문), `arch` (아치), `gate` (대문), `garage` (차고문)
- `season`: `all` (연중), `semester` (학기 중), `vacation` (방학 중), `vacation_edge` (방학 첫·마지막 2주)
- `confidence`: `high` (높음), `medium` (보통), `low` (낮음)
- `source_kind`: `dataset` (공개 자료), `page` (웹 쪽), `model` (누비가 만든 모형·값)
- `route_flag`: `steps` (계단), `road` (보도 없는 차도), `indoor` (건물 안), `covered` (지붕 있음), `lit` (조명), `est` (추정한 연결), `crossing` (횡단보도), `bridge` (다리), `under` (지하), `unpaved` (비포장), `lift` (엘리베이터·급경사로)

## Example queries

Buildings with parking below ground, deepest first:

```sql
SELECT b.name, MIN(l.ordinal) AS lowest, SUM(l.area_m2) AS parking_m2
FROM levels l JOIN named_buildings b USING (building_id)
WHERE l.ordinal < 0 AND l.use_kind = 'parking'
GROUP BY b.building_id ORDER BY lowest;
```

Everything within 100 m of a point (the R-tree):

```sql
SELECT i.table_name, i.item_id, i.name
FROM spatial_index s JOIN spatial_items i USING (geo_id)
WHERE s.x_min_m <= :x + 100 AND s.x_max_m >= :x - 100
  AND s.y_min_m <= :y + 100 AND s.y_max_m >= :y - 100;
```

A facility's hours by season:

```sql
SELECT f.name, h.season, h.rule, h.description_ko
FROM facilities f JOIN facility_hours h USING (facility_id)
WHERE f.category = 'cafeteria' ORDER BY f.name, h.ord;
```

`rule` uses a subset of OpenStreetMap's `opening_hours` syntax (`Mo-Fr 08:00-19:00; Sa 10:00-15:00`); `season` says
when it applies.

Step-free edges of the walking graph are those with `is_steps = 0`; `flags` packs the `route_flag` bits, and each bit
also has its own boolean column.

## Versioning

- The **schema version** follows the columns: major for a rename or removal, minor for new tables or columns, patch
  for meanings and vocabularies.
- The **release number** follows what the data is: a new kind of data or a new source.

| Schema | Release | Date | Change |
|---|---|---|---|
| 1.0.0 | 1.0 | 2026-10-31 | first release |
| 1.1.0 | 1.1 | 2026-10-31 | `context_areas` (forests and clearings around the campus that OpenStreetMap lacks: 안산 and its slopes); vocabularies `context_kind`, `clearing_ground` |
