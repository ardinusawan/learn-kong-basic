# learn-kong-basic

## Preparations
1. Create docker network for kong
```sh
docker network create kong-net
```

2. Start database for kong
```sh
docker run -d --name kong-database \
--network=kong-net \
-p 5555:5432 \
-e "POSTGRES_USER=kong" \
-e "POSTGRES_DB=kong" \
-e "POSTGRES_PASSWORD=kong" \
postgres:9.6
```

3. Prepare database and run Kong migration
```sh
docker run --rm \
--network=kong-net \
-e "KONG_DATABASE=postgres" \
-e "KONG_PG_HOST=kong-database" \
-e "KONG_PG_PASSWORD=kong" \
kong:latest kong migrations bootstrap
```

4. Start the kong, it depend on `kong-database` container and migration
```sh
docker run -d --name kong \
--network=kong-net \
-e "KONG_LOG_LEVEL=debug" \
-e "KONG_DATABASE=postgres" \
-e "KONG_PG_HOST=kong-database" \
-e "KONG_PG_PASSWORD=kong" \
-e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
-e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
-e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
-e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
-e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
-p 9000:8000 \
-p 9443:8443 \
-p 9001:8001 \
-p 9444:8444 \
kong:latest
```

5. Check Kong Instance
```sh
curl -i http://localhost:9001
```

The response should be
```
HTTP/1.1 200 OK
Date: Sat, 28 Mar 2020 13:32:50 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Access-Control-Allow-Origin: *
Server: kong/2.0.2
Content-Length: 8821
X-Kong-Admin-Latency: 1

{"plugins":{"enabled_in_clus.....
```

Kong is up and ready to use.


## Setup API Server
We will use nodejs. To make it simple, clone from [here](https://github.com/faren/NodeJS-API-KONG)

Inside project folder, lets build with docker using command:
```sh
docker build -t node_kong .
```
When build finish, run it
```sh
docker run -d --name=node_kong --network=kong-net node_kong
```
Check if the all docker container we need run with command `docker ps`

We should have three container running
```sh
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                                                                            NAMES
6331440480d5        node_kong           "docker-entrypoint.s…"   56 seconds ago      Up 56 seconds       10000/tcp                                                                                        node_kong
d4495855da83        kong:latest         "/docker-entrypoint.…"   44 minutes ago      Up 44 minutes       0.0.0.0:9000->8000/tcp, 0.0.0.0:9001->8001/tcp, 0.0.0.0:9443->8443/tcp, 0.0.0.0:9444->8444/tcp   kong
e12e70d1f830        postgres:9.6        "docker-entrypoint.s…"   56 minutes ago      Up 56 minutes       0.0.0.0:5555->5432/tcp                                                                           kong-database

```
Get IP container on docker network kong-net.
```sh
docker network inspect kong-net
```
Get `IPv4Address` on `node_kong` container. Save for later

Go inside kong container
```sh
docker exec -it kong sh
```
Call `node_kong` with curl
```sh
curl -i <your_node_kong_ip>:10000/api/v1/customers
```
Response should be like this
```
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: application/json; charset=utf-8
Content-Length: 110
ETag: W/"6e-Tf3vAGLC3XH0dFR2pCIzWdG8/5c"
Date: Sat, 28 Mar 2020 14:12:48 GMT
Connection: keep-alive

[{"id":5,"first_name":"Dodol","last_name":"Dargombez"},{"id":6,"first_name":"Nyongot","last_name":"Gonzales"}]
```
Above respond showing the node.js server API is alive and can serve GET method REST /api/v1/customers

## Setup KONG as API-gateway to API server routing
This is current structure

![kong structure](https://raw.githubusercontent.com/ardinusawan/learn-kong-basic/master/kong%20structure.png)

Kong is gateway. Service that has been defined in Kong will be direct to API server that is ready to serve.

For instance:

API server (inside kong_net network): http://192.168.192.4:10000/api/v1/customers

API Kong (host network): http://localhost:9000

On Kong, we set routes path to /api/v1/customers

And set services host to 192.168.192.4:10000, and path /api/v1/customers

So, when client hit Kong http://localhost:9000/api/v1/customers, Kong will proxy it to 172.19.0.4:10000/api/v1/customers

To start please import postman collection file on github [NodeJS-API-KONG](https://github.com/faren/NodeJS-API-KONG) — kong.postman_collection.json.

Import it, and on your postman folder named `Kong` will exist

### Add service: customers
Select Kong -> Services -> Services - Create

On postman, make sure your request like this, and got successfully response:

![Create new services](https://raw.githubusercontent.com/ardinusawan/learn-kong-basic/master/create.png)

### List service
On postman, select Kong -> Services -> Services - List

Response should be:
```json
{
    "next": null,
    "data": [
        {
            "host": "192.168.192.4",
            "created_at": 1585491636,
            "connect_timeout": 60000,
            "id": "7da29f51-4a2a-49ba-b3ab-696ef2c87246",
            "protocol": "http",
            "name": "api-v1-customers",
            "read_timeout": 60000,
            "port": 10000,
            "path": "/api/v1/customers",
            "updated_at": 1585491636,
            "retries": 5,
            "write_timeout": 60000,
            "tags": null,
            "client_certificate": null
        }
    ]
}
```
Now we successfully created service customers! Huray!!

### Add Routes: customer
On postman, select Kong -> Routes -> POST Routes — Create

Edit body data, it should be like this:
```json
{
  "hosts": ["localhost:9000"],
  "paths": ["/api/v1/customers"]
}
```
Response:
```json
{
    "id": "1e136893-0608-45d0-9abd-a41708e966dd",
    "path_handling": "v0",
    "paths": [
        "/api/v1/customers"
    ],
    "destinations": null,
    "headers": null,
    "protocols": [
        "http",
        "https"
    ],
    "methods": null,
    "snis": null,
    "service": {
        "id": "7da29f51-4a2a-49ba-b3ab-696ef2c87246"
    },
    "name": null,
    "strip_path": true,
    "preserve_host": false,
    "regex_priority": 0,
    "updated_at": 1585495033,
    "sources": null,
    "hosts": [
        "localhost:9000"
    ],
    "https_redirect_status_code": 426,
    "tags": null,
    "created_at": 1585495033
}
```
