# learn-kong-basic

## Step

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
-e "POSTGRES_DB=kon" \
-e "POSTGRES_PASSWORD=kong" \
postgres:9.6
```
