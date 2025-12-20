## Данный  гайд описывает создание и подключение MinIO к ранее развернутому [JupyterLab со SPARK и Trino](URL "https://github.com/MaxKots/installation_guides/blob/main/Jupyter%20on%20Linux/readme.md")

### 1. Запуск MinIO отдельным контейнером
```bash
# Создание директории для данных MinIO
mkdir -p ~/minio-storage

# Запуска MinIO
docker run -d \
  --name MinIO \
  --hostname minio \
  -p 9000:9000 \
  -p 9001:9001 \
  -v ~/minio-storage:/data \
  -e MINIO_ROOT_USER=****USER**** \
  -e MINIO_ROOT_PASSWORD=****PASSWORD**** \
  --restart unless-stopped \
  minio/minio:latest \
  server /data --console-address ":9001"
```
### 2. Проверка работы MinIO:
   
`MinIO Web UI: http://localhost:9001`

`API Endpoint: http://localhost:9000`

### 3. Подключение из JupyterLab
<details>
<summary><b>Вариант A: Через сеть Docker (рекомендуется)</b></summary>
  
```bash
# 1. Подключение JupyterLab к сети MinIO
docker network connect bridge JupyterLab

# 2. Проверка соединения из контейнера JupyterLab
docker exec JupyterLab ping -c 2 minio
```
</details>

<details>
<summary><b>Вариант B: Через IP хоста</b></summary>

```bash
# Получение IP хоста внутри контейнера
HOST_IP=$(docker exec JupyterLab sh -c "ip route | grep default | awk '{print \$3}'")
echo "Host IP для контейнера: $HOST_IP"
```
</details>

### 4. Работа с MinIO из JupyterLab ноутбука
```python
# Установка клиента MinIO
!pip install minio boto3

# Импорт библиотек
from minio import Minio
from minio.error import S3Error
import pandas as pd
from io import BytesIO

# Настройка подключения (вариант для сети Docker)
minio_client = Minio(
    endpoint="host.docker.internal:9000",  # или IP хоста
    access_key="minioadmin",
    secret_key="minioadmin",
    secure=False
)

# Альтернативный вариант с использованием IP хоста
import socket
host_ip = socket.gethostbyname('host.docker.internal')
print(f"Host IP: {host_ip}")

# Создание bucket
bucket_name = "my-bucket"
try:
    if not minio_client.bucket_exists(bucket_name):
        minio_client.make_bucket(bucket_name)
        print(f"Bucket '{bucket_name}' создан")
    else:
        print(f"Bucket '{bucket_name}' уже существует")
except S3Error as e:
    print(f"Ошибка: {e}")

# Загрузка файла
minio_client.put_object(
    bucket_name=bucket_name,
    object_name="test.txt",
    data=BytesIO(b"Hello from JupyterLab!"),
    length=22
)

# Список файлов
objects = minio_client.list_objects(bucket_name)
for obj in objects:
    print(f"Файл: {obj.object_name}, размер: {obj.size} байт")
```
### 5. Работа с Spark и MinIO
```python
import findspark
findspark.init()

from pyspark.sql import SparkSession
import socket

# Определение IP хоста для контейнера
host_ip = socket.gethostbyname('host.docker.internal')

# Spark сессия с поддержкой S3
spark = SparkSession.builder \
    .appName("MinIO-Spark") \
    .config("spark.hadoop.fs.s3a.endpoint", f"http://{host_ip}:9000") \
    .config("spark.hadoop.fs.s3a.access.key", "minioadmin") \
    .config("spark.hadoop.fs.s3a.secret.key", "minioadmin") \
    .config("spark.hadoop.fs.s3a.path.style.access", "true") \
    .config("spark.hadoop.fs.s3a.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem") \
    .config("spark.hadoop.fs.s3a.connection.ssl.enabled", "false") \
    .getOrCreate()

# Пример работы
df = spark.range(10).toDF("id")
df.show()

# Сохранение в MinIO
df.write \
    .mode("overwrite") \
    .parquet(f"s3a://my-bucket/spark-data/test.parquet")

# Чтение из MinIO
df_read = spark.read.parquet(f"s3a://my-bucket/spark-data/test.parquet")
df_read.show()
```
### 6. Полезные команды
```bash
# Проверка работы MinIO
curl http://localhost:9000/minio/health/live

# Создание alias для быстрого доступа
alias minio-mc='docker run -it --network host minio/mc:latest'

# Использование MinIO Client (mc)
docker run -it --network host minio/mc:latest \
  alias set myminio http://localhost:9000 minioadmin minioadmin

docker run -it --network host minio/mc:latest \
  ls myminio
```
### 7. Быстрый старт (все в одном скрипте)
Создай start-minio.sh:

```bash
#!/bin/bash

# Запуск MinIO
docker run -d \
  --name MinIO \
  -p 9000:9000 \
  -p 9001:9001 \
  -v ~/minio-storage:/data \
  -e MINIO_ROOT_USER=****USER**** \
  -e MINIO_ROOT_PASSWORD=****PASSWORD**** \
  minio/minio:latest \
  server /data --console-address ":9001"

echo "MinIO запущен:"
echo "Web UI: http://localhost:9001"
echo "API: http://localhost:9000"
echo "Login: minioadmin"
echo "Password: minioadmin"
Основные настройки подключения:
Параметр	Значение для JupyterLab
Endpoint	host.docker.internal:9000
Access Key	****USER****
Secret Key	****PASSWORD****
```
Bucket URL	s3a://bucket-name/path
Web UI	http://localhost:9001
#### **Примечание:**
Если host.docker.internal не работает, используй IP хоста, полученный через socket.gethostbyname('host.docker.internal')
