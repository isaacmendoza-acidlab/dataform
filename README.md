# dataform
# PoC Embeddings

[![pre-commit](https://img.shields.io/badge/pre--commit-enabled-brightgreen?logo=pre-commit)](https://github.com/pre-commit/pre-commit)

## install pre-commit hooks

This repository uses [pre-commit](https://pre-commit.com) to ensure that your code is formatted correctly and that the
minimum quality standards are met.

So, before committing any changes:

* Install the `pre-commit` library in your environment (if not already installed)

```bash
# install pre-commit (required)
pip install --upgrade --no-cache-dir pre-commit
```

* Run the following commands at the root of the repository, where the `.pre-commit-config.yaml` file is located

  * install the `pre-commit` hooks

    ```bash
    # install the pre-commit hooks (required)
    pre-commit install
    ```

  * install the environments for the `pre-commit` hooks

    ```bash
    # install environments for all available hooks now (optional)
    # otherwise they will be automatically installed when they are first executed
    pre-commit install-hooks
    ```

  * run the `pre-commit` hooks on all files

    ```bash
    # run the hooks on all files with the current configuration (optional)
    # this is what will happen automatically
    pre-commit run --all-files
    ```

The following are optional commands that can be run at any time if needed:

* Auto-update the `pre-commit` hooks (optional)

    ```bash
    # auto-update the version of the hooks (optional)
    pre-commit autoupdate
    ```

* Installs hook environments overriding existing environments (optional)

    ```bash
    # idempotently replaces existing git hook scripts with pre-commit, and also installs hook environments (optional)
    pre-commit install --install-hooks --overwrite
    ```

* Run individual hooks (optional)

    ```bash
    # run individual hooks with the current configuration (optional)
    pre-commit run <hook_id>
    ```

* Store "frozen" hashes of hook repositories in the configuration file (optional)

    ```bash
    # store "frozen" hashes in rev instead of tag names (optional)
    pre-commit --freeze
    ```

    ```bash
    # alternatively, use autoupdate to update the revs in the config file to the latest versions of the hooks
    # and store "frozen" hashes in rev instead of tag names
    pre-commit autoupdate --freeze
    ```

* Hard clean-up of the local repository (optional)

    ```bash
    # Hard cleanup of the local repository (optional)
    pre-commit uninstall
    pre-commit clean
    pre-commit gc
    pre-commit autoupdate
    pre-commit install
    pre-commit install-hooks --overwrite
    pre-commit run --all-files
    pre-commit gc
    ```

## References

* Text Embedding Semantic Search:
  * <https://cloud.google.com/bigquery/docs/text-embedding-semantic-search#sql>

* BigQueryML:
  * <https://cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-create-kmeans>

* BigQuery Syntax for `CREATE MODEL`:
  * <https://cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-create-remote-model>

  * Connection Requirement
    * <https://cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-create-remote-model#connection>

* BigQuery Syntax for `GENERATE_TEXT_EMBEDDING`
  * <https://cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-generate-text-embedding>

* BigQueryML Syntax for `ML.DISTANCE`
  * : <https://cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-distance>

## Setup

NOTE: The following are the values of PoC experiment. Please change them accordingly.

```bash
REGION=US
PROJECT_ID=ai-lab-plygrd-9d5c
CONNECTION_NAME=semantic_search_connection
DATASET_NAME=semantic_search
DATASET_ID=ai-lab-plygrd-9d5c.semantic_search
MODEL_NAME=embedding_model

TEXT_DATA_TABLE_NAME=lisa_hackathons
QUESTIONS_EMBEDDING_TABLE_NAME=questions_embedding
ANSWER_EMBEDDING_TABLE_NAME=answers_embedding
```

## Experiment

* Create a connection `semantic_search_connection`

  1. Go to the `BigQuery` page (<https://console.cloud.google.com/bigquery?project=ai-lab-plygrd-9d5c>)
  2. Go to `BigQuery`
  3. To create a connection, click add `+ Add data`, and then click `Connections to external data sources`.
  4. In the `Connection type` list, select `BigLake and remote functions (Cloud Resource)`.
  5. In the `Connection ID` field, enter a name for your connection
  6. Click `Create` connection (`semantic_search_connection`)
  7. Click `Go to connection`.
  8. In the `Connection info pane`, copy the `service account ID` for use in a later step.

```bash
EXTERNAL_CONNECTION_NAME: semantic_search_connection

SERVICE_ACCOUNT_ID: bqcx-421316969106-cv9o@gcp-sa-bigquery-condel.iam.gserviceaccount.com

DATASET_NAME: semantic_search

DEFAULT_ANSWER: "No puedo encontrar una respuesta ya que la pregunta no está relacionada con los temas de Ayuda de este Chat. \nPuedes probar lo siguiente,\nSé más específico en la pregunta, agrega palabras que estén relacionadas con el tema en cuestión que me ayuden a encontrar una respuesta\nSi aún no puedo responderte bien, puedes llamar a la Mesa de Ayuda Latam del Portal IT."
```

* Give the service account access

1. Go to the `IAM & Admin` page (<https://console.cloud.google.com/iam-admin/iam?project=ai-lab-plygrd-9d5c>)
2. Click `+ Grant access`.
   1. The Add principals dialog opens.
3. In the `New principals` field, enter the `service account ID` that you copied earlier.
4. In the `Select a role` field, select `Vertex AI`, and then select `Vertex AI User`.
5. Click `Save`.

* Create dataset

Create a dataset named `semantic_search` to store the tables and models that you create

```sql
CREATE SCHEMA `ai-lab-plygrd-9d5c.semantic_search`;
```

* Create a model

Create a remote model, specifying `CLOUD_AI_TEXT_EMBEDDING_MODEL_V1` ( `textembedding-gecko` ) for `REMOTE_SERVICE_TYPE`

```sql
CREATE OR REPLACE MODEL semantic_search.embedding_model
REMOTE WITH CONNECTION `ai-lab-plygrd-9d5c.us.semantic_search_connection`
OPTIONS (REMOTE_SERVICE_TYPE = 'CLOUD_AI_TEXT_EMBEDDING_MODEL_V1');
```

* Materialize text data

**NOTE:** The http status value is wrongly stored as 500 for successful requests (instead of 200)

```sql
CREATE OR REPLACE TABLE semantic_search.lisa_hackathons AS (
    WITH
        -- Create a Common Table Expression (CTE) for the "1st Hackathon LISA [Google Chat]"
        FirstHackathonLisaGoogleChat AS (
            SELECT
                -- *
                user AS question,   -- STRING
                gpt AS answer,      -- STRING
                email,              -- STRING
                creation_time,      -- TIMESTAMP
                http_status         -- INTEGER
            FROM
                `ai-lab-plygrd-9d5c.logs_chat_google_lisa.google_chat_lisa_backend_report`  -- 1st Hackathon LISA [Google Chat]
        ),
        -- Create a Common Table Expression (CTE) for the "2nd Hackathon LISA [Dialogflow]"
        SecondHackathonLisaDialogflow AS (
            SELECT
                -- *
                question,       -- STRING
                answer,         -- STRING
                email,          -- STRING
                session_id,     -- STRING
                creation_time   -- TIMESTAMP
            FROM
                `ai-lab-plygrd-9d5c.logs_chat_google_lisa.logs_hackaton_v2_clean`           -- 2nd Hackathon LISA [Dialogflow]
        ),
        -- Create a Common Table Expression (CTE) for the "Concatenated Tables"
        ConcatenatedTables AS (
            SELECT
                -- Select and transform data from the CTE: FirstHackathonLisaGoogleChat
                REPLACE(CAST(question AS STRING), '"', '') AS question,
                REPLACE(CAST(answer AS STRING), '"', '') AS answer,
                creation_time,
                '1st Hackathon LISA [Google Chat]' AS origin    -- Add an "origin" column to indicate the source table
            FROM
                FirstHackathonLisaGoogleChat
            WHERE
                http_status IN (200, 500)                       -- Filter rows with http_status of 200 or 500

            UNION ALL   -- Concatenate the tables using UNION ALL to keep duplicate rows

            SELECT
                -- Select and transform data from the CTE: SecondHackathonLisaDialogflow
                REPLACE(CAST(question AS STRING), '"', '') AS question,
                REPLACE(CAST(answer AS STRING), '"', '') AS answer,
                creation_time,
                '2nd Hackathon LISA [Dialogflow]' AS origin     -- Add an "origin" column to indicate the source table
            FROM
                SecondHackathonLisaDialogflow
        )

    SELECT
        question,       -- STRING
        answer,         -- STRING
        creation_time,  -- TIMESTAMP
        origin          -- STRING
    FROM
        ConcatenatedTables
);
```

* Embed text

  * <https://cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-generate-text-embedding>

Create a new table that contains the text embedding of (user) questions by using the `ML.GENERATE_TEXT_EMBEDDING` function.

```sql
CREATE OR REPLACE TABLE semantic_search.questions_embedding AS (
    SELECT *
    FROM ML.GENERATE_TEXT_EMBEDDING(
        MODEL semantic_search.embedding_model,
        (
            -- It must be a single column with the name "content" and the type STRING
            SELECT question AS content  -- CAST(question AS STRING) AS content
            FROM semantic_search.lisa_hackathons
        ),
        STRUCT(TRUE AS flatten_json_output)
    )
);
```

Create a new table that contains the text embedding of (bot) answers by using the `ML.GENERATE_TEXT_EMBEDDING` function.

```sql
CREATE OR REPLACE TABLE semantic_search.answers_embedding AS (
    SELECT *
    FROM ML.GENERATE_TEXT_EMBEDDING(
        MODEL semantic_search.embedding_model,
        (
            -- It must be a single column with the name "content" and the type STRING
            SELECT answer AS content  -- CAST(answer AS STRING) AS content
            FROM semantic_search.lisa_hackathons
        ),
        STRUCT(TRUE AS flatten_json_output)
    )
);
```

Create a table with a single row that contains the search content and its text embedding.

In this case, we use the default answer as the search content, since we want to find the most similar questions to the default answer (i.e. the questions that the bot cannot answer)

```sql
CREATE OR REPLACE TABLE semantic_search.search_embedding AS (
    SELECT *
    FROM ML.GENERATE_TEXT_EMBEDDING(
        MODEL semantic_search.embedding_model,
        (
            -- It must be a single column with the name "content" and the type STRING
            SELECT
                "No puedo encontrar una respuesta ya que la pregunta no está relacionada con los temas de Ayuda de este Chat. \nPuedes probar lo siguiente,\nSé más específico en la pregunta, agrega palabras que estén relacionadas con el tema en cuestión que me ayuden a encontrar una respuesta\nSi aún no puedo responderte bien, puedes llamar a la Mesa de Ayuda Latam del Portal IT."
            AS
                content  -- CAST("${DEFAULT_ANSWER}" AS STRING) AS content
        ),
        STRUCT(TRUE AS flatten_json_output))
);
```

* Most similar using `cosine distance` threshold

  * BigQueryML Syntax Distance: <https://cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-distance>

Use the `ML.DISTANCE` function to find semantically similar answers to the search content.

```sql
-- most similar contents (answers) for a given search content (default answer) and similarity threshold (0.8) using cosine similarity
SELECT *
FROM (
    SELECT
        s.content AS search_content,                                                    -- v1: default answer
        c.content AS content,                                                           -- v2: answer
        -- cosine similarity between search content (default answer) and content (answers)
        -- The cosine similarity is a number between 0 and 1, where 1 means the two vectors are identical and 0 means the two vectors are as different as possible
        1 - ML.DISTANCE(s.text_embedding, c.text_embedding, 'COSINE') AS similarity     -- similarity(v1, v2) = 1 - distance(v1, v2)
    FROM
        semantic_search.search_embedding AS s,                                          -- search content (default answer)
        semantic_search.answers_embedding AS c                                          -- content        (answers)
    ) AS subquery
WHERE
    similarity > 0.8                                                                    -- similarity threshold for most similar i.e. similarity(v1, v2) > threshold
ORDER BY
    similarity DESC;                                                                    -- descending order for similarity
    -- similarity ASC;                                                                  -- ascending order for disimilarity
```

```sql
-- number of most similar answers for a given search content (default answer) and similarity threshold (0.8) using cosine similarity
SELECT COUNT(*) AS result_count
    FROM (
    SELECT
        s.content AS search_content,
        c.content AS content,
        1 - ML.DISTANCE(s.text_embedding, c.text_embedding, 'COSINE') AS similarity
    FROM
        semantic_search.search_embedding AS s,
        semantic_search.answers_embedding AS c
    WHERE
        1 - ML.DISTANCE(s.text_embedding, c.text_embedding, 'COSINE') > 0.8
    ) AS subquery;
```

```sql
-- percentage of most similar answers for a given search content (default answer) and similarity threshold (0.8)  using cosine similarity
SELECT
    (result_count * 100.0) / total_count AS percentage
FROM (
    SELECT
        COUNT(*) AS result_count
    FROM (
        SELECT
            s.content AS search_content,
            c.content AS content,
            1 - ML.DISTANCE(s.text_embedding, c.text_embedding, 'COSINE') AS similarity
        FROM
            semantic_search.search_embedding AS s,
            semantic_search.answers_embedding AS c
        WHERE
            1 - ML.DISTANCE(s.text_embedding, c.text_embedding, 'COSINE') > 0.8
        ) AS subquery
    ) AS result_count_subquery
CROSS JOIN (
    SELECT
        COUNT(*) AS total_count
    FROM
        semantic_search.search_embedding AS s,
        semantic_search.answers_embedding AS c
    ) AS total_count_subquery;
```

* Top-K most similar using `cosine distance`

The following query returns the top `20` answers from the `LISA HACKATHONS` data that are most semantically similar to the default answer

```sql
-- top k most similar contents (answers) for a given search content (default answer) and similarity threshold (0.8) using cosine distance
SELECT
    s.content AS search_content,                                                    -- v1: default answer
    c.content AS content,                                                           -- v2: answer
    -- cosine distance between search content (default answer) and content (answers)
    -- the cosine distance is a number between 0 and 1, where 0 means the two vectors are identical and 1 means the two vectors are as different as possible
    ML.DISTANCE(s.text_embedding, c.text_embedding, 'COSINE') AS distance           -- distance(v1, v2) = 1 - similarity(v1, v2)
FROM
    semantic_search.search_embedding AS s,                                          -- search content (default answer)
    semantic_search.answers_embedding AS c                                          -- content        (answers)
ORDER BY
    distance ASC                                                                    -- ascending order for distance
LIMIT 20;                                                                           -- top k most similar
```
