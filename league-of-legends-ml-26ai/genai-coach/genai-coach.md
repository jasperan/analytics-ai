# An AI Coach with OCI Generative AI (Optional)

## Introduction

A statistical model tells you **how likely** a team was to win. It cannot tell you **what to work on**. A large language model can, but only if you hand it the facts and forbid it from inventing anything. In this lab you combine both: the win probability of the post-game model, plus the percentile of a player's per-minute stats compared with everybody else in the same position, become a prompt for an OCI Generative AI chat model.

The coach is a **post-game** report on purpose. It reads a finished game from the dataset and never touches a running match, which keeps it clear of Riot's restrictions on in-game advice.

Estimated Time: 15 minutes

### Objectives

In this lab, you will:

* Turn a finished game into per-minute stats and percentiles against players in the same position
* Build a constrained prompt from model output and statistics
* Create an OCI Generative AI API key and call a chat model with it
* Run the whole notebook without a key and inspect the prompt

### Prerequisites

This lab assumes you have:

* Completed Labs 1, 2 and 4
* For Task 4 only: an Oracle Cloud tenancy where OCI Generative AI is available, and permission to create an API key and a policy

## Task 1: Build the prompt

1. Open `notebooks/07_genai_coach.ipynb` and run all cells. The notebook trains the post-game model from Lab 4, picks one game from the test matches, and prints the model's view of it.

    ```text
    Karma UTILITY, 32 min, actual result: loss, model: 2% win
    ```

2. Read the percentile table. For each of six per-minute stats it computes the share of players in the same position (support, here) who did worse. For deaths, fewer is better, so the percentile is inverted.

    ```text
                                 stat   value  percentile
    0               deaths per minute    0.31          15
    1      kills + assists per minute    0.47          39
    2                 gold per minute  226.15          10
    3  damage to champions per minute  257.76          44
    4       minions killed per minute    0.19           4
    5         vision score per minute    1.94          52
    ```

    A percentile of 100 is the best among players in the same position. The chosen game is a loss with many deaths and little gold.

3. Read the prompt that the notebook builds from the table.

    ```text
    [system]
    You are a concise League of Legends coach. Use ONLY the numbers you are given. Do not invent events, items or objectives that are not in the data. Reply with a two-sentence summary and exactly three bullet points of advice, most important first.

    [user]
    Game: Karma (UTILITY), 32 minutes, result: loss.
    A statistical model that sees the final stats gave this player's team a 2% chance of winning.
    Per-minute stats against other UTILITY players:
    - deaths per minute: 0.31 (better than 15 % of UTILITY players)
    ...
    ```

    The system message is the guardrail. A language model that does not see the timeline of the game would otherwise happily make up "you were caught out of position at the dragon". The facts, and only the facts, are in the user message.

4. The last cell prints `LOLML_GENAI_KEY is not set: skipping the model call`. The notebook is complete without a key, and the prompt it prints is exactly what would be sent.

## Task 2: Create an OCI Generative AI API key

> **Note:** The OCI steps in Tasks 2 to 4 were not executed while this workshop was written, because they create cloud resources and send data to a model. The code that builds the request was tested against a local stub server that checks the URL, the `Authorization` header and the JSON body.

An OCI Generative AI API key is a secret that identifies you to the Generative AI service. It is **not** the same as an OCI IAM API key.

1. In the OCI console, choose a region where Generative AI API keys are supported (for example **US Midwest (Chicago)** or **Germany Central (Frankfurt)**; the list is in [API Keys](https://docs.oracle.com/en-us/iaas/Content/generative-ai/api-keys.htm)).

2. Open the navigation menu, click **Analytics & AI**, and then click **Generative AI**. Open the **API keys** list page and click **Create API key** (see [Creating an API Key](https://docs.oracle.com/en-us/iaas/Content/generative-ai/create-api-key.htm) if the menu differs in your console).

3. Enter a name (for example `lol-coach`), keep the default compartment, name the two secrets (`key-one` and `key-two`), optionally set an expiration date, and click **Create**. Copy the secret that starts with `sk-` now: it is shown only once.

4. Add a permission for the key. Without it, calls fail with an authorization error. Ask your tenancy administrator to add this policy, with your compartment name (see [Adding Key Permissions](https://docs.oracle.com/en-us/iaas/Content/generative-ai/add-api-permission.htm) for narrower variants):

    ```text
    <copy>
    allow any-user to use generative-ai-family in compartment <compartment-name> where ALL {request.principal.type='generativeaiapikey'}
    </copy>
    ```

## Task 3: Configure the notebook

1. Put the key in your environment **before you start JupyterLab**, in the terminal that you will start it from. The key stays in memory and is never written to the notebook. `LOLML_GENAI_REGION` defaults to `us-chicago-1`; set it to your key's region. `LOLML_GENAI_MODEL` defaults to `xai.grok-3`; set another chat model that is available in that region if you prefer, for example `meta.llama-3.3-70b-instruct`.

    ```bash
    <copy>
    read -rs LOLML_GENAI_KEY && export LOLML_GENAI_KEY
    export LOLML_GENAI_REGION=us-chicago-1
    </copy>
    ```

    On Windows PowerShell 7, use `$env:LOLML_GENAI_KEY = Read-Host -MaskInput "key"` and `$env:LOLML_GENAI_REGION = "us-chicago-1"`.

2. Start JupyterLab from that terminal (`python -m jupyterlab notebooks`), open `07_genai_coach.ipynb` and run all cells again. The last cell now sends the prompt to `https://inference.generativeai.<region>.oci.oraclecloud.com/20231130/actions/v1/chat/completions` with your key as a bearer token and prints the answer.

## Task 4: Judge the answer

1. Read the reply against the table. Does each bullet follow from a number in the prompt? A good answer points at the low gold and minion numbers and at the deaths, in that order of importance, and does not mention anything that is not in the data.

2. Change the game (`random_state` in `test.sample(1, random_state=11)`) or edit the system message, and compare. This is **prompt engineering**: the model is the same, and the quality of the answer comes from what you give it and what you forbid.

3. Think about what you would add next: the neighbours found by AI Vector Search in Lab 10 ("games like this one, and how they ended"), or the answers of Select AI to questions about the champion. They would become more lines in the prompt, retrieved by the database instead of typed by you.

## Learn More

* [OCI Generative AI: API Keys](https://docs.oracle.com/en-us/iaas/Content/generative-ai/api-keys.htm)
* [OCI Chat Completions API](https://docs.oracle.com/en-us/iaas/Content/generative-ai/chat-completions-api.htm)
* [Oracle Generative AI service](https://docs.oracle.com/en-us/iaas/Content/generative-ai/home.htm)

## Acknowledgements

* **Author** - Ignacio Martinez, Oracle
* **Last Updated By/Date** - Ignacio Martinez, October 2026
