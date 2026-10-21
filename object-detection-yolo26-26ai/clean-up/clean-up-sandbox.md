# Clean Up

## Introduction

This lab removes what the workshop created on your computer and in the database. Your reservation, and the database in it, end automatically, so the database cleanup is optional. Do it if you want to leave the database as you found it, or to run the workshop again.

> **Warning:** The steps below permanently delete data and cannot be undone. They remove:
>
> * the Python environment, the downloaded dataset, the training runs and your trained model on your computer
> * the `YOLO26` database user and everything in it: the detection tables and views, the embeddings, the Select AI credential and profile, and the network access entry for the Generative AI endpoint
>
> Copy `runs/ppe_v1/weights/best.pt` somewhere safe first if you want to keep your model.

Estimated Time: 5 minutes

### Objectives

In this lab, you will:

* Remove the local files and environment
* Remove the workshop user from the database

### Prerequisites

This lab assumes you completed the earlier labs, with the workshop folder as your current directory.

## Task 1: Remove the local files

1. Leave the Python environment and delete the generated files. The first command runs only if the environment is active.

    ```bash
    <copy>
    deactivate
    rm -rf .venv datasets runs events.jsonl frames demo.mp4 out_demo.mp4 *.pt
    </copy>
    ```

2. Press **Ctrl+C** in any terminal that still runs Label Studio, and delete `~/labelstudio-venv` if you created it in Lab 2.

## Task 2: Remove the workshop user from the database

Dropping the user removes all of its tables and objects.

1. Close every terminal that holds a connection as `YOLO26`, and sign out of Database actions as that user. A user with an open session can not be dropped.

2. In your reservation, open the database, then **Database actions** and **SQL** as `ADMIN`.

3. Open the contents of `db/cleanup_admin.sql` and run it as a script. Enter the Generative AI region you used in Lab 9, or press Enter if you skipped that lab. The script removes the network access entry and drops the `YOLO26` user with `CASCADE`.

You have removed the workshop's files and database objects.

## Learn More

* [Autonomous AI Database documentation](https://docs.oracle.com/en/cloud/paas/autonomous-database/)

## Acknowledgements

* **Author** - Ignacio Martinez, Oracle
* **Last Updated By/Date** - Ignacio Martinez, October 2026
