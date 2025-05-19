Проект: my-kafka-app
Описание проекта: 
Проект my-kafka-app демонстрирует использование Kafka для потоковой обработки данных, а также интеграцию с CI/CD и Kubernetes. 
В рамках практической демонстрации мы создали приложение, которое:
            Отправляет сообщения в Kafka.
            Читает сообщения из Kafka.
            Использует Docker для контейнеризации приложения.
            Автоматизирует процесс сборки и развертывания через CI/CD.
            Развертывает приложение в Kubernetes.
Структура проекта:
                    my-kafka-app/
                    ├── app/
                    │   ├── main.py
                    │   
                    ├── Dockerfile
                    ├── my-kafka-dev.yaml
                    ├── requirements.txt
                    ├── docker-compose.yml
                    └── .gitlab-ci.yml
Подробное описание файлов:
1. app/main.py
    Описание:
    Это основной файл приложения, который запускает логику работы с Kafka. В нем реализованы функции отправки и получения сообщений.
Содержимое:

from kafka import KafkaProducer, KafkaConsumer
import time

def produce_message():
    produce = KafkaProducer(bootstrap_server='localhost:9092')
    message = b'Hello, Kafka!'
    produce.send('test-topic', message)
    produce.flush()
    print('Сообщение отправлено: ' + message)
    

def consume_message():
    consumer = KafkaConsumer(
        'test-topic',
        bootstrap_servers='localhost:9092',
        auto_offset_reset='earliest',
        consumer_timeout_ns=5000)
    
    for message in consumer:
        print('Сообщение получено: ' + str(message.value))
        time.sleep(1)
        
if __name__ == ' __main__ ':
    produce_message()
    time.sleep(2)
    consume_message()
    
 
2. Dockerfile
Этот файл описывает процесс создания Docker-образа для приложения. Он используется для контейнеризации Python-приложения.
Содержимое:

FROM python:3.10.2

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD [ "python", "app/main.py"]

3. my-kafka-dev.yaml
Этот файл содержит конфигурацию для развертывания Kafka в Kubernetes. Он описывает Deployment и Service для Kafka.
Содержимое:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-demo-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kafka-demo-app
  template:
    metadata:
      labels:
        app: kafka-demo-app
    spec:
      containers:
      - name: kafka-demo-app
        image: kafka-demo-app:latest
        ports:
        - containerPort: 80


4. `requirements.txt`
Этот файл содержит список зависимостей Python, которые необходимы для работы приложения.
Содержимое:

kafka-python == 2.0.2


5. `docker-compose.yml`
Этот файл используется для запуска Kafka и Zookeeper в Docker. Он описывает сервисы, сети и тома.
Содержимое:

version: '3'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    ports:
      - "2181:2181"
  kafka:
    image: confluentinc/cp-kafka:latest
    container_name: kafka
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
    - "9092:9092"
  

6. .gitlab-ci.yml
Этот файл содержит конфигурацию для CI/CD в GitLab. Он описывает этапы сборки, тестирования и развертывания приложения.
Содержимое:

stages:
  - build
  - test
  - deploy

variables:
  DOCKER_IMAGE: my-kafka-app
  DOCKER_TAG: $CI_COMMIT_SHORT_SHA

build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - echo "Сборка Docker-образа..."
    - docker build -t $DOCKER_IMAGE:$DOCKER_TAG .
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
    - docker push $DOCKER_IMAGE:$DOCKER_TAG
  only:
    - master
    - develop

unit_tests:
  stage: test
  image: python:3.10.2
  script:
    - pip install --no-cache-dir -r requirements.txt
    - pytest tests/unit
  only:
    - master
    - develop

integration_tests:
  stage: test
  image: docker:latest
  services:
    - docker:dind
  before_script:
    # Установка docker-compose внутри контейнера (если не установлен)
    - apk add --no-cache docker-compose
    # Поднимаем Kafka и ZooKeeper с помощью docker-compose
    - docker-compose up -d
    # Ожидаем инициализации сервисов (можно заменить на скрипт wait-for-it)
    - sleep 15
  script:
    - echo "Запуск интеграционных тестов..."
    - pip install --no-cache-dir -r requirements.txt
    - pytest tests/integration
  only:
    - master
    - develop

deploy:
  stage: deploy
  image: docker:latest
  script:
    - echo "Деплой в тестовую/продакшн среду..."
    - ./deploy_script.sh $DOCKER_IMAGE:$DOCKER_TAG
  only:
    - master

