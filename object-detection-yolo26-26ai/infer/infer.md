# Real-Time Inference

## Introduction

A model on disk is not a detector yet. In this lab you run it on live input: images, a video file and a webcam. The script `scripts/infer.py` reads frames, runs YOLO26, draws the boxes, counts the objects of each class on every frame, and reports the frame rate it actually achieves. It can also write every detection to a file, which is what Lab 7 loads into the database.

It is short enough to read in one sitting, and it avoids three mistakes that are easy to make when you write this loop yourself:

* **Colour channels.** OpenCV delivers images as BGR (blue, green, red), and Ultralytics expects BGR for arrays, so the frames go to the model unchanged. Converting them "to RGB" first would make the model see swapped colours. The drawing colours in the script are BGR tuples too (green is `(0, 200, 0)`), so `mask` shows in green and `no_mask` in red.
* **Frame rate.** The FPS on screen is the wall-clock rate of the whole loop (reading, inference, drawing and display) averaged over the last 30 frames, not the time of the model alone. The model's own time is shown separately as `inf` in milliseconds.
* **Silent overlays.** The class counters are drawn on every frame, including frames where nothing is detected.

Estimated Time: 20 minutes

### Objectives

In this lab, you will:

* Run the detector on images and on a video
* Run it on a webcam
* Read the frame rate and the class counts it reports
* See how the confidence threshold changes what you detect
* Write detection events to a file

### Prerequisites

This lab assumes you have:

* Completed Lab 3, with the trained model at `runs/ppe_v1/weights/best.pt`

## Task 1: Detect on images

1. Run the detector on the test images and save annotated copies.

    ```bash
    <copy>
    python scripts/infer.py --weights runs/ppe_v1/weights/best.pt --source datasets/ppe_mask/test/images --save runs/annotated
    </copy>
    ```

2. Read the summary line. In the run used to write this lab, with `device=cpu`:

    ```text
    frames=15 wall=1.60s avg_fps=9.4 avg_inference_ms=38.0 detections={'mask': 78, 'no_mask': 1}
    ```

    The device is chosen automatically: an NVIDIA GPU first, then an Apple GPU, then the CPU. Add `--device cpu` to force one. The average frame rate over only 15 images includes the start-up of the model, so it is lower than the rate you see in a long video.

3. Open a few files in `runs/annotated`. Each box carries the class and the confidence, and the top-left corner shows `FPS`, the model time and one counter per class.

## Task 2: Detect on a video

You can use any `.mp4` file. If you do not have one, build a test video from the test images. Each image is shown for one second.

1. Create the video and run the detector on it. The second command saves the annotated result.

    ```bash
    <copy>
    python scripts/make_demo_video.py datasets/ppe_mask/test/images demo.mp4 --seconds-per-image 1
    python scripts/infer.py --weights runs/ppe_v1/weights/best.pt --source demo.mp4 --save out_demo.mp4
    </copy>
    ```

2. Compare the speed on your devices. The same 375-frame video gave these results on an Apple M3 Pro, running the detector with no display window:

    | Device | Average FPS | Model time per frame |
    | --- | --- | --- |
    | `--device mps` (Apple GPU) | 74.6 to 77.9 (two runs) | 5.1 to 5.3 ms |
    | `--device cpu` | 13.9 and 20.0 (two runs) | 44 to 66 ms |

    The CPU numbers vary a lot because other programs were using the processor at the time, and the model time is not the whole story: the reported FPS also includes reading, drawing and writing each frame. Measure your own machine. Real-time video needs about 25 to 30 FPS: the Apple GPU is far above that, while the CPU did not reach it in this run.

3. Open `out_demo.mp4`. The `FPS` counter is the number you would show to a user, and the counters per class change as each photo comes and goes.

## Task 3: Detect on a webcam

1. Connect a webcam, or use your laptop's, and run the detector with a window. Press **q** in the window to quit.

    ```bash
    <copy>
    python scripts/infer.py --weights runs/ppe_v1/weights/best.pt --source 0 --show
    </copy>
    ```

