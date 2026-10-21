# Clean Up

## Introduction

This lab removes what the workshop created on your computer and in your tenancy. The database (and any GPU machine you started in Lab 3) keeps incurring charges until you terminate it, so Task 3 is the step that matters for cost.

> **Warning:** The steps below permanently delete data and cannot be undone. They remove:
>
> * the Python environment, the downloaded dataset, the training runs and your trained model on your computer
> * the `YOLO26` database user and everything in it: the detection tables and views, the embeddings, the Select AI credential and profile, and the network access entry for the Generative AI endpoint
> * the Autonomous AI Database itself, with everything stored in it (Task 3)
>
> Copy `runs/ppe_v1/weights/best.pt` somewhere safe first if you want to keep your model.

Estimated Time: 10 minutes

### Objectives

In this lab, you will:

* Remove the local files and environment
* Remove the workshop user from the database
* Terminate the database and any compute resources to stop charges

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

Dropping the user removes all of its tables and objects. Terminating the database in Task 3 deletes them as well, so this task matters only if you want to keep the database.

1. Close every terminal that holds a connection as `YOLO26`, and sign out of Database actions as that user. A user with an open session can not be dropped.

2. In the Console, open your database, then **Database actions** and **SQL** as `ADMIN`.

3. Open the contents of `db/cleanup_admin.sql` and run it as a script. Enter the Generative AI region you used in Lab 9, or press Enter if you skipped that lab. The script removes the network access entry and drops the `YOLO26` user with `CASCADE`.

## Task 3: Terminate the database and compute resources

Stop the charges by deleting what you created for this workshop. Terminating permanently deletes the data and you can not get it back. Make sure it is the right resource before you confirm.

1. In the Console, open the navigation menu, select **Oracle AI Database**, then **Autonomous AI Database**, and choose your compartment.

2. Open `YOLO26ADB`, click **More actions**, then **Terminate**. Type the database name to confirm and click **Terminate Autonomous AI Database**.

3. If you used a GPU virtual machine or a Data Science notebook session in Lab 3, terminate the instance (and its boot volume) or delete the notebook session, from **Compute**, **Instances** or **Data Science**, **Notebook sessions**.

4. Delete the OCI Generative AI API key you created in Lab 9 if you will not use it again.

You have removed the workshop's files, database objects and billable resources.

## Learn More

* [Terminate an Autonomous Database](https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/autonomous-terminate.html)

## Acknowledgements

* **Author** - Ignacio Martinez, Oracle
* **Last Updated By/Date** - Ignacio Martinez, October 2026
