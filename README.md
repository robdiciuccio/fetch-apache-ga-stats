<!--
    Licensed to the Apache Software Foundation (ASF) under one
    or more contributor license agreements.  See the NOTICE file
    distributed with this work for additional information
    regarding copyright ownership.  The ASF licenses this file
    to you under the Apache License, Version 2.0 (the
    "License"); you may not use this file except in compliance
    with the License.  You may obtain a copy of the License at

      http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing,
    software distributed under the License is distributed on an
    "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
    KIND, either express or implied.  See the License for the
    specific language governing permissions and limitations
    under the License.
-->

# Fetch Apache GitHub Actions Statistics

![Test the build](https://github.com/TobKed/fetch-apache-ga-stats/workflows/Test%20the%20build/badge.svg)
[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Context and motivation](#context-and-motivation)
- [Statistics](#statistics)
    - [Json files](#json-files)
    - [CSV file](#csv-file)
    - [Processing existing json files to csv and pushing it to BigQuery](#processing-existing-json-files-to-csv-and-pushing-it-to-bigquery)
- [Determining ASF repositories which uses GitHub Actions (matrix.json)](#determining-asf-repositories-which-uses-github-actions-matrixjson)
- [GitHub Actions Secrets:](#github-actions-secrets)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Context and motivation

For [The Apache Software Foundation [ASF]](https://github.com/apache/) the limit for concurrent jobs in GitHub Actions [GA] equals 180
([usage limits](https://docs.github.com/en/free-pro-team@latest/actions/reference/usage-limits-billing-and-administration#usage-limits)).
The GItHub does not provide statistics related to GA and this repo was created to collect some basic data to make it possible.

## Statistics

Statistics data is fetched in the scheduled action [Fetch GitHub Action queue](.github/workflows/fetch-github-actions-queue.yml).
This action makes series of "snapshots" of GA workflow runs for every ASF repository which uses GA
(list of them is stored in [matrix.json](./matrix.json), described [here]((#determining-asf-repositories-which-uses-github-actions-matrixjson))).

Statistics consists of:

 * json files - workflow runs for every repo in seperate files (described [here]((#json-files)))
 * csv file - simple statistics in single file (described [here](#csv-file))

These files are uploaded as workflow artifact.

#### Json files

The json files contain list of repository workflow runs in `queued` and `in_progress` state.
File titles contain timestamp when fetching this list started.
The json schema is described in GitHub API documentation [here](https://docs.github.com/en/free-pro-team@latest/rest/reference/actions#list-workflow-runs-for-a-repository).

Files are uploaded to Google Cloud Storage for later processing.

Example structure of objects in Google Cloud Storage bucket:

```shell script
example-bucket-name
└── apache
    ├── airflow
    │   ├── 20201103_130148Z.json
    │   └── 20201103_131641Z.json
    └── beam
        ├── 20201103_130148Z.json
        └── 20201103_131641Z.json
```

#### CSV file

Single `bq.csv` file is created and contains simple statistics for all fetched repositories.
This file is used in the [Fetch GitHub Action queue](.github/workflows/fetch-github-actions-queue.yml)
to efficiently upload data to BigQuery table.

CSV file headers: `workflow_id`, `status`, `created_at`, `timestamp`.


#### Processing existing json files to csv and pushing it to BigQuery

Helper script `scripts/parse_existing_json_files.py` can be used to process existing json files into single csv.

Example use:
```shell script
gsutil -m cp -r gs://example-bucket-name/apache gcs

python parse_existing_json_files.py \
    --input-dir gcs \
    --output bq_csv.csv

bq load --autodetect \
    --source_format=CSV \
    dataset.table bq_csv.csv
```

## Determining ASF repositories which uses GitHub Actions (matrix.json)

There is no single endpoint to obtain list of ASF repositories which uses GA and since ASF consists of 2000+
repositories it is not trivial task to obtain it.

This list of repositories which uses GitHub Actions is stored in [matrix.json](./matrix.json)
and can be updated in three ways:
 * manually editing `matrix.json` and committing changes
 * by using [fetch_apache_projects_with_ga.py](scripts/fetch_apache_projects_with_ga.py) python script and committing changes
 * automatically by [Fetch Apache Repositories with GA](.github/workflows/fetch-apache-repos-with-ga.yml) action (changes committed automatically when occur).

Running python script and action cause many requests in behalf of used GitHub Access token which may cause in exceeding quota limits.

## GitHub Actions Secrets:

| Secret           | Required | Description                                                                                                                                                                                                                                                                                                                                                  |
|------------------|----------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `PERSONAL_TOKEN` | True     | [Personal GitHub access token](https://docs.github.com/en/free-pro-team@latest/github/authenticating-to-github/creating-a-personal-access-token) used to authorize requests. It has bigger quota than [`GITHUB_TOKEN  secret`](https://docs.github.com/en/free-pro-team@latest/actions/reference/authentication-in-a-workflow#about-the-github_token-secret) |
| `BQ_TABLE`       | -        | BigQuery table reference to which simple statistics will be pushed (e.g. `dataset.table`).                                                                                                                                                                                                                                                                   |
| `GCP_BUCKET`     | -        | Google Cloud storage bucket to which json files with workflows payload will be pushed (e.g. `example-bucket-name`).                                                                                                                                                                                                                                          |
| `GCP_PROJECT_ID` | -        | Google Cloud Project ID.                                                                                                                                                                                                                                                                                                                                     |
| `GCP_SA_KEY`     | -        | Google Cloud Service Account key (Service Account with permissions to Google Cloud Storage and BigQuery).                                                                                                                                                                                                                                                    |
| `GCP_SA_EMAIL`   | -        | Google Cloud Service Account email (Service Account with permissions to Google Cloud Storage and BigQuery).                                                                                                                                                                                                                                                  |
