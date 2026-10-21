# Store Detection Events in Oracle AI Database 26ai

## Introduction

A detector that only draws boxes on a screen forgets everything the moment the frame is gone. Storing what it saw turns it into a data source: you can count, trend, audit and search. In this lab you create three tables and two views for detection events, load the events file from Lab 5 together with an embedding of each saved frame, and query the result with SQL.

The design is deliberately small:

* `DETECTION_RUNS`: one row per run of the detector (which model, which source).
* `FRAMES`: one row per frame, with an `EMBEDDING` column of type `VECTOR`.
* `DETECTIONS`: one row per box, with the class, the confidence and the pixel coordinates.
* `DETECTION_EVENTS` and `FRAME_COMPLIANCE`: views that join them and compute, per frame, how many people wore a mask and the compliance percentage.

An **embedding** is a list of numbers (here 256) that summarizes an image so that similar images get similar numbers. You compute it with the same YOLO26 model that made the detections, so no extra model or key is needed.

Estimated Time: 20 minutes

### Objectives

In this lab, you will:

* Create the detection schema
* Load detection events and frame embeddings with Python
* Query the data with SQL
* Read the results with their limits in mind

### Prerequisites

This lab assumes you have:

* Completed Labs 1 to 6: a trained model at `runs/ppe_v1/weights/best.pt`, the files `events.jsonl` and `frames/` from Lab 5, and the `DB_DSN`, `DB_USER` and `DB_PASSWORD` variables set in your terminal

## Task 1: Create the schema

1. Run the schema script. The `--define DIM=256` option sets the size of the embedding column to the size of the YOLO26 embedding. A different model size (for example `yolo26s`) produces a different size, so set `DIM` accordingly.

    ```bash
    <copy>
    python db/run_sql.py db/schema.sql --define DIM=256
    </copy>
    ```

2. Every statement should print `ok:`. The script also adds comments to the tables and columns, for example `comment on column detections.class_name is 'Detected class: mask = person wearing a face mask, no_mask = person without a face mask'`. These comments are not decoration: Select AI sends them to the language model in Lab 9 so that it understands your schema.

> **Note:** To start over, run `python db/run_sql.py db/drop_schema.sql` and then this task again.

## Task 2: Load the events

`store_events.py` reads `events.jsonl`, computes one embedding for every frame that has a saved image, and inserts the rows. It normalizes each embedding to length 1 so that cosine distance behaves well.

1. Load the events. Use the weights that produced the events, because the embeddings come from the same model.

    ```bash
    <copy>
    python db/store_events.py --events events.jsonl --weights runs/ppe_v1/weights/best.pt --conf 0.4
    </copy>
    ```

2. In the run used to write this lab, the output was:

    ```text
    375 frames, 75 embeddings computed (256 dims)
    stored run 1: 375 frames, 1953 detections
    ```

    375 frames were stored, but only 75 have an embedding: Lab 5 saved every fifth frame to disk (`--frame-every 5`), and frames without an image file are stored without a vector. Your numbers differ if you used a webcam or another video.

> **Note:** The demo video from Lab 5 shows each of the 15 test photos for one second, which is 25 frames per photo. A "frame" here is therefore not a distinct photo. When you read the counts below, remember that 25 identical frames are one photo. With a real camera, consecutive frames are similar, not identical, and Lab 8 shows how to find them.

## Task 3: Query the data with SQL

You can use `python db/run_sql.py` or paste the statements into **Database actions**, then **SQL**, for the same schema.

1. Run the prepared queries.

    ```bash
    <copy>
    python db/run_sql.py db/analytics_queries.sql
    </copy>
    ```

2. Read the answers. With the run described above you should see:

    * Detections per class: `mask` 1928 boxes with an average confidence of 0.715, and `no_mask` 25 boxes with 0.58.
    * Frames with at least one person without a mask: 25, which is exactly one of the 15 photos shown for 25 frames.
    * The five frames with the lowest compliance (`compliance_pct` of 80, one person without a mask among five detected), from frames 275 to 279.
    * Boxes with a confidence below 0.5: 201, all of them `mask`.
    * An average compliance of 98.7 percent across the 375 frames.

3. Be careful with that last number. It is a percentage of *detections*, and the model found 1928 masks against 25 people without one because that is what the test photos show and because it misses many people without masks: Lab 4 measured a recall of 0.376 for `no_mask` on the test split. A compliance score is only as trustworthy as the recall of the class that signals a violation.

Here is the query behind the compliance percentage, so you can adapt it. It lives in the `FRAME_COMPLIANCE` view:

```sql
<copy>
select count(case when d.class_name = 'mask' then 1 end)    as mask_count,
       count(case when d.class_name = 'no_mask' then 1 end) as no_mask_count,
       round(100 * count(case when d.class_name = 'mask' then 1 end) / count(d.detection_id), 1) as compliance_pct
  from frames f join detections d on d.frame_id = f.frame_id
 where f.frame_id = 276
</copy>
```

You now have detection events and frame embeddings in Oracle AI Database 26ai, and a way to ask questions about them with SQL.

You may now **proceed to the next lab**.

## Learn More

* [python-oracledb documentation](https://python-oracledb.readthedocs.io/en/latest/)
* [Oracle AI Vector Search: vector data type](https://docs.oracle.com/en/database/oracle/oracle-database/26/vecse/)

## Acknowledgements

* **Author** - Ignacio Martinez, Oracle
* **Last Updated By/Date** - Ignacio Martinez, October 2026
