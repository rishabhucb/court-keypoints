# court-keypoints
Basketball court keypoint detection for homography
# Court Keypoint Detection

Goal: detect basketball court keypoints and compute a homography
to map player and ball positions onto a top-down court.

## Dataset
- Source: Roboflow (forked), keypoint detection, half court
- 1 class (`basketball_court`), 13 keypoints per court
- 150 train / [X] valid / [Y] test images
- License: [check on Roboflow]

## Keypoint schema
| ID | Name |
|---|---|
| 0 | baseline_top_corner |
| 1 | baseline_bottom_corner |
| 2 | sideline_top_halfcourt |
| 3 | paint_top_left |
| 4 | paint_bottom_left |
| 5 | paint_bottom_right |
| 6 | paint_top_right |
| 7 | threept_arc_top_baseline |
| 8 | threept_arc_top_wing |
| 9 | threept_arc_apex |
| 10 | threept_arc_bottom_wing |
| 11 | threept_arc_bottom_baseline |
| 12 | sideline_bottom_halfcourt |

"Left" = baseline end of the paint, "right" = free-throw line end,
"top"/"bottom" = the two sidelines.

## Progress
- **Day 1:** Downloaded Roboflow keypoint dataset into Google Drive, set up Colab and GitHub.
- **Day 2:** Dataset audit.
  - Keypoint schema confirmed from COCO export and visual checks.
  - Paint and arc points (3–10) visible in 100% of train images; court corners less often (0: 60%, 1: 37%, 2: 58%).
  - Images are elevated high-school gym views, closer to a tripod setup than NBA broadcast.
  - `flip_idx` in data.yaml is incorrect; correct version is `[1, 0, 12, 4, 3, 6, 5, 11, 10, 9, 8, 7, 2]`. To fix before training.
  - Many images are frames from the same game videos; checking for train/test overlap.
-  **Day 3:** Data cleanup and baseline training (YOLO11n-pose).
  - Original Roboflow split leaked near-duplicate video frames: 86% of validation images had a near-twin in train.
  - Grouped near-duplicates with perceptual hashing: the dataset is effectively 2 camera views (two ends of a court) plus ~20 extra images.
  - Re-split so validation/test come from the view not used for training (`court_resplit`).
  - Fixed labels: off-screen keypoints had negative coordinates, causing YOLO to skip 23 of 27 test images. Marked them as not visible.
  - Fixed `flip_idx` in data.yaml.

  | Run | What changed | Test pose mAP50 | Lesson |
  |---|---|---|---|
  | v1 | 100 epochs, early stopping | 0 | Stopped before keypoints were learned |
  | v2 | 300 epochs | 0 (4 images) | Learned training view (train mAP 0.96); most test images skipped |
  | v3 | Fixed off-screen labels | 0 (27 images) | Does not transfer to an unseen camera view |
  | v4 | Both court ends in training | 0.995 (mAP50-95 0.63) | Works once the view is in training |

  **Takeaway:** the model works, but only on camera views it has seen. We need our own footage from more courts and angles. (v4 test is easy: same view, later frames, 11 images.)

## Open questions
- Target court standard: NBA or high school?
- Half court only?
- Fixed tripod/phone camera?
- Is there a larger or more varied court keypoint dataset we can use?
