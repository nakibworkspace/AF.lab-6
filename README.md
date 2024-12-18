# Airflow for ETL and Data Lake Management

Tags: Airflow, Data Lake, Docker, ETL
Created: November 13, 2024 10:47 PM

# Table of contents:

1. Overview on ETL. 
2. Implementations of Apache Airflow on ETL project scheduling.
3. Launching the project using Load Balancer.

# What is ETL?

ETL stands for Extract, Transform, Load, a process that is foundational in data engineering and used in data warehousing, data analysis, and MLOps.

1. **Extract**
    - Involves gathering data from various sources like databases, APIs, CSV or even real-time.
2. **Transform**
    - In this stage, the raw data is cleaned, normalised, aggregated, or enriched to ensure consistency and accuracy. Common transformation tasks include handling missing values, converting data types and even creating new features.
3. **Load**
    - Loading is the process of storing the transformed data into a target system including data warehouse, database and data lake.

![Untitled Diagram.drawio.svg](Airflow%20for%20ETL%20and%20Data%20Lake%20Management%2013dd4718bce880e2b672dac3cac5ae97/Untitled_Diagram.drawio.svg)

ETL in MLOps

- Data Consistency
- Feature Engineering
- Model Retraining
- Automation and Scalability

# Implementation of Apache Airflow in ETL and Data Lake Management

### Step 01 : Open VS Code and create the files that needed & Create a virtual environment.

- Update and upgrade the sudo package list.
    
    ```markup
    sudo apt update
    ```
    
    ```markup
    sudo apt upgrade -y
    ```
    
- Install python3-venv.
    
    ```markup
    sudo apt install python3-venv
    ```
    
- Create virtual environment.
    
    ```markup
    python3 -m venv etl
    ```
    
    ```markup
    source etl/bin/activate
    ```
    
- Install Apache Airflow.

```markup
AIRFLOW_VERSION=2.7.3
PYTHON_VERSION="$(python3 --version | cut -d " " -f 2 | cut -d "." -f 1-2)"
CONSTRAINT_URL="[https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt](https://raw.githubusercontent.com/apache/airflow/constraints-$%7BAIRFLOW_VERSION%7D/constraints-$%7BPYTHON_VERSION%7D.txt)"
pip install "apache-airflow==${AIRFLOW_VERSION}" --constraint "${CONSTRAINT_URL}"
```

### Step 02 : Initilaise the docker compose yaml file.

- Use this Cli to create the docker compose yaml file.

```python
curl -LfO '[https://airflow.apache.org/docs/apache-airflow/2.10.3/docker-compose.yaml](https://airflow.apache.org/docs/apache-airflow/2.10.3/docker-compose.yaml)'
```

- Will create a yaml file automatically to your directory and get this output.

![Screenshot 2024-11-14 at 2.11.36 AM.png](Airflow%20for%20ETL%20and%20Data%20Lake%20Management%2013dd4718bce880e2b672dac3cac5ae97/Screenshot_2024-11-14_at_2.11.36_AM.png)

or, can use this as docker-compose file.

```
# docker-compose.yml
version: '3'
services:
  airflow:
    build: .
    restart: always
    environment:
      - AIRFLOW__CORE__EXECUTOR=LocalExecutor
      - AIRFLOW__CORE__SQL_ALCHEMY_CONN=postgresql+psycopg2://airflow:airflow@postgres:5432/airflow
      - AIRFLOW__CORE__LOAD_EXAMPLES=False
    user: "${AIRFLOW_UID:-50000}:0"
    volumes:
      - ./dags:/opt/airflow/dags
      - ./logs:/opt/airflow/logs:rw
    ports:
      - "8081:8080"
    depends_on:
      - postgres
    command: bash -c "airflow db init && airflow users create --username admin --password admin --firstname Anonymous --lastname Admin --role Admin --email admin@example.com && airflow webserver & airflow scheduler"

  postgres:
    image: postgres:13
    environment:
      - POSTGRES_USER=airflow
      - POSTGRES_PASSWORD=airflow
      - POSTGRES_DB=airflow
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

- Make some directories in it.
    
    ```python
    
    mkdir -p ./dags ./logs ./plugins ./config
    ```
    

### Step 03 : Write the code.

```python
from airflow import DAG
from airflow.providers.http.hooks.http import HttpHook
from airflow.providers.postgres.hooks.postgres import PostgresHook
from airflow.operators.python import PythonOperator
from airflow.decorators import task
from airflow.utils.dates import days_ago
import requests
import json

