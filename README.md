# Simple Demo for Building Scaling Celery Backend System

### A high level overview for my current project. All codes are removed since it's not opensourced yet, only for demo purpose.

[![made-with-python](https://img.shields.io/badge/Made%20with-Python-1f425f.svg)](https://www.python.org/)

## Prerequisite
* **Operating System**
  * Any Linux environment should be ok for local developement
  * CentOS 7.7 for production environment

* **Python**
  * 3.6.8 or 3.6.9

* **Docker** (for containerizing our app)
  * Python official image: "python:3.6.8-buster" or "python:3.6.9-buster" 
  * Docker 1.13.1
  * Docker-Compose 1.18.0

* **MySQL** (where our data lives in)
  * 5.7 or 8.0
  * currently using 8.0 for better performance

* **Redis** (act as broker for Celery)
  * 5.0.7 

* **Cloud Deployment**
  * Managed AWS services are simple
  * Swap MySQL for AWS RDS
  * Swap Redis for AWS Elasticache or SQS

## Installation
```bash
pip install -r requirements.txt
```

## What is Celery
* Celery is a task/queue system which communicate via message brokers between clients/schedulers and workers. 
* To initiate a task the client/scheduler adds a message to the queue, the broker then delivers that message to a worker. 
* A Celery system can consist of multiple workers and brokers, giving way to high availability and horizontal scaling.
* Official Document: https://docs.celeryproject.org/en/latest/index.html

## What is a Task
* A task is just as simple as a function which is executed by the celery worker. 
* For example, the following task queries a person from the database, add age and write back to database
* the @app.task decorator transform the function into a task executable by celery
```python
@app.task()
def simple_etl_job():
    session = db_session()
    person = session.query(Person).first()
    person.age += 1
    session.merge(person)
    session.close()
    return True
```

## System Architecture
* The whole Celery system communicates through the redis broker
* Celery beat send etl/crawler tasks periodically and workers fetch task from redis
* Crawled data are structured through SQLAlchemy and stored in RDS
* Worker logs are real-time streamed by kinesis agent through kinesis firehose and stored in S3 (optional)

<img src="https://lnsocial.s3-ap-northeast-1.amazonaws.com/Screenshot+from+2020-01-15+17-01-53.png" width="1024"></img>


## Project Structure
```bash
modules/
├── celery
│   ├── config.ini
│   ├── db.py
│   ├── main.py
│   └── settings.py
├── core
│   ├── alert
│   │   ├── config.ini
│   │   ├── stopwords.txt
│   │   ├── tasks.py
│   │   ├── utils
│   │   │   ├── image_generator.py
│   │   │   └── mailservice_new.py
│   │   └── wqy-microhei.ttc
│   ├── common
│   │   └── models.py
│   ├── crawler
│   │   ├── config
│   │   │   ├── config.py
│   │   │   └── config.yaml
│   │   ├── crawlers
│   │   │   ├── cookie_crawler.py
│   │   │   ├── idata_crawler.py
│   │   │   └── template.py
│   │   ├── models
│   │   │   ├── cookie_crawler.py
│   │   │   └── idata_crawler.py
│   │   ├── tasks.py
│   │   ├── test_script.py
│   │   └── utils
│   │       ├── dateutils.py
│   │       └── urlutils.py
│   └── etl
│       ├── config
│       │   ├── config.py
│       │   └── config.yaml
│       ├── tasks.py
│       └── utils
│           └── threshold_finder.py
└── __init__.py

```
   **modules.celery**
   * contains basic setting files for the Celery framework.
   * **modules.celery.db**   -> define database settings including user, pwd, url 
   * **modules.celery.main** -> initiate celery app which imports settings from settings.py
   * **modules.celery.settings** -> defines all the settings for Celery. Including broker url, result backed url, task schedules...etc
    
   **modules.core** 
   * contains backend core functions.
   * all components except **common** include a task.py which defines all the tasks that will be executed by Celery.
   * **modules.core.crawler**  -> crawler component which contains crawler classes and crawler data models
   * **modules.celery.alert**  -> alert component
   * **modules.celery.etl**    -> define daily batch calcutlation tasks, including keyword mapping and target calculation
   * **modules.celery.common** -> common files across components, only contains shared data models for now
   

## Basic Usage

### Run celery worker
```bash
celery -A modules.celery.main worker -l info -O fair --autoscale=16,1  
```
* Creates a celery worker which has the ability to autoscale from 1 processes to a maximum of 16 processes.
* Automatically connects to the broker specified in modules/celery/settings.py.
* -O Fair indicates to use the "fair" optimization (see 2. Use -Ofair for your preforking workers from https://medium.com/@taylorhughes/three-quick-tips-from-two-years-with-celery-c05ff9d7f9eb).

### Run celery scheduler
```bash
celery -A modules.celery.main beat -l info  
```
* Creates a celery beat process which send task periodically.
* Automatically connects to the broker specified in modules/celery/settings.py.
* Task schedules are defined in modules/celery/settings.py.

### Run celery flower
```bash
celery -A modules.celery.main flower --max-tasks=100000 --purge_offline_workers --port=9999 
```
* Creates a simple web monitor console for celery on port 9999.
* Automatically connects to the broker specified in modules/celery/settings.py.
* Look at https://flower.readthedocs.io/en/latest/ for more info.

## Containerize
* Build the docker image 

```bash
docker build -t dockerhub/imagename:tag .
```

* Push to our docker hub

```bash
docker push dockerhub/imagename:tag
```

* Run container with docker-compose
```bash
docker-compose up -d
```
* Let's take a look at the following docker-compose file

```yaml
version: '2.2'

services:
  sii-worker:
    image: dockerhub/imagename:tag
    container_name: worker
    privileged: false
    restart: always
    command: celery -A modules.celery.main worker -O fair --autoscale=32,1 -l warning
    mem_limit: 3g
    mem_reservation: 1g
    volumes:
      - /tmp/logs/:/app/log:z
      - /etc/localtime:/etc/localtime:ro
```
* This file creates a docker container called **worker** with the image **dockerhub/imagename:tag**
* The command **celery -A modules.celery.main worker -O fair --autoscale=32,1 -l warning** will be executed once the container is up
* The container will try to restart whenever it shuts down or when ec2 starts or reboots since the flag **restart** is set to **always **
* 1GB memory is reserved while the limit is 3GB 
* Mount the generated logs to the system's /tmp/logs folder for streaming into S3

