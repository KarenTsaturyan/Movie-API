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


# Middleware
```go 
func (app *application) exampleMiddleware(next http.Handler) http.Handler {
// Any code here will run only once, when we wrap something with the middleware.
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    // Any code here will run for every request that the middleware handles.
        next.ServeHTTP(w, r)
    })
}
```


# MAIL SMTP
### used https://mailtrap.io/
### Activaiton
    1. As part of the registration process for a new user we will create a cryptographicallysecure random activation token that is impossible to guess.
    2. We will then store a hash of this activation token in a new tokens table, alongside the
    new user’s ID and an expiry time for the token.
    3. We will send the original (unhashed) activation token to the user in their welcome email.
    4. The user subsequently submits their token to a new PUT /v1/users/activated
    endpoint.
    5. If the hash of the token exists in the tokens table and hasn’t expired, then we’ll update
    the activated status for the relevant user to true.
    6. Lastly, we’ll delete the activation token from our tokens table so that it can’t be used
    again.

### AUTH

#### 1. Basic Authentication
- Client sends credentials (username:password) base64-encoded in Authorization header
- **Example:** `Authorization: Basic YWxpY2VAZXhhbXBsZS5jb206cGE1NXdvcmQ=`
- **Not ideal for hashed passwords** — requires expensive password hash verification on every request

#### 2. Stateful Token Authentication
- Client submits credentials → API generates & returns a bearer token
- Token stored server-side (in DB) with user ID and expiry time
- Client includes token in subsequent requests: `Authorization: Bearer <token>`
- **Pros:** Full control over tokens, easy to revoke, better than basic auth
- **Cons:** Requires database lookup on every request

#### 3. Stateless Token Authentication
- Token encodes user ID & expiry time, cryptographically signed/encrypted (JWT, PASETO, etc.)
- No database lookup needed — all info contained in token
- **Pros:** No DB lookup required, faster
- **Cons:** Can't easily revoke tokens once issued (unless you maintain a blocklist)

#### 4. API Key Authentication
- User has non-expiring secret key; hash stored in DB alongside user ID
- Client passes key in header: `Authorization: Key <key>`
- **Pros:** Simple for clients, no token expiry management
- **Cons:** Adds complexity (key regeneration), two secrets to manage (password + key)

#### 5. OAuth 2.0 / OpenID Connect
- User info stored with third-party provider (Google, Facebook, etc.)
- **Note:** OAuth 2.0 is NOT an auth protocol — use OpenID Connect instead
- Flow: Redirect to provider → user consents → receive ID token (JWT) → validate & decode
- **Pros:** No need to store passwords, delegated auth
- **Cons:** Complex implementation, requires user account with provider, needs browser interaction

    * If your API doesn’t have ‘real’ user accounts with slow password hashes, then HTTP basic
    authentication can be a good — and often overlooked — fit.
    * If you don’t want to store user passwords yourself, all your users have accounts with a
    third-party identity provider that supports OpenID Connect, and your API is the back-end
    for a website… then use OpenID Connect.
    * If you require delegated authentication, such as when your API has a microservice
    architecture with different services for performing authentication and performing other
    tasks, then use stateless authentication tokens.
    * Otherwise use API keys or stateful authentication tokens. In general:
        * Stateful authentication tokens are a nice fit for APIs that act as the back-end for a website or single-page application, as there is a natural moment when the user logsin where they can be exchanged for user credentials.
        * In contrast, API keys can be better for more ‘general purpose’ APIs because they’re permanent and simpler for developers to use in their applications and scripts.

### CORS

**Cross-Origin Resource Sharing (CORS)** is a security mechanism that allows resources on a web page to be requested from another domain outside the domain from which the first resource was served.

#### Simple Requests
A request is considered **simple** if it meets ALL of the following criteria:
- **Method:** GET, HEAD, or POST
- **Headers:** Only the following are allowed:
  - `Accept`
  - `Accept-Language`
  - `Content-Language`
  - `Content-Type` (with specific MIME types only: `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain`)
  - `DPR`, `Downlink`, `Save-Data`, `Viewport-Width`, `Width`
- **Credentials:** Request does NOT include cookies or HTTP authentication
- **No event listeners:** No `XMLHttpRequestUpload` event listeners are attached

For simple requests, the browser sends the request directly with an `Origin` header. The server responds with `Access-Control-Allow-Origin` and the browser checks if the origin matches before making data available to JavaScript.

#### Preflight Requests
If a request does **NOT** meet the criteria for a simple request, the browser automatically sends a **preflight** request first (an OPTIONS request) to check if the actual request is allowed.

Preflight is triggered by:
- **Methods:** PUT, DELETE, PATCH, or other custom methods
- **Custom Headers:** Any header not in the simple list (e.g., `Authorization`, `X-Custom-Header`)
- **Content-Type:** `application/json`, `application/xml`, or other non-simple MIME types
- **Credentials:** Request includes credentials (cookies, authentication)

**Preflight Flow:**
```
1. Browser sends OPTIONS request with headers:
   - Origin: https://example.com
   - Access-Control-Request-Method: PUT
   - Access-Control-Request-Headers: Content-Type, Authorization

2. Server responds with CORS headers:
   - Access-Control-Allow-Origin: https://example.com
   - Access-Control-Allow-Methods: GET, POST, PUT, DELETE
   - Access-Control-Allow-Headers: Content-Type, Authorization
   - Access-Control-Max-Age: 86400 (caches preflight for this duration)

3. If preflight succeeds, browser sends the actual request
4. If preflight fails, browser blocks the actual request
```

#### Key CORS Response Headers
- **Access-Control-Allow-Origin:** Specifies which origins can access the resource
  - Example: `Access-Control-Allow-Origin: https://example.com` or `*` (all origins)
- **Access-Control-Allow-Methods:** Specifies allowed HTTP methods
  - Example: `Access-Control-Allow-Methods: GET, POST, PUT, DELETE`
- **Access-Control-Allow-Headers:** Specifies allowed request headers
  - Example: `Access-Control-Allow-Headers: Content-Type, Authorization`
- **Access-Control-Max-Age:** How long (in seconds) preflight responses can be cached
  - Example: `Access-Control-Max-Age: 86400`
- **Access-Control-Allow-Credentials:** Whether credentials (cookies, auth) are allowed
  - Example: `Access-Control-Allow-Credentials: true`
- **Vary: Origin:** Indicates the response varies based on the Origin header (important for caching)

#### CORS with Credentials
If your API needs to handle requests with credentials (cookies, HTTP auth):
1. Server must set: `Access-Control-Allow-Credentials: true`
2. Client must set credentials in the request (e.g., `credentials: 'include'` in fetch)
3. Server **cannot** use `*` for `Access-Control-Allow-Origin` — must specify the exact origin

#### CORS Middleware Implementation
A typical CORS middleware should:
1. Extract the `Origin` header from the request
2. Check if the origin is in your allowed list
3. For preflight (OPTIONS) requests, respond with appropriate CORS headers
4. For actual requests, include CORS headers in the response
5. Always include `Vary: Origin` to prevent caching issues

# INFO
https://pgtune.leopard.in.ua/

So because of this we should make sure to always set a Vary: Origin response header to
warn any caches that the response may be different. This is actually really important, and it
can be the cause of subtle bugs like this one if you forget to do it. As a rule of thumb:
If your code makes a decision about what to return based on the content of a request header,
you should include that header name in your Vary response header — even if the request
didn’t include that header.

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