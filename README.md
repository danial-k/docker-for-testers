# Introduction
This repository is a collection of tutorials and starter projects to help test software more efficiently using Docker.
Docker is an application containerisation technology that shares the underlying operating system kernel between isolated processed.
It provides three key benefits to teams that ship software:
- **Multiple runtimes simulataneously:**  For example, Python 3.4, 3.5, 3.6 and 3.7 or Ruby 2.2, 2.3 and 2.4 may all run at the same time in isolation without version managers.
- **Consistent deployments:** A built Docker image is almost guaranteed to run anywhere in the deployment pipeline.  A container that builds in Dev should run in QA/Test, Staging/Pre-prod and Live/Prod.  Key things to bear in mind are Docker engine version (e.g. older Docker versions didn't support multi-stage builds), significant difference in OS kernel versions (especially in [Windows containers](https://docs.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/version-compatibility)) and [well-defined](https://12factor.net/) configuration (e.g. environment variables).
- **Higher application densities:**  By removing the overhead of memory, CPU and storage allocation associated with virtual machines, significantly fewer resources are required to run the same number of applications.  Do remember that unless limits are specified, containers could consume all available memory leading to unexpected behaviour.

One of the factors contributing to the success of Docker is the [Docker Hub](https://hub.docker.com). This is a catalog of available iamges and includes images from the leading vendors including Microsoft, Oracle, IBM and Red Hat.

# Tutorial 1: Databases
In this example, the ease at which a variety of different databases may be set up will be demonstrated.
We will set up three different database technologies on the same machine and see how much faster it is than a traditional deployment.
We will use the Flyway database migration tool to run a collection of sample SQL scripts against each database

## Creating a Docker network
A docker network is a virtual bridge network that provides DNS resolution based on container names.  Create a new network with the following:
```shell
docker network create tutorial-1
```

## Starting a Microsoft SQL Server database
```shell
docker run \
--name sqlserver \
--network tutorial-1 \
-e 'ACCEPT_EULA=Y' \
-e 'SA_PASSWORD=Password123!' \
-e 'MSSQL_PID=express' \
-p 9100:1433 \
mcr.microsoft.com/mssql/server:2017-CU14-ubuntu
```

## Starting a MySQL database
```shell
docker run \
--name mysql \
--network tutorial-1 \
-p 9101:3306 \
-e 'MYSQL_ROOT_PASSWORD=Password123!' \
-e 'MYSQL_DATABASE=tutorial1' \
mysql:5.7.27
```

## Populating the databases
The Java-based (Flyway)[https://flywaydb.org/] database migration tool is also available as a Docker container.
The SQL files will be mounted to the container.
Once the operation has completed, the container will stop and will be automatically be removed (```--rm``` flag).
Flyway will create a migration tracking table that is used to manage ongoing changes to a database's schema.
Each database engine will have a different configuration
To populate the SQL Server database run:
```shell
docker run \
--rm \
--network tutorial-1 \
--mount type=bind,src=`pwd`/sql,dst=/flyway/sql \
flyway/flyway:6.0.3 \
-user='sa' \
-password='Password123!' \
-url='jdbc:sqlserver://sqlserver:1433' \
-schemas='tutorial1' \
migrate
```

To populate the MySQL database run:
```shell
docker run \
--rm \
--network tutorial-1 \
--mount type=bind,src=`pwd`/sql,dst=/flyway/sql \
flyway/flyway:6.0.3 \
-user='root' \
-password='Password123!' \
-url='jdbc:mysql://mysql:3306/tutorial1' \
-schemas='tutorial1' \
migrate
```

## Connecting to the databases with Grafana
Grafana is a visualisation platform that allows reading from multiple data sources, including relational databases.
The platforms allows rapid creation of dashboards with a variety of different charts.  To start an instance of Grafana in the same network as the databases, run:
```shell
docker run \
--name grafana \
--network tutorial-1 \
-p 9102:3000 \
grafana/grafana:6.3.5
```
Log in with the default credentials ```admin/admin```.
Create a database connection and configure the SQL Server endpoint.
Add the following to a query:
```SQL
SELECT
  $__timeEpoch(event_timestamp),
  event_value as value,
  'sample' as metric
FROM
  tutorial1.events
WHERE
  $__timeFilter(event_timestamp)
ORDER BY
  event_timestamp ASC
```
Adjust the date range accordingly to view entries.