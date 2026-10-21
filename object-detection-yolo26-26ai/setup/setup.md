# Set Up Your Workstation and Get the Dataset

## Introduction

Everything up to Lab 5 runs on your own computer. In this lab you get the workshop code, create a Python environment with the exact package versions the workshop was tested with, prove that YOLO26 runs, and download the dataset. The dataset comes from a public Hugging Face repository and needs no account or API key.

The dataset is *mask-wearing*, a set of 149 photos of people with and without a face mask, with a box drawn around each face. You treat it as a PPE (personal protective equipment) compliance check: "is this person wearing the required equipment?" The same steps work for helmets or safety vests.

Estimated Time: 20 minutes

### Objectives

In this lab, you will:

* Get the workshop scripts
* Create a Python 3.12 environment and install the pinned packages
* Check that YOLO26 runs on a sample image
* Download the dataset and convert it to the YOLO format
* Check the dataset layout and the `data.yaml` file

### Prerequisites

This lab assumes you have:

* A macOS or Linux terminal (on Windows, use WSL 2) with Python 3.12 and Git installed
* About 3 GB of free disk space

## Task 1: Get the workshop code

The scripts live in the `workshops/yolo26_object_detection_26ai` folder of the `ai-solutions` repository. A sparse checkout downloads only that folder.

1. Open a terminal in the folder where you keep projects and clone the folder.

    ```bash
    <copy>
    git clone --depth 1 --filter=blob:none --sparse https://github.com/oracle-devrel/ai-solutions.git
    cd ai-solutions
    git sparse-checkout set workshops/yolo26_object_detection_26ai
    cd workshops/yolo26_object_detection_26ai
    </copy>
    ```

2. List the folder. You should see `scripts`, `db`, `requirements.txt` and a `README.md`.

    ```bash
    <copy>
    ls
    </copy>
    ```

> **Note:** Every command in the next labs assumes your terminal is in this folder (`workshops/yolo26_object_detection_26ai`). If you open a new terminal, go back to it and activate the environment again.

## Task 2: Create the Python environment

A virtual environment keeps the workshop's packages separate from the rest of your system. The `requirements.txt` file pins the versions that were tested together, including `ultralytics` (the YOLO26 library) and `oracledb` (the Oracle Database client used in Lab 7).

1. Create and activate the environment, then install the packages. The download is large because it includes PyTorch, so allow a few minutes.

    ```bash
    <copy>
    python3.12 -m venv .venv
    source .venv/bin/activate
    pip install -r requirements.txt
    </copy>
    ```

2. Confirm the versions. You should see `ultralytics 8.4.183` and `torch 2.14.1`.

    ```bash
    <copy>
    python -c "import ultralytics, torch; print('ultralytics', ultralytics.__version__, 'torch', torch.__version__, 'cuda', torch.cuda.is_available(), 'mps', torch.backends.mps.is_available())"
    </copy>
    ```

    The last two values tell you which accelerator PyTorch can use: `cuda` is an NVIDIA GPU and `mps` is the GPU of an Apple silicon Mac. If both are `False`, training still works on the CPU, just slower, and Lab 3 tells you how to shorten it.

## Task 3: Check that YOLO26 runs

Before touching your own data, run the pretrained YOLO26 nano model on a sample photo. This confirms that the install works and downloads the pretrained weights once.

1. Run the detector on a public sample image.

    ```bash
    <copy>
    yolo predict model=yolo26n.pt source=https://ultralytics.com/images/bus.jpg device=cpu
    </copy>
    ```

2. Read the last lines of the output. For this sample image the model found `4 persons, 1 bus`, and it prints the folder where the annotated image was saved (`runs/detect/predict`). Open that image to see the boxes.

    This model was trained on the 80 everyday classes of the COCO dataset. It knows what a person and a bus are, but it has never heard of `mask` or `no_mask`. Fine-tuning teaches it.

## Task 4: Download the dataset

`download_dataset.py` downloads the dataset from a pinned revision on Hugging Face, verifies its SHA-256 checksum and converts it to the layout YOLO expects. It uses only the Python standard library.

1. Run the script.

    ```bash
    <copy>
    python scripts/download_dataset.py
    </copy>
    ```

2. Read the summary at the end. You should see 105 training, 29 validation and 15 test images:

    ```text
    split     images      mask   no_mask
    train        105       572       123
    valid         29       142        20
    test          15        91         5
    ```

    Two things in this table matter later. There are about **five times more `mask` boxes than `no_mask` boxes**, so the model sees far fewer examples of the class you probably care about most. And the test split holds only **5** `no_mask` boxes, so any number measured on it for that class is very uncertain.

3. Look at the files the script created in `datasets/ppe_mask`.

    ```text
    datasets/ppe_mask/
      data.yaml
      train/images  train/labels
      valid/images  valid/labels
      test/images   test/labels
    ```

    Each image has a `.txt` file with the same name in `labels`. Every line is one box: `class_id center_x center_y width height`, with the four numbers between 0 and 1 relative to the image size.

## Task 5: Understand `data.yaml`

The `data.yaml` file tells YOLO where the images are and what the class numbers mean. Two mistakes here cause most "my model is broken" reports, and the script avoids both.

1. Open the file.

    ```bash
    <copy>
    cat datasets/ppe_mask/data.yaml
    </copy>
    ```

    You should see a `path:` that is an **absolute** path on your computer, the three split folders, and the class names in index order:

    ```yaml
    path: /home/you/ai-solutions/workshops/yolo26_object_detection_26ai/datasets/ppe_mask
    train: train/images
    val: valid/images
    test: test/images
    nc: 2
    names:
      0: mask
      1: no_mask
    ```

2. Note why each detail is there.

    * An **absolute `path:`** means training works from any folder. Exports from labeling tools often contain relative paths such as `../train/images`, which YOLO resolves from your current folder rather than from the file, and fail with "dataset not found".
    * The **class order** is the contract between your labels and your model. The source dataset has a placeholder class `mask-wearing` with no boxes at index 0, so `mask` is 1 and `no-mask` is 2. The script maps classes **by name**, drops the placeholder and renumbers, so `mask` is 0 and `no_mask` is 1. Never fix class problems by editing the `names:` list: that relabels the numbers without changing the boxes, so every box silently gets the wrong class. Lab 2 gives you a script that remaps by name.

You now have a working environment, a tested YOLO26 install and a dataset in the exact format used by the rest of the workshop.

You may now **proceed to the next lab**.

## Learn More

* [Ultralytics quickstart](https://docs.ultralytics.com/quickstart/)
* [YOLO detection dataset format](https://docs.ultralytics.com/datasets/detect/)
* [mask-wearing dataset on Hugging Face](https://huggingface.co/datasets/Francesco/mask-wearing-608pr)

## Acknowledgements

* **Author** - Ignacio Martinez, Oracle
* **Last Updated By/Date** - Ignacio Martinez, October 2026
