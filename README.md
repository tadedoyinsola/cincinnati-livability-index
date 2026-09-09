# Cincinnati Apartment Livability Index

A reproducible geospatial workflow that assigns a livability score to apartment buildings in Cincinnati, Ohio, from open data. Built with Python (GeoPandas, osmnx) as an applied exercise in agentic geospatial data science.

> Status: analysis complete for three factors (parks, transit, schools). Scores and figures below were produced from OpenStreetMap data retrieved via `osmnx`.
> See [Reproducing the analysis]
---

## 1. Summary

This project calculates a composite livability score for every apartment building in Cincinnati, so that a prospective resident can compare buildings by how well their surroundings support daily life. The current index combines three neighborhood-access factors, each derived from OpenStreetMap:

- proximity to green space (parks within 1 km),
- access to public transit (distance to the nearest stop), and
- access to schools (distance to the nearest school).

Each factor is normalized to a 0 to 100 scale, combined using explicit, adjustable weights, and written back to the building layer. The output is a GeoPackage of apartment buildings enriched with the individual factor values and the final `livability_score`.

The methodology mirrors the construction of composite spatial indices used in public health and urban planning (for example social vulnerability or amenity-access indices): choose indicators, measure each per spatial unit, normalize to a common scale, weight, combine, and validate.

---

## 2. Study area and data

**Study area:** the City of Cincinnati, Ohio, United States.

**Data source:** OpenStreetMap (OSM), accessed programmatically with the `osmnx` Python package. OSM is an open, community-maintained dataset; its completeness varies by feature type and neighborhood, which is discussed as a limitation in Section 6.

| Layer | OSM query | Count retrieved |
|---|---|---|
| Apartment buildings | `building=apartments` (polygons only) | 4,307 |
| Parks | `leisure=park` | 215 |
| Transit stops | `highway=bus_stop`, `railway=stop`/`station`/`tram_stop` | 1,854 |
| Schools | `amenity=school` | 133 |

> Reconcile these counts against your live notebook before publishing; they are taken from the analysis runs and may differ slightly if OSM data has updated.

All distance and buffer computations were performed in a projected coordinate system, UTM Zone 16N (EPSG:32616), so that distances are in true meters rather than degrees. Data was reprojected to EPSG:4326 only for web display.

---

## 3. Factors and how they were measured

### 3.1 Parks within 1 km (`parks_within_1km`)

For each apartment building, the number of park polygons whose geometry falls within a 1 km buffer of the building. Higher is better.

| Statistic | Value |
|---|---|
| min | 0 |
| 25th percentile | 1 |
| median | 3 |
| 75th percentile | 6 |
| max | 30 |
| mean | 4.85 |

### 3.2 Distance to nearest transit stop (`distance_to_transit_m`)

Straight-line distance, in meters, from each building to the nearest transit stop. Lower is better.

| Statistic | Value (m) |
|---|---|
| min | 0.0 |
| 25th percentile | 72.9 |
| median | 182.2 |
| 75th percentile | 372.3 |
| max | 3,210.5 |
| mean | 264.6 |

### 3.3 Distance to nearest school (`distance_to_school_m`)

Straight-line distance, in meters, from each building to the nearest school. Lower is better.

| Statistic | Value (m) |
|---|---|
| min | 0.0 |
| 25th percentile | 210.2 |
| median | 411.7 |
| 75th percentile | 706.8 |
| max | 4,058.9 |
| mean | 494.4 |

---

## 4. Normalization and weighting

### 4.1 Normalization

Each raw factor is rescaled to a 0 to 100 scale, where 100 is always the most desirable value. For `higher_is_better` factors (parks) the raw value is scaled directly; for `lower_is_better` factors (transit distance, school distance) the scale is inverted, so that a building sitting on top of a stop scores near 100 and a distant building scores near 0.

Each factor carries an explicit direction flag in a single `FACTOR_CONFIG` structure, so the scoring logic is transparent and a reader can change a direction, weight, or factor without touching the rest of the pipeline.

**Note on skew.** All three factors have a long right tail: park counts reach a lone maximum of 30 against a median of 3, and both distance factors have a single far outlier (a building near Kellogg Avenue) well beyond the next cluster. Because min-max normalization stretches the scale to the maximum, these outliers compress the differences among the majority of buildings. A percentile-rank normalization or a capped maximum would produce a more balanced spread and is noted as a recommended refinement.

### 4.2 Weighting

