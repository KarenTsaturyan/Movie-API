# Setup
## Postgres
sudo -u postgres psql

CREATE DATABASE movies_api;

\c movies_api

movies_api=# CREATE ROLE movies_api WITH LOGIN PASSWORD 'pa55word';

movies_api=# CREATE EXTENSION IF NOT EXISTS citext;

## Connecting as a new user
psql --host=localhost --dbname=movies_api --username=movies_api

## DNS 
### postgres://{DBname}:{pass}@{localhost}/greenlight.
### OR
### postgres://{DBname}:{pass}@{localhost}/greenlight?sslmode=disable


### ENV
Terminal:
```bash
    nano $HOME/.profile
    export DB_DSN=.....
    ...save
    source $HOME/.profile
    echo $DB_DSN
``` 




# INFO
https://pgtune.leopard.in.ua/
https://www.enterprisedb.com/postgres-tutorials/how-tune-postgresql-memory
## So how does the sql.DB connection pool work?
    The most important thing to understand is that a sql.DB pool contains two types of
    connections — ‘in-use’ connections and ‘idle’ connections. A connection is marked as in-use
    when you are using it to perform a database task, such as executing a SQL statement or
    querying rows, and when the task is complete the connection is then marked as idle.
    When you instruct Go to perform a database task, it will first check if any idle connections
    are available in the pool. If one is available, then Go will reuse this existing connection and
    mark it as in-use for the duration of the task. If there are no idle connections in the pool
    when you need one, then Go will create a new additional connection.
    When Go reuses an idle connection from the pool, any problems with the connection are
    handled gracefully. Bad connections will automatically be re-tried twice before giving up, at
    which point Go will remove the bad connection from the pool and create a new one to carry
    out the task.