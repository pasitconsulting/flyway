# sql code migrations using Flyway

## prereqs:-
docker installed locally

## intro
this repo contains a docker-compose file with 2 containers:- 

Flyway (built from local Dockerfile, based on docker hub flyway/flyway)

Postgresql (dockerhub image)

# Install Instructions:
1) git clone this repo
    git clone https://github.com/pasitconsulting/liquibase.git

2) build image into local repo:- 

 ```docker build -t myname/flyway:1.0 . ```

3) edit ```docker-compose.yml``` file to refer to above tag name
```
version: '3.2'
services:
  flyway:
    image: myname/flyway:1.0                                  <===MODIFY THIS LINE WITH CORRECT LOCAL DOCKER REPO IMAGE
    command: migrate -X                                       <===ACTUAL FLYWAY COMMAND; values can be 'info' or 'migrate'
    volumes:
      - type: bind
        source: /Users/paulsenior/scratch/docker/flyway/sql
        target: /flyway/sql
      - type: bind
        source: /Users/paulsenior/scratch/docker/flyway/conf/flyway.conf
        target: /flyway/flyway.conf
    depends_on:
      - db
  db:
    image: postgres
    restart: always
    environment:
      - POSTGRES_PASSWORD:hello123
    ports:
      - 5432:5432
```

start both containers as an app:-

    docker-compose up


## To verify your db schema changes are successful:-

1) you should get the following to stdout when running docker-compose up:-
here is end of stdout if you run 'command: info -X'  
```
flyway_1  | Schema version: 5
flyway_1  | 
flyway_1  | +-----------+---------+---------------------------+------+---------------------+---------+
flyway_1  | | Category  | Version | Description               | Type | Installed On        | State   |
flyway_1  | +-----------+---------+---------------------------+------+---------------------+---------+
flyway_1  | | Versioned | 1       | initial setup             | SQL  | 2020-01-24 10:27:19 | Success |
flyway_1  | | Versioned | 2       | added town column         | SQL  | 2020-01-24 10:27:19 | Success |
flyway_1  | | Versioned | 3       | add test2 schema          | SQL  | 2020-01-24 10:27:19 | Success |
flyway_1  | | Versioned | 4       | add test5 schema          | SQL  | 2020-01-24 10:35:34 | Success |
flyway_1  | | Versioned | 5       | undo V4 drop test5 schema | SQL  | 2020-01-24 10:49:54 | Success |
flyway_1  | +-----------+---------+---------------------------+------+---------------------+---------+
```

here is end of stdout if you run 'command: migrate -X'
```
flyway_1  | Successfully validated 5 migrations (execution time 00:00.057s)
flyway_1  | Current version of schema "public": 4
flyway_1  | DEBUG: Parsing V5__undo_V4_drop_test5_schema.sql ...
flyway_1  | DEBUG: Found statement at line 1: drop schema test5
flyway_1  | DEBUG: Starting migration of schema "public" to version 5 - undo V4 drop test5 schema ...
flyway_1  | Migrating schema "public" to version 5 - undo V4 drop test5 schema
flyway_1  | DEBUG: Executing SQL: drop schema test5
flyway_1  | DEBUG: Update Count: 0
flyway_1  | DEBUG: Successfully completed migration of schema "public" to version 5 - undo V4 drop test5 schema
flyway_1  | DEBUG: Schema History table "public"."flyway_schema_history" successfully updated to reflect changes
flyway_1  | Successfully applied 1 migration to schema "public" (execution time 00:00.121s)
```

2) run a psql session in the running postgres container to verify schema objects, e.g. to confirm tables:-

``docker ps``   <==FIND CONTAINER ID OF RUNNING POSTGRES

```docker exec -it [POSTGRES CONTAINER ID] bash```

```psql -h localhost -p 5432 -U postgres```

```postgres=# \dt```  <== RUN '\dt' on POSTGRES PROMPT



# Flyway Configuration
check these configuration files:-

`/conf`    THIS IS MAIN CONFIG FILE FOR FLYWAY - JDBC CONNECTION STRING, CREDENTIALS ETC
```
/Users/paulsenior/scratch/docker/flyway/conf
PaulSenior-MacBook-Pro:conf paulsenior$ cat flyway.conf | grep -v "^#" | grep -v "^$"
flyway.url=jdbc:postgresql://db:5432/postgres?user=postgres&password=hello123
 flyway.user="postgres"
 flyway.password=
```

`/sql`    THIS DIRECTORY CONTAINS ANY SQL STATEMEMTS TO PUSH TO THE DATABASE - see sample changelog below

```
PaulSenior-MacBook-Pro:sql paulsenior$ ls
V1__initial_setup.sql			V3__add_test2_schema.sql		V5__undo_V4_drop_test5_schema.sql
V2__added_town_column.sql		V4__add_test5_schema.sql		put-your-sql-migrations-here.txt

PaulSenior-MacBook-Pro:sql paulsenior$ cat V4__add_test5_schema.sql 
create schema test5;
```

##Making schema changes

to make SQL schema changes you will need to create & modify a file under /conf with the filename format `V[n]__[description].sql`  and then rebuild the dockerfile and rerun docker-compose (as per above instructions)



## Reference
https://flywaydb.org/getstarted/


## Tips & Tricks
### rollbacks
Rollback sql statements - these need the Pro license, i.e if you try to run a `U[n]__[description].sql` statement you will get a warning to upgrade to Pro license

Rollbacks can be acheived just my creating a new `V[n]__[description].sql` file and putting a rollback sql statement (drop table etc) in there and rerun
