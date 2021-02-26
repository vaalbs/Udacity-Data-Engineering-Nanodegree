# Project: Data Warehouse

### Introduction:

A music streaming startup, Sparkify, has grown their user base and song database and want to move their processes and data onto the cloud. Their data resides in S3, in a directory of JSON logs on user activity on the app, as well as a directory with JSON metadata on the songs in their app.

As their data engineer, you are tasked with building an ETL pipeline that extracts their data from S3, stages them in Redshift, and transforms data into a set of dimensional tables for their analytics team to continue finding insights in what songs their users are listening to. You'll be able to test your database and ETL pipeline by running queries given to you by the analytics team from Sparkify and compare your results with their expected results.

### Project Description:

In this project, I applied what I've learned on data warehouses and AWS to build an ETL pipeline for a database hosted on Redshift. To complete the project, I loaded data from S3 to staging tables on Redshift and execute SQL statements that create the analytics tables from these staging tables.

### How to run

1. To run this project you will need to fill the following information, and save it as _dwh.cfg_ in the project root folder.

```
[CLUSTER]
HOST=''
DB_NAME=''
DB_USER=''
DB_PASSWORD=''
DB_PORT=5439

[IAM_ROLE]
ARN=

[S3]
LOG_DATA='s3://udacity-dend/log_data'
LOG_JSONPATH='s3://udacity-dend/log_json_path.json'
SONG_DATA='s3://udacity-dend/song_data'

[AWS]
KEY=
SECRET=
```

2. Run the following command to install the dependencies ( for macos )

   `$brew install awscli`
   `$pip install boto3 botocore pandas psycopg2 psycopg2-binary`

3. Follow along _create_cluster_ notebook to set up the needed infrastructure for this project.

4. Run the _create_tables_ script to set up the database staging and analytical tables

   `$ python create_tables.py`

5. Finally, run the _etl_ script to extract data from the files in S3, stage it in redshift, and finally store it in the dimensional tables.

   `$ python create_tables.py`

### Data Warehouse Schema Definition

#### Staging Tables

- staging_events
- staging_songs

#### Fact Table

- songplays - records in event data associated with song plays i.e. records with page NextSong -
  _songplay_id, start_time, user_id, level, song_id, artist_id, session_id, location, user_agent_

#### Dimension Tables

- users - users in the app -
  _user_id, first_name, last_name, gender, level_
- songs - songs in music database -
  _song_id, title, artist_id, year, duration_
- artists - artists in music database -
  _artist_id, name, location, lattitude, longitude_
- time - timestamps of records in songplays broken down into specific units -
  _start_time, hour, day, week, month, year, weekday_

### Project structure

The structure is:

- <b> create_tables.py </b> - This script will drop old tables (if exist) ad re-create new tables.
- <b> etl.py </b> - This script orchestrate ETL.
- <b> sql_queries.py </b> - This is the ETL. All the transformatios in SQL are done here.
- <b> /img </b> - Directory with images that are used in this markdown document.
- <b> dhw.cfg - File with AWS credentials.
