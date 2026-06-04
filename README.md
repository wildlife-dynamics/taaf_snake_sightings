# Snake Sightings Workflow

The **Snake Sightings Workflow** processes iNaturalist snake observation data from EarthRanger, produces distribution maps at species and community level, and compiles everything into a structured Word report and interactive dashboard.

---

## Overview

This workflow:

1. Connects to **EarthRanger** and fetches iNaturalist snake observation events within a user-defined time range.
2. Parses and cleans observation details — extracting species names, common names, and observer information.
3. Filters records to **valid binomial species** and removes spatial outliers.
4. Builds a **species summary table** with observation counts, observer counts, unique locations, date ranges, and estimated range area.
5. Generates a **15 km observation effort grid** and a **15 km species richness grid** across the study area.
6. Produces three **community-level maps**: all-species distribution, observation effort, and species richness.
7. Dynamically generates **per-species distribution maps** — one for every species found in the data — without any hardcoded species list.
8. Assembles a **Word report** containing:
   - Cover page with logos and generation date
   - All-species, effort, and richness maps
   - Species summary table
   - Per-species distribution map pages
9. Publishes an **interactive dashboard** with all map widgets and the summary table.

---

## Connecting to a Data Source

This workflow requires an active **EarthRanger** connection to fetch observations.

### Prerequisites

- Access to an EarthRanger site that has iNaturalist observations ingested as events of type `inat_observation`.
- Valid credentials (API token or username/password) for that EarthRanger site.

### Steps

1. Open the workflow in the **EcoScope Desktop** app.
2. In the **Connect to EarthRanger** step, select your EarthRanger site from the data source list, or enter your site URL and credentials if connecting for the first time.
3. Ensure your account has at minimum **read access** to events on that site.
4. Confirm the connection is active before running — the workflow will fail at the fetch step if the connection is invalid or the site is unreachable.

### Observations Required

The workflow expects events with:

| Field | Description |
|-------|-------------|
| `event_type` | Must be `inat_observation` |
| `details.taxon_name` | Scientific name of the observed species |
| `details.taxon_common_name` | Common name |
| `details.user_name` | Observer username |
| `details.inat_id` | iNaturalist observation ID |
| `geometry` | Point geometry (latitude/longitude of the sighting) |

Events missing `taxon_name` or geometry are automatically excluded during filtering.

---

## User Inputs

### Required

| Input | Description |
|-------|-------------|
| **EarthRanger Connection** | The EarthRanger site to fetch observations from. |
| **Time Range** | Start (`since`) and end (`until`) dates for the observation window. Default: `2000-01-01` → `2026-04-01`. |

### Optional (advanced users)

These are pre-configured for standard use but can be adjusted if needed:

| Input | Default | Description |
|-------|---------|-------------|
| **Grid Size (effort & richness)** | `15 000 m` | Cell size of the effort and richness grids. |
| **Outlier Removal IQR Factor** | `3.0` | Controls how aggressively spatial outliers are filtered. Higher = less filtering. |
| **Map Max Zoom** | `5` | Maximum zoom level for community-level maps. |
| **Per-Species Buffer Fraction** | `0.15` | Padding added around each species' bounding box when fitting the map viewport. |
| **Maps Per Row / Col (report)** | `2 × 3` | Grid layout of per-species maps in the Word report pages. |

---

## Workflow Steps

| Step | Purpose |
|------|---------|
| **Set Workflow Details** | Records workflow name and description metadata. |
| **Define Time Range** | Sets the observation date window for the analysis. |
| **Configure Grouping Strategy** | Defines how results are grouped (default: none). |
| **Connect to EarthRanger** | Authenticates and opens a connection to the EarthRanger data source. |
| **Fetch iNaturalist Observations** | Pulls snake observation events of type `inat_observation` from EarthRanger. |
| **Parse Event Details** | Extracts structured fields (`taxon_name`, `taxon_common_name`, `user_name`, `inat_id`) from raw event detail JSON. |
| **Filter to Valid Binomial Species** | Keeps only records with a two-part scientific name and removes spatial outliers using IQR-based filtering. |
| **Build Species Summary Table** | Computes per-species statistics: observation count, unique observers, unique grid locations, first/last observation date, and range area (km²). |
| **Save Summary Table as Word Doc** | Exports the summary table as a formatted `.docx` file for inclusion in the final report. |
| **Render Summary Table as HTML** | Creates an interactive sortable HTML table for the dashboard. |
| **Prepare All-Species Points GDF** | Builds a combined GeoDataFrame of all observations with a convex hull overlay. |
| **Create All-Species Layer** | Styles the all-species points as a colour-coded map layer. |
| **Compute All-Species View State** | Auto-fits the map viewport to the all-species data extent. |
| **Draw All-Species Map** | Renders the interactive HTML map with terrain and hillshade base layers. |
| **Create Observation Effort Grid** | Aggregates observations into a 15 km grid and colours cells by observation count. |
| **Create Effort Layer** | Styles the effort grid as a choropleth map layer. |
| **Compute Effort View State** | Auto-fits the viewport to the effort grid extent. |
| **Draw Observation Effort Map** | Renders the effort map as interactive HTML. |
| **Create Species Richness Grid** | Counts unique species per 15 km grid cell and colours by richness. |
| **Create Richness Layer** | Styles the richness grid as a choropleth map layer. |
| **Compute Richness View State** | Auto-fits the viewport to the richness grid extent. |
| **Draw Species Richness Map** | Renders the richness map as interactive HTML. |
| **Generate All Per-Species Outputs** | For every species discovered at runtime: builds a points GDF, renders a per-species HTML map, converts to PNG, and creates a dashboard widget. |
| **Extract Per-Species Widgets** | Unpacks the widget list from the per-species outputs bundle. |
| **Extract Per-Species PNG Paths** | Unpacks the PNG file paths for use in the Word report. |
| **Extract Per-Species Species Names** | Unpacks the species name list for labelling map pages. |
| **Build Per-Species Distribution Pages** | Lays out per-species PNG maps in a 2-column × 3-row grid per page as a `.docx` file. |
| **Generate Word Report Template** | Creates the base Word document with TAAF and EcoScope logos, section headings, and map placeholders. |
| **Build Report Template Context** | Assembles the map HTML paths and generation date into a context object for rendering. |
| **Render Main Report Sections** | Screenshots the three HTML maps via Playwright and embeds them into the Word document. |
| **Assemble Final Report** | Appends the summary table and per-species pages to the rendered report, producing the single final `.docx`. |
| **Snake Sightings Dashboard** | Publishes all widgets — summary table, community maps, and per-species maps — into a single interactive dashboard. |