#Latitude and longitude for the desired location (London in this case)
LATITUDE= '51.5074'
LONGITUDE= '-0.1278'
POSTGRES_CONN_ID= 'postgres_default'
API_CONN_ID= 'open_meteo_api'

default_args= {
    'owner': 'airflow',
    'start_date': days_ago(1)
}

def extract_weatherdata():
        """Extract weather data from Open meteo api using airflow connection"""

        #Use http hook to get connection details from airflow connection
        http_hook= HttpHook(http_conn_id=API_CONN_ID, method='GET')

        ##Build api endpoint
        ## https://api.open-meteo.com/v1/forecast?latitude=51.5074&longitude=-0.1278&current_weather=true
        endpoint= f'/v1/forecast?latitude={LATITUDE}&longitude={LONGITUDE}&current_weather=true'

        #Make the request via httphook
        response= http_hook.run(endpoint)

        if response.status_code == 200:
            return response.json()
        else:
            raise Exception(f"Failed to fetch data: {response.status_code}")
        

    
def transform_weatherdata(weather_data):
        """Transform the extracted weather data"""
        current_weather= weather_data['current_weather']
        transformed_data={
            'latitude' : float(LATITUDE),
            'longitude': float(LONGITUDE),
            "temperature": current_weather['temperature'],
            'windspeed': current_weather['windspeed'],
            'winddirection': current_weather['winddirection'],
            'weathercode': current_weather['weathercode'],

        }
        return transformed_data
    
    
def load_weatherdata(transformed_data):
        """Load transformed data to postgres"""
        pg_hook= PostgresHook(postgres_conn_id=POSTGRES_CONN_ID)
        conn= pg_hook.get_conn()
        cursor= conn.cursor()

        #create table if it doesn't exist
        cursor.execute(
            """create table if not exist weather_data(
            latitude FLOAT
            longitude FLOAT
            temperature INT
            windspeed INT
            winddirection INT
            weathercode INT
            timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            );"""
        )

        #Insert transformed data into the table
        cursor.execute(
            """INSERT INTO weather_data (latitude, longitude, temperature, windspeed, winddirection, weathercode)
            VALUES (%s, %s, %s, %s, %s, %s)
            """, (
                transformed_data['latitude'],
                transformed_data['longitude'],
                transformed_data['temperature'],
                transformed_data['windspeed'],
                transformed_data['winddirection'],
                transformed_data['weathercode'],
            )
        )
        conn.commit()
        cursor.close()
        conn.close()

with DAG(dag_id='etl_weather_pipeline',
         default_args=default_args,
         schedule_interval='@daily',
         catchup=False
         ) as dag:
        
        extract_task= PythonOperator(
               task_id='extract_weather',
               python_callable=extract_weatherdata
        )

        transform_task= PythonOperator(
               task_id='transform_weatherdata',
               python_callable=transform_weatherdata,
               op_args=[extract_task.output]
        )

        load_task= PythonOperator(
               task_id='load_weatherdata',
               python_callable=load_weatherdata,
               op_args=[transform_task.output]
        )
        
        #DAG workflow
        
        extract_task >> transform_task >> load_task

