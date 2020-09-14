## Steps to demonstrate data pinning in Cockroach DB 

More Info: https://www.cockroachlabs.com/docs/stable/topology-geo-partitioned-replicas.html



1. Apply an enterprise license

   `SET CLUSTER SETTING cluster.organization = 'Pinning Demo';`
   `SET CLUSTER SETTING enterprise.license = 'crl-0-actualvaluehere';`




2. Look at locality flags for each node in the cluster

   ```
   --locality=region=eu,datacenter=eu-west-2,az=eu-west-2a
   --locality=region=eu,datacenter=eu-west-2,az=eu-west-2b
   --locality=region=eu,datacenter=eu-west-2,az=eu-west-2c
   
   --locality=region=us,datacenter=us-east-1,az=us-east-1c
   --locality=region=us,datacenter=us-east-1,az=us-east-1b
   --locality=region=us,datacenter=us-east-1,az=us-east-1a
   
   --locality=region=apac,datacenter=ap-southeast-1,az=ap-southeast-1a
   --locality=region=apac,datacenter=ap-southeast-1,az=ap-southeast-1c
   --locality=region=apac,datacenter=ap-southeast-1,az=ap-southeast-1b
   ```

3. Initialize the movr workload

   ```
   cockroach workload init movr
   ```

   Results:

   ```
   I200914 17:54:08.338877 1 workload/workloadsql/dataload.go:140  imported users (0s, 50 rows)
   I200914 17:54:08.372447 1 workload/workloadsql/dataload.go:140  imported vehicles (0s, 15 rows)
   I200914 17:54:08.481126 1 workload/workloadsql/dataload.go:140  imported rides (0s, 500 rows)
   I200914 17:54:08.556945 1 workload/workloadsql/dataload.go:140  imported vehicle_location_histories (0s, 1000 rows)
   I200914 17:54:08.695815 1 workload/workloadsql/dataload.go:140  imported promo_codes (0s, 1000 rows)
   I200914 17:54:08.704895 1 workload/workloadsql/workloadsql.go:113  starting 8 splits
   I200914 17:54:10.339302 1 workload/workloadsql/workloadsql.go:113  starting 8 splits
   I200914 17:54:12.178740 1 workload/workloadsql/workloadsql.go:113  starting 8 splits
   ```

4. Inspect zone configs

   ```
   root@:26257/defaultdb> show zone configurations;
   ```

   Verify that no constraints or lease_preferences exist on any of the entities. Verify that the replication factor is 3 by default.

   Results:

   ```
                          target                      |                               raw_config_sql
   ---------------------------------------------------+------------------------------------------------------------------------------
     RANGE default                                    | ALTER RANGE default CONFIGURE ZONE USING
                                                      |     range_min_bytes = 134217728,
                                                      |     range_max_bytes = 536870912,
                                                      |     gc.ttlseconds = 90000,
                                                      |     num_replicas = 3,
                                                      |     constraints = '[]',
                                                      |     lease_preferences = '[]'
     DATABASE system                                  | ALTER DATABASE system CONFIGURE ZONE USING
                                                      |     range_min_bytes = 134217728,
                                                      |     range_max_bytes = 536870912,
                                                      |     gc.ttlseconds = 90000,
                                                      |     num_replicas = 5,
                                                      |     constraints = '[]',
                                                      |     lease_preferences = '[]'
     RANGE meta                                       | ALTER RANGE meta CONFIGURE ZONE USING
                                                      |     range_min_bytes = 134217728,
                                                      |     range_max_bytes = 536870912,
                                                      |     gc.ttlseconds = 3600,
                                                      |     num_replicas = 5,
                                                      |     constraints = '[]',
                                                      |     lease_preferences = '[]'
     RANGE system                                     | ALTER RANGE system CONFIGURE ZONE USING
                                                      |     range_min_bytes = 134217728,
                                                      |     range_max_bytes = 536870912,
                                                      |     gc.ttlseconds = 90000,
                                                      |     num_replicas = 5,
                                                      |     constraints = '[]',
                                                      |     lease_preferences = '[]'
     RANGE liveness                                   | ALTER RANGE liveness CONFIGURE ZONE USING
                                                      |     range_min_bytes = 134217728,
                                                      |     range_max_bytes = 536870912,
                                                      |     gc.ttlseconds = 600,
                                                      |     num_replicas = 5,
                                                      |     constraints = '[]',
                                                      |     lease_preferences = '[]'
     TABLE system.public.replication_constraint_stats | ALTER TABLE system.public.replication_constraint_stats CONFIGURE ZONE USING
                                                      |     gc.ttlseconds = 600,
                                                      |     constraints = '[]',
                                                      |     lease_preferences = '[]'
     TABLE system.public.replication_stats            | ALTER TABLE system.public.replication_stats CONFIGURE ZONE USING
                                                      |     gc.ttlseconds = 600,
                                                      |     constraints = '[]',
                                                      |     lease_preferences = '[]'
   (7 rows)
   ```

