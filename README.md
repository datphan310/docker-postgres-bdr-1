### PostgreSQL with [BDR](https://2ndquadrant.com/en/resources/bdr/) B-Directional Replication in a docker.

[Docker Image](https://registry.hub.docker.com/u/polinux/postgres-bdr/) with PostgreSQL server with [BDR](https://2ndquadrant.com/en/resources/bdr/) support for database **Bi-Directional** replication. Based on [CentOS with Supervisor](https://hub.docker.com/r/million12/centos-supervisor/).

User can specify if database should work as `stand-alone` or `master/slave` mode. Even though BDR is more `master/master` solution we need to deploy first image which will create BDR setup inside of PostgreSQL. So for this purpose on `run` we need to specify one image to be so called master.

### Environmental Variables


| Variable     | Meaning     |
| :-----------:| :---------- |
|`POSTGRES_PASSWORD`|Self explanatory|
|`POSTGRES_USER`|Self explanatory|
|`POSTGRES_DB`|Self explanatory|
|`POSTGRES_PORT`|Self explanatory|
|`MODE`|Mode in which this image should work. Options: `single/master/slave` (default=single)|
|`MASTER_ADDRESS`|Address of master node|
|`MASTER_PORT`|Master node port|
|`SLAVE_PORT`|Slave node port|
|`POSTGRES_SHAREDBUFFERS`|Shared memory|
|`POSTGRES_MAXCONNECTIONS`|Max connections - [postgres max connection issue](https://stackoverflow.com/questions/30778015/how-to-increase-the-max-connections-in-postgres?rq=1)|


### Note:
If you using docker on windows, open all ".sh" files, then use notepad++, go to edit -> EOL conversion -> change from CRLF to LF

### Usage

#### Build docker image
	docker build -t postgres-bdr .

#### Stand-Alone mode

    docker run \
      -d \
      --name postgres \
      -p 5432:5432 \
      postgres-bdr

#### Master + 2 slaves

Use `docker-compose.yml` exmple.

#### Deploy all at once from `docker-compose-yml` file.

    docker compose up


#### Deploy master only

    docker compose up master


#### Deploy slave1 only

    docker compose up slave1

#### Deploy slave2 only

    docker compose up slave2

Docker troubleshooting
======================

Use docker command to see if all required containers are up and running:
```
$ docker ps
```

Check logs of postgres-bdr server container:
```
$ docker logs postgres-bdr
```

Sometimes you might just want to review how things are deployed inside a running
 container, you can do this by executing a _bash shell_ through _docker's
 exec_ command:
```
docker exec -ti postgres-bdr /bin/bash
```

History of an image and size of layers:
```
docker history --no-trunc=true postgres-bdr | tr -s ' ' | tail -n+2 | awk -F " ago " '{print $2}'
```

### Config PostgreSQL global sequences (IMPORTANT)
Run scripts below on each node target to bdr database
```
DO $$
DECLARE
i TEXT;
BEGIN
  FOR i IN (
    SELECT 'SELECT SETVAL('
        || quote_literal(quote_ident(PGT.schemaname) || '.' || quote_ident(S.relname))
        || ', COALESCE(MAX(' ||quote_ident(C.attname)|| '), 1) ) FROM '
        || quote_ident(PGT.schemaname)|| '.'||quote_ident(T.relname)|| ';'
      FROM pg_class AS S,
           pg_depend AS D,
           pg_class AS T,
           pg_attribute AS C,
           pg_tables AS PGT
     WHERE S.relkind = 'S'
       AND S.oid = D.objid
       AND D.refobjid = T.oid
       AND D.refobjid = C.attrelid
       AND D.refobjsubid = C.attnum
       AND T.relname = PGT.tablename
  ) LOOP
            raise notice 'seqname: %', i;
      EXECUTE i;
  END LOOP;
END $$;
```