2. If it fails to open the camera, try `--source 1` for a second camera. On macOS, allow your terminal to use the camera in **System Settings**, **Privacy & Security**, **Camera**, and run the command again.

3. Add `--stop-after 30` to end after 30 seconds, and `--save webcam.mp4` to record the annotated stream. A webcam source needs `--show`, `--save` or `--events`, otherwise the output would be discarded and the script tells you so.

4. Test the model the way it will be used. Look at the camera with and without a mask, hold the mask below your nose, turn your head, and watch the confidence on each box. A model trained on 105 stock photos meets a real webcam for the first time here, and the gap between the two is worth seeing. If your webcam image is much darker or more blurred than the training photos, expect lower confidence and more misses.

## Task 4: Choose the confidence threshold

The model reports a confidence between 0 and 1 for every box, and `--conf` decides how sure it must be before the box is kept. The right value depends on what a mistake costs.

1. Run the test images at three thresholds.

    ```bash
    <copy>
    python scripts/infer.py --weights runs/ppe_v1/weights/best.pt --source datasets/ppe_mask/test/images --conf 0.25
    python scripts/infer.py --weights runs/ppe_v1/weights/best.pt --source datasets/ppe_mask/test/images --conf 0.4
    python scripts/infer.py --weights runs/ppe_v1/weights/best.pt --source datasets/ppe_mask/test/images --conf 0.6
    </copy>
    ```

2. Compare the detection counts. In the run used to write this lab, the 15 images contain 91 `mask` and 5 `no_mask` boxes, and the detector reported:

    | `--conf` | `mask` boxes | `no_mask` boxes |
    | --- | --- | --- |
    | 0.25 | 82 | 5 |
    | 0.4 | 78 | 1 |
    | 0.6 | 64 | 1 |

    Raising the threshold drops the uncertain boxes, and the first to go are the `no_mask` ones: the model is least sure about the class you most want to catch. At 0.4, the default, it reports one person without a mask in 15 images that contain five. For a safety application, a lower threshold such as 0.25 with a human checking the alerts is the safer choice. Lab 4 explains why the extra boxes are not all correct.

## Task 5: Write detection events

1. Run the detector with `--events` to write one JSON line per frame, and `--save-frames` to keep every fifth frame as an image. Lab 7 loads these files into the database.

    ```bash
    <copy>
    python scripts/infer.py --weights runs/ppe_v1/weights/best.pt --source demo.mp4 --save out_demo.mp4 --events events.jsonl --save-frames frames --frame-every 5
    </copy>
    ```

2. Look at the first line of `events.jsonl`.

    ```bash
    <copy>
    head -n 1 events.jsonl
    </copy>
    ```

    Each line has the time (UTC), the source, the frame number, the saved frame image if there is one, the frame size, and a list of detections with the class, the confidence and the box in pixels:

    ```json
    {"ts": "2026-10-21T03:26:18.455+00:00", "source": "demo.mp4", "frame_no": 0, "frame_file": "frames/demo_000000.jpg", "width": 640, "height": 640, "detections": [{"class": "mask", "confidence": 0.7018, "box": [300.4, 8.2, 372.2, 201.7]}, {"class": "mask", "confidence": 0.4989, "box": [187.7, 350.2, 221.6, 425.5]}]}
    ```

3. Count the lines and the saved frames. For the 375-frame demo video there are 375 events and 75 saved frames.

    ```bash
    <copy>
    wc -l events.jsonl
    ls frames | wc -l
    </copy>
    ```

You now have a detector that runs on live input, an honest frame rate, and a file of detections ready for the database.

You may now **proceed to the next lab**.

## Learn More

* [Ultralytics predict mode](https://docs.ultralytics.com/modes/predict/)
* [Ultralytics inference sources](https://docs.ultralytics.com/modes/predict/#inference-sources)

## Acknowledgements

* **Author** - Ignacio Martinez, Oracle
* **Last Updated By/Date** - Ignacio Martinez, October 2026
