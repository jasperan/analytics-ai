# Query, Search and Score in the Database

## Introduction

So far the games lived in parquet files and the model in Python. In this lab you put them into Oracle AI Database 26ai and use four of its features on the same data:

* **SQL** answers questions about 227 thousand player-games and one million in-game snapshots with one statement.
* **Select AI** turns a question in plain English into SQL (Autonomous AI Database only).
* **AI Vector Search** finds the games that look most like a game you give it.
* **In-database scoring** runs your scikit-learn model inside the database, from SQL, through ONNX (optional).

Every script reads the connection details you saved in Lab 9. The scripts are in the `scripts` folder and the SQL they run is in the `sql` folder, numbered in the order you use it. You can also paste the SQL into Database Actions (SQL worksheet) on an Autonomous AI Database.

> **Note:** The authors ran this lab on Oracle AI Database 26ai Free 23.26.3, with the full datasets. The outputs below come from that run: `PLAYER_MATCH` holds 226,910 rows and `BY_MINUTE_SNAPSHOT` 1,000,000 rows. **Select AI (Task 4) was not run by the authors because it requires Autonomous AI Database.** Your numbers match if you load the full datasets; with `LOLML_SMOKE=1` (the small sample) they differ.

Estimated Time: 45 minutes

### Objectives

In this lab, you will:

* Create the `LOLML` user and the tables, and load the data
* Ask questions about the games in SQL
* Set up Select AI and ask questions in English (Autonomous AI Database only)
* Find the five games most similar to a game the database has never seen
* Export your model to ONNX and score rows with SQL (optional)

### Prerequisites

This lab assumes you have:

* Completed Lab 9, with `LOLML_ADMIN_*` and `LOLML_DB_*` set in a terminal in the `lol-ml-26ai` folder with the Python environment from Lab 1 active
* The full datasets from the data lab (`python scripts/fetch_data.py`). Without them the scripts use the small committed sample, and the results differ from this page.

## Task 1: Create the user and the tables

Create the `LOLML` schema that owns everything in this lab. The administrator account from Lab 9 runs the first script, because creating a user needs it; the second script runs as `LOLML`.

1. Run the setup script.

    ```
    <copy>
    python scripts/db_setup.py
    </copy>
    ```

2. The script runs `sql/01_create_user.sql` as the administrator and `sql/02_tables.sql` as `LOLML`. You should see output like this on the Free container:

    ```
    connected as admin: Oracle AI Database 26ai Free Release 23.26.3.0.0 - Develop, Learn, and Run for F
    == 01_create_user.sql
      | created user LOLML (tablespace USERS)
      | DBMS_CLOUD_AI not present (Free container): skipped
      | DBMS_CLOUD not present (Free container): skipped
    connected as LOLML: Oracle AI Database 26ai Free Release 23.26.3.0.0 - Develop, Learn, and Run for F
    == 02_tables.sql
    ```

    On Autonomous AI Database the two lines about `DBMS_CLOUD` say `granted EXECUTE on ... to LOLML` instead, because those packages exist there. The scripts can run again: they create what is missing and change nothing else.

3. Look at what the second script created. `PLAYER_MATCH` has one row per player per match and 31 columns (30 picked from the 98 of the performance dataset, plus a row id): the champion, the position, the result (`WIN`), kills, deaths, assists, gold, damage, minions, vision and more. `BY_MINUTE_SNAPSHOT` has the in-game stats at each minute with the final result of the game. Every table and column has a comment, which Select AI sends to the language model later. Print the table comments.

    ```
    <copy>
    python scripts/db_sql.py "select table_name, comments from user_tab_comments where table_name in ('PLAYER_MATCH', 'BY_MINUTE_SNAPSHOT') order by table_name"
    </copy>
    ```

## Task 2: Load the data

Load the two datasets with `executemany` in batches. The loader reads the column names from the table definition and fills every column it finds in the data.

1. Run the loader.

    ```
    <copy>
    python scripts/db_load.py
    </copy>
    ```

2. It loads both tables and checks the row count. The authors saw this, in about 10 seconds on the container:

    ```
    PLAYER_MATCH: loading 226,910 rows x 31 columns from 'performance' in batches of 20,000
    PLAYER_MATCH: done in 3.3s
    PLAYER_MATCH: verified 226,910 rows
    BY_MINUTE_SNAPSHOT: loading 1,000,000 rows x 27 columns from 'by_minute' in batches of 20,000
    BY_MINUTE_SNAPSHOT: done in 7.0s
    BY_MINUTE_SNAPSHOT: verified 1,000,000 rows
    ```

