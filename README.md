# climate_impact_curves

## 1. Aims and problem description

Building impact curves for tree mortality / biomass loss from two climate
hazards, drought and storm, following on from the `fire-impact-curves`
project's approach (dNBR -> loss share, fitted as a logistic curve against
fire severity). The two hazards here have no single intensity metric as
clean as dNBR, so each needs its own working definition of "event" and
"intensity" before a curve can be fit at all - see Section 3.

- **Drought**: tree mortality / biomass loss vs. SPEI (drought severity index).
- **Storm**: tree loss vs. wind speed. No wind/storm dataset identified yet
  (candidates: IBTrACS tropical cyclone tracks, or a reanalysis wind
  product) - open task, not started.

Core method carried over from `fire-impact-curves`:
`loss share = (AGB_before - AGB_after) / AGB_before`, from ESA CCI
Above-Ground Biomass, fit as a logistic curve against hazard intensity.
Uncertainty computed by event (by fire there; by drought period/storm here),
not by pixel. Validation discipline to carry over: leave-one-event-out
cross-validation, a true holdout set, and a placebo/control check before
trusting any result.

## 2. Infrastructure (already set up, reused as-is)

- GCP project `tree-fire-510209` ("tree-fire"), Vertex AI Workbench instance
  `tree-fire-workbench` (`europe-west2-a`).
- Earth Engine authorised via `ee.Authenticate()` + `ee.Initialize(project="tree-fire-510209")`.
- GCS export bucket `gs://tree-fire-510209-data` (not yet used in the
  drought pilot - small test pulls so far have gone through
  `getDownloadURL` directly rather than batch export).
- `src/grids.py`, `src/curves.py` copied over from `fire-impact-curves`,
  unmodified, both generic (not fire-specific).

## 3. Data sources

- **ESA CCI Above-Ground Biomass V6.0** (`ESA/CCI/Above_Ground_Biomass/V6_0`
  on Earth Engine). Annual mosaics, but *not every year*: available years
  are 2007, 2010, 2015, 2016, 2017, 2018, 2019, 2020, 2021, 2022 - no 2011-2014,
  which matters for picking before/after pairs (Section 5). Each image has
  2 bands: band 1 is AGB mean (Mg/ha, uint16), band 2 is AGB
  uncertainty/SD (confirmed empirically, not from file metadata - band 2's
  mean and std both scale with band 1's rather than being small/flat, no
  band descriptions or tags were present in the downloaded GeoTIFFs to
  confirm this directly).
- **SPEI-12**: global 0.5 deg monthly grid, 1901-2023, copied from
  `fire-impact-curves/data/drought/spei12.nc` (already existed there from
  an earlier test). Variable name `spei`, `lat` ascending -89.75 to 89.75,
  `lon` -179.75 to 179.75 - standard orientation, no surprises.
- **Hansen Global Forest Change** (`UMD/hansen/global_forest_change_2023_v1_11`),
  used only as an independent disturbance check (Section 6), not as a
  primary data source - `lossyear` band flags canopy-cover loss per pixel,
  year-coded (e.g. 17 = 2017).
- Storm/wind dataset: not yet identified (open task).

## 4. Pilot choice: California 2012-2016 drought

