# airflow-ha-mariadb-10.6

This repo is to quickly spin up Airflow 2.2.5 (latest as of 2022.04.28) with MariaDB 10.6 to see if `Scheduler HA (High availability)` works properly.



## Writing docker-compose.yaml

1. download `docker-compose.yaml` from Airflow official website (`https://airflow.apache.org/docs/apache-airflow/stable/start/docker.html#docker-compose-yaml`)

2. update `AIRFLOW__CORE__SQL_ALCHEMY_CONN` and `AIRFLOW__CELERY__RESULT_BACKEND` from postgres to mariadb

```
 environment:
    &airflow-common-env
    AIRFLOW__CORE__EXECUTOR: CeleryExecutor
    # AIRFLOW__CORE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
    # AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
    AIRFLOW__CORE__SQL_ALCHEMY_CONN: mysql://airflow:airflow@mariadb:3306/airflow
    AIRFLOW__CELERY__RESULT_BACKEND: db+mysql://airflow:airflow@mariadb:3306/airflow
...

```
3. update db service part from postgres to mariadb and add command `--explicit_defaults_for_timestamp=1`
```
services:
  mariadb:
    image: mariadb:10.6
    hostname: airflow
    environment:
      MYSQL_DATABASE: airflow
      MYSQL_HOSTNAME: airflow
      MYSQL_PASSWORD: airflow
      MYSQL_PORT: 3306
      MYSQL_ROOT_PASSWORD: airflow
      MYSQL_USER: airflow
    command: --explicit_defaults_for_timestamp=1
    ports:
      - "3306:3306"
```

## Setting the right Airflow user

On Linux, the quick-start needs to know your host user id and needs to have group id set to 0. Otherwise the files created in dags, logs and plugins will be created with root user. You have to make sure to configure them for the docker-compose:

```
mkdir -p ./dags ./logs ./plugins
echo -e "AIRFLOW_UID=$(id -u)" > .env
```
For other operating systems, you will get warning that AIRFLOW_UID is not set, but you can ignore it. You can also manually create the .env file in the same folder your docker-compose.yaml is placed with this content to get rid of the warning:

```
AIRFLOW_UID=50000
```

## spin up mariadb first
```
❯ docker-compose up mariadb
[+] Running 2/0
 ⠿ Network airflow-ha-mariadb-106_default      Created                                                                                                                                                                                           0.0s
 ⠿ Container airflow-ha-mariadb-106-mariadb-1  Created                                                                                                                                                                                           0.0s
Attaching to airflow-ha-mariadb-106-mariadb-1
airflow-ha-mariadb-106-mariadb-1  | 2022-04-29 01:33:37+00:00 [Note] [Entrypoint]: Entrypoint script for MariaDB Server 1:10.6.7+maria~focal started.
...
```

## start airflow init
```
❯ docker compose up airflow-init
[+] Running 1/0
 ⠿ Container airflow-ha-mariadb-106-airflow-init-1  Created                                                                                                                                                                                      0.0s
Attaching to airflow-ha-mariadb-106-airflow-init-1
airflow-ha-mariadb-106-airflow-init-1  | The container is run as root user. For security, consider using a regular user account.
airflow-ha-mariadb-106-airflow-init-1  |
airflow-ha-mariadb-106-airflow-init-1  | [2022-04-29 01:34:54,052] {cli_action_loggers.py:105} WARNING - Failed to log action with (MySQLdb._exceptions.ProgrammingError) (1146, "Table 'airflow.log' doesn't exist")
airflow-ha-mariadb-106-airflow-init-1  | [SQL: INSERT INTO log (dttm, dag_id, task_id, event, execution_date, owner, extra) VALUES (%s, %s, %s, %s, %s, %s, %s)]
airflow-ha-mariadb-106-airflow-init-1  | [parameters: (datetime.datetime(2022, 4, 29, 1, 34, 53, 989382), None, None, 'cli_upgradedb', None, 'root', '{"host_name": "1ac59708bfd4", "full_command": "[\'/home/airflow/.local/bin/airflow\', \'db\', \'upgrade\']"}')]
airflow-ha-mariadb-106-airflow-init-1  | (Background on this error at: http://sqlalche.me/e/13/f405)
airflow-ha-mariadb-106-airflow-init-1  | DB: mysql://airflow:***@mariadb:3306/airflow
airflow-ha-mariadb-106-airflow-init-1  | [2022-04-29 01:34:55,238] {db.py:919} INFO - Creating tables
...
...
airflow-ha-mariadb-106-airflow-init-1  | [2022-04-29 01:36:08,044] {manager.py:214} INFO - Added user airflow
airflow-ha-mariadb-106-airflow-init-1  | User "airflow" created with role "Admin"
airflow-ha-mariadb-106-airflow-init-1  | 2.2.5
airflow-ha-mariadb-106-airflow-init-1 exited with code 0
```


