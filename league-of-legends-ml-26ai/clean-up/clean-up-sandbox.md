# Clean Up

## Introduction

This lab removes what the workshop created on your computer and, optionally, in the database. Your reservation, and the database in it, end automatically, so there is nothing to terminate and the database cleanup is optional. Do it if you want to leave the database as you found it, or to run the workshop again.

> **Warning:** The steps below permanently delete data and cannot be undone. They remove:
>
> * the tables, vectors, ONNX models and Select AI profile and credential of the `LOLML` schema, and the `LOLML` user itself (Task 2, optional)
> * the Oracle AI Database 26ai Free container and its stored data, if you started one (Task 3)
> * the OCI Generative AI API key and the IAM policy for it, if you created them in your own tenancy (Task 4)
> * the Python environment, the downloaded datasets, the trained models and the outputs on your computer (Task 5)

Estimated Time: 10 minutes

### Objectives

In this lab, you will:

* Stop the programs the workshop left running
* Remove the workshop objects from the database (optional)
* Remove the files on your computer

### Prerequisites

This lab assumes you completed the earlier labs, with the `LOLML_ADMIN_*` and `LOLML_DB_*` environment variables from Lab 9 set in a terminal in the `lol-ml-26ai` folder with the Python environment from Lab 1 active. If you open a new terminal, set the variables again as in Lab 9.

## Task 1: Stop the running programs

Stop everything that is still running, then close any other database sessions. A database user cannot be dropped while it is connected.

1. Press **Ctrl+C** in every terminal that runs JupyterLab, the model server from Lab 8 or the mock Live Client, and close the notebook tabs that you connected to the database in.

2. If the terminal shows a notebook kernel that is still running, shut it down from JupyterLab (**Kernel**, then **Shut Down All Kernels**) before you continue.

## Task 2: Remove the database objects (optional)

Drop everything the labs created in the `LOLML` schema. The scripts can run again safely.

1. Drop the tables, the vector index, the ONNX models and the Select AI profile and credential.

    ```
    <copy>
    python scripts/db_setup.py 06
    </copy>
    ```

2. If you set up Select AI in Lab 10, remove the network access entry that it added. Set `LOLML_GENAI_REGION` to the region you used there if the variable is no longer set, then run the script. It runs as `ADMIN`.

    ```
    <copy>
    python scripts/db_setup.py 07a
    </copy>
    ```

3. Drop the `LOLML` user. The script runs as `ADMIN` and prints `0` when the user is gone. If it fails with `ORA-01940: cannot drop a user that is currently connected`, a session of `LOLML` is still open: stop it (Task 1) and run the script again.

    ```
    <copy>
    python scripts/db_setup.py 07b
    </copy>
    ```

You do not need to delete the database. The reservation and its database are removed automatically when the reservation ends.

## Task 3: Remove the Free container (if you started one)

Skip this task if you used only the reservation's database. Removing the container with its volume deletes the data and frees several gigabytes.

1. From the `lol-ml-26ai` folder, remove the container and its data volume.

    ```
    <copy>
    docker compose -f docker/compose.yaml down -v
    </copy>
    ```

    If you started the container with `docker run` instead of Compose, use `docker rm -f lolml-oracle-free` and `docker volume rm lolml_oracle_free_data`.

## Task 4: Delete the OCI Generative AI API key

Skip this task if you did not create an API key. If you created an OCI Generative AI API key in your own tenancy for Select AI, delete it and the policy that allows it. An API key that is no longer needed is a risk.

1. In the Oracle Cloud Console of your own tenancy, open the **Generative AI** service in the region where you created the key, open **API keys**, select the key and delete it. If you prefer to keep it, revoke or deactivate the key secret instead; see [Managing API Keys](https://docs.oracle.com/en-us/iaas/Content/generative-ai/api-keys.htm#manage).

2. Open **Identity & Security**, then **Policies**, and delete the policy statement that you added for the key (it contains `request.principal.type='generativeaiapikey'`).

## Task 5: Remove the local files

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

* [Autonomous AI Database documentation](https://docs.oracle.com/en/cloud/paas/autonomous-database/)
* [OCI Generative AI API Keys](https://docs.oracle.com/en-us/iaas/Content/generative-ai/api-keys.htm)

## Acknowledgements

* **Author** - Ignacio Martinez, Oracle
* **Last Updated By/Date** - Ignacio Martinez, October 2026
