# AliCloud_Demo_ODPS_DataWarehouse

# Dataworks

DataWorks is a Big Data platform product launched by Alibaba Cloud. It provides one-stop Big Data development, data permission management, offline job scheduling, and other features. You can read more on [product page](https://www.alibabacloud.com/product/ide). It includes key features, such as: 

1. Development Visualization: You can drag and drop nodes to create a workflow. You can also edit and debug your code online, and ask other developers to join you.
2. Multiple Task Types: Supports data integration, MaxCompute SQL, MaxCompute MR, machine learning, and shell tasks.
3. Strong Scheduling Capability: Runs millions of tasks concurrently and supports hourly, daily, weekly, and monthly schedules.
4. Task Monitoring and Alarms: Supports task monitoring and sends alarms when errors occur to avoid service interruptions.


# Data Model
![Alt text](/demo_screenshot/data_model.png)


# Workshop
## create Dataworks workspace
* It is recommended to create a workspace in __China East 2__ region (Shanghai). 
* It is recommended to create a workspace in __Standard__ mode. Standard will create a seperate Dev and Prod envionrment and would allow project control. 
![Alt text](/demo_screenshot/dataworks_create_workspace.jpg)
![Alt text](/demo_screenshot/dataworks_standard_mode.jpg)

## configure external data source for ingestion
### connect to mysql: [configuration](/config_mysql_in.sql)
![Alt text](/demo_screenshot/datasource_mysql_in.jpg)
```
Data Source Type: ApasaraDB for RDS
Data Source Name: rds_workshop_log
Description：rds log ingest
RDS Instance ID: rm-bp1z69dodhh85z9qa
RDS Instance Account: 1156529087455811
Database name: workshop
Username: workshop
Password: workshop#2017
```
### connect to oss: [configuration](/datasource_oss_in.jpg)
![Alt text](/demo_screenshot/datasource_oss_in.jpg)
```
Data Source Name：oss_workshop_log
Endpoint：http://oss-cn-shanghai-internal.aliyuncs.com
bucket：dataworks-workshop
AccessKey ID：LTAINEhd4MZ8pX64
AccessKey Key：lXnzUngTSebt3SfLYxZxoSjGAK6IaF
```
## create a virtual node for starting point
* make sure the very first virtual node has setup __root node__ dependency. 
![Alt text](/demo_screenshot/virtual_node_root.jpg)

## data ingestion for mysql [ODS]
### create table to host mysql data ingestion: [sql](sql_dd_e2e_mysql.sql)
* table partition is always recommended. 
```
CREATE TABLE IF NOT EXISTS ods_mysql_s_user (
    uid STRING COMMENT 'user ID',
    gender STRING COMMENT 'gender',
    age_range STRING COMMENT 'age range, e.g. 30-40 year old',
    zodiac STRING COMMENT 'zodiac'
)
PARTITIONED BY (
    dt STRING
);
```
### configure mysql ingestion task
![Alt text](/demo_screenshot/ingest_mysql.jpg)

### ods_user: [sql](/demo_screenshot/sql_ods_s_user_dd.sql)
```
CREATE TABLE IF NOT EXISTS ods_s_user_dd (
    uid STRING COMMENT 'user ID',
    gender STRING COMMENT 'gender',
    age_range STRING COMMENT 'age range, e.g. 30-40 year old',
    zodiac STRING COMMENT 'zodiac'
)
PARTITIONED BY (
    dt STRING
);

-- DML
INSERT OVERWRITE TABLE ods_s_user_dd PARTITION (dt=${bdp.system.bizdate})
SELECT uid, gender, age_range, zodiac
FROM ods_mysql_s_user
WHERE dt = ${bdp.system.bizdate};
```

## data ingestion for oss [ods]

### create table to host mysql data ingestion: [sql](sql_dd_e2e_mysql.sql)
* table partition is always recommended. 
```
CREATE TABLE IF NOT EXISTS ods_visit_log_dd (
    uid STRING COMMENT 'user ID',
    ip STRING COMMENT 'ip address',
    time STRING COMMENT 'time yyyymmddhh:mi:ss',
    http_status STRING COMMENT 'server responsed status code',
    traffic_bytes STRING COMMENT 'client responsed bite count',
    http_method STRING COMMENT 'http request type',
    url STRING COMMENT 'url',
    http_protocol STRING COMMENT 'http protocal version',
    host STRING COMMENT ' source url',
    device STRING COMMENT 'client type ',
    visit_type STRING COMMENT 'request type crawler feed user unknown',
    region STRING COMMENT 'geogrpahical location according to IP address'
) PARTITIONED BY (
    dt STRING
);

INSERT OVERWRITE TABLE ods_visit_log_dd PARTITION (dt=${bdp.system.bizdate})
SELECT 
    uid, ip, time, http_status, traffic_bytes, http_method, url, 
    http_protocol, host, device, visit_type,
    get_region_from_ip(ip) as region
FROM ods_tmp_visit_log_dd
WHERE dt = ${bdp.system.bizdate};
```
### configure oss ingestion task
![Alt text](/demo_screenshot/ingest_oss.jpg)

### UDF

### ods_visit_log: [sql](sql_dd_e2e_mysql.sql)
```
CREATE TABLE IF NOT EXISTS ods_tmp_visit_log_dd (
    uid STRING COMMENT 'user ID',
    ip STRING COMMENT 'ip address',
    time STRING COMMENT 'time yyyymmddhh:mi:ss',
    http_status STRING COMMENT 'server responsed status code',
    traffic_bytes STRING COMMENT 'client responsed bite count',
    http_method STRING COMMENT 'http request type',
    url STRING COMMENT 'url',
    http_protocol STRING COMMENT 'http protocal version',
    host STRING COMMENT ' source url',
    device STRING COMMENT 'client type ',
    visit_type STRING COMMENT 'request type crawler feed user unknown'
) PARTITIONED BY (
    dt STRING
);

INSERT OVERWRITE TABLE ods_tmp_visit_log_dd PARTITION (dt=${bdp.system.bizdate})
SELECT uid, ip, time, status, bytes, 
    regexp_substr(request, '(^[^ ]+ )') AS method, 
    regexp_extract(request, '^[^ ]+ (.*) [^ ]+$') AS url, 
    regexp_substr(request, '([^ ]+$)') AS protocol, --parse refer to get more accurate url, 
    regexp_extract(referer, '^[^/]+://([^/]+){1}') AS referer, --parse agent to get client infor and request method, 
    CASE
        WHEN TOLOWER(agent) RLIKE 'android' THEN 'android'
        WHEN TOLOWER(agent) RLIKE 'iphone' THEN 'iphone'
        WHEN TOLOWER(agent) RLIKE 'ipad' THEN 'ipad'
        WHEN TOLOWER(agent) RLIKE 'macintosh' THEN 'macintosh'
        WHEN TOLOWER(agent) RLIKE 'windows phone' THEN 'windows_phone'
        WHEN TOLOWER(agent) RLIKE 'windows' THEN 'windows_pc'
        ELSE 'unknown'
    END AS device, 
    CASE
        WHEN TOLOWER(agent) RLIKE '(bot|spider|crawler|slurp)' THEN 'crawler'
        WHEN TOLOWER(agent) RLIKE 'feed' OR regexp_extract(request, '^[^ ]+ (.*) [^ ]+$') RLIKE 'feed' THEN 'feed'
        WHEN TOLOWER(agent) NOT RLIKE '(bot|spider|crawler|feed|slurp)' AND agent RLIKE '^[Mozilla|Opera]' AND regexp_extract(request, '^[^ ]+ (.*) [^ ]+$') NOT RLIKE 'feed' THEN 'user'
        ELSE 'unknown'
    END AS identity
FROM (
    SELECT 
        SPLIT(text, '##@@')[0] AS ip, 
        SPLIT(text, '##@@')[1] AS uid, 
        SPLIT(text, '##@@')[2] AS time, 
        SPLIT(text, '##@@')[3] AS request, 
        SPLIT(text, '##@@')[4] AS status, 
        SPLIT(text, '##@@')[5] AS bytes, 
        SPLIT(text, '##@@')[6] AS referer, 
        SPLIT(text, '##@@')[7] AS agent
    FROM ods_oss_log_dd
    WHERE dt = ${bdp.system.bizdate}
) a;
```

## worktask overview
![Alt text](/demo_screenshot/workflow_overview.jpg)