---

## Outputs

All outputs are written to:

```
$ECOSCOPE_WORKFLOWS_RESULTS/
```

### Word Report

| File | Description |
|------|-------------|
| `taaf_snake_distribution_report.docx` | **Final combined report** — cover page, three community maps, species summary table, and per-species distribution pages. |
| `report_template.docx` | Intermediate: base template with logos and placeholders. |
| `snake_distribution_report_<hash>.docx` | Intermediate: rendered cover + maps section before combining. |
| `species_summary_table.docx` | Intermediate: summary table section. |
| `per_species_maps.docx` | Intermediate: per-species distribution pages. |

### Maps (HTML)

| File | Description |
|------|-------------|
| `*_all_snakes_distribution.html` | Interactive all-species distribution map. |
| `*_observation_effort.html` | Interactive observation effort choropleth map. |
| `*_species_richness.html` | Interactive species richness choropleth map. |
| `map_<species_slug>.html` | One interactive map per species (e.g. `map_bitis_arietans.html`). |

### Images (PNG)

| File | Description |
|------|-------------|
| `map_<species_slug>.png` | High-resolution screenshot of each per-species map, embedded in the Word report. |

### Dashboard Widgets

| Widget | Description |
|--------|-------------|
| Species Summary Table | Sortable table of all species statistics. |
| All Snake Species Distribution | Community-level distribution map. |
| Snake Observation Effort | 15 km effort grid choropleth. |
| Species Richness Map | 15 km richness grid choropleth. |
| Per-Species Maps | One widget per species, dynamically generated at runtime. |

---

## Running the Workflow

1. Launch the Snake Sightings Workflow from the EcoScope Desktop.
2. Connect to your **EarthRanger data source** (see [Connecting to a Data Source](#connecting-to-a-data-source) above).
3. Set the **time range** (since / until dates).
4. Click **Run** and wait for the workflow to complete.
5. Retrieve outputs from your configured results directory:
   - Open `taaf_snake_distribution_report.docx` for the full printable report.
   - Open the `.html` map files in any browser for interactive exploration.
   - Load the **dashboard** in EcoScope Desktop for the full widget view.

> **Note:** Per-species map generation scales with the number of species in the data. For large datasets spanning many species, the workflow may take several minutes to complete the per-species rendering step.

---

## How the Report is Built

The final Word report is assembled in three stages:

```
Stage 1 — Template generation
  create_report_template  →  report_template.docx
  (logos, headings, map placeholders)

Stage 2 — Map rendering
  build_maps_context  →  create_docx
  (screenshots the 3 HTML maps via Playwright and embeds them)
  →  snake_distribution_report_<hash>.docx

Stage 3 — Assembly
  combine_docx_files
  [rendered report + species_summary_table + per_species_maps]
  →  taaf_snake_distribution_report.docx
```

Final document order:

1. Cover page with TAAF and EcoScope logos
2. All Snake Species Distribution map
3. Observation Effort map
4. Species Richness map
5. Species Summary Table
6. Per-Species Distribution Maps (2 per row, 3 rows per page, alphabetical order)

---

## Technical Notes

### Dynamic Species Discovery
Species are discovered at runtime from the filtered observations — there is no hardcoded species list. The workflow generates maps and report pages for every valid binomial species present in the data for the selected time range.

### Spatial Outlier Removal
Observations are filtered using IQR-based outlier detection on coordinates. This prevents a single erroneous GPS point from distorting the distribution map or inflating the range area calculation.

### Grid Coordinate System
Effort and richness grids are computed in **EPSG:32737** (UTM Zone 37S — East Africa) for accurate area calculations, then reprojected to **EPSG:4326** for mapping.

---

## Future Enhancements

- **Configurable grid size** — expose the 15 km cell size as a user-facing parameter.
- **Species filter** — allow users to limit the report to a specific subset of species.
- **Date annotations** — add the observation date range as a subtitle on each report map.
- **PDF export** — generate a PDF version of the final report alongside the Word document.
- **Multi-site support** — aggregate observations from multiple EarthRanger sites in a single run.