5. Inspect movr.vehicles table

   ```
   root@:26257/defaultdb> use movr;
   SET
   
   Time: 13.1124ms
   
   root@:26257/movr> show tables;
   
             table_name
   ------------------------------
     promo_codes
     rides
     user_promo_codes
     users
     vehicle_location_histories
     vehicles
   (6 rows)
   
   Time: 12.513ms
   
   root@:26257/movr> select * from vehicles;
                      id                  |     city      |    type    |               owner_id               |       creation_time       |  status   |        current_location        |                   ext
   ---------------------------------------+---------------+------------+--------------------------------------+---------------------------+-----------+--------------------------------+------------------------------------------
     88888888-8888-4800-8000-000000000008 | san francisco | skateboard | 80000000-0000-4000-8000-000000000019 | 2019-01-02 03:04:05+00:00 | in_use    | 69721 Noah River               | {"color": "blue"}
     11111111-1111-4100-8000-000000000001 | new york      | scooter    | 147ae147-ae14-4b00-8000-000000000004 | 2019-01-02 03:04:05+00:00 | in_use    | 86667 Edwards Valley           | {"color": "black"}
     cccccccc-cccc-4000-8000-00000000000c | paris         | skateboard | c7ae147a-e147-4000-8000-000000000027 | 2019-01-02 03:04:05+00:00 | in_use    | 19202 Edward Pass              | {"color": "black"}
     dddddddd-dddd-4000-8000-00000000000d | paris         | skateboard | cccccccc-cccc-4000-8000-000000000028 | 2019-01-02 03:04:05+00:00 | available | 2505 Harrison Parkway Apt. 89  | {"color": "red"}
     eeeeeeee-eeee-4000-8000-00000000000e | rome          | bike       | fae147ae-147a-4000-8000-000000000031 | 2019-01-02 03:04:05+00:00 | in_use    | 64935 Matthew Flats Suite 55   | {"brand": "Pinarello", "color": "blue"}
     55555555-5555-4400-8000-000000000005 | seattle       | scooter    | 6b851eb8-51eb-4400-8000-000000000015 | 2019-01-02 03:04:05+00:00 | available | 91427 Steven Spurs Apt. 49     | {"color": "blue"}
     22222222-2222-4200-8000-000000000002 | boston        | scooter    | 2e147ae1-47ae-4400-8000-000000000009 | 2019-01-02 03:04:05+00:00 | in_use    | 19659 Christina Ville          | {"color": "blue"}
     33333333-3333-4400-8000-000000000003 | boston        | scooter    | 33333333-3333-4400-8000-00000000000a | 2019-01-02 03:04:05+00:00 | in_use    | 47259 Natasha Cliffs           | {"color": "green"}
     99999999-9999-4800-8000-000000000009 | los angeles   | scooter    | 9eb851eb-851e-4800-8000-00000000001f | 2019-01-02 03:04:05+00:00 | in_use    | 43051 Jonathan Fords Suite 36  | {"color": "red"}
     00000000-0000-4000-8000-000000000000 | new york      | skateboard | 051eb851-eb85-4ec0-8000-000000000001 | 2019-01-02 03:04:05+00:00 | in_use    | 64110 Richard Crescent         | {"color": "black"}
     aaaaaaaa-aaaa-4800-8000-00000000000a | amsterdam     | scooter    | c28f5c28-f5c2-4000-8000-000000000026 | 2019-01-02 03:04:05+00:00 | in_use    | 62609 Stephanie Route          | {"color": "red"}
     bbbbbbbb-bbbb-4800-8000-00000000000b | amsterdam     | scooter    | bd70a3d7-0a3d-4000-8000-000000000025 | 2019-01-02 03:04:05+00:00 | available | 57637 Mitchell Shoals Suite 59 | {"color": "blue"}
     77777777-7777-4800-8000-000000000007 | san francisco | skateboard | 75c28f5c-28f5-4400-8000-000000000017 | 2019-01-02 03:04:05+00:00 | in_use    | 49164 Anna Mission Apt. 38     | {"color": "black"}
     66666666-6666-4800-8000-000000000006 | seattle       | skateboard | 570a3d70-a3d7-4c00-8000-000000000011 | 2019-01-02 03:04:05+00:00 | lost      | 81472 Morris Run               | {"color": "green"}
     44444444-4444-4400-8000-000000000004 | washington dc | bike       | 4ccccccc-cccc-4c00-8000-00000000000f | 2019-01-02 03:04:05+00:00 | available | 37754 Farmer Extension         | {"brand": "Merida", "color": "yellow"}
   (15 rows)
   
   Time: 20.1204ms
   
   root@:26257/movr> select city, COUNT(*) AS vehicle_count from vehicles group by city order by COUNT(*) DESC;
         city      | vehicle_count
   ----------------+----------------
     amsterdam     |             2
     boston        |             2
     new york      |             2
     paris         |             2
     san francisco |             2
     seattle       |             2
     los angeles   |             1
     rome          |             1
     washington dc |             1
   (9 rows)
   
   
   root@:26257/defaultdb> SHOW CREATE TABLE movr.vehicles;
          table_name      |                                               create_statement
   -----------------------+----------------------------------------------------------------------------------------------------------------
     movr.public.vehicles | CREATE TABLE vehicles (
                          |     id UUID NOT NULL,
                          |     city VARCHAR NOT NULL,
                          |     type VARCHAR NULL,
                          |     owner_id UUID NULL,
                          |     creation_time TIMESTAMP NULL,
                          |     status VARCHAR NULL,
                          |     current_location VARCHAR NULL,
                          |     ext JSONB NULL,
                          |     CONSTRAINT "primary" PRIMARY KEY (city ASC, id ASC),
                          |     CONSTRAINT fk_city_ref_users FOREIGN KEY (city, owner_id) REFERENCES users(city, id),
                          |     INDEX vehicles_auto_index_fk_city_ref_users (city ASC, owner_id ASC),
                          |     FAMILY "primary" (id, city, type, owner_id, creation_time, status, current_location, ext)
                          | );
   (1 row)
   
   Time: 334.0393ms
   ```

   Notice that there are 9 distinct cities.  (You may get different cities based on the data inserted during the movr initialization process).
   
   Also notice that there is no partitioning on the table or its indexes.



