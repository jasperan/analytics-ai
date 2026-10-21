# Write a Compliance Summary with OCI Generative AI (Optional)

## Introduction

Numbers in a table are not a report. In this optional lab a Python script reads *aggregated* figures from your detection tables and asks a generative AI chat model on OCI to write a short summary for a site manager. Only the totals leave the database, never images or frames.

The script calls OCI Generative AI through its OpenAI-compatible endpoint with a plain HTTPS request, so it needs no extra Python package. It asks the model to use only the numbers it is given and to include a caveat about confidence, and you check the result against the figures from Lab 7.

Estimated Time: 15 minutes

### Objectives

In this lab, you will:

* See exactly which facts the script sends to the model
* Generate a summary with an OCI Generative AI chat model
* Check a generated text against the data

### Prerequisites

This lab assumes you have:

* Completed Lab 7, with the database variables set in your terminal
* The region, chat model name and API key from Lab 9, Task 1. This lab does not need the Select AI profile

## Task 1: Preview the prompt

1. Run the script with `--dry-run`. It queries the database and prints the prompt, but does not call the model.

    ```bash
    <copy>
    python db/genai_summary.py --dry-run
    </copy>
    ```

2. Read the facts block. For the run described in Lab 7 it was:

    ```text
    {
      "frames_processed": 375,
      "frames_with_people": 375,
      "frames_with_at_least_one_person_without_mask": 25,
      "average_mask_compliance_percent": 98.7,
      "detections_below_0.5_confidence": 201,
      "per_class": [
        {"class": "mask", "boxes": 1928, "avg_confidence": 0.715},
        {"class": "no_mask", "boxes": 25, "avg_confidence": 0.58}
      ]
    }
    ```

    (The script prints it with one value per line.) This is everything the model will know. It does not know that 25 frames are one repeated photo, or that the detector misses many people without masks. A generated report can only be as honest as its inputs, which is why the prompt asks for a caveat.

## Task 2: Generate the summary

1. Set the region, model name and key for this terminal. The key prompt does not echo.

    ```bash
    <copy>
    export GENAI_REGION=<region>
    export GENAI_MODEL=<chat model name from the catalog>
    printf 'OCI Generative AI API key: '; read -rs GENAI_API_KEY; echo; export GENAI_API_KEY
    </copy>
    ```

2. Generate the summary.

    ```bash
    <copy>
    python db/genai_summary.py
    </copy>
    ```

3. Read the four sentences and verify every number against the facts block above. The text varies from one run to the next. Typical problems to look for are a number that was not in the input, a claim of "full compliance" that ignores the low recall of `no_mask`, and a missing caveat. If you see one, tighten the prompt in `db/genai_summary.py` and run it again.

> **Note:** If the call fails, the script prints the HTTP status and the start of the response. A `401` means the key is wrong or from another region, and a `404` usually means the model name is not available in that region.

You now have the full path from camera frames to a written report, with the facts visible at every step.

You may now **proceed to the next lab**.

## Learn More

* [OCI Generative AI](https://docs.oracle.com/en-us/iaas/Content/generative-ai/home.htm)

## Acknowledgements

* **Author** - Ignacio Martinez, Oracle
* **Last Updated By/Date** - Ignacio Martinez, October 2026
