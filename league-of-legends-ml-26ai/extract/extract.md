# Extract Your Own Matches from the Riot API (Optional)

## Introduction

The workshop ships with a dataset of ranked League of Legends matches, so the core path needs no Riot account and no API key. This lab is optional. Do it if you want to see where such data comes from, or if you want to run the notebooks on fresh matches that you downloaded yourself.

Riot changed its API in a way that broke most older tutorials. On 2025-07-07 Riot removed `summonerId` and `accountId` from its API payloads. Everything now keys on the **PUUID**, a long opaque player identifier. The extraction in this lab follows the current flow:

1. **league-v4** lists the players of an apex tier (Challenger, Grandmaster or Master). Each entry carries a `puuid`.
2. **match-v5** `by-puuid/{puuid}/ids` returns the recent ranked match ids of one player.
3. **match-v5** `matches/{matchId}` returns the full result of one match, with 10 participants.

The scripts store the matches in a local SQLite file, then convert them to the same table the notebooks already use, so notebook 01 can run on your own file.

Estimated Time: 20 minutes

### Objectives

In this lab, you will:

* Get a Riot development API key and keep it out of files and logs
* Check that the key works
* Download a small set of ranked matches into SQLite
* Convert them to the workshop's performance table and run notebook 01 on them

### Prerequisites

This lab assumes you have:

* Completed Lab 1, so the repository is cloned and its Python environment is active
* A terminal inside the `lol-ml-26ai` folder of the cloned `leagueoflegends-optimizer` repository
* A Riot Games account (free) to sign in to the Riot developer portal

> **Note:** This lab was tested against a local mock of the Riot API only, not against the live Riot API. The mock is `scripts/mock_riot.py`, and `tests/test_riot_mock.py` runs the whole extraction against it. The mock serves the three endpoints above, checks the `X-Riot-Token` header (401 when missing, 403 when wrong) and can answer one request with a 429 and a `Retry-After` header. Its match data are real rows copied from the workshop sample, with made-up players and ids. It does not copy Riot's real rate limits or timing. If Riot's live responses differ from what this lab shows, the Troubleshooting task tells you what to check.

> **Riot's rules:** Development keys are for personal and learning use. A product or public service needs a registered application and a production key from Riot. Do not publish players' identifiers (PUUIDs, names or Riot IDs) from your extraction, and do not commit the SQLite file.

## Task 1: Get a development key and keep it secret

A development key is a token that Riot issues to your account. It expires after 24 hours, and it is limited to 20 requests per second and 100 requests per 2 minutes. The scripts stay below both limits on their own.