6. Pin vehicles data to regions by city

   ```
   ALTER TABLE movr.vehicles
   PARTITION BY LIST (city) ( --note that city is already part of the primary key
       PARTITION us VALUES IN ('boston', 'new york', 'washington dc'),
       PARTITION eu VALUES IN ('amsterdam', 'paris', 'rome'),
       PARTITION apac VALUES IN ('san francisco', 'seattle', 'los angeles')
   );
   
   ALTER PARTITION us OF TABLE movr.vehicles CONFIGURE ZONE USING
     constraints = '{"+region=us": 3}',
     lease_preferences = '[[+region=us]]',
     num_replicas = 3;
   
   ALTER PARTITION eu OF TABLE movr.vehicles CONFIGURE ZONE USING
     constraints = '{"+region=eu": 3}',
     lease_preferences = '[[+region=eu]]',
     num_replicas = 3;
   
   ALTER PARTITION apac OF TABLE movr.vehicles CONFIGURE ZONE USING
     constraints = '{"+region=apac": 3}',
     lease_preferences = '[[+region=apac]]',
     num_replicas = 3;
   
   ALTER INDEX movr.vehicles_auto_index_fk_city_ref_users
   PARTITION BY LIST (city) (
       PARTITION us VALUES IN ('boston', 'new york', 'washington dc'),
       PARTITION eu VALUES IN ('amsterdam', 'paris', 'rome'),
       PARTITION apac VALUES IN ('san francisco', 'seattle', 'los angeles')
   );
   
   ALTER PARTITION us OF INDEX movr.vehicles_auto_index_fk_city_ref_users CONFIGURE ZONE USING
     constraints = '{"+region=us": 3}',
     lease_preferences = '[[+region=us]]',
     num_replicas = 3;
   
   ALTER PARTITION eu OF INDEX movr.vehicles_auto_index_fk_city_ref_users CONFIGURE ZONE USING
     constraints = '{"+region=eu": 3}',
     lease_preferences = '[[+region=eu]]',
     num_replicas = 3;
   
   ALTER PARTITION apac OF INDEX movr.vehicles_auto_index_fk_city_ref_users CONFIGURE ZONE USING
     constraints = '{"+region=apac": 3}',
     lease_preferences = '[[+region=apac]]',
     num_replicas = 3;
   ```

   

