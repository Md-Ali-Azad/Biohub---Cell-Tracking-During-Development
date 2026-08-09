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

## Methodology

The obvious first instinct for a task like this is to reach for deep learning, and that's clearly the direction most of the competition is heading. I wanted to try the other path first: how far can plain image analysis actually get you here, without a GPU, without training data, without any of that cost? No neural nets in any of the three notebooks below, just Gaussian filtering, watershed segmentation, and distance-based matching. Partly this was a cost and speed thing. Also, I just wanted an honest answer to whether classical methods still hold up on real, messy microscopy data before deciding a learned model was actually necessary.
 
Three attempts so far, each one fixing a specific thing I didn't like about the last.
 
### `azad-biohub-first-solution.ipynb` — the baseline
 
This one is deliberately simple, and it doesn't do anything with deep learning. Detection is a multi-scale difference-of-Gaussians blob detector: normalize each frame, run three pairs of Gaussian blurs at different sigmas (tuned in physical microns, then converted to voxels using the z/y/x scale), subtract each pair, and keep whatever comes out as a local maximum above a threshold. Centroids get refined afterward with a quick weighted-average pass around each peak.
 
For linking, I used a straightforward Hungarian assignment between consecutive frames: predict each track's next position from its current velocity, build a cost matrix out of the scaled centroid distances, and solve it. There's a gap-closing step afterward that stitches together tracks separated by a missed detection or two, matching endpoints across a short time window instead of just adjacent frames.
 
The obvious gap here, and I knew it going in, is that this linker can only ever produce one edge per node. A cell that splits into two daughters has nowhere to put the second edge, so every division in the ground truth was guaranteed to score as a miss no matter how good the detection was. This version exists mostly to get a working end-to-end pipeline (detect → link → write submission.csv) and a number on the board before touching anything more complicated.
 
### `azad-biohub-second-solution.ipynb` — adding divisions
 
Same detector as the first pass. The change here is entirely in the linker. After the normal one-to-one Hungarian assignment runs, there's a second pass that looks at whatever detections in the next frame didn't get claimed by anyone, and checks if one of them sits close to a track that already has a match. If it does, and the two candidate daughters are roughly symmetric in their distance from the parent (not one right next to it and one far away, which usually means the second point is just an unrelated neighboring cell), it gets treated as a division and both daughters get an edge from the parent.
 
I also added a few bookkeeping functions on top: one that walks the finished graph looking for nodes with exactly two outgoing edges and tags them as divisions in the metadata, and one that prints out a quick summary (how many divisions got found, a sample of them with their timestamps) so I could sanity-check the output without opening the CSV by hand every time.
 
The CONFIG in this notebook is tuned a bit differently from the first one too — smaller `xy_downsample`, tighter division radius, wider symmetry tolerance — mostly from trial and error trying to cut down on false-positive divisions, since early on the division pass was a little too eager to call two nearby cells siblings when they were really just two cells that happened to be close.
 
### `azad-biohub-third-solution-watershed.ipynb` — dealing with touching cells
 
The peak-picking detector has a real weakness: when two cells sit close enough together that their brightness profiles overlap, the local-maximum step either merges them into a single peak or, worse, finds two peaks inside what's actually one blob. This notebook swaps that out for a marker-controlled watershed instead. The same DoG peaks are still computed, but now they're used as seed points for a watershed segmentation rather than being taken directly as the final cell centers, so the boundary between two touching cells gets resolved by where the intensity actually dips between them instead of by however NMS happened to break the tie. I also added per-z-block intensity normalization here, since the signal gets dimmer with depth in these volumes, and a single global brightness threshold was under-detecting cells in the deeper slices.
 
The linker got an upgrade to match: alongside centroid distance, the Hungarian cost now also factors in how close a candidate's brightness is to what the track has been showing. In a crowded region where two cells are nearly the same distance from a track's predicted position, this appearance term is often the only thing that tells them apart.
 
Both of these are behind config flags (`use_watershed`, `use_appearance_cost`), defaulting to on in this notebook but easy to flip off to fall back to the second solution's behavior for comparison.
 
One honest note: this notebook imports PyTorch and sets up a GPU check at the top, but there's no actual model being trained or run here yet; that import is left over from where I was planning to add a learned detector next. Everything in this version is still classical image processing (watershed, distance-based cost matrices), not deep learning.



## Citation

Thibaut Goldsborough, Jordão Bragantini, Xiang Zhao, Gordon Leary, Teun Huijben, Ilan da Silva Theodoro, Kyle Harrington, Chi-Li Chiu, Walter Reade, María Cruz, and Loïc A. Royer. Biohub - Cell Tracking During Development. https://kaggle.com/competitions/biohub-cell-tracking-during-development, 2026. Kaggle.
