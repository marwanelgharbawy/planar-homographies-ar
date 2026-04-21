# AGENTS.md — Planar Homographies & AR Project

## Project Overview

This project implements an Augmented Reality (AR) application using **planar homographies** in a Jupyter notebook (`planar_homographies.ipynb`). The goal is to overlay frames from an AR source video (`ar_source.mov`) onto a computer-vision textbook that appears in a book video (`book.mov`), producing a composited output video.

### Data Files (`data/`)
| File | Description |
|------|-------------|
| `data/book.mov` | Input video showing a CV textbook |
| `data/ar_source.mov` | Source video to overlay onto the book cover |
| `data/cv_cover.jpg` | Static reference image of the book cover |

---

## Implementation Breakdown

The notebook is structured across **Parts 1.1 – 1.5**, each in its own cell group:

| Part | Title | Cells | Status |
|------|-------|-------|--------|
| 1.1 | Getting Correspondences | Cells 3–4 | ✅ |
| 1.2 | Compute Homography Parameters | Cells 5–7 | ✅ |
| 1.3 | Calculate Book Coordinates | Cells 8–9 | ✅ |
| 1.4 | Crop AR Video Frames | Cells 10–13 | ✅ |
| 1.5 | Overlay the First Frame | Cells 13–16 | ✅ |

### Key Variables Produced by Each Part

| Variable | Type | Produced In | Description |
|----------|------|-------------|-------------|
| `book_frame` | ndarray (RGB) | Cell 2 | First frame of book video |
| `ar_frame` | ndarray (RGB) | Cell 2 | First frame of AR source video |
| `book_cover` | ndarray (BGR) | Cell 2 | Book cover reference image |
| `h_book`, `w_book` | int | Cell 2 | Book cover dimensions |
| `h_ar`, `w_ar` | int | Cell 2 | AR video frame dimensions |
| `top_50_matches` | list | Cell 4 | Top 50 SIFT correspondences |
| `kp1`, `kp2` | list | Cell 4 | Keypoints for cover and video frame |
| `H` | ndarray (3×3) | Cell 7 | Homography matrix: cover → video frame |
| `corners_transformed` | list of [x,y] | Cell 9 | Book corners projected into video frame |
| `corners_dst` | ndarray int32 | Cell 9 | Integer corners for `cv2.polylines` / `fillConvexPoly` |
| `cropped_ar_frame` | ndarray (RGB) | Cell 13 | AR frame cropped to book aspect ratio, resized to cover size |

---

## Part 1.4 Logic (Cell 13)

The proper crop procedure:

1. Derive the **book's projected width and height** in the video frame from `corners_transformed` using Euclidean norms.
2. Compute the **book aspect ratio** (`book_w_in_vid / book_h_in_vid`).
3. Compare with the AR video aspect ratio and decide which axis to crop:
   - If the AR frame is **wider** than the book ratio → preserve full height, crop width centrally.
   - If the AR frame is **taller** → preserve full width, crop height centrally.
4. Call `crop_frame(ar_frame, crop_w, crop_h)` (defined in Cell 11) to extract the central region.
5. Resize the cropped result to `(w_book, h_book)` so it aligns pixel-for-pixel with the homography warp in Part 1.5.

---

## Plan Directory

All task plans produced by Atlas are stored in:

```
plans/
```

---

## Agent Guidance

### Oracle (Planner)
- Focus research on the notebook cells and the `data/` directory.
- The homography pipeline must remain intact; never alter Cells 2–12 or 14–18.
- The `crop_frame` helper (Cell 11) accepts `(frame, target_width, target_height)` and crops the **central region**.

### Sisyphus / Frontend-Engineer (Implementer)
- **DO NOT edit existing cells** (Cells 0–12, 14–18). Only add new cells or append to Cell 18.
- When inserting cells, place them immediately after the relevant markdown cell for that part.
- Variables like `H`, `corners_transformed`, `corners_dst`, `book_frame`, and `ar_frame` are notebook-global; use them freely.
- Always import `cv2`, `numpy`, and `matplotlib` via Cell 2; do not add duplicate imports.

### Code-Review (Reviewer)
- Verify that `cropped_ar_frame.shape == (h_book, w_book, 3)` before the overlay step.
- Confirm `corners_dst` is `dtype=np.int32` and has shape `(4, 2)` for `cv2.polylines` / `fillConvexPoly`.
- Check that the homography `H` maps **cover coordinates → video frame coordinates** (not the inverse).

---

## ⚠️ Assignment Constraints (HARD RULES — NEVER VIOLATE)

The following OpenCV built-ins are **banned** by the assignment. Any agent that introduces them will cause an automatic fail.

| Banned function | Reason | Allowed alternative |
|----------------|--------|---------------------|
| `cv2.findHomography` | Built-in homography solver | `compute_homography_parameters` + `ransac_homography` |
| `cv2.warpPerspective` | Built-in projective warp | `inverse_warp` (custom numpy) |
| `cv2.getPerspectiveTransform` | Built-in H from 4 points | `compute_homography_parameters` on 4 pts |

Custom implementations live in cell `6c2d2d58` (the warp-functions cell):
- `forward_warp(src, H, output_shape)` — must remain even if unused in the pipeline
- `inverse_warp(src, H, output_shape)` — used everywhere warping is needed
- `ransac_homography(src_pts, dst_pts, n_iters, threshold)` — manual RANSAC, calls `compute_homography_parameters` internally
- `is_valid_homography(H, frame_w, frame_h, w_book, h_book)` — geometry sanity check (convexity + area + bounds)

**Audio**: The output video `data/ar_output.mp4` must include audio from `ar_source.mov`. Audio is muxed using `ffmpeg` (via `subprocess`) after the video writer closes — no Python audio library needed.

---

## Coding Conventions

- **Language**: Python 3, inside a Jupyter notebook.
- **Image format**: All images are **RGB** when displayed with matplotlib; **BGR** when passed directly to OpenCV functions. Convert explicitly with `cv2.cvtColor`.
- **Homogeneous coordinates**: Normalise by the third component after any `np.dot(H, p)` call.
- **No magic numbers**: Derive crop dimensions from `corners_transformed`; do not hard-code pixel values.
