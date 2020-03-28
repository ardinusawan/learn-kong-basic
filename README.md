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