Chosen over Amazon 2015-16 or a European drought because an independent
ground-truth mortality dataset exists to sanity-check against (USFS Aerial
Detection Survey, not yet pulled in - deliberately deferred until after the
first AGB-loss pass, so it doesn't bias how the event/threshold is tuned).

**SPEI sanity check** (mean SPEI-12 over lat 32-42, lon -124 to -114):
drought clearly visible, trough around early 2014 at approx -1.6, sustained
below the -1.5 threshold line for a stretch around then, broader dry period
roughly 2012-2015, recovering by 2016. Matches the documented event.

**Assumption, not yet verified**: "event" is being operationalised as
SPEI-12 <= -1.5 for several consecutive months in a region - a convention
from the SPEI drought literature, not independently confirmed as the right
threshold/duration for *this* purpose.

## 5. Pilot region and before/after year choice

Test bbox: Sierra Nevada, lon -119.5 to -118.5, lat 35.5 to 37.5 - the
area most associated with the documented 2012-2016 bark-beetle die-off
(USFS reported ~129M dead trees statewide, concentrated in the southern/
central Sierra).

Because of the AGB mosaic gaps (Section 3), before/after years are
constrained, not freely chosen:

- **Before**: 2010 (nearest available pre-drought mosaic - leaves a 2-4
  year gap before the drought even starts, since there's no 2011-2013
  mosaic).
- **After**: tested 2015, 2016, 2017, 2018 rather than committing to one -
  drought-driven bark-beetle mortality in California peaked after the
  trough, continuing into 2016-2017, so a single fixed "after" year was not
  assumed upfront.

**Assumption, flagged not verified**: using 2010 as baseline means any
non-drought disturbance in 2010-2012 in the test bbox (fire, logging) would
get misattributed as drought-era loss. Not yet checked against a fire-
perimeter or disturbance layer for that specific window.

## 6. Dead end (so far): raw year-to-year CCI AGB differencing looks unreliable here

First pass: downloaded AGB for 2010 and 2017 over the Sierra bbox, computed
`AGB_2010 - AGB_2017` per pixel. The resulting map does **not** look like a
drought-mortality footprint - it shows a strong, spatially coherent band of
biomass *gain* running along the high Sierra crest, with *loss* concentrated
in a lower-elevation band to its northwest. The pattern tracks terrain/
elevation, not any known mortality distribution.

This reproduces an issue `fire-impact-curves` already flagged and left
unresolved (its README Section 7, known issue #2): *"Noise in annual CCI
AGB: needs a proper control... use of ESA's AGB-change maps with their
uncertainty and quality flags. [Not yet] checked ESA's guidance on
year-to-year change detection."* Treating this pilot's raw before/after
difference as a trustworthy change-detection product was an unexamined
assumption carried over from the fire pipeline, where it worked because
fire gives a real, abrupt before/after; drought does not have an
equivalently sharp before/after, making CCI's year-to-year noise more
visible/more likely to dominate.

**In progress**: testing whether the banding persists in pixels that Hansen
Global Forest Change's `lossyear` band shows no detected canopy loss
2010-2018 (i.e., pixels with no independently-confirmed disturbance of any
kind). If the banding survives that mask, it's most likely a CCI processing
artifact rather than real change. If it disappears, the pattern was
probably tracking something real, worth checking against drought
specifically from there. Chose this over picking a new "undisturbed"
control region, because in this window no forested California region can
be assumed disturbance-free just by inspection - fire, logging and
beetle mortality are scattered across nearly all of it - whereas Hansen
gives an independent, pixel-level check on the exact data already pulled.

Caveat on this check, flagged in advance: Hansen's `lossyear` is tuned to
canopy-cover loss (fire, clearcut) and may under-detect slower, partial-
canopy drought/beetle mortality - so a clean result here rules out "obvious
disturbance contamination" but doesn't by itself prove the remaining
pattern is drought-specific.

**Result: confirmed artifact, not real change.** Masked out any pixel
where Hansen's `lossyear` shows *any* detected canopy loss 2010-2018 (6.2%
of the bbox) and recomputed the same 2010-2017 difference map on the
remaining 93.8%. The coherent, terrain-following banding survived
essentially unchanged. Since these pixels have zero independently-detected
disturbance of any kind, this rules out real mortality as the explanation -
confirmed CCI year-to-year artifact, not a drought signal. This is more
serious than ordinary noise: the bias is systematic and terrain-correlated,
which risks a spurious SPEI-loss correlation (terrain correlates with both
forest type/density and often drought exposure) rather than just adding
imprecision.

**Follow-up: ESA publishes an official change product for exactly this
problem**, confirmed via its product fact sheet (Santoro, v5.0, March
2024) - not something `fire-impact-curves` had used. Consecutive-year and
one decadal (2020-2010) change layers are available, each with a standard
deviation and quality-flag band (`AGB_DIFF_SD`, `AGB_DIFF_QF`), via the
community Earth Engine asset `projects/sat-io/open-datasets/ESA/ESA_CCI_AGB_DIFF`
(not an official Google/ESA-maintained EE catalog entry - worth treating
the exact version/vintage as unconfirmed). QF legend, quoted directly from
the fact sheet: `0` AGB=0 in both maps; `1` AGB loss; `2` potential AGB
loss; `3` improbable change; `4` potential AGB gain; `5` AGB gain. The
actual change value is still computed by the user (not provided
pre-computed) - QF/SD are reliability layers to screen it with, not a
replacement for it.

**New, more serious problem surfaced by the same document**: ESA's own
fact sheet states the **2010 AGB map specifically** is lower-quality than
later years and warns against trend comparisons that include it: *"The
2010 dataset may present AGB > 0 in contrast to AGB = 0 in more recent
years... due to the lower quality of the EO data used to estimate AGB in
2010... Trends... may need to be critically investigated, in particular
when comparing 2010 with more recent years."* 2010 is the only available
pre-drought mosaic for this pilot (Section 3's gap years). This is not a
maskable/fixable-with-QF problem the way the Hansen check was - it's ESA
flagging the one baseline year we have access to as unreliable for this
exact kind of comparison. Open decision, not yet resolved: whether to
proceed with 2010 as baseline anyway (QF-filtered, caveated heavily),
switch the pilot's dependent variable away from CCI AGB entirely for the
"before" side (e.g., a dataset with full annual coverage through the
actual drought years), or accept a narrower pilot scope that avoids 2010
altogether (e.g. SPEI-severity vs. loss only *within* 2015-2020, forgoing
a true pre-drought baseline).

**Decision**: narrow the pilot to 2015-2020 (the years ESA's fact sheet
rates "equally reliable", excluding 2010), rather than fighting the 2010
baseline. This gives up a true pre-drought baseline for this specific
event - the comparison now captures the tail of the 2014-peak die-off and
post-drought trajectory (2015 was already within the drought's later
stage), not loss from a genuinely pre-drought state. Chosen over hunting
for an alternative pre-2012 biomass source (real research cost, uncertain
payoff) or using Artio's in-house quarterly biomass model as a
replacement baseline (considered and deferred - see below).

**Follow-on, deliberately deferred, not dropped**: Artio has an in-house
quarterly biomass model (Sentinel-2 + radar + land cover + elevation),
still being scaled in `fire-impact-curves`, with a known, documented bias
outside conifer forest (systematically 2-4x low vs. CCI on non-Needleleaf
types) and unresolved infrastructure issues. Two reasons not to bring it
into this pilot yet: (1) it's very likely Sentinel-2/-1 dependent, meaning
its time series probably doesn't extend earlier than ~2015-2016 anyway -
not yet confirmed - so it likely wouldn't solve the 2010-baseline problem
even if used; (2) using an unvalidated biomass model and a first-pass
drought methodology together would tangle two sources of uncertainty, with
no way to tell which one explains any odd result. Worth revisiting
specifically because the Sierra Nevada pilot region is Evergreen
Needleleaf-dominated - exactly the forest type where the in-house model
held up well against CCI in the fire project (correlation 0.71-0.72) - so
once a CCI-based drought methodology works end to end, re-running this
same Sierra pilot with the in-house model as a second, comparative pass
would be a clean, well-isolated test of whether its quarterly resolution
improves on CCI's annual mosaics for drought timing specifically.

**Status: decided, proceeding with 2015-2020 CCI-only pilot.**

## 8. The artifact is not a 2010-specific or resolution problem

Checked the 2015-2020 pair (both ESA-rated "equally reliable", no 2010
involved) unfiltered: the same coherent, terrain-following banding is
still there. This rules out 2010 specifically as *the* cause (Section 6's
original hypothesis) - something more general is going on.

Tested whether coarser spatial aggregation fixes it, per ESA's own
guidance ("the magnitude of changes is more reliable at coarser
resolution") - aggregated the 2015/2020 pair to ~1km, ~5km, ~25km, and
~55km (matching SPEI's native 0.5 deg grid) using `grids.py`'s
`reproject_to_grid` (degrees, not metres - CRS is EPSG:4326, an early
attempt using metre cell sizes silently produced 0x0 output grids).
**Result: the artifact survives even at the 55km SPEI-matched
resolution** - adjacent cells sit at opposite extremes of the value range
even with only a handful of cells across the whole bbox.

This changes the diagnosis. Random, independent pixel noise should shrink
towards zero as more pixels are averaged into a cell (central limit
theorem) - a pattern that survives 100m to 55km aggregation unchanged is
evidence of **systematic bias**, not noise, most likely tied to terrain
itself (consistent sign within a given slope/aspect, so averaging more
pixels of the *same* terrain doesn't cancel it). CCI AGB is SAR-derived,
and radar layover/foreshortening/shadow on steep terrain, dependent on
slope and aspect relative to the satellite's fixed look geometry, is a
plausible physical mechanism - not confirmed, a hypothesis to test next.

**Next**: test for a direct slope/aspect correlation, reusing
`fire-impact-curves`' own validated approach (its README Section 26, SRTM
slope vs. implausible readings, already run successfully in this same
Sierra Nevada region) rather than inventing a new method.

## 9. Confirmed: terrain/radar-geometry artifact, not drought signal

Pulled SRTM slope and aspect for the Sierra bbox, compared against the
unfiltered 100m 2015-2020 loss map.

Raw pixel-level Pearson correlation is ~0 for both (0.0035 for slope) -
this does **not** mean no relationship. Two reasons: (1) correlation only
captures straight-line relationships, and the slope relationship is
hump-shaped, not monotonic; (2) a single pixel is dominated by noise, so a
pixel-level correlation is swamped by that noise - the same reason the
fire project computes uncertainty by fire, not by pixel.

Binned means (averaging out per-pixel noise within each slope/aspect
bucket) tell a clearer story:
- **Slope**: non-monotonic - loss rises through moderate slopes, then
  swings to apparent *gain* on the steepest slopes (40-50 deg).
- **Aspect**: clear directional split. East/northeast-facing slopes
  (30-150 deg) show strong apparent loss (up to +10 Mg/ha); west/
  southwest-facing slopes (210-330 deg) show the opposite, apparent gain
  (down to -5 Mg/ha).

A clean flip tied to compass direction has no obvious biological
explanation (no reason drought would kill more trees facing one direction
and favour growth facing the opposite one, across an entire mountain
range), but is exactly the signature expected from SAR radar
layover/foreshortening, a geometric distortion that depends on slope
direction relative to the satellite's fixed look angle. CCI AGB is
SAR-derived.

**Conclusion: this is very likely a terrain/radar-geometry artifact, not
a real drought-mortality signal**, specific to steep, mountainous terrain
like the Sierra Nevada. It does not necessarily affect flatter forested
regions - not yet tested. This is independent of, and more fundamental
than, the year-pair (Section 6) and resolution (Section 8) issues found
earlier - all three may be compounding in this particular pilot region.

**Status: pilot region choice (Sierra Nevada) now itself in question, not
just the CCI differencing method.**

## 10. Gentle-slope filter does not fix it either

Tested whether restricting to gentle terrain (slope < 12 deg, using the
same SRTM layer as Section 9) removes the artifact, on the logic that if
the effect is layover/foreshortening-driven, it should be much weaker on
flat ground. Kept 40.6% of the bbox. The coherent banding persisted
essentially unchanged in the retained pixels - gentle-slope filtering
alone is not a viable fix, at least not at this threshold. Narrowing
further would shrink the usable area more, with no evidence yet that a
stricter cutoff would clean it up rather than just leaving less data.

## 11. Testing Artio's in-house biomass model as an alternative

Given Section 9's conclusion, the open question became whether the
Sierra Nevada *site* is unsalvageable for any SAR/optical biomass product,
or whether it was CCI specifically. Decided to test Artio's in-house
quarterly biomass model (Section 6's deferred follow-on) now rather than
wait, since it's a direct, relatively cheap way to tell those two
explanations apart: same bbox, different underlying biomass product.

Chose **2017-Q1 vs. 2018-Q4** as the comparison pair, not 2016-Q1 as
originally planned - 2016-Q1's 6-month lookback window reaches back to
2015-12-02, inside Sentinel-2A's still-sparse early-coverage period. This
was flagged as a risk beforehand and confirmed: the 2016-Q1 job failed
outright with "No STAC items found for any AOI feature" after exhausting
both the direct AOI search and a bbox-grid fallback. 2017-Q1 ran without
that failure, consistent with 2017 being inside the model's confirmed
reliable window.

**Operational notes** (see also `[[artio-biomass-model-gotchas]]` in
memory for the reusable version of these): the Cloud Function request
body uses all-caps keys (`PROJECT_ASSESSMENT_ID`, `AOI_GS`,
`PIPELINE_COMMAND`, `MODEL_IDS` as a comma-joined string, `YEARS`/
`QUARTERS` as lists) - guessing at snake_case keys produced a `400` with
`PROJECT_ASSESSMENT_ID is required`. Stage 1 (data prep) and Stage 2
(inference) are separate Vertex AI jobs with different display-name
prefixes (`biomassagbd-stage1-...` / `biomassagbd-stage2-...`); checking
only the stage1 prefix will show `SUCCEEDED` while Stage 2 is still
queued or running, giving a false impression of completion. Stage 2 sat
in `JOB_STATE_PENDING` (no error) for roughly two days before running -
consistent with the known shared-GPU-queue congestion, not a failure.

**Result: a different artifact, not a cleaner signal.** The 2017-Q1 to
2018-Q4 AGB-loss map shows no terrain/aspect-correlated banding like
CCI - but it has its own problem: sharp, straight-line discontinuities in
value, confirmed (by overlaying each tile's reprojected footprint on the
same coordinate frame) to align exactly with MGRS tile boundaries (e.g.
the horizontal step at lat ~37.0, matching the `11SKB`/`11SLA`/`11SLB`
tile edge). This is a tile-mosaic stitching artifact - adjacent
Sentinel-2 tiles processed under different conditions (cloud cover,
atmospheric correction, composite date) producing a visible seam, not an
ecological signal.

A second, separate problem with this specific comparison: **Q1 (winter)
vs. Q4 (autumn) is a seasonal mismatch**. Winter Sentinel-2 imagery at
Sierra Nevada elevations can carry snow cover and very different sun
angles/shadows than Q4 imagery, which could itself produce a systematic,
non-mortality difference - not confirmed, but a real competing
hypothesis to the actual drought signal.

Checked for a per-tile quality/cloud-cover summary to help disentangle
this (`_metrics.json`, expected per `[[artio-biomass-model-gotchas]]`):
**none exists for this run** - `pipeline-state.json`'s `metrics` field is
an empty object (`{}`). No QC signal is available from the pipeline's own
output to check cloud or snow contamination directly.

**Status: inconclusive.** The in-house model avoids CCI's specific
terrain/radar artifact, but this particular comparison is confounded by
tile seams and a seasonal mismatch, with no QC data available to check
either. Does not yet answer whether the Sierra Nevada site itself is
salvageable - a same-quarter, year-over-year comparison (e.g. Q4 2017 vs.
Q4 2018, or Q1 2017 vs. Q1 2018) would remove the seasonal confound and
is the natural next test, not yet run.

## 12. Same-quarter rerun: tile seam is structural, not seasonal

Re-ran the in-house model comparison as Q4 2017 vs. Q4 2018 (same quarter,
both years), removing the Q1/Q4 seasonal mismatch from Section 11
entirely. The same tile-stitching seam appeared, in the same location
(aligned with the same MGRS tile boundary at lat ~37.0), essentially
unchanged from the Q1/Q4 version. **This rules out season as the cause -
the seam is structural**, most likely tied to the pilot bbox spanning two
UTM zones (10N/11N, visible in the tile set: `10SGG`, `11SKA`, `11SKB`,
`11SKV`, `11SLA`, `11SLB`, `11SLV`), which is exactly the kind of AOI
placement that triggers tile-stitching artifacts in any Sentinel-2-tiled
pipeline.

This means the two biomass products now each have a *confirmed, distinct*
artifact mechanism on this same bbox: CCI has a terrain/radar-geometry
artifact (Section 9), the in-house model has a tile-stitching seam
(this section) - different causes, same site. That convergence is itself
informative: it's more evidence the specific AOI, not just one data
source, is the problem.

**Follow-up test**: re-ran the in-house model on a smaller AOI chosen to
sit entirely inside a single MGRS tile (`11SLA`: lon -119.248 to -118.475,
lat 36.036 to 37.037; used lon -119.2 to -118.5, lat 36.05 to 36.95 with a
safety margin from the tile edges), covering part of the southern-central
Sierra within the documented bark-beetle die-off zone. Result pending.

In parallel, re-checked **CCI** (not the in-house model) on this same
single-tile sub-region, reusing the already-downloaded 2015/2020 mosaics
(no new pull needed). **The terrain-following artifact is still present**,
same fine-grained, ridge-following pattern as the full bbox. This is not
a contradictory result - CCI is not MGRS-tiled, so narrowing to one
Sentinel-2 tile was never going to address CCI's terrain/SAR-geometry
mechanism, only the in-house model's tile-seam mechanism. Reinforces
Section 9's conclusion (terrain-driven, not AOI-shape-driven) rather than
undermining it.

**Status**: tile-seam cause confirmed structural, not seasonal. Waiting
on the single-tile in-house model result to know whether that specific
fix works for the in-house model. CCI's terrain artifact confirmed
independent of AOI size/shape - a single-tile AOI will not rescue CCI as
a usable data source here regardless of how the in-house model result
turns out.

**Single-tile result (2026-11-17): the fix did not work.** The
single-tile AOI (fully inside `11SLA`'s own bounds) still pulled 6
distinct MGRS tiles in its Sentinel-2 composite (`11SKA`, `11SKB`,
`11SKV`, `11SLA`, `11SLB`, `11SLV` - the same set as the original
problematic run, minus `10SGG`), confirmed by listing the actual
composite filenames rather than trusting the tile *count* alone. The
pipeline's tile-selection logic evidently works from some
buffered/padded search area rather than strict AOI-polygon containment,
and Sentinel-2 tiles overlap at their edges by design - so even an AOI
placed centrally within one tile's bounds still pulls in all its
neighbors. Shrinking the AOI, as attempted, cannot produce a seam-free
result from this pipeline; a real fix would need either a pipeline-level
change (outside this project's control) or clipping to one tile's
footprint *after* a full multi-tile pull, accepting a smaller usable
area.

**Conclusion: the in-house model avenue for the Sierra Nevada site is
exhausted.** Two independent biomass products have now failed for this
specific site through two genuinely different mechanisms - CCI's
terrain/radar-geometry artifact (Sections 6-9, tied to the site's steep
terrain) and the in-house model's tile-stitching seam (Sections 11-12,
tied to the site's bbox straddling multiple Sentinel-2 tile boundaries,
itself a consequence of the site spanning two UTM zones). Both failures
trace back to properties of this specific site's geography, not to a
fundamental impossibility of building a drought curve at all - see the
options below.

**Next steps, not yet started**: (1) pull USFS Aerial Detection Survey
ground truth directly (already planned since Section 4, bypasses
satellite biomass differencing entirely); (2) check whether a published,
field-validated remote-sensing mortality map already exists for the
California 2012-2016 bark-beetle drought specifically, mirroring the
approach that worked for the storm pilot (Section 18's St. Peter et al.
dataset) rather than continuing to derive a signal from general-purpose
biomass products; (3) as a fallback, move the drought pilot to a
different, flatter, single-tile site - the grids/curves methodology
itself is reusable, only the pilot site/event would change.

## 13. Storm track started: wind dataset decided, first sanity check passed (with a caveat)

Decided the wind-intensity metric for storm: **ERA5 10m instantaneous wind
gust (`i10fg`)**, not IBTrACS. IBTrACS only covers tropical cyclones and
records sustained wind, not gust, which is what actually drives
windthrow; it would also miss extratropical windstorms entirely, the
dominant tree-damage mechanism outside the tropics. ERA5 gust is global,
hourly, and the basis of the closest existing precedent found (C3S's
European windstorm-footprint product, which defines a storm footprint as
max gust per cell over a 72-hour passage window). No published, validated
chain from ERA5 gust directly to a tree-mortality/windthrow curve was
found in the literature search below - this is new synthesis for this
project, not reuse of an existing method.

**Pilot event chosen**: Hurricane Michael (Oct 2018), Florida Panhandle -
Category 5 landfall, ~3M acres of documented timber damage (USFS Southern
Research Station, Florida Forest Service), falls inside both CCI's and
the in-house model's reliable windows, pine-dominated and relatively flat
terrain (avoids the SAR/terrain-artifact problem found in the Sierra
pilot).

**Access path, new for this project**: the gust variable is not available
via Earth Engine at all (its ERA5 catalog entries only carry mean u/v
wind components, and are dated through mid-2020 anyway) - it only exists
in the **ARCO-ERA5** public Zarr archive on GCS
(`gs://gcp-public-data-arco-era5`). Checked via `gsutil ls` (safe, no
memory risk) before touching `xarray`: the smaller, curated `co/` stores
(`single-level-reanalysis`, `single-level-forecast`, `single-level-surface`)
do **not** contain gust at all - only the full `ar/full_37-1h-0p25deg-
chunk-1.zarr-v3` archive (37 variables, decades of global hourly data)
has both `instantaneous_10m_wind_gust` and
`10m_wind_gust_since_previous_post_processing`. Opening this store
wholesale with `xr.open_zarr(..., consolidated=True)` crashed the
Workbench kernel outright. Fix: open just the one needed variable (plus
`latitude`/`longitude`/`time` coordinate arrays) directly via the
lower-level `zarr` library, bypassing xarray's whole-store metadata
parse entirely - confirmed this is lazy/safe, then sliced only a small
bbox x 3-day window before pulling any data into memory. Coordinate
conventions confirmed empirically: latitude descending (90 to -90),
longitude in 0-360 format (not -180/180), time as hours since
1900-01-01.

**Sanity check result**: pulled Florida Panhandle gust for Oct 9-11,
2018. Peak gust 37.7 m/s at **2018-10-10 15:00 UTC** - timing matches
Michael's documented landfall (~17:30 UTC same day near Mexico Beach)
closely, confirming the indexing/coordinate-conversion pipeline is
correct. **Magnitude does not match**: Michael's documented sustained
wind at landfall was ~69 m/s (155 mph, Category 5), well above the 37.7
m/s peak found here. Not a bug - a known, literature-flagged limitation:
ERA5's 0.25 deg (~28km) grid cannot resolve the tight eyewall of an
intense, compact tropical cyclone, so it systematically underestimates
peak TC intensity. Directly analogous to the CCI and in-house-model
artifacts: a real, documented data-source limitation that needs stating
explicitly, not something to quietly work around.

**Implication for curve-fitting, not yet acted on**: fitting a windthrow
curve directly against raw ERA5 gust would systematically underestimate
true wind intensity for compact, intense storms (hurricanes especially).
Options, not yet decided: bias-correct ERA5 gust against station/
best-track data (e.g. HURDAT2) for the most intense events, restrict the
storm pilot to weaker/larger-footprint events where ERA5 resolves
intensity reasonably, or explicitly scope the resulting curve as valid
only in ERA5's resolvable intensity range.

**Status**: wind dataset and pilot event decided, access path working,
timing-validated. Magnitude caveat flagged, not yet resolved.

## 14. Florida biomass pull: AOI placement and the logging/salvage confound

Pulled CCI AGB (2017 vs. 2019, bracketing Michael's Oct 2018 landfall) for
an initial Florida Panhandle AOI (lon -85.6 to -84.9, lat 30.0 to 30.6),
chosen from county names associated with documented timber damage
(Bay, Calhoun, Liberty, Jackson). Raw and aggregated (~1km, ~5km, ~25km)
loss maps showed only weak, diffuse signal with no coherent storm-damage
swath - unlike Sierra, the noise visibly shrank with aggregation rather
than persisting (evidence against a systematic terrain-type bias here,
consistent with Florida's flat terrain), but also no clear real signal
emerged.

**Checked AOI placement against actual damage data**: Florida Forest
Service classed Bay, Calhoun and Gulf counties as ~347,000 acres at 95%
("catastrophic") damage - the original AOI covered Bay and part of
Calhoun but missed Gulf County (just south of the original bbox) and
most of Liberty (just east). Picking counties from memory rather than
checking a damage map first was the issue. **Widened/shifted the AOI**
to lon -85.7 to -84.6, lat 29.6 to 30.6 to cover Bay, Gulf, Calhoun and
part of Liberty. Re-ran the same CCI pull/aggregation - result was
similarly weak and diffuse, not materially better. AOI placement was not
the main problem.

**Checked against Hansen `lossyear` instead** (reusing the same
cross-check method validated for drought, Section 6) - chosen because
Hansen is tuned to abrupt canopy-cover loss, a better match for sudden
wind damage than it was for drought's slower dieback. Result: 2018 shows
4.08% of the (v2) bbox with detected canopy loss, clearly above a
2016/2017 baseline of ~1.8-2.1%, and 2019 is higher still (5.03%).

**New confound, specific to storm (not previously relevant for drought
at this site): routine timber harvest.** This AOI is intensively managed
Southern pine plantation land, clearcut on short (15-25 year) rotations
regardless of storms - Hansen's `lossyear` cannot distinguish storm
damage from ordinary logging, and the 2018 loss map's shape (a dense,
branching, scattered network, not one contiguous swath) looks more
consistent with harvest patterns than a single storm's footprint.

**Why 2019 being higher than 2018 does not undermine the finding**:
plausibly real - Hansen's detection can lag damage into the following
year depending on post-storm imagery availability, and major hurricanes
typically trigger a wave of **salvage logging** the following year
(clearing storm-damaged timber before it rots/becomes a pest risk),
which itself counts as storm-attributable loss, just indirect. The
likely real picture: storm-attributable disturbance is elevated above
the ~2% baseline in *both* 2018 and 2019, not concentrated in a single
clean year - which also explains why the straight 2017-vs-2019 CCI
difference looked weak: the true signal is spread across two years and
layered on top of both Hansen's routine-harvest noise floor and CCI's
own known pixel-level noisiness (Section 6).

**Status**: real storm-attributable disturbance likely present (2018 and
2019 both ~2x the pre-storm baseline), but not cleanly isolable with a
single before/after year pair the way the Sierra drought pilot assumed
for CCI. Decision not yet made: whether to use 2017 vs. a later year
(e.g. 2020) to capture the full storm-plus-salvage window, or treat the
2018+2019 excess-over-baseline itself as the event's disturbance
footprint. Either way, the plantation-logging confound needs an explicit
control/placebo check (e.g. comparing against a nearby, undamaged
plantation region with the same land-use pattern) before trusting any
resulting curve, mirroring the discipline already planned in Section 1.

## 15. Placebo check confirms the Florida spike is storm-attributable, not routine logging

Ran the control/placebo check flagged as needed at the end of Section
14: pulled the same Hansen `lossyear` check for a nearby pine-plantation
region in Georgia (lon -83.5 to -82.8, lat 31.0 to 31.6 - Tifton/Valdosta
area), chosen for similar forest type and harvest practice but outside
Michael's damage path, rather than assuming the comparison would hold
without checking it.

**Result: clean pass.** The control region shows a flat ~1.6-2.1% canopy
loss baseline across 2016-2019, with no 2018/2019 elevation - if
anything, slightly *lower* in those years. This is the opposite of
Florida's pattern (4.08% in 2018, 5.03% in 2019, both roughly double its
own 2016/2017 baseline of ~1.8-2.1%).

This rules out the alternative explanation flagged in Section 14 (a
region-wide logging/market trend that happened to coincide with the
storm by chance) - since a same-forest-type, same-practice control region
shows no such spike, Florida's elevated 2018/2019 loss is very likely
genuinely storm-attributable (direct wind damage plus the salvage-logging
wave that typically follows a major hurricane), not routine harvest
noise.

**Status**: the "is this real storm damage or just logging noise"
question from Section 14 is resolved - it's real. The separate,
still-open question (which year pair or framing to use for a clean
before/after comparison, given the signal spans both 2018 and 2019) is
unchanged by this result and still needs deciding before building a
curve.

## 16. Year-pair decision: 2017 vs. 2020, not excess-over-baseline

Decided to use **2017 (before) vs. 2020 (after)** as the storm pilot's
biomass comparison pair, rather than trying to build a curve from the
2018+2019 excess-over-baseline Hansen signal (Section 14-15).

**Reasoning**: Hansen's `lossyear` only gives a binary per-pixel
disturbance flag, not a continuous biomass amount - useful for the
diagnostic/placebo checks just done, but translating "excess
disturbance-year fraction over baseline" into an actual loss-share
magnitude would mean inventing a storm-specific methodology, rather than
reusing the one already built (`loss share = (AGB_before - AGB_after) /
AGB_before`, logistic fit). A 2017-to-2020 CCI comparison captures the
full storm-plus-salvage disturbance window (both 2018's direct damage
and 2019's salvage logging) in one differencing step, using the existing
method unchanged.

There is direct precedent for this kind of tradeoff: Section 6 made the
same call for drought, accepting a less temporally precise comparison
(no true pre-drought baseline, settled for 2015-2020) over fighting for
perfect event isolation.

**Known tradeoff, not yet checked**: widening the window to three years
(2017-2020) increases the risk of unrelated disturbance - further
harvest/replanting cycles, anything else unrelated to Michael - creeping
into the "after" mosaic. Same class of contamination risk already
flagged for drought's 2010-2012 window (known issue 4). Deferred until
there's an actual curve to validate against, same as that drought-side
check remains deferred.

**Next step**: pull CCI AGB 2017 and 2020 for the Florida Panhandle AOI
and repeat the before/after loss-map method validated for drought.

## 17. CCI 2017-2020 shows a plausible signal, but cross-check against Hansen is inconclusive - pivoting to a validated external dataset

Ran the 2017-2020 CCI comparison decided in Section 16. Unlike the
2017-2019 version (Section 14), the loss pattern **strengthened rather
than washing out** under spatial aggregation (~1km to ~5km) - a
spatially coherent loss-dominated region emerged, the opposite behavior
from both the Sierra terrain artifact (persisted unchanged at all
scales) and the earlier Florida attempt (shrank to near-zero). On its
own, encouraging.

**Cross-checked against Hansen's 2018+2019 disturbance mask** (the same
one used for the placebo check, Section 14-15) to see if the two
independent signals spatially agreed. **They did not line up cleanly** -
Hansen's heaviest disturbance concentrates in the left-center of the
bbox, CCI's strongest apparent loss concentrates center-right, with only
partial overlap in between. Read as a genuine, if disappointing, result
rather than declaring the earlier aggregation result confirmed just
because it looked encouraging on its own.

**Decision: look for an existing, validated dataset rather than keep
deriving a storm-damage signal from general-purpose biomass products.**
Both CCI and the in-house model have now each hit multiple distinct
reliability problems across two pilot regions (Sierra terrain artifact,
in-house tile seam, this ambiguous CCI/Hansen mismatch) - each new check
has cost real effort for an inconclusive result. Searched for whether
Hurricane Michael specifically already has a validated forest-damage
product, since it's a well-studied, high-profile event.

**Found**: St. Peter et al. (2020), *Remote Sensing* 12, 1138 -
"Spatially Quantifying Forest Loss at Landscape Scale Following a Major
Storm Event." Sentinel-2-based basal area loss map at 10m resolution for
Hurricane Michael specifically, built from 248 pre-landfall field plots
and ground-truthed against 70 post-storm revisit plots. Hosted on Dryad
(doi:10.5061/dryad.905qfttgc) and Zenodo, ~7.5GB including the damage
raster, source Sentinel-2 mosaics, and field plot data. A materially
stronger ground truth than anything derived in-house from Hansen or CCI
differencing - peer-reviewed, field-validated, and specific to this
exact event rather than a general-purpose biomass product being
repurposed.

**Decision: use this dataset as the storm pilot's primary biomass-loss
layer** (not just a validation cross-check), rather than continuing to
fight CCI/in-house-model artifacts for this event. Basal area loss is a
reasonable, directly interpretable proxy for the loss-share metric this
project's curve-fitting method needs.

**Scope, explicitly flagged**: this remains a single-event pilot
(Hurricane Michael only), mirroring the drought pilot's single-event
scope (California 2012-2016). The curve-fitting method itself (leave-
one-event-out cross-validation) needs multiple events eventually -
single-event work now is pilot/validation-stage, consistent with how
the drought track has been scoped throughout, not an attempt to
generalize across all storms yet.

**Status**: dataset identified, not yet downloaded or integrated. Next
step: pull the Dryad/Zenodo package, confirm it covers this project's
Florida Panhandle AOI, and check its data format/structure before
deciding how to fold it into the existing curve-fitting pipeline.

## 18. St. Peter et al. dataset obtained and integrated - first clean storm signal

**Access path, several real obstacles, each resolved**:
- Dryad's direct download is behind an Anubis JavaScript proof-of-work
  anti-bot challenge - `wget`/`curl` can't solve it (requires executing
  JS). Fix: the same dataset is mirrored on **Zenodo**
  (record 8237469), which allows direct downloads. Found via the
  Zenodo API (`https://zenodo.org/api/records/<id>`) rather than
  guessing the web UI's download path, after a first guess 403'd.
- The archive bundles 4 files (7.5GB total) - only `Hurricane.gdb.zip`
  (851MB) is needed; `Sentinel_norm_mosaics.zip` (6.67GB) is the
  authors' own source imagery, not their output, and was skipped.
- **The raster layers inside the `.gdb` could not be read by
  `rasterio.open()` or by `gdal.OpenEx()` with the GDAL bundled in this
  project's own venv (GDAL 3.12.2)**, nor by a fresh `conda-forge` GDAL
  install (3.13.3, via `micromamba`, installed specifically to rule out
  a venv-specific limitation) - both returned `None`/failed until the
  actual bug was found: an earlier attempt used a path relative to the
  wrong working directory (confusingly, failing in a way that looked
  identical to "this GDAL build can't read FileGDB rasters at all").
  Once run with the correct path, `gdal.OpenEx(..., gdal.OF_RASTER)`
  **did** successfully list 4 raster subdatasets via the open-source
  `OpenFileGDB` driver - `BAA_Pre_Hurricane`, `BAA_Post_Hurricane`,
  `BAA_Diff_absolute_brnmsk`, `BAA_Diff_percent_brnmsk` (the "Diff"
  rasters are the authors' own precomputed loss layers - exactly what's
  needed, no further differencing required). Worth remembering for next
  time: a GDAL "can't read this format" failure should be checked
  against a plain path error first, before concluding the driver/library
  itself is the problem.
- Converted the two "Diff" subdatasets to plain GeoTIFFs with
  `gdal_translate` (via the same `micromamba` environment, since the
  OpenFileGDB raster driver worked there), after which they're readable
  normally by this project's own `rasterio` install - no further need
  for the extra environment once converted.
- **A separate, serious infrastructure problem surfaced during this
  work, unrelated to the dataset itself**: this notebook's Jupyter
  kernel was discovered to have been running under the `fire-impact-
  curves` project's Python venv and working directory this entire
  session (confirmed via `sys.executable` pointing at
  `~/fire-impact-curves/.venv/...`), not `climate_impact_curves`'s own
  venv - despite the notebook file itself being correctly located.
  `climate_impact_curves` had a venv on disk but no registered Jupyter
  kernel for it; one had to be created
  (`python -m ipykernel install --user --name climate_impact_curves`)
  and manually selected in the notebook's kernel dropdown. **Every
  relative-path operation in this notebook prior to the fix was
  silently reading/writing `fire-impact-curves/data/raw/`, not this
  project's own data folder** - worth checking the other notebooks
  (01-04) for the same issue, not yet done.
- Reading the full raster (`23040x16384` pixels) into memory at once
  crashed the kernel - same class of memory problem as earlier large-
  tile reads. Fixed by reprojecting the AOI bbox into the raster's own
  CRS (`ESRI:102039`, USA Contiguous Albers - not EPSG:4326 like
  everything else so far) and reading only a small windowed subset via
  `rasterio.windows.from_bounds`, the same technique already used for
  the ARCO-ERA5 pull (Section 13).
- First plot looked like a near-uniform dark mass with no visible
  structure - not a real result, just `imshow`'s default full min/max
  color scaling being crushed by the data's heavy right-skew (min 0,
  median 1, 95th percentile 77, max 422). Fixed by scaling the color
  range to the 95th percentile instead of the full range.

**Result: a clean, spatially coherent, physically plausible damage
signal** - heaviest basal-area loss concentrated in the western/coastal
part of the AOI and along the river corridor, tapering off toward the
east. This matches Hurricane Michael's actual track (landfall near
Mexico Beach, Bay County - the AOI's western/coastal side - weakening as
it continued northeast inland). This is a materially cleaner result than
anything obtained from CCI (Sections 6-9, 14, 17) or the in-house model
(Sections 11-12) across either pilot region in this project so far.

**Status**: dataset successfully integrated for the Florida AOI.

**Cross-check against Hansen passed (2026-11-19).** Converted the
authors' percent-loss subdataset (`BAA_Diff_percent_brnmsk`) to GeoTIFF
the same way as the absolute layer, reprojected it onto CCI's 100m grid
via `reproject_to_grid` (so it's directly comparable to `disturbed_1819`,
Section 14), and checked: mean St. Peter et al. percent loss inside
Hansen-detected 2018+2019 disturbed pixels is **37.2%**, versus **13.5%**
outside - a clear, large separation. This is the same independent
consistency check that was inconclusive for CCI in Section 17 (hotspots
didn't line up cleanly) - here it passes cleanly. Correlation with CCI's
own AGB loss is weak (0.14), which isn't a concern given CCI was already
established as unreliable for this region (Section 9, Section 17) - the
point of this check was agreement with Hansen, not with CCI.

**Next step**: pair this loss layer with the ERA5 wind-gust data
(Section 13, with its known intensity-underestimation caveat for compact
storms) to actually fit the storm pilot's curve.

## 19. Drought pilot unblocked: USFS ADS ground-truth mortality data found via an ArcGIS Online mirror

**Context**: both CCI (Section 9) and the in-house model (Section 12) are
exhausted for the Sierra Nevada site. Next step was always to get
independent ground-truth mortality data instead of deriving it from
general-purpose biomass products (Section 7, item 6).

**The live national USFS ADS feature service only goes back to 2021.**
Queried `EDW_InsectandDiseaseSurvey_01` (`apps.fs.usda.gov/ArcX/rest/
services/EDW/...`) for the pilot bbox and 2012-2016 - got zero features
back (`200`, empty `FeatureCollection`), which looked like "no data here"
but was a silent filter mismatch: the field name is lowercase
`survey_year`, not `SURVEY_YEAR` (confirmed via the layer's own `?f=json`
metadata). Fixed the field name, still zero features - checked via
`returnDistinctValues=true` on `survey_year` and found the live service
only covers `[2021, 2022, 2023, 2024, 2025]`. The 2012-2016 data isn't
in this service at all, not a query bug.

**Found the correct historical dataset, but its usual host is
unreachable.** The right record is CDFW BIOS **DS2783**, "Aerial Tree
Death Report - 1973-2019 - USFS," which should cover 2012-2016. Its
metadata pages and direct zip download all live on `filelib.wildlife.ca.gov`
/ `filelib.fgc.ca.gov` - both refused the connection outright (`ECONNREFUSED`),
from this session's own fetch tool *and* from a `curl` run directly on the
Workbench terminal with a browser user-agent. This is a network-level
block on that host (likely restricting traffic from cloud/datacenter IP
ranges), not a bot-detection/user-agent issue like Dryad's Anubis
challenge (Section 18) - no user-agent trick fixes it.

**Resolution: the same dataset is mirrored as a live ArcGIS Online
FeatureServer on a different, reachable host.** Found via web search for
other BIOS datasets' service-naming convention (`biosds<id>_fpu`) and
confirmed it holds DS2783:
`https://services2.arcgis.com/Uq9r85Potqm3MfRV/ArcGIS/rest/services/biosds2783_fpu/FeatureServer/0`.
Per-polygon fields include `RPT_YR`, `NO_TREES_TOTAL`, `TPA_Total`
(trees per acre), `DMG_TYPE1-3`, `SEVERITY1-3`, `DCA1-3` (damage causal
agent), `HOST1-3` (species), `FOR_TYPE1-3`, `ACRES`. Queried the pilot
bbox (lon -119.5 to -118.5, lat 35.5 to 37.5) for `RPT_YR` 2012-2016:
**9,916 features**, pulled in full via `resultOffset` pagination (server
caps each response below the full count).

**Causal-agent breakdown confirms this is genuinely the drought signal,
not contaminated by fire or other disturbance**: of 9,916 features,
9,885 (99.7%) are bark-beetle/drought-stress agents - western pine
beetle (3,027), mountain pine beetle (2,513), fir engraver (1,785),
explicit `"drought"` (1,103), Jeffrey pine beetle (924), and smaller
counts of other insects/disease. Only 31 features (0.3%) are tagged
`"Fire"`. No filtering needed to exclude fire contamination in this
window.

**Status**: ground-truth data obtained for the pilot bbox and years.
Next step: convert `NO_TREES_TOTAL`/`TPA_Total`/`ACRES` per polygon into
a continuous loss-share metric comparable across years and grid cells
(not yet solved - this is a real cost of this approach vs. St. Peter et
al.'s ready-made loss layer for storm, Section 18), then pair with the
existing SPEI drought-severity data to fit the curve. See notebook
`06_usfs_ads_pilot.ipynb` for the dead-end national-service query
followed by the working ArcGIS Online pull.

## 20. Pooled loss-share metric built and tested - confounded by forest-type/elevation, not drought; species-restricted version removes the confound but the pilot is too small to fit a curve

**Method chosen**: defined loss share per SPEI grid cell (the pilot bbox
sits on SPEI's native 0.5deg grid - 8 cells total, lon -119.25/-118.75 x
lat 35.75/36.25/36.75/37.25) as a severity-weighted damaged-area
fraction: `sum(ACRES_i x min(TPA_Total_i / 80, 1)) / forested_acres`,
where 80 TPA is a literature-based severe-mortality ceiling (UC
Cooperative Extension: 20-80 dead trees/acre in high-mortality Sierra
areas) and `forested_acres` comes from NLCD 2016 (classes 41/42/43), not
CCI - deliberately avoiding CCI's known terrain artifact (Section 9) in
the denominator. A deliberate pivot from the original fire-impact-curves
definition (a true biomass fraction) since ADS gives discrete tree
counts/severity classes, not a continuous biomass value - flagged at
decision time, not discovered as a problem later.

**First result (pooled across all species) went the wrong direction.**
Plotted against mean 2012-2016 SPEI-12 per cell (7 of 8 cells usable;
one had near-zero NLCD forest cover, excluded to avoid a divide-by-zero):
loss share *increased* as SPEI became less negative (less dry) - the
opposite of what a drought-mortality relationship should show. Sharpest
evidence of a problem: two adjacent cells at lat 37.25 with nearly
identical SPEI (-0.722 vs -0.701) had a 25x difference in loss share
(0.52 vs 0.02).

**Diagnosed via species composition (`HOST1`) - confirmed confound, not
a bug.** The high-loss cell is dominated by ponderosa pine (1,060
polygons, 107,100 of its damaged acres) and other low/mid-elevation
conifers; the low-loss cell is dominated by whitebark pine, lodgepole
pine, and California red fir - high-elevation subalpine species with
much lower per-polygon `TPA_Total` regardless of drought severity. A
0.5deg SPEI cell can straddle a large elevation/forest-type gradient that
the climate grid can't resolve, and which forest type dominates which
cell is itself confounded with SPEI in this small 7-cell sample. This
explains the wrong-direction pooled result without needing to distrust
the underlying ADS data or the aggregation arithmetic (both checked and
correct).

**Species-restricted version (ponderosa pine only) removes most of the
confound, but the pilot is too small to fit a curve from what's left.**
Switched to mean `TPA_Total` (already a per-acre density, so no acreage
denominator - and no LANDFIRE species-specific vegetation-type lookup -
needed) for `HOST1 == "ponderosa pine"` polygons only, aggregated to the
same grid cells. One cell had zero ponderosa polygons at all (the driest
cell, SPEI -0.957 - plausibly elevation/moisture-band driven, ponderosa
doesn't grow everywhere in this bbox) and one had only a single polygon
(excluded as unreliable). The remaining 5 cells:

| SPEI | mean TPA (ponderosa) |
|---|---|
| -0.856 | 21.98 |
| -0.792 | 26.35 |
| -0.722 | 25.94 |
| -0.637 | 24.73 |
| -0.453 | 23.26 |

Mortality intensity now sits in a narrow band (~22-26 TPA) instead of
swinging 30x like the pooled metric - confirms species/elevation
composition was the main driver of the earlier wrong-direction result,
not a flaw in the data or method. But the relationship across these 5
points is not monotonic, and the SPEI range itself is narrow (-0.45 to
-0.96). Plausible reading: within this bbox, drought severity was
sufficient almost everywhere to push ponderosa mortality toward a
saturation/epidemic response (bark-beetle outbreaks can become
self-sustaining once established, killing at a fairly consistent rate
somewhat independent of further small SPEI differences) - a real
possible ecological finding, but not distinguishable from small-sample
noise with only 5 points.

**Conclusion: data quality and method are sound, but this single event
and bbox do not contain enough SPEI range or grid cells to fit an actual
dose-response curve.** Getting a real curve shape needs either a wider
AOI (more SPEI cells, more spread) or more drought events - both beyond
this pilot's current scope. Not a dead end in the same sense as Sections
6-12 (CCI/in-house model) - the ground-truth data itself works and the
confound is understood - but a sample-size limitation, not yet resolved.

## 21. Storm curve attempt: ERA5 gust and the St. Peter et al. loss layer point in different spatial directions

**Method**: paired the St. Peter et al. percent-loss layer (Section 18,
cross-checked against Hansen and confirmed directionally correct) with
ERA5 instantaneous 10m gust (Section 13) on ERA5's native 0.25deg grid
over the Florida v2 AOI - 16 cells (4x4), a better sample size than
drought's 8. Peak gust per cell computed over the 2018-10-09/11 landfall
window; loss aggregated to the same cells via direct array slicing
(GDAL's `reproject_to_grid` returned all-NaN at this extreme
downsampling ratio - ~10m source pixels into 28km destination cells -
for reasons not fully diagnosed; switched to manual inverse-affine
array slicing instead, which worked and is more transparent for this
kind of extreme-ratio aggregation anyway).

**Result goes backwards, and the spatial pattern explains why.** The
two highest-gust cells (34.8, 32.8 m/s - closest to actual landfall)
show the *lowest* loss (5.6%, 4.3%); several lower-gust cells (26-28
m/s) show the *highest* loss (up to 32.9%). This isn't a sign-convention
bug - the loss layer's directionality was already independently
validated against Hansen (Section 18: 37.2% mean loss inside
Hansen-disturbed pixels vs. 13.5% outside). It's a spatial mismatch:
Section 18 already documented the heaviest loss concentrated in the
AOI's western/coastal side, tapering off toward the east; this ERA5
gust grid shows peak gust increasing toward the *south-east* - opposite
spatial gradients.

**Consistent with ERA5's already-known limitation (Section 13), extended
from magnitude to location.** Section 13 established ERA5's 0.25deg grid
underestimates Michael's peak gust *magnitude* for this compact, intense
storm. This result suggests the same coarseness also *mislocates* the
wind field relative to where real damage occurred, not just softens its
peak value - a reanalysis grid this coarse can't resolve a compact TC's
eyewall position, not just its intensity.

**Conclusion: fitting a curve against this gust grid as-is would fit
misalignment, not a real intensity-damage relationship - not done.**
Next step: replace the raw ERA5 gust grid with a distance-to-track
proxy (using Michael's actual best-track position, e.g. from
IBTrACS/HURDAT2, with a simple radial decay from the track), which ties
the intensity metric directly to the storm's real physical position
rather than a coarse reanalysis grid.

## 22. Distance-to-track proxy recovers a real, physically sensible signal

**Pulled Michael's actual best track from IBTrACS** (NOAA NA-basin CSV,
filtered to `NAME == "MICHAEL"`, `SEASON == 2018`). Landfall point in
the track data (2018-10-10 17:30 UTC, 30.0N/85.5W, 140kt) matches the
documented Mexico Beach landfall almost exactly, and the peak recorded
wind (140kt ~ 72 m/s) lines up with the ~69 m/s sustained-wind figure
cited earlier (Section 13) - good confirmation the track data itself is
trustworthy.

**Computed distance-to-track per pixel on the CCI 100m grid** (haversine
distance from each pixel to the nearest of Michael's track points),
paired against the St. Peter et al. percent-loss layer. First attempt
gave a nonsensical result (distances up to 16,204km, near-antipodal -
impossible for a small Florida AOI): traced to a stale-variable bug, not
a real finding - `dst_transform` had been left pointing at the 4x4 ERA5
grid's transform from Section 21's aggregation step, and was
accidentally reused here for the much larger CCI grid, extrapolating
pixel coordinates wildly beyond that transform's valid domain. Fixed by
assigning the CCI grid's own transform to a distinct variable name
rather than reusing `dst_transform` across unrelated grids - a reminder
to use fresh names per grid/resolution.

**Corrected result is a real, physically sensible signal.** Distance
range (0-98km) is now the right order of magnitude for this AOI. Mean
percent loss by distance bin:

| Distance bin (near to far) | Mean % loss |
|---|---|
| 1 (closest) | 24.7 |
| 2 | 23.6 |
| 3 | 27.7 |
| 4 | 25.3 |
| 5 | 19.5 |
| 6 | 17.4 |
| 7 | 12.2 |
| 8 | 7.8 |
| 9 | 5.5 |
| 10 (farthest) | 2.7 |

Loss stays roughly flat (~20-28%) within the first ~30-40km of the
track - plausibly a near-eyewall saturation effect, similar in kind to
the saturation hypothesis raised for drought in Section 20 - then falls
off monotonically and substantially from there, dropping to under 3% at
the farthest bin. This is the shape a storm-damage curve should have,
and it's what ERA5 gust failed to show (Section 21).

**Status**: the distance-to-track proxy works where ERA5 gust didn't.

## 23. Storm pilot curve fitted - first completed curve in this project

**Method**: aggregated per-pixel loss into 12 fixed-width distance bins
(0 to ~90km, matching the AOI's actual distance-to-track range),
computed each bin's mean percent loss and pixel count, then fit a
logistic decay function -
`loss(d) = L_min + (L_max - L_min) / (1 + exp((d - d0) / k))` - to the
12 binned points via `scipy.optimize.curve_fit`. Fitting on bin-level
means (not raw pixels) keeps this consistent with how the drought curve
was built (grid-cell-level points, not pixel-level noise).

**Result**: `L_max = 26.04`, `L_min = 2.24`, `d0 = 49.86` km,
`k = 9.43` km, **R-squared = 0.968** on the 12 binned points. Loss
plateaus around 22-28% within roughly the first 30km of the track
(plausibly near-eyewall saturation, the same kind of effect raised for
drought in Section 20), falls off steeply between about 30-70km, and
levels out near 2-3% beyond 70km. One bin at the far tail (82km, only
2,009 pixels versus 20,000-94,000 in every other bin) sits slightly
above its neighbour - a small-sample edge effect at the AOI boundary,
not a real reversal, and doesn't meaningfully affect the fit.

**This is the first completed loss-vs-intensity curve in this project**,
for either hazard. Caveats to carry forward, all already flagged
elsewhere: single-event scope (Hurricane Michael only, Section 7 item
11); the intensity metric is distance-to-track, not wind speed directly
- a reasonable, physically interpretable proxy for this storm, but not
yet converted to an estimated wind speed via a parametric model (e.g.
Holland 1980), which would be needed to generalise this curve to other
storms with different wind-field profiles; and the loss layer itself
(St. Peter et al.) is single-storm-specific by construction, built only
for Michael, so applying this exact curve to a different storm isn't
valid without new ground-truth loss data for that event.

## Related work (literature review, 2026-11-17)

Checked for existing published curves/functions before building our own,
per request. **No existing work is directly usable as a drop-in curve
for either hazard** - summarized per-hazard below; full citations in
References.

**Drought**: Greenwood et al. (2017) (already cited below) gives a
cross-biome meta-analysis slope (log-mortality increases ~0.46 per SPEI
unit), consistent across biomes and between gymnosperms/angiosperms -
useful as a plausibility benchmark for a fitted curve's slope, not a
ready biomass-loss function, and not tied to remote-sensing inputs.
Subedi et al. (2021, East Texas) pairs FIA inventory data with a 12-month
SPEI threshold of -1.0 and reports biomass loss directly, closer in
structure to this project's output, but specific to East Texas plots, not
directly transferable to a Sierra Nevada conifer forest or to
satellite-derived AGB inputs. No one has published a direct
SPEI-to-biomass-loss-fraction function.

**Storm**: Anyomi et al. (2017) used the ForestGALES_BC mechanistic model
to simulate windthrow across wind speeds in boreal Ontario (~0-20% damage
at 5 m/s, 2-90% at 20 m/s, 10-100% at 40 m/s, varying strongly by stand
type). ForestGALES itself (Gardiner et al.) is the established
alternative approach in this field - physics-based (tree size/allometry/
wind-loading), fundamentally different from this project's empirical
curve-fitting method, not a function to import. The functional form of
empirical storm-damage curves is itself unsettled in the literature (one
source notes a reviewer explicitly challenged a sigmoid damage function
as unjustified) - this project's curve-shape choice is not obviously
wrong by not matching an established convention, but isn't backed by
consensus either.

**Conclusion**: the literature confirms the general approach (empirical
intensity-to-loss curve, fit per event) is reasonable and gives benchmark
ranges to sanity-check fitted curves against later, but building the
actual curves - tied to this project's specific data sources, forest
types, and intensity metrics - is genuine synthesis, not something
already done elsewhere.

## 7. Known issues and open questions

1. 2010 AGB baseline reliability (Section 6) - ESA's own documentation
   flags 2010 specifically as unreliable for trend/change comparisons
   against later years, and it's the only pre-drought mosaic available.
   Unresolved, blocks everything downstream until settled.
2. Event definition (SPEI threshold/duration, Section 4) not independently
   validated.
3. Before/after year choice constrained by mosaic availability gaps
   (2011-2014 missing), not freely chosen (Section 5).
4. 2010-2012 disturbance contamination in the pilot bbox not yet checked
   (Section 5).
5. Storm/wind dataset decided (ERA5 gust, Section 13) and timing-validated
   against Hurricane Michael, but ERA5's ~28km grid systematically
   underestimates peak intensity for compact, intense storms. Extended
   in Section 21: pairing ERA5 gust against the validated St. Peter et
   al. loss layer went backwards (highest-gust cells showed the lowest
   loss), and the spatial pattern shows why - ERA5's peak gust and the
   documented damage gradient point in different directions, not just
   different magnitudes. A distance-to-track proxy (IBTrACS/HURDAT2
   position + radial decay) is under consideration as a replacement -
   not yet built.
6. USFS Aerial Detection Survey ground-truth data obtained for the
   pilot bbox/years via an ArcGIS Online mirror of CDFW BIOS DS2783
   (Section 19) - 9,916 features, 99.7% bark-beetle/drought-attributable.
   A loss-share metric was built and tested (Section 20): pooled across
   species it went the wrong direction (forest-type/elevation confound,
   diagnosed and confirmed); restricted to ponderosa pine alone the
   confound mostly disappears, but only 5 usable grid cells with a
   narrow SPEI range remain - not enough to fit an actual curve yet.
   Needs a wider AOI or more drought events, not yet done.
7. In-house biomass model's tile-stitching seam (Section 11) is confirmed
   structural, not a seasonal artifact (Section 12) - tied to the pilot
   bbox spanning two UTM zones. A single-MGRS-tile rerun is in progress to
   check whether a smaller AOI fixes it; result not yet in.
8. No `_metrics.json` or equivalent QC output exists for the in-house
   model runs so far, despite being expected - worth confirming with
   whoever maintains the pipeline whether this is version-dependent or
   this run's configuration specifically omitted it.
9. CCI's terrain/radar artifact (Section 9) is confirmed present even in
   a single-tile sub-region (Section 12) - ruling out AOI size/shape as a
   fix for CCI specifically, independent of how the in-house model's
   single-tile test turns out.
10. Storm pilot pivoted to St. Peter et al. (2020)'s validated Sentinel-2
    basal-area-loss product, now obtained and integrated (Section 18) -
    gives a clean, physically plausible signal for the Florida AOI. The
    CCI 2017-2020 year-pair contamination question (previously item 10)
    is now moot - this dataset replaces that approach.
11. Storm pilot remains single-event (Hurricane Michael only), mirroring
    drought's single-event scope (Section 17) - the curve-fitting
    method's leave-one-event-out CV needs multiple events eventually;
    this is pilot/validation-stage work, not a general storm model yet.
12. **Notebook 05's Jupyter kernel was found (Section 18) to have been
    running under `fire-impact-curves`'s venv and cwd for the entire
    session**, not `climate_impact_curves`'s own - a registered kernel
    for this project's venv didn't exist until created during this
    session. Every relative-path file operation in notebook 05 before
    the fix silently touched `fire-impact-curves/data/raw/` instead of
    this project's own data folder. **Checked notebooks 01-04 for the
    same issue (cwd/`sys.executable` diagnostic) - all four confirmed
    correct**, running under `climate_impact_curves`'s own venv/cwd as
    expected. The drought-pilot findings in Sections 6-12 are unaffected
    - this was isolated to notebook 05.

## References

- Greenwood, S. et al. (2017). Tree mortality across biomes is promoted by
  drought intensity, lower wood density and higher specific leaf area.
  *Ecology Letters* 20, 539-553.
- Santoro, M., Cartus, O. (2025). ESA Biomass CCI: global datasets of forest
  above-ground biomass for 2007, 2010, 2015-2022, v6.0. NERC EDS CEDA.
  doi:10.5285/95913ffb6467447ca72c4e9d8cf30501.
- Beguería, S., Vicente-Serrano, S., Reig, F., Latorre, B. (2014). SPEI
  global 0.5deg monthly dataset.
- Subedi, S.C. et al. (2021). Tree mortality and biomass loss in
  drought-affected forests of East Texas, USA. *Journal of Forestry
  Research* 32(1), 67-80. doi:10.1007/s11676-020-01106-w.
- Anyomi, K.A. et al. (2017). Windthrow modelling in boreal Ontario using
  ForestGALES_BC.
- Gardiner, B. et al. (2016). Review of wind damage to forests.
  (ForestGALES mechanistic wind-damage model.)
- St. Peter, J. et al. (2020). Spatially Quantifying Forest Loss at
  Landscape Scale Following a Major Storm Event. *Remote Sensing* 12(7),
  1138. doi:10.3390/rs12071138. Data: Dryad,
  doi:10.5061/dryad.905qfttgc.
