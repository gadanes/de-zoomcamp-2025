# NYC Yellow Taxi Data with PostgreSQL and Docker

This guide sets up a local PostgreSQL database with Docker to analyze NYC Yellow Taxi data using SQL.

---

## ✨ Data Sources

- **Yellow Taxi Trips (January 2021)**\
  [CSV (gzip)](https://github.com/DataTalksClub/nyc-tlc-data/releases/download/yellow/yellow_tripdata_2021-01.csv.gz)

- **zone Lookup Table**\
  [TLC Website](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page) > *Taxi Zone Lookup Table (CSV)*

---

## 🚀 Start PostgreSQL with Docker (No Dockerfile needed)

```bash
docker run -it \
  -e POSTGRES_USER="root" \
  -e POSTGRES_PASSWORD="root" \
  -e POSTGRES_DB="ny_taxi" \
  -v $(pwd)/ny_taxi_postgres_data:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:13
```

This will:

- Create a PostgreSQL container with user `root` and password `root`
- Initialize a DB called `ny_taxi`
- Persist data to `./ny_taxi_postgres_data`
- Expose PostgreSQL on port `5432` locally

---

## ✨ Required Python Tools

Install the following with pip:

```bash
pip install pgcli jupyterlab pandas psycopg2 sqlalchemy
```

---

## ⚡ Connect to PostgreSQL with pgcli

```bash
pgcli -h localhost -p 5432 -u root -d ny_taxi
```

---

## ⚙️ Useful SQL Queries

### 🧵 Joining Yellow Taxi Trips with zone

#### Implicit INNER JOIN

```sql
SELECT
    tpep_pickup_datetime,
    tpep_dropoff_datetime,
    total_amount,
    CONCAT(zpu."Borough", ' | ', zpu."Zone") AS "pickup_loc",
    CONCAT(zdo."Borough", ' | ', zdo."Zone") AS "dropoff_loc"
FROM
    yellow_taxi_data t,
    zone zpu,
    zone zdo
WHERE
    t."PULocationID" = zpu."LocationID"
    AND t."DOLocationID" = zdo."LocationID"
LIMIT 100;
```

#### Explicit JOIN

```sql
SELECT
    tpep_pickup_datetime,
    tpep_dropoff_datetime,
    total_amount,
    CONCAT(zpu."Borough", ' | ', zpu."Zone") AS "pickup_loc",
    CONCAT(zdo."Borough", ' | ', zdo."Zone") AS "dropoff_loc"
FROM
    yellow_taxi_data t
JOIN
    zone zpu ON t."PULocationID" = zpu."LocationID"
JOIN
    zone zdo ON t."DOLocationID" = zdo."LocationID"
LIMIT 100;
```

---

## 🔍 Data Integrity Checks

### NULL Location IDs

```sql
SELECT
    tpep_pickup_datetime,
    tpep_dropoff_datetime,
    total_amount,
    "PULocationID",
    "DOLocationID"
FROM
    yellow_taxi_data
WHERE
    "PULocationID" IS NULL
    OR "DOLocationID" IS NULL
LIMIT 100;
```

### Location IDs Not in zone

```sql
SELECT
    tpep_pickup_datetime,
    tpep_dropoff_datetime,
    total_amount,
    "PULocationID",
    "DOLocationID"
FROM
    yellow_taxi_data
WHERE
    "DOLocationID" NOT IN (SELECT "LocationID" from zone)
    OR "PULocationID" NOT IN (SELECT "LocationID" from zone)
LIMIT 100;
```

---

## 🧰 LEFT / RIGHT / OUTER JOINS

### LEFT JOIN example

```sql
DELETE FROM zone WHERE "LocationID" = 142;

SELECT
    tpep_pickup_datetime,
    tpep_dropoff_datetime,
    total_amount,
    CONCAT(zpu."Borough", ' | ', zpu."Zone") AS "pickup_loc",
    CONCAT(zdo."Borough", ' | ', zdo."Zone") AS "dropoff_loc"
FROM
    yellow_taxi_data t
LEFT JOIN
    zone zpu ON t."PULocationID" = zpu."LocationID"
JOIN
    zone zdo ON t."DOLocationID" = zdo."LocationID"
LIMIT 100;
```

### RIGHT JOIN example

```sql
SELECT
    tpep_pickup_datetime,
    tpep_dropoff_datetime,
    total_amount,
    CONCAT(zpu."Borough", ' | ', zpu."Zone") AS "pickup_loc",
    CONCAT(zdo."Borough", ' | ', zdo."Zone") AS "dropoff_loc"
FROM
    yellow_taxi_data t
RIGHT JOIN
    zone zpu ON t."PULocationID" = zpu."LocationID"
JOIN
    zone zdo ON t."DOLocationID" = zdo."LocationID"
LIMIT 100;
```

### FULL OUTER JOIN

```sql
SELECT
    tpep_pickup_datetime,
    tpep_dropoff_datetime,
    total_amount,
    CONCAT(zpu."Borough", ' | ', zpu."Zone") AS "pickup_loc",
    CONCAT(zdo."Borough", ' | ', zdo."Zone") AS "dropoff_loc"
FROM
    yellow_taxi_data t
FULL OUTER JOIN
    zone zpu ON t."PULocationID" = zpu."LocationID"
JOIN
    zone zdo ON t."DOLocationID" = zdo."LocationID"
LIMIT 100;
```

---

## ✅ Aggregation and Grouping

### Count trips per day

```sql
SELECT
    CAST(tpep_dropoff_datetime AS DATE) AS "day",
    COUNT(1)
FROM
    yellow_taxi_data
GROUP BY
    CAST(tpep_dropoff_datetime AS DATE)
LIMIT 100;
```

### Ordered by day or count

```sql
-- Order by date
SELECT
    CAST(tpep_dropoff_datetime AS DATE) AS "day",
    COUNT(1)
FROM
    yellow_taxi_data
GROUP BY
    CAST(tpep_dropoff_datetime AS DATE)
ORDER BY
    "day" ASC
LIMIT 100;

-- Order by number of trips
SELECT
    CAST(tpep_dropoff_datetime AS DATE) AS "day",
    COUNT(1) AS "count"
FROM
    yellow_taxi_data
GROUP BY
    CAST(tpep_dropoff_datetime AS DATE)
ORDER BY
    "count" DESC
LIMIT 100;
```

### More aggregations

```sql
SELECT
    CAST(tpep_dropoff_datetime AS DATE) AS "day",
    COUNT(1) AS "count",
    MAX(total_amount) AS "total_amount",
    MAX(passenger_count) AS "passenger_count"
FROM
    yellow_taxi_data
GROUP BY
    CAST(tpep_dropoff_datetime AS DATE)
ORDER BY
    "count" DESC
LIMIT 100;
```

### Group by multiple fields

```sql
SELECT
    CAST(tpep_dropoff_datetime AS DATE) AS "day",
    "DOLocationID",
    COUNT(1) AS "count",
    MAX(total_amount) AS "total_amount",
    MAX(passenger_count) AS "passenger_count"
FROM
    yellow_taxi_data
GROUP BY
    1, 2
ORDER BY
    "day" ASC,
    "DOLocationID" ASC
LIMIT 100;
```

---

