# Installing Kong
For this tutorial, we’re going to use docker containers. Other ways to install Kong can be found here: https://konghq.com/install

To begin with, we need to create a network so that the containers can discover and communicate with each other.
```
$ docker network create kong-net
```
2. We will be using PostgreSQL as Kong’s database (it can be either that or Cassandra).
```
$ docker run -d --name kong-database \ 
                --network=kong-net \ 
                -p 5432:5432 \ 
                -e “POSTGRES_USER=kong” \ 
                -e “POSTGRES_DB=kong” \ 
                postgres:9.6
```
3. Let us now prepare our database by starting an ephemeral Kong container which will run the appropriate migrations and die!
```
$ docker run --rm \     
             --network=kong-net \     
             -e "KONG_DATABASE=postgres" \     
             -e "KONG_PG_HOST=kong-database" \     
             kong:latest kong migrations up
```
4. Everything is now set and ready to start Kong!
```
$ docker run -d --name kong \     
             --network=kong-net \     
             -e "KONG_DATABASE=postgres" \     
             -e "KONG_PG_HOST=kong-database" \         
             -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \     
             -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \     
             -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \     
             -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \     
             -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \     
             -p 8000:8000 \     
             -p 8443:8443 \     
             -p 8001:8001 \     
             -p 8444:8444 \     
             kong:latest
```
5. Test it out!

Kong’s admin API is exposed on port 8001 and the gateway on port 8000
```
$ curl -i http://localhost:8001/
$ curl -i http://localhost:8000/
```
# Installing Konga
Konga ports with it’s own file system storage and although it’s not recommended for production, it can be safely used as long as kongadata folder persists in a volume.

If you decide to go down that road, running Konga is as simple as this:
```
$ docker run -d -p 1337:1337 \
             --network=kong-net \
             --name konga \
             -v /var/data/kongadata:/app/kongadata \
             -e "NODE_ENV=production" \
             pantsel/konga
```
For this tutorial, since we already have a running PostgreSQL instance, we might as well make our lifes a bit more complicated and use it.

Like before, we will need to prepare Konga’s database by starting an ephemeral container.
```
$ docker run --rm \ 
    --network=kong-net \ 
    pantsel/konga -c prepare -a postgres -u postgresql://kong@kong-database:5432/konga_db
```
When the migrations run, we can start the app.
```
$ docker run -p 1337:1337 \
             --network=kong-net \
             -e "DB_ADAPTER=postgres" \
             -e "DB_HOST=kong-database" \
             -e "DB_USER=kong" \
             -e "DB_DATABASE=konga_db" \
             -e "KONGA_HOOK_TIMEOUT=240000" \
             -e "NODE_ENV=production" \
             --name konga \
             pantsel/konga
```
After a while, Konga will be available at:
```
http://<your-servers-public-ip-or-host>:1337
```
Login with the default credentials:
```
Login: admin | Password: adminadminadmin (don't forget to change it :p !!!)
```