The account created has the login airflow and the password airflow.



## docker compose up

This takes some time. Be patient

```
❯ docker compose up
WARN[0000] Found orphan containers ([airflow-ha-mariadb-106-airflow-worker-1 airflow-ha-mariadb-106-airflow-scheduler-1]) for this project. If you removed or renamed this service in your compose file, you can run this command with the --remove-orphans flag to clean it up.
[+] Running 10/0
 ⠿ Container airflow-ha-mariadb-106-airflow-worker-1       Created                     0.1s
 ⠿ Container airflow-ha-mariadb-106-airflow-scheduler-1    Created                     0.1s
 ⠿ Container airflow-ha-mariadb-106-flower-1               Created                     0.0s
 ⠿ Container airflow-ha-mariadb-106-mariadb-1              Running                     0.0s
 ⠿ Container airflow-ha-mariadb-106-airflow-init-1         Created                     0.0s
 ⠿ Container airflow-ha-mariadb-106-redis-1                Created                     0.0s
 ⠿ Container airflow-ha-mariadb-106-airflow-triggerer-1    Created                     0.0s
 ⠿ Container airflow-ha-mariadb-106-airflow-webserver-1    Created                     0.0s
```

```
❯ docker ps --format "{{.Names}}"
airflow-ha-mariadb-106-airflow-worker-1
airflow-ha-mariadb-106-airflow-scheduler-1
airflow-ha-mariadb-106-flower-1
airflow-ha-mariadb-106-redis-1
airflow-ha-mariadb-106-airflow-triggerer-1
airflow-ha-mariadb-106-airflow-webserver-1
airflow-ha-mariadb-106-mariadb-1

```
## Login `local:8080` -> Username: `airflow`, Password: `airflow`

![Screenshot](images/airflow(1).png)


## Scale up scheduler (1 -> 2)
```
❯ docker-compose up --scale airflow-scheduler=2
[+] Running 9/9
 ⠿ Container airflow-ha-mariadb-106-airflow-triggerer-1  Running                               0.0s
 ⠿ Container airflow-ha-mariadb-106-mariadb-1            Running                               0.0s
 ⠿ Container airflow-ha-mariadb-106-airflow-worker-1     Running                               0.0s
 ⠿ Container airflow-ha-mariadb-106-flower-1             Running                               0.0s
 ⠿ Container airflow-ha-mariadb-106-airflow-webserver-1  Running                               0.0s
 ⠿ Container airflow-ha-mariadb-106-redis-1              Running                               0.0s
 ⠿ Container airflow-ha-mariadb-106-airflow-scheduler-2  Created                               0.1s
 ⠿ Container airflow-ha-mariadb-106-airflow-scheduler-1  Recreated                             3.0s
 ⠿ Container airflow-ha-mariadb-106-airflow-init-1       Created                               0.0s
Attaching to airflow-ha-mariadb-106-airflow-init-1, airflow-ha-mariadb-106-airflow-scheduler-1, airflow-ha-mariadb-106-airflow-scheduler-2, airflow-ha-mariadb-106-airflow-triggerer-1, airflow-ha-mariadb-106-airflow-webserver-1, airflow-ha-mariadb-106-airflow-worker-1, airflow-ha-mariadb-106-flower-1, airflow-ha-mariadb-106-mariadb-1, airflow-ha-mariadb-106-redis-1
...
...

```

