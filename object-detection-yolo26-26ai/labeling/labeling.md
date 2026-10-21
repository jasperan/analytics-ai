# Label, Pre-Label and Check Your Data

## Introduction

A detector can only be as good as its labels. Most real projects spend more time on data than on training, and the mistakes are quiet: a swapped class number or a photo that appears in both the training and test sets does not raise an error, it just produces a model that looks fine and is not.

This lab covers the data side of the workflow. You choose a labeling tool, learn the labeling rules and the export traps that cause silent mistakes, see how a model can draft labels for you to correct (AI-assisted labeling), and run a script that checks a dataset before you train on it.

The dataset from Lab 1 is already labeled, so you can skip straight to Task 3 and still complete the workshop. Tasks 1, 2 and 4 are for when you add your own photos.

Estimated Time: 25 minutes

### Objectives

In this lab, you will:

* Choose a labeling tool that fits your situation
* Apply labeling rules that keep labels consistent
* Export labels in YOLO format without the common class-order and path mistakes
* Rename, merge or reorder classes by name with a script
* Check a dataset for label errors, duplicates and leakage before training
* Understand how AI-assisted labeling speeds up annotation

### Prerequisites

This lab assumes you have:

* Completed Lab 1, with your terminal in `workshops/yolo26_object_detection_26ai` and the environment active

## Task 1: Choose a labeling tool

You need a tool that draws boxes, assigns classes and exports YOLO text files. Two good options work with this workshop. Pick one.

| | RoboFlow (hosted) | Label Studio (open source) |
| --- | --- | --- |
| Where it runs | In your browser, on RoboFlow's servers | On your computer (or an OCI Compute instance) |
| Account | RoboFlow account required | Local account on your own instance, no third party |
| Cost | Free tier with a monthly credit allowance; some features need a paid plan | Free |
| AI assistance | Built-in assisted labeling (for example Smart Select, which uses the Segment Anything model) | Connect a machine learning backend that proposes boxes |
| Export | YOLO formats including the Ultralytics layout | YOLO export (a ZIP with labels and `classes.txt`) |
| Privacy | Check the workspace data-use setting before you upload photos of people | Your images stay on your machine |

> **Note:** OCI Data Labeling reached end of life on August 30, 2025. Oracle's documentation recommends moving to Label Studio, which is also available as an image in the OCI Marketplace.