```

Here we have used the weather data information for data extracting as real time data and used it for transformation and loaded it in postgres database.

The output for the data that we used. (Using the https://api.open-meteo.com/v1/forecast?latitude=51.5074&longitude=-0.1278&current_weather=true)

![Screenshot 2024-11-14 at 2.04.25 AM.png](Airflow%20for%20ETL%20and%20Data%20Lake%20Management%2013dd4718bce880e2b672dac3cac5ae97/Screenshot_2024-11-14_at_2.04.25_AM.png)

### Step 04 : Create the docker image and initialize the docker with Apache Airflow.

```python
docker-compose up airflow-init
```

This will lead to Airflow initialisation with Docker.

![Screenshot 2024-10-12 at 1.48.06 AM.png](Airflow%20for%20ETL%20and%20Data%20Lake%20Management%2013dd4718bce880e2b672dac3cac5ae97/Screenshot_2024-10-12_at_1.48.06_AM.png)

Here you can see we have created a default Airflow user with the role Admin.

### Step 05 : Run the docker file using the cli.

```python
docker-compose up -d
```

![Screenshot 2024-11-14 at 2.18.08 AM.png](Airflow%20for%20ETL%20and%20Data%20Lake%20Management%2013dd4718bce880e2b672dac3cac5ae97/Screenshot_2024-11-14_at_2.18.08_AM.png)

Successfully created the docker containers.

```python
docker ps
```

This command line will show what containers are currently active.

![Screenshot 2024-11-14 at 2.32.18 AM.png](Airflow%20for%20ETL%20and%20Data%20Lake%20Management%2013dd4718bce880e2b672dac3cac5ae97/Screenshot_2024-11-14_at_2.32.18_AM.png)

```python
docker-compose down 
```

![Screenshot 2024-11-14 at 2.18.47 AM.png](Airflow%20for%20ETL%20and%20Data%20Lake%20Management%2013dd4718bce880e2b672dac3cac5ae97/Screenshot_2024-11-14_at_2.18.47_AM.png)

Successfully stopped docker containers.

### Step 06 : Use the load balancer to expose the Airflow UI.

- Find the local ip using the command.
    
    ip addr show eth0
    
    ![Screenshot 2024-11-20 at 8.48.41 PM.png](Airflow%20for%20ETL%20and%20Data%20Lake%20Management%2013dd4718bce880e2b672dac3cac5ae97/Screenshot_2024-11-20_at_8.48.41_PM.png)
    

- Go to the load balancer.

![Screenshot 2024-11-20 at 8.47.51 PM.png](Airflow%20for%20ETL%20and%20Data%20Lake%20Management%2013dd4718bce880e2b672dac3cac5ae97/Screenshot_2024-11-20_at_8.47.51_PM.png)

- Create the load balancer and launch it.
    
    
    ![Screenshot 2024-11-20 at 8.49.13 PM.png](Airflow%20for%20ETL%20and%20Data%20Lake%20Management%2013dd4718bce880e2b672dac3cac5ae97/Screenshot_2024-11-20_at_8.49.13_PM.png)
    

### Step 07 : Login to Apache Airflow.

Go to the Airflow dashboard and login with the default user that was created.

![Screenshot 2024-10-13 at 10.46.43 PM.png](Airflow%20for%20ETL%20and%20Data%20Lake%20Management%2013dd4718bce880e2b672dac3cac5ae97/Screenshot_2024-10-13_at_10.46.43_PM.png)

username: admin

password: admin

### Step 08 : Finalising the creation of the dag.

![Screenshot 2024-11-14 at 2.35.23 AM.png](Airflow%20for%20ETL%20and%20Data%20Lake%20Management%2013dd4718bce880e2b672dac3cac5ae97/Screenshot_2024-11-14_at_2.35.23_AM.png)

As you can see we have successfully created the dag!

### Step 09 : Monitoring the dag.

Trigger the dag and see if it runs successfully.

![Screenshot 2024-11-14 at 2.15.25 AM.png](Airflow%20for%20ETL%20and%20Data%20Lake%20Management%2013dd4718bce880e2b672dac3cac5ae97/Screenshot_2024-11-14_at_2.15.25_AM.png)

# Conclusion:

In this document we have tried experimenting the ETL pipeline scheduling using Airflow and launched it using the Load Balancer on Poridhi Lab.