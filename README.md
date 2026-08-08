# Biohub — Cell Tracking During Development

This is my work on Kaggle's [Biohub - Cell Tracking During Development](https://kaggle.com/competitions/biohub-cell-tracking-during-development) competition: detecting cells in 3D microscopy videos, tracking them frame to frame, and catching the moments they divide.

The videos come from time-lapse imaging of developing embryos. A single video can start from just a handful of cells and end with hundreds, all moving, deforming, and splitting. Right now, most of that tracking work still gets done by hand in labs, which doesn't scale. That's the problem this competition is trying to fix.

<p align="center">
  <img src="https://raw.githubusercontent.com/Md-Ali-Azad/Biohub---Cell-Tracking-During-Development/main/Live%20Visualization%20GT%20(2D%20and%203D)/cells_2d_over_time_t.gif" width="48%" alt="2D cell detections over time" />
  <img src="https://raw.githubusercontent.com/Md-Ali-Azad/Biohub---Cell-Tracking-During-Development/main/Live%20Visualization%20GT%20(2D%20and%203D)/cells_3d_over_time_t.gif" width="48%" alt="3D cell positions over time" />
</p>
<p align="center"><em>Ground-truth cells over time — 2D projection on the left, the same cells in full 3D on the right.</em></p>

## What the competition is asking for

Given a 3D+time microscopy video, the task is to output every cell's position in every frame, link the same cell across consecutive frames, and flag when one cell splits into two. Get all three right and you've reconstructed the full lineage of the embryo.

It's harder than it sounds. Cells sit close together, the image is noisy, and cells don't stay round and separate the way you'd hope. A lot of existing tools break down exactly in these conditions, which is the gap this competition is trying to measure.

Scoring combines two things:

```
score = adjusted_edge_jaccard + 0.1 × division_jaccard
```

Edge Jaccard checks whether the links between frames match the ground truth — predicted cells get matched to real ones by distance (within 7.0 µm, accounting for the fact that the microscope's z-resolution is about 4x coarser than its x/y resolution), and a link only counts if both ends match correctly. There's also a penalty if you predict way more cells than actually exist.

Division Jaccard checks specifically whether splits get caught: for each real division, does the prediction have a connected chunk of graph that covers the parent and reaches into both children?

Ground truth isn't fully annotated either. Not every visible cell has a label, so the metric is built to not punish you for that.

## What's in this repo

```
.
├── Live Visualization GT (2D and 3D)/
│   ├── cells_2d_over_time_t.gif
│   └── cells_3d_over_time_t.gif
│
├── Start with Visualization/
│   └── biohub-dataset-visualization.ipynb
│
├── azad-biohub-first-solution.ipynb
├── azad-biohub-second-solution.ipynb
└── azad-biohub-third-solution-watershed.ipynb
```

If you're new to the dataset, start with `biohub-dataset-visualization.ipynb`. It covers how the data's actually stored (paired `.zarr` volumes and `.geff` ground-truth graphs, not a single CSV like you might expect), what the raw images look like, and why a "cell" in these images isn't always the obvious bright blob you'd picture. I found this step more useful than I expected before writing any detection code.

The three solution notebooks are separate attempts, roughly in the order I tried them.

## Citation

Thibaut Goldsborough, Jordão Bragantini, Xiang Zhao, Gordon Leary, Teun Huijben, Ilan da Silva Theodoro, Kyle Harrington, Chi-Li Chiu, Walter Reade, María Cruz, and Loïc A. Royer. Biohub - Cell Tracking During Development. https://kaggle.com/competitions/biohub-cell-tracking-during-development, 2026. Kaggle.

## Methodology

Write-up for each solution below.

### `azad-biohub-first-solution.ipynb`


### `azad-biohub-second-solution.ipynb`


### `azad-biohub-third-solution-watershed.ipynb`
