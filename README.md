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
