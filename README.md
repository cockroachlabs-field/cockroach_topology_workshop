# Mulit-Region CockroachDB Cluster with HAProxy
9 node multi-region CockroachDB cluster with HAProxy acting as load balancer per region

## Services
* `crdb-0` - EU CockroachDB node
* `crdb-0a` - EU CockroachDB node
* `crdb-0b` - EU CockroachDB node
* `crdb-1` - US CockroachDB node
* `crdb-1a` - US CockroachDB node
* `crdb-1b` - US CockroachDB node
* `crdb-2` - APAC CockroachDB node
* `crdb-2a` - APAC CockroachDB node
* `crdb-2b` - APAC CockroachDB node
* `lb_eu` - HAProxy acting as load balancer for EU Region
* `lb_us` - HAProxy acting as load balancer for US Region
* `lb_apac` - HAProxy acting as load balancer for APAC Region


## Getting started
1) run `docker-compose up`
2) visit the CockroachDB UI 
   1) http://localhost:8080 for EU Region
   2) http://localhost:8090 for US Region
   3) http://localhost:8070 for APAC Region
3) visit the HAProxy UI 
   1) http://localhost:8081 for EU Region
   2) http://localhost:8082 for US Region
   3) http://localhost:8083 for APAC Region
4) have fun!

## Helpful Commands

### Execute SQL
Use the following to execute arbitrary SQL on the CockroachDB cluster.  The following creates a database called `test`.
```bash
docker-compose exec crdb-0 /cockroach/cockroach sql --insecure --execute="CREATE DATABASE test;"
```

### Open Interactive Shells
```bash
docker exec -ti crdb-0 /bin/bash
docker exec -ti crdb-0a /bin/bash
docker exec -ti crdb-0b /bin/bash
docker exec -ti crdb-1 /bin/bash
docker exec -ti crdb-1a /bin/bash
docker exec -ti crdb-1b /bin/bash
docker exec -ti crdb-2 /bin/bash
docker exec -ti crdb-2a /bin/bash
docker exec -ti crdb-2b /bin/bash
docker exec -ti lb_eu /bin/sh
docker exec -ti lb_us /bin/sh
docker exec -ti lb_apac /bin/sh
```

### Stop Individual nodes
```bash
docker-compose stop crdb-0
docker-compose stop crdb-0a
docker-compose stop crdb-0b
docker-compose stop crdb-1
docker-compose stop crdb-1a
docker-compose stop crdb-1b
docker-compose stop crdb-2
docker-compose stop crdb-2a
docker-compose stop crdb-2b
```