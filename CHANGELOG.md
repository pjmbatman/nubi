# Changelog

Changes to Nubi, newest first: the service (the map, the assistant, the site) and the campus data release
(`nubi-campus`, see [DATA_FORMAT.md](DATA_FORMAT.md)).

## Service (2026-11-01)

- **Sign-in pages**: the whole campus as an architect's model on one board, with a short pre-rendered film of Nubi in
  use: a route up 백양로, a question to the assistant and its route followed from above to 백양누리, the 지하 view,
  a day's timetable route, the places open now and the afternoon shade over 노천극장. Light (morning) and dark
  (evening) versions, and a portrait framing for phones.
- **Map**: the junction of 첨단과학관 and IBS관 redrawn: the open undercroft on pillars, the glass entrances.

## Campus data 1.1 (2026-10-31, schema 1.1.0)

- `context_areas`: the forests OpenStreetMap lacks around the campus (안산 and its slopes), with canopy density and the
  share of conifers, and the clearings in them (rock, bare ground, grass).

## Campus data 1.0 (2026-10-31, schema 1.0.0)

- **First release**: one SQLite file with 42 tables: buildings and their pieces, the building register's title and
  floor records, levels and their uses, entrances, 백양누리, the underground ways, facilities and their hours, the
  academic calendar, the map's features, paths, areas, trees, the city around, the walking graph and the audit of
  every map cell. IMDF tables for indoor data, sources on every row, the schema inside the file, an R-tree index.
  The same input builds the same file byte for byte.

## Service (2026-10-31)

- **Assistant**: answers from the campus data release (what the release holds, what a column means, read-only
  queries, everything about one building), and finds restaurants and cafés near a place by walking time.
- **Nearby places**: restaurants and cafés within about 1 km of the campus, from the public business-licence records
  joined to OpenStreetMap, with a walking graph of the streets around the campus.
- **Routes**: the route drawn as a ribbon of light.
- **Map**: 안산 and its slopes forested; the building sites (제5공학관, and the halted extension of 과학원); the campus
  land between the detail zones modelled too; ground markings (lines, crossings, tactile strips), hedges, statues,
  dormers and stepped gables; a second cell-by-cell audit of every zone.

## Service (2026-10-30)

- **Site**: Nubi opens at **nubi.yonsei.ac.kr** with a Google sign-in (Yonsei accounts at once, others on approval).
  The map is the main page, with a question box in the middle and tabs at the top; the Yonsei-blue theme and the
  stitched-route mark.
- **Assistant**: finds places and flies the map there, switches between first and third person, searches the map's
  data with filters, plays guided tours, sets the time of day, and says how much of a place is in shade.
- **Routes (길찾기)**: a walking route engine over the campus paths and the map's ground, with three profiles
  (최단, 계단 없이, 오르막 적게); used by both the map and the assistant.
- **Facilities**: cafeterias, cafés, shops, reading rooms, printing, ATMs, shuttle stops, AEDs and more, with their
  hours by season and whether they are open now.
- **Underground (지하)**: the basement levels of every building from the building register's floor records, a model of
  백양누리, the underground passages, and routes that use them.
- **Map**: every building's facade (window types, bands, materials, glass, night lights) drawn in the shader; building
  entrances; the ground drawn in the shader (kerbs, grass, paving, asphalt); layered trees; gates, courts and small
  buildings checked against aerial references and the 서대문구 360° VR tour.

## Service (2026-10-29)

- **First map**: the Sinchon campus and about 500 m around it from OpenStreetMap, surveyed building outlines from
  GIS건물통합정보 joined to the building register by building id, heights from the register, terrain from the
  Copernicus DEM. Third and first person, the real sun.
- **Detail zones**: the campus divided into zones (정문 and 백양로, the central square, 본관, the science and
  engineering areas, the north campus, the dorms, 세브란스, 노천극장, the playing fields), each modelled feature by
  feature: roofs, towers, plazas, stairs, pitches, stands, statues, parking.
- **Credits page** (출처) with every data source and its licence.
