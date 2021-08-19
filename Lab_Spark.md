

### 1. Развернем облачный SQL

В GCP создадим простой MySQL.

Подключимся к базе:

`gcloud sql connect rentals --user=root --quiet`

Подготовим таблицы
```
CREATE DATABASE IF NOT EXISTS recommendation_spark;
USE recommendation_spark;

DROP TABLE IF EXISTS Recommendation;
DROP TABLE IF EXISTS Rating;
DROP TABLE IF EXISTS Accommodation;

CREATE TABLE IF NOT EXISTS Accommodation
(
  id varchar(255),
  title varchar(255),
  location varchar(255),
  price int,
  rooms int,
  rating float,
  type varchar(255),
  PRIMARY KEY (ID)
);

CREATE TABLE  IF NOT EXISTS Rating
(
  userId varchar(255),
  accoId varchar(255),
  rating int,
  PRIMARY KEY(accoId, userId),
  FOREIGN KEY (accoId)
    REFERENCES Accommodation(id)
);

CREATE TABLE  IF NOT EXISTS Recommendation
(
  userId varchar(255),
  accoId varchar(255),
  prediction float,
  PRIMARY KEY(userId, accoId),
  FOREIGN KEY (accoId)
    REFERENCES Accommodation(id)
);

SHOW DATABASES;
```

###2. Зальем данные в подготовленные таблицы в CloudeSQL

Создадим бакет в CloudStorage:

```
gsutil mb gs://$DEVSHELL_PROJECT_ID
```

Скопируем данные в Storage в виде csv:

```
gsutil cp gs://cloud-training/bdml/v2.0/data/accommodation.csv gs://$DEVSHELL_PROJECT_ID
gsutil cp gs://cloud-training/bdml/v2.0/data/rating.csv gs://$DEVSHELL_PROJECT_ID
```

Файлы на месте:

```
gsutil ls gs://$DEVSHELL_PROJECT_ID
```
Посмотрим на один из файлов:
```
gsutil cat gs://$DEVSHELL_PROJECT_ID/accommodation.csv
```

Через интерфейс GCP зальем данные в таблицы:
- Accommodation
- Rating

Можно посмотреть как данные легли в таблицы:
```
USE recommendation_spark;

SELECT * FROM Rating
LIMIT 15;
```
```
SELECT COUNT(*) AS num_ratings
FROM Rating;
```
```
SELECT
    userId,
    COUNT(rating) AS num_ratings
FROM Rating
GROUP BY userId
ORDER BY num_ratings DESC;
```

###3. Развернем "Spark-считалку" (DataProc) 

Создадим небольшой Hadoop-кластер для проекта и региона:
```
mlbasics:europe-central2:rentals
europe-central2-b
```
Назовем кластер `rentals`

Сконфигурируем кластер для работы с источником CloudSQL:

```
echo "Authorizing Cloud Dataproc to connect with Cloud SQL"
CLUSTER=rentals
CLOUDSQL=rentals
ZONE=europe-central2-b
NWORKERS=2

machines="$CLUSTER-m"
for w in `seq 0 $(($NWORKERS - 1))`; do
   machines="$machines $CLUSTER-w-$w"
done

echo "Machines to authorize: $machines in $ZONE ... finding their IP addresses"
ips=""
for machine in $machines; do
    IP_ADDRESS=$(gcloud compute instances describe $machine --zone=$ZONE --format='value(networkInterfaces.accessConfigs[].natIP)' | sed "s/\['//g" | sed "s/'\]//g" )/32
    echo "IP address of $machine is $IP_ADDRESS"
    if [ -z  $ips ]; then
       ips=$IP_ADDRESS
    else
       ips="$ips,$IP_ADDRESS"
    fi
done

echo "Authorizing [$ips] to access cloudsql=$CLOUDSQL"
gcloud sql instances patch $CLOUDSQL --authorized-networks $ips
```

### 4. Запустим расчет на Spark

Скачаем код расчета:
```
gsutil cp gs://cloud-training/bdml/v2.0/model/train_and_apply.py train_and_apply.py
```
Поправим параметры подключения к источнику:
```
cloudshell edit train_and_apply.py
```
Загрузим код в наш CloudStorage:
```
gsutil cp train_and_apply.py gs://$DEVSHELL_PROJECT_ID
```

Засабмитим джобу, расположенную:

`gs://mlbasics/train_and_apply.py`


###5. Посмотрим на результаты расчета

```
USE recommendation_spark;

SELECT COUNT(*) AS count FROM Recommendation;

SELECT
    r.userid,
    r.accoid,
    r.prediction,
    a.title,
    a.location,
    a.price,
    a.rooms,
    a.rating,
    a.type
FROM Recommendation as r
JOIN Accommodation as a
ON r.accoid = a.id
WHERE r.userid = 10;

```