7. Validate settings

   ```
   root@:26257/defaultdb> SHOW CREATE TABLE movr.vehicles;
          table_name      |                                               create_statement
   -----------------------+----------------------------------------------------------------------------------------------------------------
     movr.public.vehicles | CREATE TABLE vehicles (
                          |     id UUID NOT NULL,
                          |     city VARCHAR NOT NULL,
                          |     type VARCHAR NULL,
                          |     owner_id UUID NULL,
                          |     creation_time TIMESTAMP NULL,
                          |     status VARCHAR NULL,
                          |     current_location VARCHAR NULL,
                          |     ext JSONB NULL,
                          |     CONSTRAINT "primary" PRIMARY KEY (city ASC, id ASC),
                          |     CONSTRAINT fk_city_ref_users FOREIGN KEY (city, owner_id) REFERENCES users(city, id),
                          |     INDEX vehicles_auto_index_fk_city_ref_users (city ASC, owner_id ASC) PARTITION BY LIST (city) (
                          |         PARTITION us VALUES IN (('boston'), ('new york'), ('washington dc')),
                          |         PARTITION eu VALUES IN (('amsterdam'), ('paris'), ('rome')),
                          |         PARTITION apac VALUES IN (('san francisco'), ('seattle'), ('los angeles'))
                          |     ),
                          |     FAMILY "primary" (id, city, type, owner_id, creation_time, status, current_location, ext)
                          | ) PARTITION BY LIST (city) (
                          |     PARTITION us VALUES IN (('boston'), ('new york'), ('washington dc')),
                          |     PARTITION eu VALUES IN (('amsterdam'), ('paris'), ('rome')),
                          |     PARTITION apac VALUES IN (('san francisco'), ('seattle'), ('los angeles'))
                          | );
                          | ALTER PARTITION apac OF INDEX movr.public.vehicles@primary CONFIGURE ZONE USING
                          |     num_replicas = 3,
                          |     constraints = '{+region=apac: 3}',
                          |     lease_preferences = '[[+region=apac]]';
                          | ALTER PARTITION eu OF INDEX movr.public.vehicles@primary CONFIGURE ZONE USING
                          |     num_replicas = 3,
                          |     constraints = '{+region=eu: 3}',
                          |     lease_preferences = '[[+region=eu]]';
                          | ALTER PARTITION us OF INDEX movr.public.vehicles@primary CONFIGURE ZONE USING
                          |     num_replicas = 3,
                          |     constraints = '{+region=us: 3}',
                          |     lease_preferences = '[[+region=us]]';
                          | ALTER PARTITION apac OF INDEX movr.public.vehicles@vehicles_auto_index_fk_city_ref_users CONFIGURE ZONE USING
                          |     num_replicas = 3,
                          |     constraints = '{+region=apac: 3}',
                          |     lease_preferences = '[[+region=apac]]';
                          | ALTER PARTITION eu OF INDEX movr.public.vehicles@vehicles_auto_index_fk_city_ref_users CONFIGURE ZONE USING
                          |     num_replicas = 3,
                          |     constraints = '{+region=eu: 3}',
                          |     lease_preferences = '[[+region=eu]]';
                          | ALTER PARTITION us OF INDEX movr.public.vehicles@vehicles_auto_index_fk_city_ref_users CONFIGURE ZONE USING
                          |     num_replicas = 3,
                          |     constraints = '{+region=us: 3}',
                          |     lease_preferences = '[[+region=us]]'
   (1 row)
   
   root@:26257/defaultdb> SHOW PARTITIONS FROM TABLE movr.vehicles;
     database_name | table_name | partition_name | parent_partition | column_names |                   index_name                   |                 partition_value                 |              zone_config               |            full_zone_config
   ----------------+------------+----------------+------------------+--------------+------------------------------------------------+-------------------------------------------------+----------------------------------------+-----------------------------------------
     movr          | vehicles   | us             | NULL             | city         | vehicles@primary                               | ('boston'), ('new york'), ('washington dc')     | num_replicas = 3,                      | range_min_bytes = 134217728,
                   |            |                |                  |              |                                                |                                                 | constraints = '{+region=us: 3}',       | range_max_bytes = 536870912,
                   |            |                |                  |              |                                                |                                                 | lease_preferences = '[[+region=us]]'   | gc.ttlseconds = 90000,
                   |            |                |                  |              |                                                |                                                 |                                        | num_replicas = 3,
                   |            |                |                  |              |                                                |                                                 |                                        | constraints = '{+region=us: 3}',
                   |            |                |                  |              |                                                |                                                 |                                        | lease_preferences = '[[+region=us]]'
     movr          | vehicles   | eu             | NULL             | city         | vehicles@primary                               | ('amsterdam'), ('paris'), ('rome')              | num_replicas = 3,                      | range_min_bytes = 134217728,
                   |            |                |                  |              |                                                |                                                 | constraints = '{+region=eu: 3}',       | range_max_bytes = 536870912,
                   |            |                |                  |              |                                                |                                                 | lease_preferences = '[[+region=eu]]'   | gc.ttlseconds = 90000,
                   |            |                |                  |              |                                                |                                                 |                                        | num_replicas = 3,
                   |            |                |                  |              |                                                |                                                 |                                        | constraints = '{+region=eu: 3}',
                   |            |                |                  |              |                                                |                                                 |                                        | lease_preferences = '[[+region=eu]]'
     movr          | vehicles   | apac           | NULL             | city         | vehicles@primary                               | ('san francisco'), ('seattle'), ('los angeles') | num_replicas = 3,                      | range_min_bytes = 134217728,
                   |            |                |                  |              |                                                |                                                 | constraints = '{+region=apac: 3}',     | range_max_bytes = 536870912,
                   |            |                |                  |              |                                                |                                                 | lease_preferences = '[[+region=apac]]' | gc.ttlseconds = 90000,
                   |            |                |                  |              |                                                |                                                 |                                        | num_replicas = 3,
                   |            |                |                  |              |                                                |                                                 |                                        | constraints = '{+region=apac: 3}',
                   |            |                |                  |              |                                                |                                                 |                                        | lease_preferences = '[[+region=apac]]'
     movr          | vehicles   | us             | NULL             | city         | vehicles@vehicles_auto_index_fk_city_ref_users | ('boston'), ('new york'), ('washington dc')     | num_replicas = 3,                      | range_min_bytes = 134217728,
                   |            |                |                  |              |                                                |                                                 | constraints = '{+region=us: 3}',       | range_max_bytes = 536870912,
                   |            |                |                  |              |                                                |                                                 | lease_preferences = '[[+region=us]]'   | gc.ttlseconds = 90000,
                   |            |                |                  |              |                                                |                                                 |                                        | num_replicas = 3,
                   |            |                |                  |              |                                                |                                                 |                                        | constraints = '{+region=us: 3}',
                   |            |                |                  |              |                                                |                                                 |                                        | lease_preferences = '[[+region=us]]'
     movr          | vehicles   | eu             | NULL             | city         | vehicles@vehicles_auto_index_fk_city_ref_users | ('amsterdam'), ('paris'), ('rome')              | num_replicas = 3,                      | range_min_bytes = 134217728,
                   |            |                |                  |              |                                                |                                                 | constraints = '{+region=eu: 3}',       | range_max_bytes = 536870912,
                   |            |                |                  |              |                                                |                                                 | lease_preferences = '[[+region=eu]]'   | gc.ttlseconds = 90000,
                   |            |                |                  |              |                                                |                                                 |                                        | num_replicas = 3,
                   |            |                |                  |              |                                                |                                                 |                                        | constraints = '{+region=eu: 3}',
                   |            |                |                  |              |                                                |                                                 |                                        | lease_preferences = '[[+region=eu]]'
     movr          | vehicles   | apac           | NULL             | city         | vehicles@vehicles_auto_index_fk_city_ref_users | ('san francisco'), ('seattle'), ('los angeles') | num_replicas = 3,                      | range_min_bytes = 134217728,
                   |            |                |                  |              |                                                |                                                 | constraints = '{+region=apac: 3}',     | range_max_bytes = 536870912,
                   |            |                |                  |              |                                                |                                                 | lease_preferences = '[[+region=apac]]' | gc.ttlseconds = 90000,
                   |            |                |                  |              |                                                |                                                 |                                        | num_replicas = 3,
                   |            |                |                  |              |                                                |                                                 |                                        | constraints = '{+region=apac: 3}',
                   |            |                |                  |              |                                                |                                                 |                                        | lease_preferences = '[[+region=apac]]'
   (6 rows)
   ```