The three factors are combined as a weighted average of their normalized values. Rather than weight all factors equally, this index gives transit and school access slightly more weight than park count:

| Factor | Weight | Rationale |
|---|---|---|
| Distance to transit | 0.35 | Transit access is a strong, consistently valued determinant of daily livability. |
| Distance to school | 0.35 | School proximity is a primary factor for a large share of home buyers. |
| Parks within 1 km | 0.30 | Green space matters, but this count is partly an artifact of OSM tagging density (see limitations). |

This is a deliberate, transparent choice, not a neutral default. The weights are defined in one place and are trivial to change. As a sensitivity check, the index was also computed with equal weights (1/1/1); the two schemes produce broadly similar rankings, which indicates the score is reasonably robust to the weighting decision.

> Confirm in your notebook which weighting the final GeoPackage was written with, and set `FACTOR_CONFIG` accordingly, so the published score matches this table.

---

## 5. Validation

Validation was performed factor by factor, and then on the composite, by checking whether extreme values fall where local knowledge of Cincinnati expects them.

**Parks.** The highest park counts belong to buildings in Over-the-Rhine and downtown (Vine Street, Republic Street, 12th Street, Bakery Lofts), which are genuinely dense with small parks and plazas. Zero-count buildings sit on outer residential streets (Cambridge Avenue, North Bend Road). Direction and location both check out.

**Transit.** The nearest-to-transit buildings sit essentially on top of stops along Vine Street and Broadway downtown (sub-meter distances). The farthest is 5920 Kellogg Avenue, far east along the river where transit coverage is sparse. A median distance of ~182 m is consistent with a reasonably dense urban bus network.

**Schools.** Median nearest-school distance is ~412 m. The farthest buildings cluster on Disney Street and Spindlehill Drive, neighboring streets that are both far from any mapped school (see limitations). Notably, 5920 Kellogg Avenue is both farthest from transit and near-farthest from a school, an independent cross-check indicating it is a genuinely underserved location rather than a data artifact.

**Composite.** After blending, high-scoring buildings concentrate in the transit- and amenity-rich urban core, and low-scoring buildings fall in outer, underserved pockets, consistent with the per-factor results.

---

## 6. Limitations

**OSM completeness.** All factors depend on how thoroughly features are tagged in OpenStreetMap. Counts and distances measure the mapped environment, not necessarily the real one.

**Park count is tagging-sensitive.** The high park counts in Over-the-Rhine partly reflect that dense, well-mapped areas have many small greens, plazas, and playgrounds tagged as separate `leisure=park` polygons. "Parks within 1 km" should be read as "count of OSM park polygons," not distinct named parks.

**Possible school-coverage gap.** The Disney Street / Spindlehill Drive cluster shows large nearest-school distances. This may be a real access gap or an OSM tagging gap (an unmapped school); it should be checked against the `schools.geojson` layer before the finding is relied upon.

**Straight-line distance.** Distances are Euclidean, not network or walking distances. Real access is constrained by streets, rivers, and barriers, so true travel distances are longer. A network-distance or isochrone version is a natural refinement.

**Outlier-sensitive normalization.** Min-max scaling lets single far outliers compress the rest of the distribution; a percentile-based normalization would be more robust.

---

## 7. Outputs

- `outputs/livability/apartment_buildings_scored.gpkg` — apartment buildings with all factor columns (`parks_within_1km`, `distance_to_transit_m`, `distance_to_school_m`), their normalized counterparts, and the final `livability_score`.
- `outputs/livability/apartment_buildings_scored.geojson` — same layer in EPSG:4326 for web viewers.
- Intermediate layers in `data/livability_score/`: `apartment_buildings`, `parks`, `transit_stops`, `schools`.

![Livability score map](outputs/livability/livability_score_map.png)

---

## 8. Reproducing the analysis

```bash
# 1. Create and activate the environment
conda create --name claude_code_workshop -y
conda activate claude_code_workshop
conda install -c conda-forge pandas geopandas matplotlib jupyterlab osmnx -y

# 2. Open the notebook
jupyter lab scripts/livability_score.ipynb

# 3. Run all cells. Outputs are written to outputs/livability/.
```

The notebook downloads all data live from OpenStreetMap, so no data files need to be committed to the repository.

---

## 9. License and credits

Data: © OpenStreetMap contributors, ODbL.

Built as part of the Agentic Coding for Geospatial workshop by Spatial Thoughts (Ujaval Gandhi), courses.spatialthoughts.com.

Analysis and write-up by Tolulope Oladeji.
