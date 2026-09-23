# dls-pipeline

Quality control, replicate-aware statistics, and publication figures for dynamic light scattering (DLS) data exported from a Malvern Zetasizer.

![Z-average by group, SuperPlot style](examples/output/figures/z_average.png)

*Synthetic example. Small points are individual measurements; large points are batch means, which are what the statistics use.*

## Why

DLS analysis is usually done by hand: export, paste into a spreadsheet, delete the runs that look wrong, paste the rest into a stats program. Three things tend to go wrong along the way.

**Technical replicates get counted as n.** Three readings of the same cuvette are not three independent experiments. Treating them as if they were shrinks the error bars and makes differences look more significant than they are. Here the statistical unit is the independent batch, set explicitly in a sample sheet: records are averaged within each sample, samples (aliquots) within each batch, and only batch values enter the tests.

**Excluded runs leave no trace.** Every record gets a pass or fail status with the reason, against thresholds written in a config file. Nothing is dropped silently, and the summary lists every exclusion.

**Nobody can say later exactly what was run.** Each analysis writes a manifest with the input file hashes (SHA-256), the full configuration, and software versions, so a result can be traced back to its data and settings.

## What it does

1. Reads one or more Zetasizer exports (tab, comma, or semicolon separated; UTF-8 or UTF-16; decimal points or commas).
2. Joins each record to a sample sheet that says which group and batch it belongs to.
3. Flags records by PDI, correlogram intercept, cumulants fit error, and optionally count rate, then flags samples with too few valid records or poor agreement between technical replicates.
4. Averages up to batch level and compares groups with Welch's t-test (or a paired t-test when batches share starting material), reporting the difference with its confidence interval, raw and adjusted p-values (Holm by default).
5. Writes tables, SuperPlot-style figures (PNG at 300 dpi and editable SVG), a readable summary, and the run manifest.

## Quick start

Requires Python 3.11 or newer.

```bash
python3 -m pip install -r requirements.txt

# try it on the synthetic example
python3 examples/make_synthetic_data.py
python3 -m dls_pipeline --data examples/synthetic/zetasizer_export.txt \
                        --samples examples/synthetic/sample_sheet.csv \
                        --config examples/config.toml \
                        --out results
```

Open `results/summary.md` first.

## Using it on your own data

**1. Export from the Zetasizer software** as a text or CSV file, including at least Sample Name, Z-Average, and PdI. Intercept, fit error, and derived count rate enable the corresponding QC checks. If your column names aren't recognized, add them under `[columns]` in the config.

**2. Write a sample sheet** (CSV) with one row per sample name:

| sample | group | batch | aliquot | exclude | note |
|---|---|---|---|---|---|
| Cit-B1 | AuNP-citrate | B1 | | | |
| PEG5k-B2 | AuNP-PEG5k | B2 | | yes | bubble in cuvette |

`batch` is the independent replicate, for example a separate synthesis. Repeat records of the same sample name are technical replicates. `aliquot`, `exclude`, and `note` are optional; a manual exclusion is recorded with its note.

**3. Copy `config.example.toml` to `config.toml`** and set the group order, QC thresholds, and comparisons. Comparing each group against one control (`mode = "reference"`) instead of every pair means fewer tests and a lighter correction.

**4. Run it** with your files in place of the example paths.

Put your own exports in a folder called `data/` and write results to `results/`. Both are listed in `.gitignore`, so real data never gets pushed to GitHub by accident.

## Outputs

| File | Contents |
|---|---|
| `summary.md` | Plain-language overview: QC failures, exclusions, group summaries, comparisons |
| `qc_records.csv` | Every record with pass/fail and reasons, traceable to file and row |
| `qc_samples.csv` | Sample-level checks: valid record count, Z-average RSD |
| `batch_values.csv` | The batch means used in the statistics |
| `group_summary.csv` | n, mean, SD, SEM per group and metric |
| `comparisons.csv` | Test, n, difference, CI, t, df, raw and adjusted p |
| `figures/` | One SuperPlot per metric, PNG and SVG |
| `run_manifest.json` | Input hashes, full config, software versions, timestamp |

## Statistical notes

With three batches per group, any test has little power. A non-significant result is not evidence of no difference; look at the confidence interval. The output flags comparisons with fewer than three batches.

Welch's t-test is used rather than Student's because it doesn't assume equal variances, and costs almost nothing when variances are equal (Delacre et al. 2017). Multiple-comparison correction is applied separately for each metric across its comparisons.

Z-average and PDI come from the cumulants analysis and describe the intensity-weighted distribution. They are the ISO 22412 quantities, but for multimodal samples the size distribution tells a different story. Distribution analysis is not in this version.

## QC thresholds

The defaults (PDI ≤ 0.3, intercept 0.1–1.0, fit error ≤ 0.005, Z-average RSD ≤ 5%) are starting points. Check them against your instrument's quality guidance and your lab's SOP, and report the values you used; they are saved in every run manifest.

## Tests

```bash
python3 -m pip install -r requirements-dev.txt
python3 -m pytest
```

Test results are checked against independent references: scipy for the t-tests and confidence intervals, hand-worked values for the p-value corrections, and hand calculations for the aggregation, including a test that technical replicates never count as n.

## How this was built

TBC

## References

- Lord SJ, Velle KB, Mullins RD, Fritz-Laylin LK. SuperPlots: communicating reproducibility and variability in cell biology. *J Cell Biol* 2020, 219, e202001064.
- Lazic SE. The problem of pseudoreplication in neuroscientific studies: is it affecting your analysis? *BMC Neurosci* 2010, 11, 5.
- Delacre M, Lakens D, Leys C. Why psychologists should by default use Welch's t-test instead of Student's t-test. *Int Rev Soc Psychol* 2017, 30, 92–101.
- Holm S. A simple sequentially rejective multiple test procedure. *Scand J Stat* 1979, 6, 65–70.
- ISO 22412:2017. Particle size analysis — Dynamic light scattering (DLS).

## License

MIT