**RoboFlow:** create a workspace and an *object detection* project at [roboflow.com](https://roboflow.com), upload images, draw boxes, then use **Versions** and **Export Dataset** with a YOLO format. RoboFlow's current plans, credits and options change often, so follow its [documentation](https://docs.roboflow.com/) for the exact screens.

**Label Studio:** run it in its own environment, because it pins different package versions than the rest of this workshop.

1. In a **new terminal**, create a separate environment, install and start Label Studio.

    ```bash
    <copy>
    python3.12 -m venv ~/labelstudio-venv
    source ~/labelstudio-venv/bin/activate
    pip install label-studio
    label-studio start
    </copy>
    ```

2. A browser window opens at `http://localhost:8080`. Create a local account with any email and password, then create a project and import your images.

3. Under **Settings**, then **Labeling Interface**, choose **Code** and paste this configuration. It defines the two classes of this workshop and tells Label Studio to draw rectangles.

    ```xml
    <copy>
    <View>
      <Image name="image" value="$image"/>
      <RectangleLabels name="label" toName="image">
        <Label value="mask" background="#00c800"/>
        <Label value="no_mask" background="#e60000"/>
      </RectangleLabels>
    </View>
    </copy>
    ```

4. Label your images, then use **Export** and choose **YOLO**. Label Studio's documentation states that YOLO and COCO exports **sort the class names alphabetically** unless you add a `category` attribute to each `<Label>`. For these two names, alphabetical order happens to match this workshop (`mask` is 0, `no_mask` is 1), but always read the exported `classes.txt` before you trust it.

## Task 2: Apply labeling rules

Two annotators will label the same image differently unless they follow the same rules. Write the rules down before you start, and keep them with the dataset. These are the rules the starter dataset needs:

* **One box per visible face**, tight around the face and the mask, including partly hidden faces if you can tell what they wear.
* **`mask`** means the nose and mouth are covered. **`no_mask`** means they are not, including a mask pulled down to the chin.
* **Do not label** faces that are too small or blurred to judge. A missing label for an unclear face is better than a guess.
* **Label every instance** in an image. An unlabeled person teaches the model that people without boxes exist.
* **Keep the split clean.** Decide train, validation and test membership *before* augmenting, and keep photos from the same video, or of the same person on the same day, in the same split. Otherwise the test result is flattered by near-copies of training images.

Privacy matters when the subjects are people. Photograph only people who agree to it, and read the data-use settings of any hosted tool before uploading.

## Task 3: Export, remap and the traps in between

Exports from labeling tools fail in three quiet ways. This task shows each one and the fix, using the dataset you already have. Nothing here needs a labeling tool.

1. **Class order.** Suppose your tool exported the classes alphabetically as `no_mask` then `mask`. Simulate that export by rewriting the dataset with the two classes swapped.

    ```bash
    <copy>
    python scripts/remap_classes.py datasets/ppe_mask/data.yaml datasets/swapped --names no_mask mask
    </copy>
    ```

    Compare the first label file of both datasets. The boxes are identical, but the class numbers are exchanged: `0` is now `no_mask`.

    ```bash
    <copy>
    f=$(ls datasets/swapped/train/labels | head -1)
    head -2 datasets/swapped/train/labels/$f
    head -2 datasets/ppe_mask/train/labels/$f
    </copy>
    ```

    ```text
    1 0.184766 0.369141 0.041406 0.066406
    1 0.438281 0.362891 0.117188 0.210156
    0 0.184766 0.369141 0.041406 0.066406
    0 0.438281 0.362891 0.117188 0.210156
    ```

2. **Fix it by name, not by number.** The tempting fix is to open `data.yaml` and type the class names in the order you want. That changes only the names: every box keeps its number, so every `no_mask` box becomes `mask` and the other way round, with no error. `remap_classes.py` looks up each source class by *name* and writes the labels again.

    ```bash
    <copy>
    python scripts/remap_classes.py datasets/swapped/data.yaml datasets/fixed --names mask no_mask
    diff -r datasets/ppe_mask/train/labels datasets/fixed/train/labels && echo identical
    </copy>
    ```

    The script prints how each class was mapped, and `diff` confirms that the round trip restored the original labels:

    ```text
    class remap: {'0:no_mask': '1:no_mask', '1:mask': '0:mask'}
    wrote datasets/fixed/data.yaml; boxes per class: {'mask': 805, 'no_mask': 148}
    identical
    ```

    The same script merges classes (`--map "No Mask=no_mask" --map "NoMask=no_mask"`) and drops classes (`--map "face="`, an empty target). Use it whenever you combine datasets from different sources.

3. **Paths.** `remap_classes.py` and `download_dataset.py` both write an absolute `path:` into `data.yaml`. If a tool gives you a `data.yaml` with relative paths such as `../train/images`, open it and replace the first lines with an absolute `path:` as shown in Lab 1, Task 5.

## Task 4: Check label quality

`check_labels.py` is a gate you run before every training. It reads `data.yaml` and checks every image and label file.

* **Errors (the script exits with a failure code):** missing label files, labels without an image, class numbers outside the class list, malformed lines, coordinates outside 0 to 1, and the same image appearing in more than one split.
* **Warnings:** boxes that are tiny, duplicate boxes on one image, images without any box, near-identical images in different splits (a perceptual hash catches resized or re-compressed copies), and a strong imbalance between classes.

1. Run it on the dataset you will train on.

    ```bash
    <copy>
    python scripts/check_labels.py datasets/ppe_mask/data.yaml
    </copy>
    ```

    ```text
    dataset: datasets/ppe_mask/data.yaml   classes: {0: 'mask', 1: 'no_mask'}
    split     images   bg        mask     no_mask
    train        105    0         572         123
    val           29    0         142          20
    test          15    0          91           5

    WARN  class imbalance: {'mask': 805, 'no_mask': 148} (ratio 5.4:1) - consider more data for the rare class and check per-class metrics

    0 error(s), 1 warning(s)
    ```

    No errors, and one warning you already know about. The warning is the reason Lab 4 reports metrics per class instead of a single number.

2. Now break a label on purpose, so you see what a failure looks like. This appends a box with class number 7 to one label file of the swapped copy from Task 3, then checks it.

    ```bash
    <copy>
    echo "7 0.5 0.5 0.2 0.2" >> datasets/swapped/train/labels/$f
    python scripts/check_labels.py datasets/swapped/data.yaml
    echo "exit code: $?"
    </copy>
    ```

    You should see `ERROR [train] 0002526673_jpg.txt:3: class id 7 outside 0..1` and a non-zero exit code. A training run would have started anyway and either crashed in the middle or silently ignored the box.

3. Remove the practice copies.

    ```bash
    <copy>
    rm -rf datasets/swapped datasets/fixed
    </copy>
    ```

> **Note:** The script finds *mistakes in the files*. It can not tell that a box carries the wrong class when the number is valid. For that, look at a sample of images with their boxes drawn and compare per-class counts with what you expect: a dataset of mostly masked faces that suddenly shows more `no_mask` than `mask` boxes has a swapped order.

## Task 5: Let a model draft your labels

Drawing every box by hand is slow. **AI-assisted labeling** lets a model propose boxes that you then accept or fix. Reviewing a good proposal takes seconds, drawing from scratch takes minutes. Four approaches exist, from the least to the most set-up effort.

* **Built-in assistants in the labeling tool.** RoboFlow offers assisted labeling features, and Label Studio can call a machine learning backend that proposes boxes. Use these for click-to-segment help on objects the assistant already understands.
* **A model you train yourself (bootstrapping).** Label a first few hundred images by hand, train a model (Lab 3), use it to pre-label the next batch, correct the mistakes, retrain. Each round needs less correction. `scripts/prelabel.py` does the pre-labeling step: it writes YOLO label files and a `review.csv` that sorts every box into a confidence band, so you spot-check confident boxes and review the uncertain ones.
* **Open-vocabulary detectors.** Models such as YOLO-World and YOLOE take a text prompt ("person wearing a safety helmet") and need no labeled data for common objects, which is a quick way to get the first batch. They work well for ordinary things and poorly for fine distinctions like a mask worn below the nose, so expect to correct a lot.
* **A multimodal language model.** A vision-capable model on OCI Generative AI can be asked to return boxes for an image. Treat the result as a draft: language models are less precise at box coordinates than detectors.

Whatever you use, a human must still review the result, and your review needs a measure. In Lab 4 you run `prelabel.py` with the model you train in this workshop and compare its drafts against the human labels of images the model never saw.

You now know how to choose a labeling tool and avoid the quiet traps, and you have a script that checks a dataset before it is used.

You may now **proceed to the next lab**.

## Learn More

* [Label Studio export formats](https://labelstud.io/guide/export.html)
* [Start Label Studio](https://labelstud.io/guide/start)
* [RoboFlow documentation](https://docs.roboflow.com/)
* [OCI service change announcements (Data Labeling end of life)](https://docs.oracle.com/en-us/iaas/Content/servicechanges.htm)
* [Ultralytics YOLO detection dataset format](https://docs.ultralytics.com/datasets/detect/)

## Acknowledgements

* **Author** - Ignacio Martinez, Oracle
* **Last Updated By/Date** - Ignacio Martinez, October 2026