8. View vehicle data by various city combinations

   ```
   root@:26257/system> select * from movr.vehicles WHERE city = 'san francisco'; -- APAC data
                      id                  |     city      |    type    |               owner_id               |       creation_time       | status |      current_location      |        ext
   ---------------------------------------+---------------+------------+--------------------------------------+---------------------------+--------+----------------------------+---------------------
     77777777-7777-4800-8000-000000000007 | san francisco | skateboard | 75c28f5c-28f5-4400-8000-000000000017 | 2019-01-02 03:04:05+00:00 | in_use | 49164 Anna Mission Apt. 38 | {"color": "black"}
     88888888-8888-4800-8000-000000000008 | san francisco | skateboard | 80000000-0000-4000-8000-000000000019 | 2019-01-02 03:04:05+00:00 | in_use | 69721 Noah River           | {"color": "blue"}
   (2 rows)
   
   Time: 3.5489ms
   
   root@:26257/system> select * from movr.vehicles WHERE city = 'boston'; -- US data
                      id                  |  city  |  type   |               owner_id               |       creation_time       | status |   current_location    |        ext
   ---------------------------------------+--------+---------+--------------------------------------+---------------------------+--------+-----------------------+---------------------
     22222222-2222-4200-8000-000000000002 | boston | scooter | 2e147ae1-47ae-4400-8000-000000000009 | 2019-01-02 03:04:05+00:00 | in_use | 19659 Christina Ville | {"color": "blue"}
     33333333-3333-4400-8000-000000000003 | boston | scooter | 33333333-3333-4400-8000-00000000000a | 2019-01-02 03:04:05+00:00 | in_use | 47259 Natasha Cliffs  | {"color": "green"}
   (2 rows)
   
   Time: 3.3878ms
   
   root@:26257/system> select * from movr.vehicles WHERE city = 'rome'; -- EU data
                      id                  | city | type |               owner_id               |       creation_time       | status |       current_location       |                   ext
   ---------------------------------------+------+------+--------------------------------------+---------------------------+--------+------------------------------+------------------------------------------
     eeeeeeee-eeee-4000-8000-00000000000e | rome | bike | fae147ae-147a-4000-8000-000000000031 | 2019-01-02 03:04:05+00:00 | in_use | 64935 Matthew Flats Suite 55 | {"brand": "Pinarello", "color": "blue"}
   (1 row)
   
   Time: 1.9016ms
   
   root@:26257/system> select city, COUNT(*) AS vehicle_count from movr.vehicles group by city order by COUNT(*) DESC; -- all data
         city      | vehicle_count
   ----------------+----------------
     new york      |             2
     boston        |             2
     san francisco |             2
     paris         |             2
     amsterdam     |             2
     seattle       |             2
     rome          |             1
     los angeles   |             1
     washington dc |             1
   (9 rows)
   
   Time: 11.9099ms
   ```

   All the data is served up correctly without errors.