3. A table that already has rows is left alone, so you can run the script again safely. To empty the tables and load them again, for example after a change of data, add `--truncate`. The documentation of `DBMS_CLOUD.COPY_DATA` describes a way to load from Object Storage on Autonomous AI Database; the script's header explains that alternative.

4. Check the row counts yourself, with SQL.

    ```
    <copy>
    python scripts/db_sql.py "select 'PLAYER_MATCH' as table_name, count(*) as row_count from player_match union all select 'BY_MINUTE_SNAPSHOT', count(*) from by_minute_snapshot"
    </copy>
    ```

    ```
            TABLE_NAME  ROW_COUNT
          PLAYER_MATCH     226910
    BY_MINUTE_SNAPSHOT    1000000
    ```

The two tables take about 350 MB in the database together with the vectors you create in Task 5.

## Task 3: Ask questions in SQL

Each question below is one statement. The commands use `scripts/db_sql.py`, which runs a statement as `LOLML` and prints the result; it works in bash, zsh and PowerShell. On Autonomous AI Database you can paste the same statements into the SQL worksheet of Database Actions.

1. Which position has the most vision and the fewest kills? Average the main stats by position.

    ```
    <copy>
    python scripts/db_sql.py "select team_position, count(*) as games, round(avg(kills), 1) as avg_kills, round(avg(deaths), 1) as avg_deaths, round(avg(assists), 1) as avg_assists, round(avg(vision_score), 1) as avg_vision
    from player_match
    where team_position is not null
    group by team_position
    order by avg_vision desc"
    </copy>
    ```

    ```
    TEAM_POSITION  GAMES  AVG_KILLS  AVG_DEATHS  AVG_ASSISTS  AVG_VISION
          UTILITY  45312        2.3         5.6         12.8        53.0
           JUNGLE  45313        6.9         5.4          8.0        24.1
           BOTTOM  45311        6.6         5.6          6.9        19.9
           MIDDLE  45311        6.6         5.6          6.4        19.4
              TOP  45310        5.3         5.6          5.7        18.2
    ```

    Supports (`UTILITY`) place wards and help: the most vision (53) and assists, the fewest kills (2.3). Each position has about the same number of games, because almost every match has one player per position on each team. For the same reason the win rate of every position is 50 %, so a win rate by position tells you nothing here.

2. Which champions have the highest win rate? Ignore champions with fewer than 500 games, whose win rate says little.

    ```
    <copy>
    python scripts/db_sql.py "select champion_name, count(*) as games, round(avg(win), 3) as win_rate
    from player_match
    group by champion_name
    having count(*) >= 500
    order by win_rate desc
    fetch first 5 rows only"
    </copy>
    ```

    ```
    CHAMPION_NAME  GAMES  WIN_RATE
            Swain    828     0.551
           Singed    500     0.542
       Cassiopeia    849     0.542
             Rell    732     0.537
          Karthus   1344     0.536
    ```

3. How do the stats of a player change during a game? Group the one million snapshots by ten-minute blocks.

    ```
    <copy>
    python scripts/db_sql.py "select floor(game_minute / 10) * 10 as from_minute,
           count(*) as snapshots,
           round(avg(max_health)) as avg_max_health,
           round(avg(attack_damage)) as avg_attack_damage,
           round(avg(ability_power)) as avg_ability_power
    from by_minute_snapshot
    group by floor(game_minute / 10) * 10
    order by from_minute"
    </copy>
    ```

    ```
     FROM_MINUTE  SNAPSHOTS  AVG_MAX_HEALTH  AVG_ATTACK_DAMAGE  AVG_ABILITY_POWER
               0     348139             795                 74                 13
              10     340343            1396                121                 43
              20     245260            1916                170                 87
              30      61768            2376                216                137
              40       4450            2729                255                192
              50         40            2916                328                393
    ```

    Health, attack damage and ability power grow steadily during a game. The last row comes from only 40 snapshots, so ignore its averages.

## Task 4: Ask in English with Select AI (Autonomous AI Database only)

> **Not run by the authors:** Select AI requires `DBMS_CLOUD_AI`, which only Autonomous AI Database has, so the authors could not run this task. On the Free container the authors saw only this: script `03a` ran (it uses a standard network package), and script `03b` failed at its first statement with `PLS-00201: identifier 'DBMS_CLOUD.DROP_CREDENTIAL' must be declared`. The steps below follow the Oracle documentation. If you use the Free container, skip to Task 5.

A **Select AI profile** is a stored object that names the language model, the credential to reach it and the tables the model may see. When you ask a question, the database sends the question and the metadata of those tables (names, columns and the comments from Task 1) to the model provider, gets SQL back and, depending on the action, runs it. The action `narrate` and other actions that explain results can send more than the metadata; read the Select AI documentation before you use them on data that is not public.

