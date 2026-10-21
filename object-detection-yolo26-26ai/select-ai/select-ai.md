# Ask the Database in Plain English with Select AI (Optional)

## Introduction

SQL is precise, but a site manager does not want to write it. **Select AI** lets Autonomous AI Database turn a question in plain language into SQL. In this lab you set it up over the detection views from Lab 7, ask questions about your detections, and check each answer against the SQL you already trust.

A **Select AI profile** is a stored database object that names the language model provider, the credential to reach it and the list of tables and views the model may see. When you ask a question, the database sends the question and *metadata* about those objects (names, columns and the comments you added in Lab 7) to the model, receives SQL back, and runs it. The model never needs your rows to write the query.

The profile you create is named `YOLO_GENAI`. It uses OCI Generative AI through its OpenAI-compatible endpoint with an API key. Because the database calls that endpoint itself, an administrator must allow the outbound connection once.

This lab is optional. Nothing later depends on it, and Lab 10 uses a different route to the same model.

Estimated Time: 25 minutes

### Objectives

In this lab, you will:

* Collect the region, a chat model name and an API key for OCI Generative AI
* Allow the database user to reach the Generative AI endpoint (as `ADMIN`)
* Create the credential and the Select AI profile (as `YOLO26`)
* Ask questions about your detections and verify the generated SQL

### Prerequisites

This lab assumes you have:

* Completed Lab 7, with the detection tables in your Autonomous AI Database
* Access to OCI Generative AI in your tenancy, and permission to create an API key for it
* The `ADMIN` password of the database

> **Note:** Select AI needs an Autonomous AI Database. A local Oracle AI Database Free container does not include the Select AI package, so this lab can not be done there.

## Task 1: Collect the Generative AI settings

1. In the Oracle Cloud Console, open **Generative AI** and note the region you are using, for example `us-chicago-1`. The endpoint host name is `inference.generativeai.<region>.oci.oraclecloud.com`.

2. Open the model catalog and choose a **chat** model that is available in your region. Copy its model name exactly as shown. Model availability changes over time, so take it from the catalog rather than from this guide.

3. In the Generative AI service, create an **API key** (it starts with `sk-`) and copy it right away. You see it only once. Do not paste it into a file you commit or share.

## Task 2: Allow access to the endpoint (ADMIN)

The database blocks outbound network calls by default. `db/select_ai_acl.sql` adds a network access control entry for the Generative AI host and grants `YOLO26` the right to use Select AI.

1. In the Console, open your database, then **Database actions** and **SQL**. You are signed in as `ADMIN`.

2. Open the contents of `db/select_ai_acl.sql` in the worksheet and run it as a script (the **Run Script** button). When it asks for the region, enter the region from Task 1.

3. The script ends with a query on the host access entries. Confirm that the `inference.generativeai.<region>.oci.oraclecloud.com` host appears for the `YOLO26` principal.

## Task 3: Create the credential and profile (YOLO26)

1. Sign out of Database actions, then sign in as `YOLO26` with the password you chose in Lab 6. Open **SQL** again.

2. Open `db/select_ai_profile.sql` and run it as a script. It asks for the region, the model name and the API key (the key is hidden as you type). It then:

    * creates the credential `GENAI_CRED`, which stores the key
    * creates the profile `YOLO_GENAI` with provider `openai`, your endpoint and model, and an object list that allows only `DETECTION_EVENTS`, `FRAME_COMPLIANCE` and `DETECTION_RUNS`
    * activates the profile for your session and asks a first question

3. The profile deliberately excludes the `FRAMES` table, which holds the embeddings. The model does not need vectors to answer questions about counts, and fewer objects means fewer chances of a wrong join.

> **Note:** A Select AI profile is a stored object, so editing a script does not change it. After you add tables or views, run the script again.

## Task 4: Ask questions

1. In the SQL worksheet, ask for the SQL first, without running it. `showsql` returns the SQL the model wrote.

    ```sql
    <copy>
    select dbms_cloud_ai.generate(
             prompt       => 'How many frames had at least one person without a mask?',
             profile_name => 'YOLO_GENAI',
             action       => 'showsql') as generated_sql
      from dual;
    </copy>
    ```

2. Read the SQL before you trust the answer. A correct query counts frames where `NO_MASK_COUNT` is greater than zero in `FRAME_COMPLIANCE`, or counts distinct frames in `DETECTION_EVENTS` where `CLASS_NAME` is `no_mask`. Both give the same number. The exact text differs from one run to the next, because a language model writes it.

3. Run the question and the others. In the `select ai` form the action comes first: `runsql` (the default), `showsql`, or `narrate`.

    ```sql
    <copy>
    exec dbms_cloud_ai.set_profile('YOLO_GENAI')
    select ai how many boxes were detected per class and what is the average confidence;
    select ai showsql which 5 frames have the lowest mask compliance percentage among frames where someone has no mask;
    select ai narrate what is the average mask compliance percentage across frames that contain people;
    </copy>
    ```

4. Compare each answer with the SQL you ran yourself in Lab 7 (`db/analytics_queries.sql`). If you loaded the same events as in that lab, the per-class counts, the frame count with a missing mask and the average compliance should match what Lab 7 listed. When they do not, look at the SQL the model wrote: wrong joins and wrong filters are the usual cause, and they are easy to see in `showsql`.

> **What leaves your database:** Select AI sends your question and metadata about the objects in the profile (names, columns, comments) to the model provider. With `narrate`, it also sends the query *result* so the model can describe it. Do not include objects with sensitive data in a profile for an external model.

You now have natural-language access to your detection data, and a habit worth keeping: always compare a generated query with one you can read.

You may now **proceed to the next lab**.

## Learn More

* [Select AI for Autonomous AI Database](https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/sql-generation-ai-autonomous.html)
* [OCI Generative AI](https://docs.oracle.com/en-us/iaas/Content/generative-ai/home.htm)

## Acknowledgements

* **Author** - Ignacio Martinez, Oracle
* **Last Updated By/Date** - Ignacio Martinez, October 2026
