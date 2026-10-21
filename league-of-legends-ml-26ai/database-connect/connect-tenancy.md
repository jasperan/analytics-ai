# Connect to Autonomous AI Database 26ai

## Introduction

The earlier labs kept the data in files and the models in Python. The next lab moves both into Oracle AI Database 26ai: you load the games into tables, ask questions in SQL, search for similar games with vectors and score rows with your model. This lab prepares the connection.

The Python scripts reach the database with the `python-oracledb` driver in *thin mode*. Thin mode needs no Oracle Client install and, with plain TLS, no wallet (a download with client certificates). It needs three things from you: a database user, its password and a connect string. You collect them in environment variables, which every later script reads.

You can use an **Autonomous AI Database** in your own tenancy (Option A, Tasks 1 to 3) or a **local Oracle AI Database 26ai Free container** (Option B, Task 4). Select AI, the natural-language part of Lab 10, works only on Autonomous AI Database. Everything else works on both.

> **Note:** The authors tested the connection and every script in Lab 10 on the Free container (Option B). The console steps for Option A were written from the current Oracle documentation and have not been run by the authors.

Estimated Time: 20 minutes

### Objectives

In this lab, you will:

* Create an Autonomous AI Database 26ai in your tenancy, or start a local Free container
* Allow connections from your computer with plain TLS, without a wallet
* Copy the TLS connect string
* Store the connection details in environment variables and test the connection

### Prerequisites

This lab assumes you have:

* Completed Lab 1, with a terminal open in the `lol-ml-26ai` folder of the repository and the Python environment from Lab 1 active
* For Option A: an Oracle Cloud tenancy with permission to create Autonomous Databases
* For Option B: Docker with the `docker compose` command

## Task 1: Create the database (Option A)

Create an Autonomous AI Database that holds the workshop tables. The data takes about 350 MB.

1. Sign in to the Oracle Cloud Console, open the navigation menu, select **Oracle AI Database**, then **Autonomous AI Database**.

2. Choose your region and compartment, then click **Create Autonomous AI Database**.

3. Enter the following settings.

    * Display name and database name: `LOLML26AI`
    * Workload type: **Transaction Processing**. **Lakehouse** works as well, but its storage is entered in terabytes.
    * Database version: **26ai**. The Select AI and vector features this workshop uses need 26ai, not 19c.
    * Compute: **ECPU**, ECPU count `2`, and clear **Compute auto scaling**
    * Storage: `50` GB, unit **GB**. With the Lakehouse workload enter `1` TB instead. **Storage auto scaling** stays off.

4. Under **Create administrator credentials**, enter a password for `ADMIN` and confirm it. The password must meet the strong-password rules of Autonomous AI Database (see [About User Passwords](https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/manage-users-create.html)). Lab 10 needs it, so remember it.

5. Under **Choose network access**, select **Secure access from allowed IPs and VCNs only**.

6. In the access control rule, keep **IP notation type** set to **IP address** and enter your public IP address as the value, or select **Add my IP address**. To find your address, search "what is my IP" in a browser.

7. Clear **Require mutual TLS (mTLS) authentication**. With it cleared, the database accepts TLS connections that need no wallet.

8. Click **Create Autonomous AI Database** and wait until the state is **Available**.

> **Always Free option:** The Always Free option, which is shown in the same form, creates a free database that has 20 GB of storage and at most 30 simultaneous sessions. The documentation says you can create it only in your tenancy's home region and that Always Free with Oracle AI Database 26ai is not offered in every region (see [Always Free Autonomous AI Database](https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/autonomous-always-free.html)). The authors did not test the workshop on an Always Free database; the 350 MB of data fit within 20 GB.

## Task 2: Check the network settings (Option A)

Confirm that the database accepts TLS connections from your computer. If your public IP address changes later, for example on another network, repeat step 3.

1. On the **Autonomous AI Database details** page, find the **Network** area.

2. Check that **Access control list** shows **Enabled** and that **Mutual TLS (mTLS) authentication** shows **Not Required**.

3. To add or change an address, click **Edit** next to **Access control list**, add the IP address and click **Update**. The state changes to **Updating** for a short time. If **Mutual TLS (mTLS) authentication** shows **Required**, click **Edit** next to it, clear **Require mutual TLS (mTLS) authentication** and click **Update**.

## Task 3: Copy the TLS connect string (Option A)

The connect string tells the driver where the database is. You need the TLS variant.

1. On the details page, click **Database connection**.

2. In the dialog, under **Connection Strings**, set the **TLS Authentication** drop-down list to **TLS**. If it shows **Mutual TLS**, the strings are different and do not work without a wallet.

3. Copy the connection string of the **LOW** service. The long text that starts with `(description=` is the connect string. Keep it in a text editor for Task 5.

    > **Note:** Each database has several services named after their resource limits (high, medium, low and so on). `LOW` is enough here and leaves the most capacity for other work.

## Task 4: Use a local Free container instead (Option B)

Skip Tasks 1 to 3 and start Oracle AI Database 26ai Free in Docker. It runs on your computer and costs nothing. The authors ran the commands of Lab 10 on this container (version 23.26.3), except Select AI.