9. Shutdown "eu" nodes

   ```
   $ docker-compose stop crdb-0
   Stopping crdb-0 ... done
   $ docker-compose stop crdb-0a
   Stopping crdb-0a ... done
   $ docker-compose stop crdb-0b
   Stopping crdb-0b ... done
   ```

10. Verify nodes are down

    ```
    docker exec -ti crdb-1 /bin/bash
    
    root@crdb-1:/cockroach# cockroach node status --insecure
      id |    address    |  sql_address  |  build  |            started_at            |            updated_at            |                         locality                         | is_available | is_live
    -----+---------------+---------------+---------+----------------------------------+----------------------------------+----------------------------------------------------------+--------------+----------
       1 | crdb-0:26257  | crdb-0:26257  | v20.1.5 | 2020-09-14 17:24:39.879427+00:00 | 2020-09-14 20:03:19.556761+00:00 | region=eu,datacenter=eu-west-2,az=eu-west-2a             | false        | false
       2 | crdb-1b:26257 | crdb-1b:26257 | v20.1.5 | 2020-09-14 17:24:40.664089+00:00 | 2020-09-14 20:04:14.76274+00:00  | region=us,datacenter=us-east-1,az=us-east-1c             | true         | true
       3 | crdb-2:26257  | crdb-2:26257  | v20.1.5 | 2020-09-14 17:24:40.750817+00:00 | 2020-09-14 20:04:14.824629+00:00 | region=apac,datacenter=ap-southeast-1,az=ap-southeast-1a | true         | true
       4 | crdb-0a:26257 | crdb-0a:26257 | v20.1.5 | 2020-09-14 17:24:40.764554+00:00 | 2020-09-14 20:03:20.888378+00:00 | region=eu,datacenter=eu-west-2,az=eu-west-2b             | false        | false
       5 | crdb-1a:26257 | crdb-1a:26257 | v20.1.5 | 2020-09-14 17:24:41.208715+00:00 | 2020-09-14 20:04:15.298313+00:00 | region=us,datacenter=us-east-1,az=us-east-1b             | true         | true
       6 | crdb-1:26257  | crdb-1:26257  | v20.1.5 | 2020-09-14 17:24:41.216681+00:00 | 2020-09-14 20:04:15.301226+00:00 | region=us,datacenter=us-east-1,az=us-east-1a             | true         | true
       7 | crdb-2b:26257 | crdb-2b:26257 | v20.1.5 | 2020-09-14 17:24:41.268288+00:00 | 2020-09-14 20:04:15.714252+00:00 | region=apac,datacenter=ap-southeast-1,az=ap-southeast-1c | true         | true
       8 | crdb-2a:26257 | crdb-2a:26257 | v20.1.5 | 2020-09-14 17:24:41.424473+00:00 | 2020-09-14 20:04:15.517653+00:00 | region=apac,datacenter=ap-southeast-1,az=ap-southeast-1b | true         | true
       9 | crdb-0b:26257 | crdb-0b:26257 | v20.1.5 | 2020-09-14 17:24:41.47008+00:00  | 2020-09-14 20:03:30.807541+00:00 | region=eu,datacenter=eu-west-2,az=eu-west-2c             | false        | false
    (9 rows)
    ```

    Notice the "is_live" column showing the EU nodes as down.

