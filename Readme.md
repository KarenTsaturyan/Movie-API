# Setup
## Postgres
sudo -u postgres psql

CREATE DATABASE movies_api;

\c movies_api

movies_api=# CREATE ROLE movies_api WITH LOGIN PASSWORD 'pa55word';

movies_api=# CREATE EXTENSION IF NOT EXISTS citext;

## Connecting as a new user
psql --host=localhost --dbname=movies_api --username=movies_api

### DNS 
    postgres://{username}:{pass}@{host}/{DBname}.
 
  OR

    postgres://{username}:{pass}@{host}/{DBname}?sslmode=disable


### ENV
Terminal:
```bash
    nano $HOME/.profile
    export DB_DSN=.....
    ...save
    source $HOME/.profile
    echo $DB_DSN
``` 
## Migrations
 ### tool
 ```bash
 $ curl -L https://github.com/golang-migrate/migrate/releases/download/v4.14.1/migrate.linux-amd64.tar.gz | tar xvz

$ mv migrate.linux-amd64 $GOPATH/bin/migrate

$ migrate -version

# Create a table
$ migrate create -seq -ext=.sql -dir=./migrations create_movies_table
```
ex.
Update up and down files, 
up -> CREATE TABLE ...
down -> DROP TABLE ...

* NOTE: Might need permission GRANT for new user from ROOT
* Migrate
```bash 
 $ migrate -path=./migrations -database=$DB_DSN up
```

* Version check
```bash 
$ migrate -path=./migrations -database=$EXAMPLE_DSN version
```

* You can also migrate up or down to a specific version by using the goto command:
```bash 
$ migrate -path=./migrations -database=$EXAMPLE_DSN goto 1
```
* You can use the down command to roll-back by a specific number of migrations. For
example, to rollback the most recent migration you would run:

```bash 
$ migrate -path=./migrations -database =$EXAMPLE_DSN down 1
```

If the migration file which failed contained multiple SQL statements, then it’s possible that
the migration file was partially applied before the error was encountered. In turn, this
means that the database is in an unknown state as far as the migrate tool is concerned.
Accordingly, the version field in the schema_migrations field will contain the number for
the failed migration and the dirty field will be set to true. At this point, if you run another
migration (even a ‘down’ migration) you will get an error message similar to this:

```Dirty database version {X}.Fix and force version.```
 What you need to do is investigate the original error and figure out if the migration file
which failed was partially applied. If it was, then you need to manually roll-back the
partially applied migration.
Once that’s done, then you must also ‘force’ the version number in the schema_migrations
table to the correct value. For example, to force the database version number to 1 you
should use the force command like so:
```$ migrate -path=./migrations -database=$EXAMPLE_DSN force 1```


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