1. Open [developer.riotgames.com](https://developer.riotgames.com) and sign in with your Riot account.

2. On the dashboard, accept the terms if asked and click **Regenerate API Key**. Copy the key, which starts with `RGAPI-`. You need a new key every 24 hours.

3. In your terminal, read the key into an environment variable. The `-s` option keeps the key from being shown on screen, and nothing is written to a file or to your shell history.

    ```bash
    <copy>
    printf 'Riot API key: '; read -rs RIOT_API_KEY; echo; export RIOT_API_KEY
    </copy>
    ```

    Paste the key and press **Enter**. The variable only exists in this terminal window.

    On Windows, run the commands in WSL2, or set the variable in PowerShell with `$env:RIOT_API_KEY="RGAPI-..."`. The Python commands are identical.

> **Note:** Never put the key in a notebook, a script or a Git commit, and never paste it into a chat or an issue. If it leaks, regenerate it on the portal; the old key stops working.

## Task 2: Check the key

Before downloading anything, test the key with one cheap request. `check_riot.py` reads `RIOT_API_KEY`, calls the platform status endpoint once and never prints the key.

1. Run the check.

    ```bash
    <copy>
    python scripts/check_riot.py
    </copy>
    ```

2. You should see this line and an exit code of 0.

    ```
    PASS: Riot API key accepted
    ```

    If you see `FAIL` instead, the message says why. Task 5 explains each reason.

3. Other regions use the same command with `--platform`, for example `--platform na1` or `--platform kr`. The default is `euw1`.

## Task 3: Download a small set of matches

`extract_live.py` lists the best players of a tier by league points, gets each player's recent ranked solo/duo match ids, removes duplicates and downloads each match once. It is safe to stop and run again, because matches that are already stored are not downloaded twice.

1. Run a small extraction: the top 5 Challenger players of EUW and their last 10 ranked matches each.

    ```bash
    <copy>
    python scripts/extract_live.py --platform euw1 --tier challenger --players 5 --matches 10 --db riot_extract.db
    </copy>
    ```

2. The script prints its progress and a summary. The numbers in this example come from the mock, so yours will differ.

    ```
    5 challenger players
    50 new match ids (50 to download)
      10/50 matches, 100 performance rows
      ...
      50/50 matches, 500 performance rows
    players 5  matches downloaded this run 50 (skipped 0)  new performance rows 500  API calls 57  429s 1
    riot_extract.db now holds 50 matches and 500 performance rows
    ```

    Each match gives 10 rows, one per player, so rows are always matches times 10. The script stops with a non-zero exit code and a message if the key is rejected or if no new rows were written.

3. Work out how long a bigger run takes. The number of API calls is `1 + players + matches`, where `matches` is the number of different match ids (players who played together share matches, so it can be lower than `players × matches per player`). The client waits whenever it would exceed 20 calls per second or 100 calls per 2 minutes. This is arithmetic from that limiter with no network delay, not a measurement:

    | Players | Matches each | Calls (at most) | Waiting caused by the limits |
    | --- | --- | --- | --- |
    | 5 | 10 | 56 | about 2 seconds |
    | 10 | 10 | 111 | about 2 minutes |
    | 20 | 10 | 221 | about 4 minutes |
    | 20 | 20 | 421 | about 8 minutes |

    Real requests also take time to answer, so expect somewhat longer. Start small.

4. Optional: look at what was stored. The SQLite file holds the PUUIDs, so keep it private.

    ```bash
    <copy>
    python -c "import sqlite3; c = sqlite3.connect('riot_extract.db'); print(c.execute('select championName, teamPosition, kills, deaths, assists, win from performance limit 5').fetchall())"
    </copy>
    ```

## Task 4: Convert to the workshop table and run notebook 01

`export_extract.py` turns the SQLite table into a parquet file with the same columns and types as the workshop dataset. It uses the numeric part of the match id (for example `EUW1_7234567890` becomes `7234567890`), drops the PUUID, computes `duration` in minutes from `timePlayed`, recomputes `calculated_player_performance` with the formula of the original project and keeps only complete matches (10 players, 5 winners, at least 5 minutes).

1. Export your matches.

    ```bash
    <copy>
    python scripts/export_extract.py --db riot_extract.db
    </copy>
    ```

    The file is written to `data/my_matches.parquet` (inside `$LOLML_HOME` if you set that variable). The script prints the number of rows and matches.

2. Point the notebooks at your file. When `LOLML_PERFORMANCE_FILE` is set, `lolml.data` loads it instead of the downloaded dataset.

    ```bash
    <copy>
    export LOLML_PERFORMANCE_FILE="${LOLML_HOME:-$PWD}/data/my_matches.parquet"
    </copy>
    ```

3. Start Jupyter from the same terminal, so that it sees the variable, and open notebook 01.

    ```bash
    <copy>
    jupyter lab notebooks/01_explore_and_baseline.ipynb
    </copy>
    ```

4. Run all cells. The first cell with data prints the size of your file, for example `1,000 rows, 100 matches`. When you finish, remove the variable to go back to the full dataset.

    ```bash
    <copy>
    unset LOLML_PERFORMANCE_FILE
    </copy>
    ```

## Task 5: What to expect and troubleshooting

Your own file is tiny next to the dataset of about 227,000 player rows, so treat the notebook results as an exercise in the workflow, not as a benchmark. The test set holds only a few dozen matches, so accuracy and AUC move by several points from one split to the next, and the draft-only model stays near a coin flip. Players of one apex tier also look alike, so the score levels differ from the full dataset. Download more players and matches before you read anything into a number.

If something fails, find the message below.

| Message | Cause and fix |
| --- | --- |
| `RIOT_API_KEY is not set` | The variable only lives in the terminal where you ran Task 1. Run the `read -rs` command again in the terminal you use now. |
| `HTTP 401` | No key was sent. The variable is empty. |
| `HTTP 403 ... invalid or EXPIRED` | The key is wrong, was regenerated or is older than 24 hours. Regenerate it on the portal and repeat Task 1. A 403 on one endpoint only can mean that the key is not allowed to use it. |
| `HTTP 429 ... Retry-After` | You are over a rate limit, possibly because another program uses the same key. The client waits for the time Riot asks and retries up to 6 times, then stops with a rate-limit error. Wait a few minutes and run the script again. |
| `no new rows were written` | Everything was already stored, the tier list was empty, or the matches were not ranked solo/duo. Raise `--players` or `--matches`, or use another `--tier` or `--platform`. |
| `FAIL: cannot reach ...` | No network, or a proxy or firewall blocks `api.riotgames.com`. |
| `unknown platform` | Use a platform code such as `euw1`, `eun1`, `na1`, `kr`, `br1` or `oc1`. |

Because this lab was only tested against the mock, these are the points to check first if the live API behaves differently: the field names in the league-v4 and match-v5 answers (compare with Riot's API reference), the key and the status code of the error, and whether Riot changed a limit. The error messages name the failing call, so you can find it in the reference.

You may now **proceed to the next lab**.

## Learn More

* [Riot Developer Portal](https://developer.riotgames.com)
* [Riot API policies and rate limits](https://developer.riotgames.com/docs/portal)
* [League of Legends API reference (match-v5, league-v4)](https://developer.riotgames.com/apis)

## Acknowledgements

* **Author** - Ignacio Martinez, Oracle
* **Last Updated By/Date** - Ignacio Martinez, October 2026