```
❯ docker ps --format "{{.Names}}"
airflow-ha-mariadb-106-airflow-scheduler-1
airflow-ha-mariadb-106-airflow-scheduler-2  <----- newly added
airflow-ha-mariadb-106-airflow-webserver-1
airflow-ha-mariadb-106-flower-1
airflow-ha-mariadb-106-airflow-worker-1
airflow-ha-mariadb-106-airflow-triggerer-1
airflow-ha-mariadb-106-redis-1
airflow-ha-mariadb-106-mariadb-1
```

```
airflow-ha-mariadb-106-airflow-scheduler-1               |   ____________       _____________
airflow-ha-mariadb-106-airflow-scheduler-1               |  ____    |__( )_________  __/__  /________      __
airflow-ha-mariadb-106-airflow-scheduler-1               | ____  /| |_  /__  ___/_  /_ __  /_  __ \_ | /| / /
airflow-ha-mariadb-106-airflow-scheduler-1               | ___  ___ |  / _  /   _  __/ _  / / /_/ /_ |/ |/ /
airflow-ha-mariadb-106-airflow-scheduler-1               |  _/_/  |_/_/  /_/    /_/    /_/  \____/____/|__/
airflow-ha-mariadb-106-airflow-scheduler-1               | [2022-04-29 04:00:14,533] {scheduler_job.py:694} INFO - Starting the scheduler
airflow-ha-mariadb-106-airflow-scheduler-1               | [2022-04-29 04:00:14,535] {scheduler_job.py:699} INFO - Processing each file at most -1 times
airflow-ha-mariadb-106-airflow-scheduler-1               | [2022-04-29 04:00:17,095] {manager.py:163} INFO - Launched DagFileProcessorManager with pid: 176
airflow-ha-mariadb-106-airflow-scheduler-1               | [2022-04-29 04:00:17,120] {scheduler_job.py:1212} INFO - Resetting orphaned tasks for active dag runs
airflow-ha-mariadb-106-airflow-scheduler-1               | [2022-04-29 04:00:17,178] {settings.py:55} INFO - Configured default timezone Timezone('UTC')
...
...
airflow-ha-mariadb-106-airflow-scheduler-2               |   ____________       _____________
airflow-ha-mariadb-106-airflow-scheduler-2               |  ____    |__( )_________  __/__  /________      __
airflow-ha-mariadb-106-airflow-scheduler-2               | ____  /| |_  /__  ___/_  /_ __  /_  __ \_ | /| / /
airflow-ha-mariadb-106-airflow-scheduler-2               | ___  ___ |  / _  /   _  __/ _  / / /_/ /_ |/ |/ /
airflow-ha-mariadb-106-airflow-scheduler-2               |  _/_/  |_/_/  /_/    /_/    /_/  \____/____/|__/
airflow-ha-mariadb-106-airflow-scheduler-2               | [2022-04-29 04:00:18,962] {scheduler_job.py:694} INFO - Starting the scheduler
airflow-ha-mariadb-106-airflow-scheduler-2               | [2022-04-29 04:00:18,964] {scheduler_job.py:699} INFO - Processing each file at most -1 times
airflow-ha-mariadb-106-airflow-scheduler-2               | [2022-04-29 04:00:21,339] {manager.py:163} INFO - Launched DagFileProcessorManager with pid: 171
airflow-ha-mariadb-106-airflow-scheduler-2               | [2022-04-29 04:00:21,342] {scheduler_job.py:1212} INFO - Resetting orphaned tasks for active dag runs
airflow-ha-mariadb-106-airflow-scheduler-2               | [2022-04-29 04:00:21,367] {settings.py:55} INFO - Configured default timezone Timezone('UTC')
```


## Run an example dag

### example_python_operator

![Screenshot](images/airflow(2).png)


### Scheduler-1

![Screenshot](images/scheduler-1.png)
![Screenshot](images/scheduler-1-log.png)
### Scheduler-2

![Screenshot](images/scheduler-2.png)
![Screenshot](images/scheduler-2-log.png)
