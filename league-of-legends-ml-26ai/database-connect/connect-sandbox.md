# Connect to Autonomous AI Database 26ai

## Introduction

The earlier labs kept the data in files and the models in Python. The next lab moves both into Oracle AI Database 26ai: you load the games into tables, ask questions in SQL, search for similar games with vectors and score rows with your model. This lab prepares the connection to the Autonomous AI Database that your reservation already created.

The Python scripts reach the database with the `python-oracledb` driver in *thin mode*. Thin mode needs no Oracle Client install and, with plain TLS, no wallet (a download with client certificates). It needs three things from you: a database user, its password and a connect string. You collect them in environment variables, which every later script reads. Your reservation's database accepts connections only from allowed IP addresses, so you first make sure that your computer's address is allowed.

> **Note:** The authors tested the connection and every script in Lab 10 on a local Oracle AI Database 26ai Free container. The console steps in this lab were written from the current Oracle documentation and the reservation's login information, and have not been run by the authors on a reservation.

Estimated Time: 15 minutes

### Objectives

In this lab, you will:

* Locate your reservation's Autonomous AI Database
* Allow your public IP address and make sure mutual TLS (mTLS) is not required
* Copy the TLS connect string
* Store the connection details in environment variables and test the connection

### Prerequisites

This lab assumes you have:

* A started LiveLabs reservation
* Completed Lab 1, with a terminal open in the `lol-ml-26ai` folder of the repository and the Python environment from Lab 1 active

## Task 1: Get the database

Open your reservation's database and allow connections from your computer.

1. In your LiveLabs reservation, click **View Login Info** and open the Autonomous AI Database link.

2. Copy the `ADMIN` password from the login information and keep it. Lab 10 needs it.

3. On the database details page, check that the database version is 26ai. The vector and ONNX features of Lab 10 need 26ai.

4. In the **Network** area, check **Access control list**. If your public IP address is not in the list (the addresses that are allowed to connect), click **Edit** next to **Access control list**, add your address and click **Update**. To find your address, search "what is my IP" in a browser. You can also select **Add my IP address** in the dialog.

5. In the same area, check **Mutual TLS (mTLS) authentication**. If it shows **Required**, click **Edit** next to it, clear **Require mutual TLS (mTLS) authentication** and click **Update**. mTLS needs a client wallet; plain TLS does not.

6. Wait until the **Lifecycle state** shows **Available** again. If you can change neither setting, your reservation's database is already configured for you; continue with Task 2 and use the error messages in Task 4 to find out.

## Task 2: Copy the TLS connect string

The connect string tells the driver where the database is. You need the TLS variant.

1. On the details page, click **Database connection**.

2. In the dialog, under **Connection Strings**, set the **TLS Authentication** drop-down list to **TLS**. If it shows **Mutual TLS**, the strings are different and do not work without a wallet.

3. Copy the connection string of the **LOW** service. The long text that starts with `(description=` is the connect string. Keep it in a text editor for Task 3.

## Task 3: Set the environment variables

Save the connection details so that every script in Lab 10 finds them. `LOLML_ADMIN_*` is the `ADMIN` account that creates the workshop user; `LOLML_DB_*` is the workshop user `LOLML` that Lab 10 creates, with the password you choose now.

1. Replace `<TLS connect string from Task 2>` with the connect string and keep the single quotes around it. Run the block for your shell. The scripts ask for the passwords without showing them: enter the `ADMIN` password from Task 1 first, then a new password of your choice for the `LOLML` user.

    <if type="linux">

    ```bash
    <copy>
    export LOLML_ADMIN_USER=ADMIN
    printf 'ADMIN password from the login info: '; read -rs LOLML_ADMIN_PASSWORD; echo; export LOLML_ADMIN_PASSWORD
    export LOLML_DB_USER=LOLML
    printf 'Choose a password for the LOLML user: '; read -rs LOLML_DB_PASSWORD; echo; export LOLML_DB_PASSWORD
    export LOLML_DB_DSN='<TLS connect string from Task 2>'
    </copy>
    ```

    </if>

    <if type="windows">

    ```powershell
    <copy>
    $env:LOLML_ADMIN_USER = 'ADMIN'
    $env:LOLML_ADMIN_PASSWORD = [System.Net.NetworkCredential]::new('', (Read-Host 'ADMIN password from the login info' -AsSecureString)).Password
    $env:LOLML_DB_USER = 'LOLML'
    $env:LOLML_DB_PASSWORD = [System.Net.NetworkCredential]::new('', (Read-Host 'Choose a password for the LOLML user' -AsSecureString)).Password
    $env:LOLML_DB_DSN = '<TLS connect string from Task 2>'
    </copy>
    ```

    </if>

    The password of the `LOLML` user must meet the strong-password rules of Autonomous AI Database: use at least 12 characters with upper-case and lower-case letters and a digit, and do not use the user name in it. Variables last only as long as the terminal. Use the same terminal for Lab 10, or run this block again in a new one.

## Task 4: Test the connection

Connect as `ADMIN` and print the database version.

1. Run the following command from the `lol-ml-26ai` folder. It is the same in every shell.

    ```
    <copy>
    python -c "from lolml import db; print(db.banner(db.connect(admin=True)))"
    </copy>
    ```

2. The command prints the version of the database. The authors saw `Oracle AI Database 26ai Free Release 23.26.3.0.0 - Develop, Learn, and Run for Free Version 23.26.3.0.0` on the container; your Autonomous AI Database prints its own release text, which should also name 26ai.

3. If the command fails, the error message names the problem. A message that lists `LOLML_DB_USER`, `LOLML_DB_PASSWORD` or `LOLML_DB_DSN` means a variable is missing in this terminal. `ORA-01017` means a wrong user name or password. A connection that times out usually means your public IP address is not in the access control list (Task 1).

You now have a connect string and the environment variables for Lab 10.

You may now **proceed to the next lab**.

## Learn More

* [Configure Network Access with Access Control Rules (ACLs)](https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/access-control-rules-autonomous.html)
* [Update Network Options to Allow TLS or Require Only Mutual TLS (mTLS) Authentication](https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/support-tls-mtls-authentication.html)
* [Connect Python Applications Without a Wallet (TLS)](https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/connecting-python-tls.html)
* [Autonomous AI Database documentation](https://docs.oracle.com/en/cloud/paas/autonomous-database/)

## Acknowledgements

* **Author** - Ignacio Martinez, Oracle
* **Last Updated By/Date** - Ignacio Martinez, October 2026