1. In your terminal, from the `lol-ml-26ai` folder, set the password of the container's administrator users (`SYS` and `SYSTEM`) and start the container. Choose your own password; it stays in your shell.

    <if type="linux">

    ```bash
    <copy>
    printf 'Choose a password for SYS and SYSTEM: '; read -rs ORACLE_PASSWORD; echo; export ORACLE_PASSWORD
    docker compose -f docker/compose.yaml up -d
    </copy>
    ```

    </if>

    <if type="windows">

    ```powershell
    <copy>
    $env:ORACLE_PASSWORD = [System.Net.NetworkCredential]::new('', (Read-Host 'Choose a password for SYS and SYSTEM' -AsSecureString)).Password
    docker compose -f docker/compose.yaml up -d
    </copy>
    ```

    </if>

2. Follow the log and wait for the message `DATABASE IS READY TO USE!`, then press **Ctrl+C**. The first start takes a few minutes. The container restarts the database once to give it vector memory, which the HNSW vector index of Lab 10 needs; wait for the message after that restart.

    ```
    <copy>
    docker compose -f docker/compose.yaml logs -f
    </copy>
    ```

3. If port 1521 is already in use on your computer, set the environment variable `LOLML_DB_PORT` to another port before step 1 (`export LOLML_DB_PORT=15210` on Linux and macOS, `$env:LOLML_DB_PORT = '15210'` in PowerShell) and use the same port in the connect string in Task 5.

    > **Note:** The folder with the repository must be on a drive that Docker can mount. With Colima, only your home folder is mounted by default. If the repository is elsewhere, the startup script is not found, the database gets no vector memory, and Lab 10 uses an IVF vector index instead of HNSW. That still works.

The connect string for this database is `localhost:1521/FREEPDB1`.

## Task 5: Set the environment variables

Save the connection details so that every script in Lab 10 finds them. `LOLML_ADMIN_*` is the account that creates the workshop user; `LOLML_DB_*` is the workshop user `LOLML` that Lab 10 creates, with the password you choose now.

1. Run the block for your shell. Replace `<TLS connect string from Task 3>` with the connect string, keep the single quotes around it, and use `localhost:1521/FREEPDB1` for Option B. For Option B, also replace `ADMIN` with `SYSTEM` and enter the container password from Task 4. The scripts ask for passwords without showing them.

    <if type="linux">

    ```bash
    <copy>
    export LOLML_ADMIN_USER=ADMIN
    printf 'Administrator password: '; read -rs LOLML_ADMIN_PASSWORD; echo; export LOLML_ADMIN_PASSWORD
    export LOLML_DB_USER=LOLML
    printf 'Choose a password for the LOLML user: '; read -rs LOLML_DB_PASSWORD; echo; export LOLML_DB_PASSWORD
    export LOLML_DB_DSN='<TLS connect string from Task 3>'
    </copy>
    ```

    </if>

    <if type="windows">

    ```powershell
    <copy>
    $env:LOLML_ADMIN_USER = 'ADMIN'
    $env:LOLML_ADMIN_PASSWORD = [System.Net.NetworkCredential]::new('', (Read-Host 'Administrator password' -AsSecureString)).Password
    $env:LOLML_DB_USER = 'LOLML'
    $env:LOLML_DB_PASSWORD = [System.Net.NetworkCredential]::new('', (Read-Host 'Choose a password for the LOLML user' -AsSecureString)).Password
    $env:LOLML_DB_DSN = '<TLS connect string from Task 3>'
    </copy>
    ```

    </if>

    The password of the `LOLML` user must also meet the strong-password rules on Autonomous AI Database: use at least 12 characters with upper-case and lower-case letters and a digit, and do not use the user name in it. Variables last only as long as the terminal. Use the same terminal for Lab 10, or run this block again in a new one.

    > **Note:** If you connect with a wallet instead (mutual TLS), also set `LOLML_DB_WALLET_DIR` to the folder of the unzipped wallet and `LOLML_DB_WALLET_PASSWORD` to its password, and use the service name from `tnsnames.ora`, for example `lolml26ai_low`, as `LOLML_DB_DSN`. Plain TLS, as set up in this lab, is simpler.

## Task 6: Test the connection

Connect as the administrator and print the database version.

1. Run the following command from the `lol-ml-26ai` folder. It is the same in every shell.

    ```
    <copy>
    python -c "from lolml import db; print(db.banner(db.connect(admin=True)))"
    </copy>
    ```

2. The command prints the version of the database. The authors saw `Oracle AI Database 26ai Free Release 23.26.3.0.0 - Develop, Learn, and Run for Free Version 23.26.3.0.0` on the container; an Autonomous AI Database 26ai prints its own release text, which should also name 26ai.

3. If the command fails, the error message names the problem. A message that lists `LOLML_DB_USER`, `LOLML_DB_PASSWORD` or `LOLML_DB_DSN` means a variable is missing in this terminal. `ORA-01017` means a wrong user name or password. On Autonomous AI Database, a connection that times out usually means your public IP address is not in the access control list (Task 2). On the container, a refused connection means it is not ready yet or listens on another port.

You now have a database, a connect string and the environment variables for Lab 10.

You may now **proceed to the next lab**.

## Learn More

* [Provision an Autonomous AI Database Instance](https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/autonomous-provision.html)
* [Configure Network Access with Access Control Rules (ACLs)](https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/access-control-rules-autonomous.html)
* [Update Network Options to Allow TLS or Require Only Mutual TLS (mTLS) Authentication](https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/support-tls-mtls-authentication.html)
* [Connect Python Applications Without a Wallet (TLS)](https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/connecting-python-tls.html)
* [Always Free Autonomous AI Database](https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/autonomous-always-free.html)

## Acknowledgements

* **Author** - Ignacio Martinez, Oracle
* **Last Updated By/Date** - Ignacio Martinez, October 2026
