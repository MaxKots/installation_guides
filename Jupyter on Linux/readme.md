# Полный гайд по установке и запуску JupyterLab с Apache Spark в Docker

## Содержание

1. [Предварительная подготовка](#предварительная-подготовка)
2. [Установка Docker (если не установлен)](#установка-docker-если-не-установлен)
3. [Первый запуск JupyterLab + Spark](#первый-запуск-jupyterlab--spark)
4. [Запуск существующего контейнера](#запуск-существующего-контейнера)
5. [Работа с JupyterLab](#работа-с-jupyterlab)
6. [Генерация и настройка токенов/паролей](#генерация-и-настройка-токеновпаролей)
7. [Работа с Apache Spark](#работа-с-apache-spark)
8. [Управление контейнерами](#управление-контейнерами)
9. [Решение проблем](#решение-проблем)
10. [Полная очистка](#полная-очистка)

---

## Важное примечание о Docker-образе

**Образ Jupyter со Spark взят из официального репозитория на Docker Hub:**
- **URL:** `https://hub.docker.com/r/jupyter/all-spark-notebook`
- **Тег:** `x86_64-ubuntu-22.04`
- **Альтернативный тег:** `latest`
- **Содержание образа:** Python, Scala, R, Apache Spark и полный стек Jupyter

Это официальный поддерживаемый образ от проекта Jupyter, который включает:
- JupyterLab и Jupyter Notebook
- Apache Spark 3.5.0
- Поддержка PySpark, SparkR и Scala Spark
- Интеграция с Apache Toree для Scala ноутбуков
- Предустановленные библиотеки для анализа данных

---

## Предварительная подготовка

### Проверка системы

```bash
# Проверка архитектуры
uname -m

# Проверка версии ядра (должно быть 3.10+)
uname -r

# Проверка свободного места (минимум 10 ГБ)
df -h /

# Проверка памяти (рекомендуется 8+ ГБ для Spark)
free -h
```

---

## Установка Docker (если не установлен)

<details>
<summary><b>Для Ubuntu/Debian</b></summary>

```bash

# Обновление пакетов
sudo apt update && sudo apt upgrade -y

# Установка зависимостей
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

# Добавление репозитория Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=\$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Установка Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Добавление пользователя в группу docker
sudo usermod -aG docker \$USER

# Применить изменения группы (или перезагрузить систему)
newgrp docker

# Проверка установки
docker --version
docker compose version
```
</details>
  
<details>
<summary><b>Для CentOS/RHEL</b></summary>

```bash

# Установка зависимостей
sudo yum install -y yum-utils

# Добавление репозитория
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# Установка Docker
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Запуск службы
sudo systemctl start docker
sudo systemctl enable docker

# Добавление пользователя в группу
sudo usermod -aG docker \$USER
newgrp docker
```
</details>

---

## Первый запуск JupyterLab + Spark
### Шаг 1: Создание рабочей структуры
```bash

# Создаем основную директорию
mkdir -p ~/jupyter_projects
cd ~/jupyter_projects

# Создаем поддиректории
mkdir -p notebooks data workspace config logs

# Создаем структуру каталогов для проектов
mkdir -p projects/{pyspark_examples,scala_examples,data_analysis}
```

### Шаг 2: Загрузка образа Jupyter со Spark
```bash

# Просмотр доступных тегов
docker pull jupyter/all-spark-notebook:latest

# Или конкретная стабильная версия (рекомендуется)
docker pull jupyter/all-spark-notebook:x86_64-ubuntu-22.04

# Проверка скачанных образов
docker images | grep jupyter
```

### Шаг 3: Запуск контейнера с правильными параметрами

<details>
<summary><b>Вариант A: Базовый запуск (рекомендуется для начала)</b></summary>

```bash

# Останавливаем старые контейнеры с таким же именем (если есть)
docker stop JupyterLab 2>/dev/null
docker rm JupyterLab 2>/dev/null

# Запускаем новый контейнер
docker run -d \
  --name JupyterLab \
  --add-host minio:host-gateway \
  -p 8888:8888 \
  -p 4040:4040 \
  -p 4041:4041 \
  -v \$(pwd)/notebooks:/home/jovyan/work \
  -v \$(pwd)/data:/home/jovyan/data \
  -v \$(pwd)/workspace:/home/jovyan/workspace \
  -v \$(pwd)/logs:/home/jovyan/.logs \
  -e JUPYTER_ENABLE_LAB=yes \
  -e RESTARTABLE=yes \
  --restart unless-stopped \
  jupyter/all-spark-notebook:x86_64-ubuntu-22.04
```
</details>

<details>
<summary><b>Вариант B: Расширенный запуск (с правами root и настройками)</b></summary>

```bash

# Удалить старый контейнер если существует
docker rm -f JupyterLab 2>/dev/null

# Запуск с расширенными параметрами
docker run -d \
  --name JupyterLab \
  --hostname jupyter-spark \
  # Это для MinIO
  --add-host minio:host-gateway \
  -p 8888:8888 \
  -p 4040:4040 \
  -p 4041:4041 \
  -p 7077:7077 \
  -v $(pwd)/notebooks:/home/jovyan/work \
  -v $(pwd)/data:/home/jovyan/data \
  -v $(pwd)/workspace:/home/jovyan/workspace \
  -v $(pwd)/config:/home/jovyan/.jupyter \
  -v $(pwd)/logs:/var/log/jupyter \
  -e JUPYTER_ENABLE_LAB=yes \
  -e GRANT_SUDO=yes \
  -e CHOWN_HOME=yes \
  -e CHOWN_HOME_OPTS=-R \
  -e NB_UID=$(id -u) \
  -e NB_GID=$(id -g) \
  -e SPARK_OPTS="--driver-memory 2G --executor-memory 2G" \
  --user root \
  --memory="4g" \
  --memory-swap="8g" \
  --cpus="2.0" \
  --restart unless-stopped \
  jupyter/all-spark-notebook:x86_64-ubuntu-22.04
```
</details>

<details>
<summary><b>Вариант C: Использование Docker Compose</b></summary>
  
```bash

# Создание docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  jupyterlab:
    container_name: JupyterLab
    image: jupyter/all-spark-notebook:x86_64-ubuntu-22.04
    hostname: jupyter-spark
    ports:
      - "8888:8888"    # JupyterLab
      - "4040:4040"    # Spark UI
      - "4041:4041"    # Дополнительный Spark UI
      - "7077:7077"    # Spark Master
    volumes:
      - ./notebooks:/home/jovyan/work
      - ./data:/home/jovyan/data
      - ./workspace:/home/jovyan/workspace
      - ./config:/home/jovyan/.jupyter
      - ./logs:/var/log/jupyter
    environment:
      - JUPYTER_ENABLE_LAB=yes
      - GRANT_SUDO=yes
      - CHOWN_HOME=yes
      - NB_UID=1000
      - NB_GID=100
      - SPARK_OPTS=--driver-memory 2G --executor-memory 2G
      # Для MinIO. Если оба контейнера в одной сети, то можно http://minio:9000, хотя надежнее - http://localhost:9000
      - MINIO_ENDPOINT=http://minio:9000
      - MINIO_ACCESS_KEY=****USER****
      - MINIO_SECRET_KEY=****PASSWORD****
    networks:
      - jupyter-network  # это для minio, чтобы работал в сети юпитера
    user: root
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 4G
          cpus: '2.0'

networks:
  jupyter-network:
    external: true  # существующая сеть для minio
```
</details>

#### Запуск через docker-compose
```
docker-compose up -d
```

### Шаг 4: Получение токена для первого входа
```bash

# Ожидание нескольких секунд для полного запуска
sleep 5

# Способ 1: Просмотр полных логов
docker logs JupyterLab

# Способ 2: Фильтрация только URL с токеном
docker logs JupyterLab 2>&1 | grep -E "http://127.0.0.1:8888/lab\\?token=|http://localhost:8888/lab\\?token="

# Способ 3: Получение токена из контейнера
docker exec JupyterLab jupyter server list

# Способ 4: Мониторинг логов в реальном времени (если контейнер только запустился)
docker logs -f JupyterLab 2>&1 | grep --line-buffered "token"
```

### Шаг 5: Доступ к JupyterLab
Открой браузер и введи один из URL из логов, например:
    
```text
http://localhost:8888/lab?token=TOKEN
```
Или перейди по адресу http://localhost:8888 и введи токен вручную

### Шаг 6: Проверка работы Spark

Создайте новый ноутбук с ядром Python 3 и выполните:
```python

# Инициализация Spark
import findspark
findspark.init()

from pyspark.sql import SparkSession

# Создание Spark сессии
spark = SparkSession.builder \\
    .appName("FirstSparkApp") \\
    .master("local[*]") \\
    .config("spark.driver.memory", "2g") \\
    .config("spark.executor.memory", "2g") \\
    .getOrCreate()

# Тестовая операция
df = spark.range(1, 1000)
df_count = df.count()
print(f"Count: {df_count}")

# Проверка версий
print(f"Spark version: {spark.version}")
print(f"Python version: {spark.sparkContext.pythonVer}")

# Не забудь остановить сессию после работы
# spark.stop()
```

---

## Запуск существующего контейнера

<details>
  <summary><b>Если контейнер уже создан и остановлен</b></summary>

```bash

# Способ 1: Простой запуск
docker start JupyterLab

# Способ 2: Запуск с просмотром логов
docker start JupyterLab && sleep 3 && docker logs JupyterLab | grep -A 2 "http://"

# Способ 3: Проверка статуса и запуск
if docker ps -a | grep -q "JupyterLab"; then
    echo "Запуск существующего контейнера JupyterLab..."
    docker start JupyterLab
    echo "URL для доступа: http://localhost:8888"
    echo "Для получения токена выполни: docker logs JupyterLab | grep token"
else
    echo "Контейнер JupyterLab не найден. Создай новый."
fi
```
</details>

<details>
  <summary><b>Если контейнер работает</b></summary>

```bash

# Проверка статуса
docker ps | grep JupyterLab

# Получение URL с токеном (если забыли)
docker exec JupyterLab jupyter server list

# Перезагрузка контейнера (если нужно)
docker restart JupyterLab
```
</details>

<details>
  <summary><b>Автоматический скрипт для запуска</b></summary>

```bash

#!/bin/bash
# Сохранить как ~/bin/jupyter-start.sh

JUPYTER_DIR="\${HOME}/jupyter_projects"
CONTAINER_NAME="JupyterLab"

# Создание директории если их нет
mkdir -p "\${JUPYTER_DIR}"/{notebooks,data,workspace,config,logs}

# Проверка существования контейнера
if docker ps -a --format '{{.Names}}' | grep -q "^\${CONTAINER_NAME}\$"; then
    echo "Запуск существующего контейнера \${CONTAINER_NAME}..."
    docker start "\${CONTAINER_NAME}"
    
    # Ожадание запуска
    sleep 5
    
    # Получение URL
    URL=\$(docker exec "\${CONTAINER_NAME}" jupyter server list 2>/dev/null | grep -o "http://[^ ]*" | head -1)
    
    if [ -n "\${URL}" ]; then
        echo "JupyterLab доступен по адресу:"
        echo "\${URL}"
    else
        echo "Получение URL из логов..."
        docker logs "\${CONTAINER_NAME}" 2>&1 | grep -E "http://[^ ]*token=" | head -1
    fi
else
    echo "Создание нового контейнера \${CONTAINER_NAME}..."
    
    # Запуск нового контейнера
    docker run -d \\
        --name "\${CONTAINER_NAME}" \\
        -p 8888:8888 \\
        -p 4040:4040 \\
        -v "\${JUPYTER_DIR}/notebooks:/home/jovyan/work" \\
        -v "\${JUPYTER_DIR}/data:/home/jovyan/data" \\
        -v "\${JUPYTER_DIR}/workspace:/home/jovyan/workspace" \\
        -e JUPYTER_ENABLE_LAB=yes \\
        --restart unless-stopped \\
        jupyter/all-spark-notebook:x86_64-ubuntu-22.04
    
    echo "Контейнер создан. Получение токена доступа..."
    sleep 7
    
    # Получерние URL с токеном
    docker logs "\${CONTAINER_NAME}" 2>&1 | grep -E "http://[^ ]*token=" | head -1
fi

# Сделать скрипт исполняемым
chmod +x ~/bin/jupyter-start.sh
```
</details>

---

## Работа с JupyterLab
<details>
  <summary><b>  Типы ядер</b></summary>

        Python 3 (ipykernel) - для Python с поддержкой Spark

        Apache Toree - Scala - для Scala с поддержкой Spark

        R - для R с поддержкой Spark
</details>

<details>
  <summary><b>  Типы ячеек</b></summary>

    Code (Код): Для исполняемого кода

    Markdown: Для документации и текста

    Raw (Текст): Неформатированный текст
</details>

<details>
  <summary><b>  Основные горячие клавиши</b></summary>
    Shift + Enter: Выполнить ячейку и перейти к следующей
  
    Ctrl + Enter: Выполнить ячейку 
    
    Alt + Enter: Выполнить ячейку и создать новую снизу
    
    Esc: Выйти из режима редактирования
    
    A: Добавить ячейку сверху
    
    B: Добавить ячейку снизу
    
    D, D: Удалить ячейку (дважды нажать D)
    
    M: Преобразовать в Markdown ячейку
    
    Y: Преобразовать в Code ячейку
    
    Ctrl + S: Сохранить ноутбук
</details>

---

### Установка дополнительных пакетов
```bash
# В терминале JupyterLab или через docker exec
pip install pandas numpy matplotlib seaborn plotly scikit-learn

# Или из ноутбука
!pip install package_name

# Установка для всех пользователей (требует прав root)
!pip install --user package_name
```

---

## Генерация и настройка токенов/паролей
Генерация нового токена:
```bash

# Войти в контейнер
docker exec -it JupyterLab bash

# Способ 1: Генерация случайного токена
jupyter server --generate-config
# Токен будет сгенерирован автоматически при запуске

# Способ 2: Установка конкретного токена
jupyter server --generate-config
echo "c.ServerApp.token = 'ваш_секретный_токен'" >> /home/jovyan/.jupyter/jupyter_server_config.py
echo "c.ServerApp.password = ''" >> /home/jovyan/.jupyter/jupyter_server_config.py

# Способ 3: Отключение токена (не рекомендуется для production)
echo "c.ServerApp.token = ''" >> /home/jovyan/.jupyter/jupyter_server_config.py
echo "c.ServerApp.password = ''" >> /home/jovyan/.jupyter/jupyter_server_config.py

# Выйти из контейнера
exit
```

Установка пароля вместо токена:
```bash

# Запускаем временный контейнер для генерации пароля
docker run -it --rm \\
  jupyter/all-spark-notebook:x86_64-ubuntu-22.04 \\
  bash -c "jupyter server password && echo 'Пароль сохранен'"

# Или генерируем хеш пароля
docker run -it --rm \\
  jupyter/all-spark-notebook:x86_64-ubuntu-22.04 \\
  python -c "from jupyter_server.auth import passwd; print(passwd('ваш_пароль'))"

# Копируем полученный хеш и создаем конфиг
mkdir -p ~/jupyter_projects/config
cat > ~/jupyter_projects/config/jupyter_server_config.py << EOF
c.ServerApp.password = 'sha1:ваш_хеш_пароля'
c.ServerApp.password_required = True
c.ServerApp.token = ''
EOF

# Запускаем контейнер с конфигом
docker run -d \\
  --name JupyterLab \\
  -p 8888:8888 \\
  -v ~/jupyter_projects/config:/home/jovyan/.jupyter \\
  -v ~/jupyter_projects/notebooks:/home/jovyan/work \\
  -e JUPYTER_ENABLE_LAB=yes \\
  jupyter/all-spark-notebook:x86_64-ubuntu-22.04
```

Сброс пароля/токена:
```bash

# Останавливаем контейнер
docker stop JupyterLab

# Удаляем конфиг
rm -f ~/jupyter_projects/config/jupyter_server_config.py

# Запускаем заново - будет сгенерирован новый токен
docker start JupyterLab
docker logs JupyterLab | grep token

Автоматическая генерация токена при запуске:
```bash

# Создаем скрипт для автоматической настройки
cat > ~/jupyter_projects/start_jupyter_with_token.sh << 'EOF'
#!/bin/bash

# Генерируем случайный токен
TOKEN=\$(openssl rand -hex 32)

# Создаем конфиг с токеном
CONFIG_DIR="./config"
mkdir -p \$CONFIG_DIR

cat > \$CONFIG_DIR/jupyter_server_config.py << TOKENCONFIG
c.ServerApp.token = '\$TOKEN'
c.ServerApp.open_browser = False
c.ServerApp.ip = '0.0.0.0'
c.ServerApp.port = 8888
TOKENCONFIG

# Запускаем контейнер
docker run -d \\
  --name JupyterLab \\
  -p 8888:8888 \\
  -v \$(pwd)/notebooks:/home/jovyan/work \\
  -v \$(pwd)/config:/home/jovyan/.jupyter \\
  -e JUPYTER_ENABLE_LAB=yes \\
  jupyter/all-spark-notebook:x86_64-ubuntu-22.04

echo "JupyterLab запущен с токеном: \$TOKEN"
echo "URL: http://localhost:8888/lab?token=\$TOKEN"
EOF

chmod +x ~/jupyter_projects/start_jupyter_with_token.sh
```

---

## Работа с Apache Spark
Настройка Spark в контейнере:
```bash

# Вход в контейнер для настройки Spark
docker exec -it JupyterLab bash

# Проверка установки Spark
ls -la /usr/local/spark

# Проверка переменных окружения Spark
env | grep SPARK

# Просмотр конфигурации Spark
cat /usr/local/spark/conf/spark-defaults.conf 2>/dev/null || echo "Файл не найден, создание..."

# Выход из контейнера
exit
```

Запуск Spark приложений из Jupyter:
PySpark ноутбук:
```python

from pyspark.sql import SparkSession
from pyspark.sql.functions import *

# Создание сессии с настройками
spark = SparkSession.builder \
    .appName("DataAnalysis") \
    .master("local[*]") \
    .config("spark.driver.memory", "2g") \
    .config("spark.executor.memory", "2g") \
    .config("spark.sql.shuffle.partitions", "4") \
    .getOrCreate()

# Пример работы с данными
data = [("Alice", 34), ("Bob", 45), ("Charlie", 29)]
df = spark.createDataFrame(data, ["name", "age"])

# Трансформации
df.show()
df.filter(df.age > 30).show()
df.groupBy().avg("age").show()

# Чтение файлов
# df_csv = spark.read.csv("/home/jovyan/data/file.csv", header=True, inferSchema=True)
# df_json = spark.read.json("/home/jovyan/data/file.json")

# Остановка сессии (в конце ноутбука)
spark.stop()
```

Scala ноутбук (Apache Toree):
```scala

// Создание SparkContext
val sc = org.apache.spark.SparkContext.getOrCreate()

// Создание RDD
val rdd = sc.parallelize(1 to 100)
val sum = rdd.reduce(_ + _)
println(s"Sum: \$sum")

// Работа с Spark SQL
val spark = org.apache.spark.sql.SparkSession.builder
    .appName("ScalaSparkExample")
    .getOrCreate()

import spark.implicits._

case class Person(name: String, age: Int)
val people = Seq(Person("Alice", 34), Person("Bob", 45))
val df = people.toDF()

df.createOrReplaceTempView("people")
val results = spark.sql("SELECT * FROM people WHERE age > 30")
results.show()
```

Доступ к Spark UI:

После запуска Spark приложения открой в браузере:

    http://localhost:4040 - интерфейс текущего приложения

    http://localhost:4041 - интерфейс для следующего приложения

## Управление контейнерами
Основные команды:
```bash

# Просмотр всех контейнеров
docker ps -a

# Просмотр только запущенных контейнеров
docker ps

# Остановка контейнера
docker stop JupyterLab

# Запуск остановленного контейнера
docker start JupyterLab

# Перезагрузка контейнера
docker restart JupyterLab

# Удаление контейнера
docker rm JupyterLab

# Принудительное удаление (если запущен)
docker rm -f JupyterLab

# Вход в контейнер
docker exec -it JupyterLab bash

# Просмотр логов
docker logs JupyterLab
docker logs -f JupyterLab  # в реальном времени

# Мониторинг ресурсов
docker stats JupyterLab

# Копирование файлов
docker cp JupyterLab:/home/jovyan/work/notebook.ipynb ./backup/
docker cp ./data/file.csv JupyterLab:/home/jovyan/data/
```

Автоматизация управления:
```bash

# Скрипт для полного управления
cat > ~/bin/jupyter-manage.sh << 'EOF'
#!/bin/bash

CONTAINER_NAME="JupyterLab"
JUPYTER_DIR="\${HOME}/jupyter_projects"

case "\$1" in
    start)
        if docker ps -a --format '{{.Names}}' | grep -q "^\${CONTAINER_NAME}\$"; then
            docker start "\${CONTAINER_NAME}"
            echo "Контейнер \${CONTAINER_NAME} запущен"
        else
            echo "Контейнер \${CONTAINER_NAME} не найден. Создайте новый."
        fi
        ;;
    stop)
        docker stop "\${CONTAINER_NAME}"
        echo "Контейнер \${CONTAINER_NAME} остановлен"
        ;;
    restart)
        docker restart "\${CONTAINER_NAME}"
        echo "Контейнер \${CONTAINER_NAME} перезапущен"
        ;;
    status)
        if docker ps --format '{{.Names}}' | grep -q "^\${CONTAINER_NAME}\$"; then
            echo "Контейнер \${CONTAINER_NAME} запущен"
            echo "URL: http://localhost:8888"
            docker exec "\${CONTAINER_NAME}" jupyter server list 2>/dev/null | grep http || \\
                echo "Используйте: docker logs \${CONTAINER_NAME} | grep token"
        elif docker ps -a --format '{{.Names}}' | grep -q "^\${CONTAINER_NAME}\$"; then
            echo "Контейнер \${CONTAINER_NAME} остановлен"
        else
            echo "Контейнер \${CONTAINER_NAME} не существует"
        fi
        ;;
    logs)
        docker logs "\${CONTAINER_NAME}" "\${@:2}"
        ;;
    bash)
        docker exec -it "\${CONTAINER_NAME}" bash
        ;;
    clean)
        docker stop "\${CONTAINER_NAME}" 2>/dev/null
        docker rm "\${CONTAINER_NAME}" 2>/dev/null
        echo "Контейнер \${CONTAINER_NAME} удален"
        ;;
    backup)
        mkdir -p "\${JUPYTER_DIR}/backup"
        tar -czf "\${JUPYTER_DIR}/backup/jupyter_backup_\$(date +%Y%m%d_%H%M%S).tar.gz" \\
            -C "\${JUPYTER_DIR}" notebooks data workspace config 2>/dev/null
        echo "Бэкап создан в \${JUPYTER_DIR}/backup/"
        ;;
    *)
        echo "Использование: \$0 {start|stop|restart|status|logs|bash|clean|backup}"
        exit 1
        ;;
esac


chmod +x ~/bin/jupyter-manage.sh
```

---

## Решение проблем
Частые проблемы и решения:
1. Порт 8888 уже используется
```bash

# Поиск процесса занимающего порт
sudo lsof -i :8888

# Освобождение порта
sudo kill -9 \$(sudo lsof -t -i:8888)

# Или запуск на другом порту
docker run -d --name JupyterLab -p 8899:8888 ...
```

2. Ошибка доступа к файлам
```bash

# Исправление прав на хосте
sudo chown -R \$USER:\$USER ~/jupyter_projects

# Или запуск с правильными UID/GID
docker run -d \\
  --name JupyterLab \\
  -p 8888:8888 \\
  -v ~/jupyter_projects/notebooks:/home/jovyan/work \\
  -e NB_UID=\$(id -u) \\
  -e NB_GID=\$(id -g) \\
  jupyter/all-spark-notebook:x86_64-ubuntu-22.04
```

3. Контейнер не запускается
```bash

# Просмотр логов ошибок
docker logs JupyterLab

# Запуск в интерактивном режиме для отладки
docker run -it --rm \\
  jupyter/all-spark-notebook:x86_64-ubuntu-22.04 \\
  bash -c "jupyter lab --debug"

# Проверка образа
docker inspect jupyter/all-spark-notebook:x86_64-ubuntu-22.04
```

4. Spark не инициализируется
```python

# Внутри ноутбука Jupyter
import os
import sys

# Принудительная установка путей
os.environ['SPARK_HOME'] = '/usr/local/spark'
os.environ['PYSPARK_PYTHON'] = '/opt/conda/bin/python'
os.environ['PYSPARK_DRIVER_PYTHON'] = '/opt/conda/bin/python'

sys.path.insert(0, os.path.join(os.environ['SPARK_HOME'], 'python'))
sys.path.insert(0, os.path.join(os.environ['SPARK_HOME'], 'python/lib/py4j-0.10.9.7-src.zip'))

import findspark
findspark.init()
```

5. Недостаточно памяти
```bash

# Увеличение памяти контейнера
docker update --memory="6g" --memory-swap="12g" JupyterLab

# Или пересоздание с большей памятью
docker run -d \\
  --name JupyterLab \\
  -p 8888:8888 \\
  -m 8g \\
  --memory-swap 16g \\
  --cpus="4.0" \\
  jupyter/all-spark-notebook:x86_64-ubuntu-22.04
```

---

## Полная очистка
Удаление контейнера и данных:
```bash

#!/bin/bash
# Полная очистка JupyterLab + Spark

echo "Начало очистки..."

# Остановка и удаление контейнера
docker stop JupyterLab 2>/dev/null
docker rm JupyterLab 2>/dev/null

# Удаление образов Jupyter
docker rmi jupyter/all-spark-notebook:x86_64-ubuntu-22.04 2>/dev/null
docker rmi jupyter/all-spark-notebook:latest 2>/dev/null

# Очистка Docker
docker system prune -a --volumes -f

# Удаление рабочих директорий
read -p "Удалить рабочие директории? (y/n): " -n 1 -r
echo
if [[ \$REPLY =~ ^[Yy]\$ ]]; then
    rm -rf ~/jupyter_projects
    rm -rf ~/.jupyter
    echo "Директории удалены"
fi

# Очистка временных файлов
find /tmp -name "*spark*" -type f -delete 2>/dev/null
find /tmp -name "*jupyter*" -type f -delete 2>/dev/null

echo "Очистка завершена!"
```

Сохранение данных перед удалением:
```bash

# Создание бэкапа
BACKUP_DIR="\${HOME}/jupyter_backups/\$(date +%Y%m%d_%H%M%S)"
mkdir -p "\${BACKUP_DIR}"

# Копирование данных из контейнера перед удалением
docker cp JupyterLab:/home/jovyan/work "\${BACKUP_DIR}/notebooks"
docker cp JupyterLab:/home/jovyan/data "\${BACKUP_DIR}/data"

# Или копирование с хоста
cp -r ~/jupyter_projects "\${BACKUP_DIR}/"

echo "Бэкап создан в: \${BACKUP_DIR}"
```

Быстрые команды для ежедневного использования
```bash

# Алиасы для добавления в ~/.bashrc
echo "alias jup-start='docker start JupyterLab && sleep 3 && docker logs JupyterLab 2>&1 | grep -E \"http://.*token=\" | head -1'" >> ~/.bashrc
echo "alias jup-stop='docker stop JupyterLab'" >> ~/.bashrc
echo "alias jup-restart='docker restart JupyterLab'" >> ~/.bashrc
echo "alias jup-logs='docker logs -f JupyterLab'" >> ~/.bashrc
echo "alias jup-bash='docker exec -it JupyterLab bash'" >> ~/.bashrc
echo "alias jup-token='docker exec JupyterLab jupyter server list 2>/dev/null | grep http || docker logs JupyterLab 2>&1 | grep -E \"token=\" | tail -1'" >> ~/.bashrc

source ~/.bashrc
```

---

# Дополнительная информация
## Источники и ссылки:

    Официальный Docker образ Jupyter со Spark:

        Docker Hub: https://hub.docker.com/r/jupyter/all-spark-notebook

        Документация: https://jupyter-docker-stacks.readthedocs.io/

        GitHub: https://github.com/jupyter/docker-stacks

    Apache Spark:

        Официальный сайт: https://spark.apache.org/

        Документация PySpark: https://spark.apache.org/docs/latest/api/python/

    JupyterLab:

        Официальный сайт: https://jupyter.org/

        Документация: https://jupyterlab.readthedocs.io/

## Версии ПО в образе (пример):

    Ubuntu 22.04

    Python 3.11+

    Apache Spark 3.5+

    JupyterLab 4.0+

    Scala 2.12/2.13

    R 4.3+

## Рекомендации по использованию:

    Для разработки: используй вариант B с правами root для установки дополнительных пакетов

    Для production: используй вариант A с ограниченными правами

    Для team работы: рассмотри JupyterHub вместо JupyterLab

    Для больших данных: увеличь лимиты памяти и CPU

    Для сохранности данных: регулярно делай бэкапы томов

## Поддержка и обновления:

Для обновления образа до последней версии:
```bash

# Остановить и удалить контейнер
docker stop JupyterLab
docker rm JupyterLab

# Скачать последнюю версию образа
docker pull jupyter/all-spark-notebook:latest

# Запустить с теми же параметрами
docker run -d ... jupyter/all-spark-notebook:latest
```
