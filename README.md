# UltraCNN — Real-Time 3D Perception and Multi-Object Tracking

UltraCNN is a stereo-vision-based 3D perception pipeline for detecting, tracking, and localizing road users in KITTI driving sequences. It combines a KITTI-fine-tuned YOLO11n detector, ByteTrack multi-object tracking, and StereoSGBM depth estimation to associate approximate 3D positions with tracked objects over time.

## Pipeline

```text
KITTI stereo pair
   ├── Left image ──> Fine-tuned YOLO11n ──> ByteTrack IDs ──┐
   └── Right image ─> StereoSGBM ──> Disparity / Depth ──────┤
                                                             v
                                               Detection-level 3D position
                                                    (X, Y, Z)
                                                             |
                                                             v
                                               3D tracks and visualization
```

## Main components

- **Dataset:** KITTI Tracking stereo images, calibration files, and tracking labels.
- **Object detector:** YOLO11n fine-tuned on KITTI Tracking.
- **Classes:** `car`, `truck`, `pedestrian`, `cyclist`. Original KITTI `Car` and `Van` labels are merged into the `car` class for training/evaluation. `DontCare`, `Misc`, `Tram`, and `Person` are excluded from the four-class detector setup.
- **Multi-object tracking:** ByteTrack with the project configuration in `configs/bytetrack_tuned.yaml`.
- **Depth:** StereoSGBM disparity converted to depth using the sequence calibration focal length and stereo baseline.
- **3D localization:** Median valid depth within the central detection region; image coordinates are projected to camera coordinates using calibration intrinsics.

## Repository layout

```text
UltraCNN/
├── configs/
│   └── bytetrack_tuned.yaml
├── notebooks/
│   ├── 02_kitti_dataset_exploration.ipynb
│   ├── 03_object_detection.ipynb
│   ├── 04_3d_localization.ipynb
│   ├── 05_multi_object_tracking.ipynb
│   ├── 06_fine_tuning_YOLO.ipynb
│   └── 07_evaluation.ipynb
├── outputs/
├── runs/detect/kitti_yolo11n/weights/best.pt
├── requirements.txt
└── README.md
```

## Setup

1. Clone the repository and enter the project directory.
2. Install PyTorch separately using the installation command appropriate for your OS, GPU, and CUDA version. The project has been developed with CUDA-enabled PyTorch on an NVIDIA GPU; CPU execution is also supported by the tracking notebook but will be slower.
3. Install the remaining dependencies:

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
```

4. Place the KITTI Tracking dataset under:

```text
data/kitti/tracking/
├── data_tracking_image_2/training/image_02/
├── data_tracking_image_3/training/image_03/
├── data_tracking_calib/training/calib/
└── data_tracking_label_2/training/label_02/
```

5. Ensure the fine-tuned checkpoint exists at:

```text
runs/detect/kitti_yolo11n/weights/best.pt
```

6. Open Jupyter and execute the notebooks in order as needed. The end-to-end stereo tracking workflow is in `05_multi_object_tracking.ipynb`. Set `SEQUENCE_ID` in its first cell to select a sequence for processing.

## Outputs

The multi-object tracking notebook generates:

- `outputs/kitti_tracking_3d.mp4` — annotated sequence video.
- `outputs/kitti_<sequence>_3d_tracks.csv` — frame-level detections, track IDs, confidence, and estimated camera-frame `(X, Y, Z)`.
- `outputs/kitti_<sequence>_performance_summary.csv` — processed frames, detection count, unique track count, valid-depth coverage, elapsed time, and throughput.
- `outputs/sample_visualizations/` — annotated sample frames.
- X–Z trajectory plots for the longest-lived tracks.

## Reported detector validation results

The fine-tuned YOLO11n validation run previously recorded:

| Metric | Value |
|---|---:|
| Precision | 0.5362 |
| Recall | 0.4109 |
| mAP@0.50 | 0.4448 |
| mAP@0.50:0.95 | 0.2501 |

These are validation-set detector metrics, not tracking or 3D localization metrics. Per-class performance varies; the truck class was particularly weak in the recorded run. Do not present these values as official KITTI benchmark results.

## Evaluation and interpretation

The project includes detector validation and custom sequence-level tracking diagnostics. Any custom IoU-based precision/recall, IDF1-like calculations, ID-switch counts, or track-interruption counts must be clearly identified as project-specific diagnostics unless the official KITTI evaluation protocol and its ignore/difficulty rules are implemented.

Depth is estimated from stereo disparity. The 3D point currently represents a robust median point from valid pixels in the central part of a 2D detection, whereas KITTI tracking labels provide object location at the 3D box bottom centre. These are different reference points. A direct coordinate error comparison is therefore not a like-for-like metric without reference-point alignment.

## Limitations

- StereoSGBM depth can be noisy around object boundaries, reflective surfaces, occlusions, and low-texture regions.
- Detector class confusion and missed detections can cause track fragmentation or ID changes.
- A 2D tracker with depth attached to detections is not equivalent to a fully 3D motion-model tracker.
- Performance depends on image resolution, GPU, and inference settings.
- Results from a small number of sequences should not be generalized to all driving conditions.

## Reproducibility notes

- Keep KITTI data out of Git; it is large and subject to dataset terms.
- Keep the trained `best.pt` checkpoint if repository size permits, or provide a separate model download location.
- Record the exact sequence, hardware, PyTorch/CUDA version, and notebook settings when reporting FPS.
- Run notebooks from the project root or use the absolute project paths already defined in the notebook.

## License and dataset

KITTI data is provided under its own terms. Check the KITTI website and dataset terms before redistributing data or derived assets. Add an explicit license for the original project code before public release if you intend others to reuse it.
