# Visualisation principles

Choose the **question, comparison and sample before the chart**. Use finding-led
titles only when supported; exploratory titles stay descriptive. Show overlap,
weak differences and uncertainty rather than selecting only striking results.
LaTeX.css controls typography, not scientific validity.

## References

| Reference | Use |
| --- | --- |
| [Data to Viz](https://www.data-to-viz.com/) | Chart selection, alternatives/pitfalls, heatmaps and dendrograms |
| [Data Visualisation Catalogue](https://datavizcatalogue.com/) | Visual forms by communication function/comparison |
| [Wilke — Fundamentals of Data Visualization](https://clauswilke.com/dataviz/) | Faithful, readable charts integrated into a narrative |
| [Wattenberg et al. — How to Use t-SNE Effectively](https://distill.pub/2016/misread-tsne/) | Parameter, apparent-size and between-group distance pitfalls |
| [scikit-learn — Silhouette analysis](https://scikit-learn.org/stable/auto_examples/cluster/plot_kmeans_silhouette_analysis.html) | K-Means diagnostics; adapt distance/method; not environmental validation |

The following are project design rules, not environmental findings.

## Common rules

- **Comparison:** favour position/length on common scales; use small multiples
  for overlapping series. Avoid 3D and distracting decoration.
- **Scales:** bars start at zero; line/point limits fit the question. Disclose
  transformations. Comparable panels share scales or clearly flag differences;
  no dual axes used to imply relationships.
- **Colour:** sequential=intensity; diverging=deviation from a justified reference;
  qualitative=categories. Cluster numbers are unordered. Use colour-vision-accessible
  palettes with labels/symbols; keep group colours consistent across figures.
- **Sample:** state period, units, station count and valid-day/year denominators.
  Show changing network composition. NA ≠ zero; do not automatically bridge gaps.
- **Bands:** distinguish between-station quantiles, temporal variability and
  estimate confidence intervals; name the represented quantity precisely.
- **Maps:** station-level data use points, not filled regions implying complete
  measurement. State coverage/aggregation; handle overlaps and small groups.
  Proximity does not establish a shared mechanism; stations do not automatically
  represent national territory/population.
- **Delivery:** English text, axis units, self-contained captions and meaningful
  Markdown alt text. Save code/derived tables; prefer SVG for vector charts.
  Check final-size readability, contrast, HTML and print.

## Clustering

Use a few question-led figures in the narrative; link full diagnostics.

| Question | Figure | Required qualifications |
| --- | --- | --- |
| What distinguishes groups? | Profile heatmap; variable points/intervals | Standardisation/reference, group sizes and spread; retain original-unit values. Means can hide heterogeneity. |
| How do they change through the year? | Monthly-profile/response-curve small multiples | Common scales/calendar, defined bands. Show predictor support for weather curves; no implicit extrapolation. |
| Where are they? | Categorical station map | Match profile colours; apply map rules above. |
| Which similarities were used? | Station × feature heatmap, optional dendrogram | Distance, transformations, linkage and cut. Dendrograms describe hierarchical methods, not K-Means; summarise unreadable 744-row plots and link detail. |
| How much do groups overlap? | Supporting PCA projection | Explained variance and actual clustering space. Two components do not establish full-dimensional separation/overlap. |
| How reliable are assignments? | Station/group silhouettes; resampling co-assignment matrix | Consistent distance; comparable samples; respect temporal/spatial dependence. Stability ≠ separation. |
| Do stations change group? | Transition matrix or assignment trajectories | Comparable features/samples, aligned labels and uncertain assignments. Independent-fit colours do not identify the same groups automatically. |

**t-SNE:** check multiple settings/initialisations; apparent sizes and inter-group
distances are not faithful original-space measures. No 2D projection replaces
profiles, diagnostics and stability checks; do not choose cluster count from
convincing “islands”.

Candidate sequence: **profiles → seasonality → map**, with linked diagnostics.
Report weak/unstable groups; consider continuous differences instead of clusters.

## Dataset constraints

Apply `DATA.md`: PM daily IQR may not represent hourly variability;
O₃ maximum ≠ MDA8; sparse SD describes imputation, not total/trend uncertainty;
shared weather is not independent replication. Separate network turnover from
within-station trends; pair dates for PM₂.₅/PM₁₀. Respect wind-label and
accumulation-timing restrictions when designing maps or temporal comparisons.

## Before adding a figure to RESULTS.md

1. Explicit question answered; surrounding text supported?
2. Sample, period, units, aggregation and missingness identifiable?
3. Scales, colours and bands consistent and defined?
4. Caption separates observations, estimates and interpretations?
5. Readable at final size; reproducible data/code linked?

Revise any unmet requirement before inclusion.
