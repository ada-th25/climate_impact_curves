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
5. Storm/wind dataset not yet identified.
6. USFS Aerial Detection Survey ground-truth comparison deliberately
   deferred, not yet pulled in.
7. In-house biomass model comparison (Section 11) is confounded by tile
   stitching seams and a Q1/Q4 seasonal mismatch, with no QC/metrics data
   available to check either - needs a same-quarter year-over-year rerun
   before it can answer whether the Sierra Nevada site is salvageable at
   all for a biomass-based drought signal.
8. No `_metrics.json` or equivalent QC output exists for the in-house
   model runs so far, despite being expected - worth confirming with
   whoever maintains the pipeline whether this is version-dependent or
   this run's configuration specifically omitted it.

## References

- Greenwood, S. et al. (2017). Tree mortality across biomes is promoted by
  drought intensity, lower wood density and higher specific leaf area.
  *Ecology Letters* 20, 539-553.
- Santoro, M., Cartus, O. (2025). ESA Biomass CCI: global datasets of forest
  above-ground biomass for 2007, 2010, 2015-2022, v6.0. NERC EDS CEDA.
  doi:10.5285/95913ffb6467447ca72c4e9d8cf30501.
- Beguería, S., Vicente-Serrano, S., Reig, F., Latorre, B. (2014). SPEI
  global 0.5deg monthly dataset.
