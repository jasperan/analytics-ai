# Find Similar Frames with AI Vector Search

## Introduction

Detections tell you *what* is in a frame. Embeddings let you ask *which frames look like this one*. That is useful for finding all the frames around an incident, removing near-identical frames before you label them (labeling the same scene 25 times teaches the model nothing), and pulling similar hard examples into your next training set.

In Lab 7 you stored one embedding per saved frame in a `VECTOR(256, FLOAT32)` column. **AI Vector Search** compares vectors with `VECTOR_DISTANCE`. In this lab you use *cosine distance*: 0 means the vectors point the same way (very similar images), and larger values mean less similar.

You combine the vector comparison with an ordinary relational filter in one SQL statement, which is what makes keeping vectors in the same database as your other data practical.

Estimated Time: 15 minutes

### Objectives

In this lab, you will:

* Search for frames similar to a query image
* Combine similarity with a relational filter on detections
* Find near-duplicate frames
* Create an approximate vector index and know when you need one

### Prerequisites

This lab assumes you have:

* Completed Lab 7, with the database variables set in your terminal

## Task 1: Search by image

`vector_search.py` embeds a query image with your model and asks the database for the closest frames.

1. Search with one of the frames saved in Lab 5.

    ```bash
    <copy>
    python db/vector_search.py --weights runs/ppe_v1/weights/best.pt --image frames/demo_000100.jpg --top 8
    </copy>
    ```

2. Read the result. In the run used to write this lab, the first five rows were frames 100, 105, 110, 115 and 120 with distances from 0 to 0.0001. These are five stored frames of the same photo, shown for 25 consecutive video frames. The sixth row was already a different photo, at a distance of 0.0117, which is about a hundred times farther.

    The gap between "same photo" (at most 0.0001) and "different photo" (0.0117) shows that the embedding separates scenes clearly, and also that the numbers are specific to your model and data. Do not copy a threshold from this lab: pick one by looking at pairs from your own frames.

## Task 2: Combine similarity with a filter

The most useful questions mix both worlds: "frames that look like this one *and* contain a person without a mask".

1. Add `--class-name no_mask`. The script adds an `EXISTS` condition on the `DETECTIONS` table to the same statement.

    ```bash
    <copy>
    python db/vector_search.py --weights runs/ppe_v1/weights/best.pt --image frames/demo_000100.jpg --top 5 --class-name no_mask
    </copy>
    ```

2. In the run used to write this lab, the five results were frames 275, 295, 280, 285 and 290 with distances from 0.0697 to 0.0751: all from the one photo that contains a `no_mask` detection. Without the filter, those frames were not near the top.

    This is the core of the SQL, so you can reuse it. Bind variable `:q` holds the query vector.

    ```sql
    select f.frame_id, f.frame_no,
           round(vector_distance(f.embedding, :q, cosine), 4) as dist
      from frames f
     where f.embedding is not null
       and exists (select 1 from detections d
                    where d.frame_id = f.frame_id and d.class_name = 'no_mask')
     order by vector_distance(f.embedding, :q, cosine)
     fetch first 5 rows only
    ```

## Task 3: Find near-duplicate frames

1. List pairs of stored frames that are closer than 0.001.

    ```bash
    <copy>
    python db/vector_search.py --duplicates 0.001
    </copy>
    ```

2. The output lists up to 20 pairs ordered by distance. In the run used to write this lab, 150 pairs fell below 0.001. That is exactly what to expect: 15 photos with 5 stored frames each give 15 times 10 pairs. When you build a dataset from video, dropping all but one frame of each near-duplicate group is a cheap way to keep the label count down and stop the same scene from landing in both the training and the test split.

## Task 4: Create a vector index (optional)

Exact search compares the query with every vector. With 75 vectors that takes no time, and with a few hundred thousand it starts to matter. An *approximate* index trades a little accuracy for speed.

1. Create an IVF (inverted file) vector index on the embedding column and run an approximate query. An IVF index needs no extra memory configuration.

    ```bash
    <copy>
    python db/run_sql.py db/vector_index.sql
    </copy>
    ```

2. The script creates `FRAMES_EMBEDDING_IX`, checks that its status is `VALID`, and then repeats a search with `fetch approximate first 3 rows only`, which lets the optimizer use the index. The result here still lists frame 1 at distance 0 and two frames of the same photo, but with an approximate index you accept that a few true neighbors can be missing at scale. Test the accuracy on your own data before relying on it.

> **Note:** These embeddings come from the layers of a *detector*, which learned to find masks and faces, not to describe images in general. They are good at "same scene" and "similar layout", and not a substitute for a general image-text embedding model if you want to search for concepts such as "crowded entrance". Oracle AI Database can also store embeddings from other models in the same kind of column.

You now have a similarity search that you can combine with any SQL filter.

You may now **proceed to the next lab**.

## Learn More

* [Oracle AI Vector Search User's Guide](https://docs.oracle.com/en/database/oracle/oracle-database/26/vecse/)
* [Ultralytics embed method](https://docs.ultralytics.com/modes/predict/)

## Acknowledgements

* **Author** - Ignacio Martinez, Oracle
* **Last Updated By/Date** - Ignacio Martinez, October 2026