11. Try to view all vehicle data again

    ```
    root@:26257/defaultdb> select * from movr.vehicles WHERE city = 'san francisco';
                       id                  |     city      |    type    |               owner_id               |       creation_time       | status |      current_location      |        ext
    ---------------------------------------+---------------+------------+--------------------------------------+---------------------------+--------+----------------------------+---------------------
      77777777-7777-4800-8000-000000000007 | san francisco | skateboard | 75c28f5c-28f5-4400-8000-000000000017 | 2019-01-02 03:04:05+00:00 | in_use | 49164 Anna Mission Apt. 38 | {"color": "black"}
      88888888-8888-4800-8000-000000000008 | san francisco | skateboard | 80000000-0000-4000-8000-000000000019 | 2019-01-02 03:04:05+00:00 | in_use | 69721 Noah River           | {"color": "blue"}
    (2 rows)
    
    Time: 6.21ms
    
    root@:26257/defaultdb> select * from movr.vehicles WHERE city = 'boston';
                       id                  |  city  |  type   |               owner_id               |       creation_time       | status |   current_location    |        ext
    ---------------------------------------+--------+---------+--------------------------------------+---------------------------+--------+-----------------------+---------------------
      22222222-2222-4200-8000-000000000002 | boston | scooter | 2e147ae1-47ae-4400-8000-000000000009 | 2019-01-02 03:04:05+00:00 | in_use | 19659 Christina Ville | {"color": "blue"}
      33333333-3333-4400-8000-000000000003 | boston | scooter | 33333333-3333-4400-8000-00000000000a | 2019-01-02 03:04:05+00:00 | in_use | 47259 Natasha Cliffs  | {"color": "green"}
    (2 rows)
    
    Time: 3.944ms
    
    root@:26257/defaultdb> select city, COUNT(*) AS vehicle_count from movr.vehicles WHERE city IN ( 'los angeles', 'san francisco', 'seattle', 'boston', 'washington dc', 'new york' ) group by city order by COUNT(*) D
    ESC;
          city      | vehicle_count
    ----------------+----------------
      boston        |             2
      seattle       |             2
      new york      |             2
      san francisco |             2
      los angeles   |             1
      washington dc |             1
    (6 rows)
    
    Time: 7.2059ms
    
    root@:26257/defaultdb> select * from movr.vehicles WHERE city = 'rome';
    
    ^^ this fails
    
    root@:26257/defaultdb> select city, COUNT(*) AS vehicle_count from movr.vehicles WHERE city IN ( 'paris', 'amsterdam', 'rome' ) group by city order by COUNT(*) DESC;
    
    ^^ this fails
    
    root@:26257/defaultdb> select city, COUNT(*) AS vehicle_count from movr.vehicles group by city order by COUNT(*) DESC;
    
    ^^ this fails
    ```

    Any queries that involve data pinned to the EU nodes fail.

