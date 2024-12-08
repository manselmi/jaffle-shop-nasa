<!-- vim: set ft=markdown : -->


# NASA pre-interview assessment

This repository is Mike Anselmi's completed pre-interview assessment for NASA's Computer Scientist,
AST, Computer Research and Development position.

## Introduction

Given the limited time available to complete the assessment, I opted
to extend an existing public project of mine found on GitHub at
[`manselmi/jaffle-shop`](https://github.com/manselmi/jaffle-shop).

I had created `manselmi/jaffle-shop` while working at the Channing Division
of Medicine at Mass General Brigham. At the time, I was introducing [dbt
Core](https://docs.getdbt.com/docs/introduction#dbt-core) to the organization
and wanted staff to have a simple "hello world"-style project to familiarize
themselves with dbt. dbt Labs has their own "hello world"-style project on GitHub at
[`dbt-labs/jaffle-shop`](https://github.com/dbt-labs/jaffle-shop), and unfortunately it depends
on their paid cloud offering [dbt Cloud](https://docs.getdbt.com/docs/introduction#dbt-cloud) —
it's a great project but not self-contained. Additionally, dbt Labs also maintains a self-contained
project on GitHub at [`dbt-labs/jaffle_shop_duckdb`](https://github.com/dbt-labs/jaffle_shop_duckdb)
that's powered by DuckDB, but it lacks feature parity when compared to their dbt Cloud project.
I ended up modifying `dbt-labs/jaffle-shop` to work with DuckDB and made significant changes,
including how the project is run and avoiding the dbt anti-pattern of relying on seeds for data
loading, as is found in `dbt-labs/jaffle_shop_duckdb`.

To add data visualization capability, I decided to utilize [Apache
Superset](https://superset.apache.org/docs/intro) as it's fairly simple to configure a bare-bones
installation as part of a Docker Compose stack. I then added the capability to run the existing dbt
project within the Docker Compose stack to simplify execution on Windows, and also to serve the dbt
project documentation (more on that later).

## Project requirements

> [!NOTE]
> Please note that I implemented the assessment on a macOS laptop with an aarch64 CPU architecture.
> The project also runs successfully on Linux. I tried testing this project with Docker Desktop
> running within a Windows virtual machine, but apparently nested virtualization is unsupported with
> an aarch64 macOS host, so I was unable to run Docker Desktop and test with Windows. That said, I
> would expect the Docker Compose stack to run successfully on Windows.

* Application capable of building and running Docker Compose stacks, such as

    * [Docker Desktop](https://www.docker.com/products/docker-desktop) (Windows, macOS)
    * [OrbStack](https://orbstack.dev) (macOS)

* Optional: [DuckDB CLI](https://duckdb.org/docs/installation/index?version=stable&environment=cli)

## Architecture diagram

``` mermaid
architecture-beta
    group sources[Data sources]
    service localDisk(disk)[Local disk] in sources
    service db(database)[Database] in sources
    service objStorage(internet)[Object storage] in sources

    group local[Local Docker Compose stack]

    group dataWarehouse[Ingestion and data warehouse] in local
    group transformation[Data cleaning and transformation] in local
    group dashboard[Analytics dashboards] in local

    service duckdb(database)[DuckDB] in dataWarehouse
    service dbt(server)[dbt Core] in transformation
    service superset(server)[Apache Superset] in dashboard

    localDisk{group}:R --> L:duckdb
    duckdb:B <--> T:dbt
    duckdb:R --> L:superset
```

## Technical architecture elements and commentary

* Data sources

    * Synthetic e-commerce datasets generated by the
      [`jafgen`](https://github.com/dbt-labs/jaffle-shop-generator) Python library, which
      describes its datasets as "suitable for analytics engineering practice or demonstrations",
      are located in [`jaffle-data`](jaffle-data) and are configured as dbt sources in
      [`models/staging/__sources.yml`](models/staging/__sources.yml).

* Data ingestion, cleaning and transformation

    * All data ingestion, cleaning and transformation is managed by [dbt
      Core](https://docs.getdbt.com/docs/introduction#dbt-core) with
      [DuckDB](https://duckdb.org/why_duckdb) as the data warehouse.

    * In real-world usage, dbt is not responsible for data ingestion — the source data
      typically resides in the data warehouse that dbt is connecting to. Tools like
      [dlt](https://dlthub.com/docs/intro) are better suited to data ingestion.

    * These are small datasets so there's no need to implement incremental processing. However, with
      large or expensive-to-process datasets it's typically better to process them incrementally.
      dbt is well-suited for incremental processing and supports multiple [incremental processing
      strategies](https://docs.getdbt.com/docs/build/incremental-strategy).

    * This project does not incorporate a workflow orchestrator like Dagster or Airflow to schedule
      dbt execution. In real-world use cases, an orchester may be configured to trigger (possibly
      incremental) dbt runs on a schedule, or whenever new source data becomes available.

* Storage

    * All files reside on the local filesystem. This obviously would not accomodate large datasets.

    * Source data are CSV files and the data warehouse is a DuckDB database file. DuckDB is great
      for a single user working locally but does not scale beyond a single system.

    * Apache Superset generates charts and dashboards on-demand by querying the aforementioned
      DuckDB database file.

* Data governance

    * Modifications to the data processing pipeline are version controlled in Git and so are
      auditable.

    * For the sake of expediency the charts and dashboards were manually defined in the Apache
      Superset web interface. Ideally changes to charts and dashboards would be version controlled,
      or at the very least regularly backed up.

    * One aspect of data governance is tracking data lineage, i.e. which datasets are dervied from
      which other datasets. Keep an eye out for the dbt data lineage graph mentioned later on in the
      project walkthrough.

* Security

    * In this project security is not top-of-mind. Data is in plaintext at rest and in transit, and
      all connections to web services are unencrypted. There are no access controls for data or web
      services aside from hardcoded credentials for Apache Superset.

      In real-world usage, data would be encrypted at rest and in transit, and all connections to
      web services would be encrypted and require authentication. Database connections would also
      be encrypted and require authentication, and ideally access to database objects would be
      minimally scoped via e.g. role-based access controls. Sensitive columns may also be encrypted
      with format-preserving encryption.

* Data consumption by machine learning models

    * Columns should be encoded in such a way that machine learning models may be trained on them.
      For example, the `main.orders` table one-hot-encodes features such as `is_food_order` and
      `is_drink_order`.

* Scalability and fault tolerance

    * dbt scales with the data warehouse, as it transforms data by running SQL queries
      in the data warehouse. For example, if the data warehouse were Snowflake, in
      [`profiles.yml`](profiles.yml) you would specify the warehouse size and concurrency limit.

    * dbt is also as fault tolerant as your data warehouse. For example, if dbt were targeting a
      Vertica database cluster and database nodes went down, dbt would keep functioning as long as
      Vertica kept functioning.

* Tools and technologies

    * DuckDB is a great choice for local data analytics, as it's fast,
      portable, feature-rich and highly extensible. Here are some [core
      extensions](https://duckdb.org/docs/extensions/core_extensions.html) I commonly use to ingest
      a wide variety of data:

        * `aws` for authenticating with AWS S3;

        * `httpfs` for reading/writing files over HTTP(S) and S3 connections;

        * `iceberg` for read-only [Apache Iceberg](https://iceberg.apache.org/docs/nightly/)
          support;

        * `json` for working with JSON data;

        * `parquet` for reading/writing Parquet files.

    * dbt is often a good choice for managing data transformations because

        * dbt is widely used;

        * dbt models are written in SQL and sometimes Python, both very common data processing
          languages;

        * dbt supports [data tests](https://docs.getdbt.com/docs/build/data-tests),
          [unit tests](https://docs.getdbt.com/docs/build/unit-tests), [model
          contracts](https://docs.getdbt.com/docs/collaborate/govern/model-contracts) etc.
          Model contracts are a nice way to ensure that data engineers aren't violating data
          specifications agreed upon with downstream data consumers.

        * dbt tracks model lineage and automatically builds models in order;

        * dbt models may be written to be database-agnostic by leveraging macros (see
          [`macros/cents_to_dollars.sql`](macros/cents_to_dollars.sql)), which helps to avoid
          database vendor lock-in.

    * Apache Superset was chosen because I could quickly get it running in Docker Compose and create
      some simple charts. There's a good chance there are better free alternatives and there are
      almost certainly better paid alternatives. My area of expertise is backend data engineering,
      so for the data visualization aspect of this assessment I just wanted something that worked.

    * Docker Compose is a nice way to run simple or complex tech stacks locally, and as I don't have
      a Windows environment to test with, Docker Compose is my best bet for this assessment to work
      on Windows.

## Project walkthrough

The dbt "staging" models, which are responsible for minimally-transforming the source data, are
located in [`models/staging`](models/staging). The dbt "marts" models, which are responsible for
transforming the staging models to be more useful for powering dashboards and other analytics, are
located in [`models/marts`](models/marts). The SQL files define the data transformations and the
YAML files define metadata such as [data tests](https://docs.getdbt.com/docs/build/data-tests) and
[unit tests](https://docs.getdbt.com/docs/build/unit-tests).

To have dbt ingest the source data into DuckDB, build the staging and marts models, and serve the
dbt project documentation, bring up the Docker Compose stack by running

``` shell
docker-compose up
```

Please see [`docker-compose/dockerfile/dbt`](docker-compose/dockerfile/dbt) and
[`docker-compose/entrypoint/dbt`](docker-compose/entrypoint/dbt) for more on how the dbt models are
built and how the dbt project documentation is served.

Open the [dbt data lineage graph](http://localhost:8080/#!/overview?g_v=1) in your web browser. For
a simplified view, open the "resources" drop-down menu in the lower-left corner and select only
"Sources" and "Models". The source data nodes are green and the models (i.e. derived data) are in
blue.

Optionally, if you have the DuckDB CLI installed you can query the staging and marts tables by
running

``` shell
duckdb -readonly docker-compose/volume/jaffle_shop.duckdb
```

(Modify the path with the appropriate path separator if using Windows.)

Bringing up the Docker Compose stack also brought up an Apache Superset instance available at
[http://localhost:8088](http://localhost:8088) with the following user credentials:

* username: `admin`
* password: `admin`

Please see [`docker-compose/dockerfile/superset`](docker-compose/dockerfile/superset) and
[`docker-compose/entrypoint/superset`](docker-compose/entrypoint/superset) for more on how the
Apache Superset environment is configured.

After logging in, you should automatically be redirected to the [home
page](http://localhost:8088/superset/welcome/), where you can click through to the charts

* Daily total customer spend
* Customer lifetime spend

and the "Jaffle Shop" dashboard that simultaneously previews both charts.

After you've finished poking around, bring down the Docker Compose stack by running

``` shell
docker-compose down
```

and consider reclaiming disk space by removing and pruning the container images by running

``` shell
docker image rm -- jaffle-shop-dbt:latest jaffle-shop-superset:latest
docker image prune
```
