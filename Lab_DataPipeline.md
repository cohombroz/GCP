### 1. Подготовим "хранилку" в BigQuery
```
bq mk taxirides
```
```
bq mk \
--time_partitioning_field timestamp \
--schema ride_id:string,point_idx:integer,latitude:float,longitude:float,\
timestamp:timestamp,meter_reading:float,meter_increment:float,ride_status:string,\
passenger_count:integer -t taxirides.realtime
```

### 2. Сконфигурируем DataPipeline

Job name: streaming-taxi-pipeline
Dataflow template: Pub/Sub Topic to BigQuery template
Input: projects/pubsub-public-data/topics/taxirides-realtime
BigQuery output table: mlbasics:taxirides.realtime
Temporary location: gs://mlbasics/tmp/

### 3. Проверим поступающие данные

```
SELECT * FROM taxirides.realtime LIMIT 10
```
С агрегацией поминутно:
```
WITH streaming_data AS (

SELECT
  timestamp,
  TIMESTAMP_TRUNC(timestamp, HOUR, 'UTC') AS hour,
  TIMESTAMP_TRUNC(timestamp, MINUTE, 'UTC') AS minute,
  TIMESTAMP_TRUNC(timestamp, SECOND, 'UTC') AS second,
  ride_id,
  latitude,
  longitude,
  meter_reading,
  ride_status,
  passenger_count
FROM
  taxirides.realtime
WHERE ride_status = 'dropoff'
ORDER BY timestamp DESC
LIMIT 100000

)

# calculate aggregations on stream for reporting:
SELECT
 ROW_NUMBER() OVER() AS dashboard_sort,
 minute,
 COUNT(DISTINCT ride_id) AS total_rides,
 SUM(meter_reading) AS total_revenue,
 SUM(passenger_count) AS total_passengers
FROM streaming_data
GROUP BY minute, timestamp
```