12. Start "EU" nodes back up

    ```
    $ docker-compose start crdb-0
    Starting crdb-0 ... done
    $ docker-compose start crdb-0a
    Starting crdb-0a ... done
    $ docker-compose start crdb-0b
    Starting crdb-0b ... done
    ```

13. Verify EU data is accessible again

    ```
    root@:26257/defaultdb> select city, COUNT(*) AS vehicle_count from movr.vehicles group by city order by COUNT(*) DESC;
          city      | vehicle_count
    ----------------+----------------
      san francisco |             2
      amsterdam     |             2
      new york      |             2
      boston        |             2
      paris         |             2
      seattle       |             2
      los angeles   |             1
      rome          |             1
      washington dc |             1
    (9 rows)
    
    Time: 13.9764ms
    
    root@:26257/defaultdb> select * from movr.vehicles WHERE city = 'rome';
                       id                  | city | type |               owner_id               |       creation_time       | status |       current_location       |                   ext
    ---------------------------------------+------+------+--------------------------------------+---------------------------+--------+------------------------------+------------------------------------------
      eeeeeeee-eeee-4000-8000-00000000000e | rome | bike | fae147ae-147a-4000-8000-000000000031 | 2019-01-02 03:04:05+00:00 | in_use | 64935 Matthew Flats Suite 55 | {"brand": "Pinarello", "color": "blue"}
    (1 row)
    
    Time: 5.7957ms
    ```

    









