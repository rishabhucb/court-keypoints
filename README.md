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

- **Day 4:** Homography demo: court keypoints → top-down court → players on a 2D map.
  - Pipeline: YOLO pose model (v4) finds court keypoints → `cv2.findHomography` maps image pixels to court coordinates in feet → standard YOLO person detector finds players → each player's feet are projected onto a 2D court diagram.
  - Measured the dataset's real court geometry from the labels (using the 4 paint corners as reference) instead of assuming it. Confirmed a high-school court (3-point arc ~19'9" from the hoop).
  - Found that the corner and half-court sideline points (IDs 0, 1, 2, 12) are unreliable: they are often off screen and labeled at the image edge (spread up to 10.8 ft). Excluded them.

  | Step | Mean court-mapping error |
  |---|---|
  | All 13 points, assumed positions | 24.4 ft |
  | Reliable points only (IDs 3–11), measured positions | **1.6 ft** |

  - OpenCV comparison: white-line mask + Hough transform found 104 lines, mostly on the scoreboard, stands and reflections, not on the court. Court lines in these gyms are dark or cream, not bright white. This shows why learned keypoints work better than classical line detection.
  - Caveat: test frames come from camera views seen in training (v4), and are similar moments of one game.
  - Figures saved in Google Drive: `court_keypoints/figures/`.

  - **3-point line + 2PT/3PT:** fitted the 3-point arc through the 5 labeled 3-point keypoints (IDs 7–11), drew it on the camera image, and classified each player's court position as 2PT, 3PT, or "too close to call" (within 1.5 ft of the line, our mapping error).
  - First version used the official high-school rule (19'9" arc) and drifted at the ends; fitting the arc to the dataset's own keypoints fixed it.
  - Limit: this classifies positions, not shots. Scoring a real shot needs ball/hoop detection and player tracking (other teams).


## Open questions
- Target court standard: NBA or high school?
- Half court only?
- Fixed tripod/phone camera?
- Is there a larger or more varied court keypoint dataset we can use?
- - Can we record our own footage at local courts to test on new gyms and camera angles?