The profile uses OCI Generative AI through its OpenAI-compatible interface. You authenticate with an **OCI Generative AI API key** (a string that starts with `sk-`) and send it as a bearer token.

1. In the OCI Console, create an OCI Generative AI API key in a region that supports it and copy the secret; it is shown only once. Then create the IAM policy that lets the key call the service. The two pages [API Keys](https://docs.oracle.com/en-us/iaas/Content/generative-ai/api-keys.htm) and [Adding Key Permissions](https://docs.oracle.com/en-us/iaas/Content/generative-ai/add-api-permission.htm) list the supported regions and models and the policy statement. The key works only in the region where you created it.

2. Set the key, the region and a model that is available in that region. The documentation lists `meta.llama-3.3-70b-instruct`, `xai.grok-3` and `openai.gpt-oss-120b` as chat models for API keys; check the model's page for your region.

    <if type="linux">

    ```bash
    <copy>
    printf 'OCI Generative AI API key: '; read -rs LOLML_GENAI_KEY; echo; export LOLML_GENAI_KEY
    export LOLML_GENAI_REGION=<region of the key, for example us-chicago-1>
    export LOLML_GENAI_MODEL=<model, for example meta.llama-3.3-70b-instruct>
    </copy>
    ```

    </if>

    <if type="windows">

    ```powershell
    <copy>
    $env:LOLML_GENAI_KEY = [System.Net.NetworkCredential]::new('', (Read-Host 'OCI Generative AI API key' -AsSecureString)).Password
    $env:LOLML_GENAI_REGION = '<region of the key, for example us-chicago-1>'
    $env:LOLML_GENAI_MODEL = '<model, for example meta.llama-3.3-70b-instruct>'
    </copy>
    ```

    </if>

3. Run the Select AI scripts.

    ```
    <copy>
    python scripts/db_setup.py 03
    </copy>
    ```

    Script `03a_select_ai_acl.sql` runs as the administrator. The database blocks outgoing network calls by default, so it adds a network access entry that lets `LOLML` reach `inference.generativeai.<region>.oci.oraclecloud.com`. Script `03b_select_ai_profile.sql` runs as `LOLML`. It stores your key as the password of a credential named `LOL_GENAI_CRED` (the key is read from the environment and is not written into any file), and creates the profile `LOL_GENAI` with provider `openai`, the OpenAI-compatible endpoint of OCI Generative AI as `provider_endpoint`, your model, and the tables `PLAYER_MATCH` and `BY_MINUTE_SNAPSHOT`. It then prints the profile and asks two questions with the `showsql` action.

4. Ask your own question and look at the SQL the model writes. With `showsql` nothing runs; with `runsql` the result comes back, and with `narrate` the model describes the result in words.

    ```
    <copy>
    python scripts/db_sql.py "select dbms_cloud_ai.generate(prompt => 'Which champion has the highest win rate among champions with at least 500 games?', profile_name => 'LOL_GENAI', action => 'showsql') as generated_sql from dual"
    </copy>
    ```

    You should get one `SELECT` statement over `PLAYER_MATCH`. Because a language model writes it, the wording differs from run to run. Compare it with the statement you wrote yourself in Task 3: if the model's answer differs from your result, the SQL is where to look.

5. In Database Actions (SQL worksheet) you can use the shorter form. Select AI keeps the profile per session, so set it first.

    ```
    <copy>
    exec dbms_cloud_ai.set_profile('LOL_GENAI');
    select ai runsql which champion has the highest win rate among champions with at least 500 games;
    </copy>
    ```

> **Note:** If the profile script stops with an error about the endpoint or the path, check the `provider_endpoint` line in `sql/03b_select_ai_profile.sql`. The documentation of the OpenAI-compatible interface gives the base URL `https://inference.generativeai.<region>.oci.oraclecloud.com/openai/v1`; the API key page also gives `.../20231130/actions/v1`. The script's header says how to switch.

## Task 5: Find games like this one

Every player-game becomes a point in a 12-dimensional space. The dimensions are kills, deaths, assists, gold, damage dealt and taken, minions, vision, objective damage, turret damage, crowd control and healing or shielding, each **per minute** and then z-scored (centred and divided by the standard deviation). Per minute means a short stomp and a long game of the same playstyle land close together. The vector is stored in a column of type `VECTOR(12, FLOAT32)`, and the nearest neighbours are found with `VECTOR_DISTANCE`, using cosine distance.

1. Build the embeddings and the vector index.

    ```
    <copy>
    python scripts/db_vectors.py
    </copy>
    ```

2. The script embeds 95 % of the matches in Python and stores the mean and standard deviation it used, in the table `VECTOR_NORM`, so that a new game can be embedded in exactly the same way. It keeps 5 % of the matches out: games the database has never seen. Then it creates an **HNSW** index, which searches a graph of neighbours held in memory. The authors saw this; the index step took between 38 and 55 seconds in their runs:

    ```
    embedded 216,310 games (10,600 held out) into 12 dimensions in 0.2s
    inserted 216,310 vectors in 2.2s
      | created HNSW vector index player_game_vec_hnsw
    vector index step took 54.6s
    ```

    The step also prints a sample query: the games closest to the first stored game. If the database has no vector memory, the script says `HNSW not available` and creates an **IVF** index instead; on a Free container without the startup script of Lab 9, the authors saw `ORA-51962: The vector memory area is out of space for the current container.` and the IVF index worked.

3. Pick a held-out game, embed it, and ask the database for the five most similar stored games.

    ```
    <copy>
    python scripts/db_similar.py --seed 7
    </copy>
    ```

    ```
    query game: PLAYER_MATCH_ID=216306  Chogath TOP  2/10/5  win=0  33.1 min  (held out: True)
    embedding: [-0.9, 0.93, -0.71, -0.55, -0.74, 2.21, 0.12, 0.05, -0.63, -0.45, 1.97, -0.38]

    nearest 5 games (vector index, 9 ms):
    champion_name team_position  win    kda  cosine_distance
          Chogath           TOP    0 4/10/5           0.0250
             Ornn        MIDDLE    0  2/9/8           0.0329
             Sion           TOP    0  1/8/4           0.0353
          Chogath        MIDDLE    0 2/15/7           0.0375
              Zac        JUNGLE    0  1/7/2           0.0474

    exact scan: 196 ms, index result overlaps the exact top-5 in 5/5 rows
    win rate of these 5 similar games: 0%
    ```

    The query game is a Cho'Gath who took a lot of damage (2.21 standard deviations above the average) and crowd-controlled a lot. The neighbours are tanks with the same kind of game, and all five lost as well. Use `--seed` with other numbers for other games.

4. The statement behind it is plain SQL. The `:q` placeholder holds the embedding of the new game, and `FETCH APPROX` lets the optimizer answer from the vector index. The index answered in 9 ms and the exact scan of all 216,310 vectors took 196 ms. The index result matched the exact result in 5 of 5 rows for this game; an approximate index can miss neighbours, which the *target accuracy* of the index controls.

    ```
    SELECT pm.champion_name, pm.team_position, pm.win,
           pm.kills || '/' || pm.deaths || '/' || pm.assists AS kda,
           ROUND(VECTOR_DISTANCE(v.embedding, :q, COSINE), 4) AS cosine_distance
    FROM   player_game_vector v
    JOIN   player_match pm ON pm.player_match_id = v.player_match_id
    ORDER  BY VECTOR_DISTANCE(v.embedding, :q, COSINE)
    FETCH APPROX FIRST 5 ROWS ONLY
    ```

## Task 6: Score rows with your model inside the database (optional)

Train in Python, export to ONNX, and let the database run the model. The scores come back next to the data with `PREDICTION` and `PREDICTION_PROBABILITY`, and no row leaves the database. The authors ran this task on the Free container. Autonomous AI Database documents the same import procedure, `DBMS_DATA_MINING.IMPORT_ONNX_MODEL`, but the authors did not run it there.

1. Train the classifier on the post-game stats that are columns of `PLAYER_MATCH` and export it. The script uses the honest post-game features of the leakage lab (the outcome columns are left out), trains on the matches that were not held out, and checks that `onnxruntime` reproduces scikit-learn.

    ```
    <copy>
    python scripts/export_onnx.py
    </copy>
    ```

    ```
    trained gradient boosting (100 trees, depth 3) on 216,310 rows x 25 features in 49.6s
    hold-out (10,600 rows of matches never used for training): accuracy 0.8594  AUC 0.9408
    onnxruntime vs scikit-learn on the hold-out: max |difference| = 1.18e-07
    wrote .../models/lol_win_postgame.onnx (54 KB) and lol_win_postgame.json
    ```

    The database accepts a model with one float32 input of shape `[rows, features]` and a classifier whose second output is a plain table of probabilities. That is why the script turns ZipMap off. It uses gradient boosting rather than `HistGradientBoostingClassifier`, because `skl2onnx` 1.20 failed to convert that one with `TypeError: Field onnx.AttributeProto.ints: Expected an int, got a boolean`.

2. Import the model and check it.

    ```
    <copy>
    python scripts/db_score.py
    </copy>
    ```

    The script stores the file as a BLOB in the table `ONNX_MODEL_STAGE` and calls `DBMS_DATA_MINING.IMPORT_ONNX_MODEL`. The import metadata names the columns of `PLAYER_MATCH` that feed the model's input and declares the classes `0` and `1`; without `"labels": [0, 1]`, Oracle numbers the classes 1 and 2 and `PREDICTION_PROBABILITY(model, 1 ...)` would be the probability of a loss. Then it fetches the hold-out rows, scores them once with SQL and once with `onnxruntime` on the same numbers, and compares:

    ```
    imported LOL_WIN_POSTGAME in 0.33s
          model_name mining_function algorithm
    LOL_WIN_POSTGAME  CLASSIFICATION      ONNX

    parity on 10,600 hold-out rows of PLAYER_MATCH:
      SQL PREDICTION_PROBABILITY vs onnxruntime: max |diff| = 2.38e-07, mean 2.37e-08
      predicted labels identical: 100.00%
      accuracy in SQL 0.8594   in Python 0.8594   (sklearn hold-out accuracy recorded at export: 0.8594)
      time: SQL fetch+score 0.44s, onnxruntime 0.006s
      parity OK
    ```

    The database and Python agree to within float32 rounding: the largest difference between the two probabilities is 0.000000238, and every predicted label is identical.

3. Score a hold-out match in SQL. `USING *` takes the model's inputs from the columns of the table by name.

    ```
    <copy>
    python scripts/db_sql.py "select champion_name, team_position, kills || '/' || deaths || '/' || assists as kda, round(prediction_probability(lol_win_postgame, 1 using *), 3) as p_win, win
    from player_match
    where match_id = (select min(match_id) from player_match where mod(match_id, 20) = 7)
    order by team_id, team_position"
    </copy>
    ```

    ```
    CHAMPION_NAME TEAM_POSITION    KDA  P_WIN  WIN
           Twitch        BOTTOM  0/5/3  0.084    0
          Skarner        JUNGLE  1/2/2  0.157    0
           Viktor        MIDDLE  2/8/2  0.017    0
       Tryndamere           TOP  2/7/4  0.026    0
             Sona       UTILITY  1/2/3  0.102    0
           Ezreal        BOTTOM  3/0/3  0.948    1
          Karthus        JUNGLE 10/2/7  0.957    1
              Zed        MIDDLE  4/1/4  0.969    1
           Darius           TOP  7/3/6  0.941    1
           Renata       UTILITY  0/0/9  0.964    1
    ```

4. Compute the accuracy of the model over all hold-out games in one SQL statement.

    ```
    <copy>
    python scripts/db_sql.py "select count(*) as holdout_rows, round(avg(case when prediction(lol_win_postgame using *) = win then 1 else 0 end), 4) as accuracy_in_sql from player_match where mod(match_id, 20) = 7"
    </copy>
    ```

    ```
     HOLDOUT_ROWS  ACCURACY_IN_SQL
            10600           0.8594
    ```

> **Note:** This model explains games that are over: it reads the final kills, gold and damage, so it cannot tell you who will win a game in progress. The leakage lab explains why that difference matters. The live model of the by-minute data (`python scripts/export_onnx.py --model live`, then `python scripts/db_score.py --model live`) predicts during the game; on the authors' run its accuracy on held-out matches was 0.576, and the SQL in `sql/05c_onnx_score.sql` shows that it improves as the game goes on.

You have loaded the games into Oracle AI Database 26ai, queried them, searched them by similarity and scored them with a model that runs inside the database. The next lab, Clean Up, removes what you created.

You may now **proceed to the next lab**.

## Learn More

* [Oracle AI Vector Search User's Guide](https://docs.oracle.com/en/database/oracle/oracle-database/26/vecse/)
* [JSON Metadata Parameters for ONNX Models](https://docs.oracle.com/en/database/oracle/oracle-database/26/vecse/json-metadata-parameters-onnx-models.html)
* [Examples of Using ONNX Models](https://docs.oracle.com/en/database/oracle/machine-learning/oml4sql/23/dmapi/examples-using-onnx-models.html)
* [Manage AI Profiles (Select AI)](https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/select-ai-manage-profiles.html)
* [DBMS_CLOUD_AI Package](https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/dbms-cloud-ai-package.html)
* [OCI Generative AI API Keys](https://docs.oracle.com/en-us/iaas/Content/generative-ai/api-keys.htm)

## Acknowledgements

* **Author** - Ignacio Martinez, Oracle
* **Last Updated By/Date** - Ignacio Martinez, October 2026
