# Provision Autonomous AI Database 26ai

## Introduction

From this lab on, your detector gets a memory. You create an Autonomous AI Database 26ai, create a database user for the workshop, and connect to it from the Python environment you built in Lab 1. The database will hold detection events and frame embeddings (Lab 7), and answer questions about them (Labs 8 to 10).

The Python scripts connect with `python-oracledb` in *thin mode*: no Oracle Client or wallet files are needed if the database accepts TLS connections from your computer's IP address. You configure that here.

Estimated Time: 20 minutes

### Objectives

In this lab, you will:

* Create an Autonomous AI Database 26ai
* Allow your public IP address and set mutual TLS (mTLS) to not required
* Create the `YOLO26` database user
* Save the connection details in environment variables and test the connection

### Prerequisites

This lab assumes you have:

* An Oracle Cloud tenancy with permission to create Autonomous Databases
* Completed Lab 1 (the Python environment is installed)

## Task 1: Create the database

Create a database that accepts connections only from your computer.

1. Sign in to the Oracle Cloud Console, open the navigation menu and select **Oracle AI Database**, then **Autonomous AI Database**.

2. Choose your compartment and click **Create Autonomous AI Database**.

3. Enter the following settings.

    * Display name and database name: `YOLO26ADB`
    * Workload type: **Transaction Processing**
    * Database version: **26ai**
    * Compute: **ECPU**, with the default count and auto scaling as you prefer
    * Administrator password: choose a strong password for `ADMIN` and remember it; Task 3 and Lab 9 need it

4. Under **Network access**, select **Secure access from allowed IPs and VCNs only**.

5. Add your computer's public IP address to the access control list (the list of addresses allowed to connect). To find your address, search "what is my IP" in a browser.

6. Clear **Require mutual TLS (mTLS) authentication** if it is selected. mTLS needs a client wallet (a download that holds the client certificates). Without it, the scripts connect with plain TLS plus the IP access list.

7. Click **Create** and wait until the state is **Available**.

## Task 2: Copy the connection string

1. On the database details page, click **Database connection** and set **TLS authentication** to **TLS**.

2. Copy the **Low** connection string of type TLS. This is the long `(description=...)` string that works without a wallet. Keep it open: you paste it in Task 4.

> **Note:** If your organization requires mTLS, download the wallet instead, unzip it, and also set `DB_WALLET_DIR` (the unzipped folder) and `DB_WALLET_PASSWORD` in Task 4. Use the wallet's `yolo26adb_low` alias as `DB_DSN`.

## Task 3: Create the workshop user

Do not run the workshop as `ADMIN`. Create a user that owns only the workshop's tables.

1. On the database details page, click **Database actions**, then **SQL**. You are signed in as `ADMIN`.

2. Paste and run the following statements, with your own password in place of `ChangeMe_Str0ng_Passw0rd#`. The password must follow the database password rules (at least 12 characters with upper case, lower case and a number).

    ```sql
    <copy>
    create user yolo26 identified by "ChangeMe_Str0ng_Passw0rd#" quota unlimited on data;
    grant db_developer_role to yolo26;
    </copy>
    ```

    The same statements are in `db/create_user.sql`.

3. Check the output pane for `User YOLO26 created.` and `Grant succeeded.`

## Task 4: Connect from Python

1. In the terminal where your Lab 1 environment is active, set the connection details. Paste the TLS connection string from Task 2 inside the single quotes. The password prompt does not echo what you type, and nothing is written to a file.

    ```bash
    <copy>
    export DB_DSN='<TLS connection string from Task 2>'
    export DB_USER=yolo26
    printf 'YOLO26 password: '; read -rs DB_PASSWORD; echo; export DB_PASSWORD
    </copy>
    ```

2. Test the connection.

    ```bash
    <copy>
    python db/ping.py
    </copy>
    ```

3. You should see the database banner (it names Oracle AI Database 26ai), the line `connected as: YOLO26`, and `AI Vector Search works, cosine distance of two orthogonal vectors = 1.0`.

    The variables last only as long as the terminal. Open every later terminal in this folder, activate the environment, and run the block in step 1 again.

You now have an Autonomous AI Database 26ai, a workshop user and a working connection from Python.

You may now **proceed to the next lab**.

## Learn More

* [Autonomous AI Database documentation](https://docs.oracle.com/en/cloud/paas/autonomous-database/)
* [Connect with python-oracledb](https://python-oracledb.readthedocs.io/en/latest/user_guide/connection_handling.html)

## Acknowledgements

* **Author** - Ignacio Martinez, Oracle
* **Last Updated By/Date** - Ignacio Martinez, October 2026
