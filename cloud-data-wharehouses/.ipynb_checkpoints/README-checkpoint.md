# Project: Data Modeling with Cassandra

### Introduction:

A music streaming startup, Sparkify, has grown their user base and song database and want to move their processes and data onto the cloud. Their data resides in S3, in a directory of JSON logs on user activity on the app, as well as a directory with JSON metadata on the songs in their app.

As their data engineer, you are tasked with building an ETL pipeline that extracts their data from S3, stages them in Redshift, and transforms data into a set of dimensional tables for their analytics team to continue finding insights in what songs their users are listening to. You'll be able to test your database and ETL pipeline by running queries given to you by the analytics team from Sparkify and compare your results with their expected results.

### Project Description:

In this project, I applied what I've learned on data warehouses and AWS to build an ETL pipeline for a database hosted on Redshift. To complete the project, I loaded data from S3 to staging tables on Redshift and execute SQL statements that create the analytics tables from these staging tables.

### How to run

1. To run this project you will need to fill the following information, and save it as *dwh.cfg* in the project root folder.

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

### Data Warehouse Schema Definition

#### Staging tables

##### TABLE staging_songs

| COLUMN | TYPE | FEATURES |
| ------ | ---- | ------- |
|num_songs| int| |
|artist_id| varchar| |
|artist_latitude | decimal | |
|artist_longitude| decimal| |
|artist_location| varchar| |
|artist_name | varchar | |
|song_id| varchar| |
|title| varchar| |
|duration | decimal | |
|year | int | |


##### TABLE staging_events

| COLUMN | TYPE | FEATURES |
| ------ | ---- | ------- |
|artist| varchar| |
|auth| varchar| |
|firstName | varchar | |
|gender| varchar| |
|itemInSession | int| |
|lastName | varchar | |
|length| decimal| |
|level| varchar| |
|location | varchar| |
|method | varchar| |
|page | varchar | |
|registration| varchar| |
|sessionId| int| |
|song | varchar| |
|status| int| |
|ts| timestamp| |
|userAgent| varchar| |
|userId| int| |


#### Dimension tables

##### TABLE users

| COLUMN | TYPE | FEATURES |
| ------ | ---- | ------- |
|user_id| int| distkey, PRIMARY KEY |
|first_name| varchar| |
|last_name | varchar | |
|gender| varchar| |
|level| varchar| |

##### TABLE songs

| COLUMN | TYPE | FEATURES |
| ------ | ---- | ------- |
|song_id| varchar| sortkey, PRIMARY KEY |
|title| varchar| NOT NULL |
|artist_id | varchar | NOT NULL|
|duration| decimal| |

##### TABLE artists

| COLUMN | TYPE | FEATURES |
| ------ | ---- | ------- |
|artist_id| varchar| sortkey, PRIMARY KEY |
|name| varchar| NOT NULL |
|location | varchar | |
|latitude| decimal| |
|logitude| decimal| |


##### TABLE time

| COLUMN | TYPE | FEATURES |
| ------ | ---- | ------- |
|start_time| timestamp| sortkey, PRIMARY KEY |
|hour| int| |
|day| int| |
|week| int| |
|month| int| |
|year| int| |
|weekday| int| |

#### Fact table

##### TABLE songplays

| COLUMN | TYPE | FEATURES |
| ------ | ---- | ------- |
|songplay_id| int| IDENTITY (0,1), PRIMARY KEY |
|start_time| timestamp| REFERENCES  time(start_time)    sortkey|
|user_id | int | REFERENCES  users(user_id) distkey|
|level| varchar| |
|song_id| varchar| REFERENCES  songs(song_id)|
|artist_id | varchar | REFERENCES  artists(artist_id)|
|session_id| int| NOT NULL|
|location| varchar| |
|user_agent| varchar| |

--------------------------------------------

### ETL process

All the transformations logic (ETL) is done in SQL inside Redshift. 

There are 2 main steps:

1. Ingest data from s3 public buckets into staging tables:
2. Insert record into a star schema from staging tables

#### Insert data into staging tables

<b>Insert into staging events:</b>
~~~~
 staging_events_copy = ("""
    COPY staging_events 
        FROM {} 
        iam_role {} 
        region {}
        FORMAT AS JSON {} 
        timeformat 'epochmillisecs'
        TRUNCATECOLUMNS BLANKSASNULL EMPTYASNULL;
""").format(LOG_DATA, ARN, REGION, LOG_JSON_PATH)
~~~~

<b>Insert into staging songs:</b>
~~~~
staging_songs_copy = ("""
    COPY staging_songs 
        FROM {}
        iam_role {}
        region {}
        FORMAT AS JSON 'auto' 
        TRUNCATECOLUMNS BLANKSASNULL EMPTYASNULL;
""").format(SONG_DATA, ARN, REGION)
~~~~

#### Insert data into star schema from staging tables

<b>Insert Dimensions:</b>
~~~~

user_table_insert = ("""
    INSERT INTO users (user_id, first_name, last_name, gender, level)
        SELECT DISTINCT se.userId, 
                        se.firstName, 
                        se.lastName, 
                        se.gender, 
                        se.level
        FROM staging_events se
        WHERE se.userId IS NOT NULL;
""")


song_table_insert = ("""
    INSERT INTO songs (song_id, title, artist_id, year, duration) 
        SELECT DISTINCT ss.song_id, 
                        ss.title, 
                        ss.artist_id, 
                        ss.year, 
                        ss.duration
        FROM staging_songs ss
        WHERE ss.song_id IS NOT NULL;
""")

artist_table_insert = ("""
    INSERT INTO artists (artist_id, name, location, latitude, logitude)
        SELECT DISTINCT ss.artist_id, 
                        ss.artist_name, 
                        ss.artist_location,
                        ss.artist_latitude,
                        ss.artist_longitude
        FROM staging_songs ss
        WHERE ss.artist_id IS NOT NULL;
""")

time_table_insert = ("""
    INSERT INTO time (start_time, hour, day, week, month, year, weekday)
        SELECT DISTINCT  se.ts,
                        EXTRACT(hour from se.ts),
                        EXTRACT(day from se.ts),
                        EXTRACT(week from se.ts),
                        EXTRACT(month from se.ts),
                        EXTRACT(year from se.ts),
                        EXTRACT(weekday from se.ts)
        FROM staging_events se
        WHERE se.page = 'NextSong';
""")
~~~~

<b>Insert Facts table:</b>
~~~~
songplay_table_insert = ("""
    INSERT INTO songplays (start_time, user_id, level, song_id, artist_id, session_id, location, user_agent) 
        SELECT DISTINCT se.ts, 
                        se.userId, 
                        se.level, 
                        ss.song_id, 
                        ss.artist_id, 
                        se.sessionId, 
                        se.location, 
                        se.userAgent
        FROM staging_events se 
        INNER JOIN staging_songs ss 
            ON se.song = ss.title AND se.artist = ss.artist_name
        WHERE se.page = 'NextSong';
""")
~~~~

--------------------------------------------

#### Project structure

The structure is:

* <b> create_tables.py </b> - This script will drop old tables (if exist) ad re-create new tables
* <b> etl.py </b> - This script orchestrate ETL.
* <b> sql_queries.py </b> - This is the ETL. All the transformatios in SQL are done here.
* <b> /img </b> - Directory with images that are used in this markdown document
* <b> dhw.cfg - Credentials of AWS resources