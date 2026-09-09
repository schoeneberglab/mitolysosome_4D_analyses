# Mitolysosome 4D analyses

Scripts used for the 4D (3D + time) mitolysosome analyses in *[manuscript title,
authors, journal, year — fill in]*.

Cells expressing a mitochondrial marker (green) and a mitophagy reporter (red)
were imaged as volumes over time on a lattice light-sheet microscope. The two
notebooks here take the processed acquisitions and produce the per-cell
mitolysosome abundance and motility measurements reported in the paper.

## Pipeline

| Notebook | Does |
|----------|------|
| `step1-crop_image.ipynb` | Crops each hand-drawn cell ROI out of every timepoint, writing one single-cell movie per cell per channel. |
| `step2-mitolysosome_tracking.ipynb` | Detects and tracks mitolysosome puncta in the red channel of each single-cell movie, then reduces each cell to six features, figures and statistics. |

Run them in order — step 2 reads what step 1 writes.

## Installation

```bash
conda env create -f environment.yml
conda activate mitolysosome-4d
jupyter lab
```

`napari` needs a display. On a headless machine, set `VISUALIZE = False` in
step 2 and skip its "inspect a single cell" section.

## Input data layout

Step 1 expects one folder per sample, each containing one TIFF per timepoint per
channel plus an ImageJ ROI set (`*.zip`) with one rectangular ROI per cell:

```
<RAW_DATA_DIR>/<sample glob>/
├── ..._488nm_..._processed_....tif     # green: mitochondrial marker (deconvolved)
├── ..._560nm_..._no_decon_....tif      # red:   mitophagy reporter (not deconvolved)
└── rois.zip                            # one ROI per cell, drawn in Fiji
```

ROIs are drawn by hand in Fiji on a maximum projection. They are 2D, so the
axial extent of the crop is set once per experiment via `LOW_Z`/`HIGH_Z`.

Step 1 writes, and step 2 reads:

```
<SAVE_ROOT>/<EXPERIMENT_NAME>/<condition>/roi_id_<n>/{green,red}/frame_<t>.tif
```

ROI ids are numbered continuously within a condition, so cells from different
samples of the same condition never collide.

## Configuration

Each notebook has a single **Configuration** cell holding every path and
parameter; nothing else needs editing. The paths shipped here are placeholders
(`/path/to/...`) — point them at your own data. The values below are those used
for the manuscript.

**Step 1**

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `SAMPLE_GLOB` | `*1121*Pham*/Sample 1/*` | Which sample folders to process |
| `CHANNELS` | 488 nm → `green`, 560 nm → `red` | Filename pattern per channel |
| `LOW_Z`, `HIGH_Z` | 50, 220 | Slices retained in the crop |
| `CONDITION_NAMES` | 6 samples × 4 conditions | Condition label per sample, in `selected_paths` order |

**Step 2**

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `PIXEL_SIZE_UM` | 0.111 | Isotropic voxel size after deskewing |
| `FRAME_INTERVAL_S` | 11.3 | Time between volumes |
| `N_FRAMES` | 60 | Timepoints per movie |
| `DOG_PARAMS` | σ 3.0–6.0, threshold 0.015 | Difference-of-Gaussians detection; lower the threshold to detect dimmer puncta |
| `INTENSITY_RADIUS` | 4 px | Sphere over which a punctum's mean intensity is taken |
| `MAX_LINK_COST` | 800 px² | Largest squared displacement allowed between linked detections |
| `MAX_GAP` | 2 | Timepoints a punctum may go undetected without breaking its track |
| `MIN_TRACK_LENGTH`, `MAX_TRACK_LENGTH` | 4, 60 | Track length filter, in timepoints |

## Output

`all_features.csv`, in tidy form — one row per measurement:

| Column | |
|--------|--|
| `condition` | Treatment group |
| `cell_id` | Cell index within its condition |
| `feature` | Which of the six features below |
| `value` | The measurement |

| # | Feature | One row per |
|---|---------|-------------|
| 1 | Mitolysosome mean intensity | punctum |
| 2 | Mitolysosome count | timepoint |
| 3 | Normalized mitolysosome intensity | timepoint |
| 4 | Normalized mitolysosome count | timepoint |
| 5 | Net mitolysosome displacement (µm) | track |
| 6 | Average mitolysosome speed (µm/s) | track |

Features 3 and 4 divide the per-timepoint mitolysosome totals by the summed
intensity and the voxel count of the Otsu-thresholded green channel at that same
timepoint, so abundance is not confounded by how much mitochondrial mass a cell
contains. Features 5 and 6 are computed on length-filtered tracks only.

Step 2 also writes `all_features_histograms.png` (one panel per feature) and
`normalized_count_boxplot.png`, and prints two-sided Mann–Whitney U p-values
comparing each condition against `PGE2-activator`.

## Notes on reproducibility

* Step 1 reads a full stack once per timepoint per channel, several hundred MB
  each. Expect the run to be I/O-bound and test on one sample first.
* Step 2 orders frames by file creation time. Copying the step 1 output with a
  tool that does not preserve timestamps can scramble frame order — re-run step 1
  instead of copying.

## Citation

If you use this code, please cite *[manuscript reference — fill in]*.
