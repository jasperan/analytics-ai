# Clean Up

## Introduction

This lab removes what the workshop created on your computer and in your tenancy. The Autonomous AI Database, the notebook session and the model deployment live in your tenancy and keep incurring charges until you remove them, so Tasks 4 to 6 are the steps that matter for cost. Do the tasks that apply to what you created and skip the others.

> **Warning:** The steps below permanently delete data and cannot be undone. They remove:
>
> * the tables, vectors, ONNX models and Select AI profile and credential of the `LOLML` schema, and the `LOLML` user itself
> * the Oracle AI Database 26ai Free container and its stored data, if you used Option B of Lab 9
> * the Autonomous AI Database itself, with everything stored in it (Task 4)
> * the OCI Generative AI API key and the IAM policy for it, if you created them in Lab 10
> * the OCI Data Science notebook session and the model deployment, if you created them in Labs 1 and 8
> * the Python environment, the downloaded datasets, the trained models and the outputs on your computer (Task 7)

Estimated Time: 15 minutes

### Objectives

In this lab, you will:

* Stop the programs the workshop left running
* Remove the workshop objects from the database, or the database itself
* Remove the cloud resources you created
* Remove the files on your computer

### Prerequisites

This lab assumes you completed the earlier labs, with the `LOLML_ADMIN_*` and `LOLML_DB_*` environment variables from Lab 9 set in a terminal in the `lol-ml-26ai` folder with the Python environment from Lab 1 active. If you open a new terminal, set the variables again as in Lab 9.

## Task 1: Stop the running programs

Stop everything that is still running, then close any other database sessions. A database user cannot be dropped while it is connected.

1. Press **Ctrl+C** in every terminal that runs JupyterLab, the model server from Lab 8 or the mock Live Client, and close the notebook tabs that you connected to the database in.

2. If the terminal shows a notebook kernel that is still running, shut it down from JupyterLab (**Kernel**, then **Shut Down All Kernels**) before you continue.

## Task 2: Remove the database objects

Drop everything the labs created in the `LOLML` schema. Terminating the Autonomous AI Database in Task 4 deletes it as well, so this task matters only if you keep the database or use the Free container. Both scripts can run again safely.

1. Drop the tables, the vector index, the ONNX models and the Select AI profile and credential.

    ```
    <copy>
    python scripts/db_setup.py 06
    </copy>
    ```

2. If you set up Select AI in Lab 10, remove the network access entry that it added. Set `LOLML_GENAI_REGION` to the region you used there if the variable is no longer set, then run the script. It runs as the administrator.

    ```
    <copy>
    python scripts/db_setup.py 07a
    </copy>
    ```

3. Drop the `LOLML` user. The script runs as the administrator, which is `ADMIN` on Autonomous AI Database and `SYSTEM` on the Free container, and prints `0` when the user is gone. If it fails with `ORA-01940: cannot drop a user that is currently connected`, a session of `LOLML` is still open: stop it (Task 1) and run the script again.

    ```
    <copy>
    python scripts/db_setup.py 07b
    </copy>
    ```

## Task 3: Remove the Free container (Option B)

Skip this task if you used an Autonomous AI Database. Stopping the container keeps your tables; removing it with its volume deletes the data and frees several gigabytes.

1. From the `lol-ml-26ai` folder, remove the container and its data volume.

    ```
    <copy>
    docker compose -f docker/compose.yaml down -v
    </copy>
    ```

    If you started the container with `docker run` instead of Compose, use `docker rm -f lolml-oracle-free` and `docker volume rm lolml_oracle_free_data`.

2. Optionally, remove the image that was downloaded.

    ```
    <copy>
    docker image rm gvenzl/oracle-free:latest-faststart
    </copy>
    ```

3. Remove the passwords from your terminal environment by closing the terminal, or unset `ORACLE_PASSWORD` and the `LOLML_*` variables.

## Task 4: Terminate the Autonomous AI Database (Option A)

Stop the charges by deleting the Autonomous AI Database you created for this workshop. Terminating permanently deletes the database and all its data, and you cannot get it back. Make sure it is the right database before you confirm. An Always Free database does not incur charges, but it counts toward your limit of two, so terminate it as well if you no longer need it.

1. In the Oracle Cloud Console, open the navigation menu, select **Oracle AI Database**, then **Autonomous AI Database**, and open your database (`LOLML26AI`).

2. Click **More actions**, then **Terminate**, and confirm by entering the database name.

## Task 5: Delete the OCI Generative AI API key and its policy

Skip this task if you did not set up Select AI. An API key that is no longer needed is a risk. Delete it and the policy that allows it.

1. In the Oracle Cloud Console, open the **Generative AI** service in the region where you created the key, open **API keys**, select the key you created for this workshop and delete it. If you prefer to keep it, revoke or deactivate the key secret instead; see [Managing API Keys](https://docs.oracle.com/en-us/iaas/Content/generative-ai/api-keys.htm#manage).

2. Open **Identity & Security**, then **Policies**, and delete the policy statement that you added for the key in Lab 10 (it contains `request.principal.type='generativeaiapikey'`), or remove only that statement if the policy holds others.

## Task 6: Delete the Data Science resources

Skip this task if you did not create an OCI Data Science notebook session (Lab 1) or model deployment (Lab 8). Both run on compute resources that are billed while they exist.

1. In the Oracle Cloud Console, open the navigation menu, select **Analytics & AI**, then **Data Science**, and open the project you created.

2. Open **Model deployments**, select the deployment from Lab 8 and delete it. If the console asks you to deactivate it first, do that first; see [Deactivating and Activating a Model Deployment](https://docs.oracle.com/en-us/iaas/Content/data-science/using/model-dep-deactivate.htm).

3. Open **Notebook sessions**, select the notebook session from Lab 1 and delete it. This stops its compute instance and deletes its block volume, so copy any file you want to keep first; see [Deleting a Notebook Session](https://docs.oracle.com/en-us/iaas/Content/data-science/using/delete-notebooks.htm).

4. Optionally delete the model you stored in the model catalog; see [Deleting a Model](https://docs.oracle.com/en-us/iaas/Content/data-science/using/delete-models.htm). If you want to delete the project as well, do so last.

## Task 7: Remove the local files

Delete the Python environment, the downloaded datasets, the trained models and the outputs. Do this last, because the earlier tasks use the environment. The small sample in `data/sample` is part of the repository and stays. If you set `LOLML_HOME` to another folder, delete the `data`, `models` and `outputs` folders in that folder instead. Deactivate the Python environment first.

1. In the `lol-ml-26ai` folder, run the commands for your shell.

    <if type="linux">

    ```bash
    <copy>
    deactivate
    rm -rf .venv models outputs data/performance.parquet data/by_minute.parquet
    </copy>
    ```

    </if>

    <if type="windows">

    ```powershell
    <copy>
    deactivate
    Remove-Item -Recurse -Force .venv, models, outputs -ErrorAction SilentlyContinue
    Remove-Item -Force data\performance.parquet, data\by_minute.parquet -ErrorAction SilentlyContinue
    </copy>
    ```

    </if>

You have removed what the workshop created. Thank you for taking the workshop.

## Learn More

* [Terminate an Autonomous AI Database Instance](https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/autonomous-terminate.html)
* [OCI Generative AI API Keys](https://docs.oracle.com/en-us/iaas/Content/generative-ai/api-keys.htm)
* [Deleting a Notebook Session](https://docs.oracle.com/en-us/iaas/Content/data-science/using/delete-notebooks.htm)

## Acknowledgements

* **Author** - Ignacio Martinez, Oracle
* **Last Updated By/Date** - Ignacio Martinez, October 2026
