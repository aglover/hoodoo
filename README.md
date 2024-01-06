
# Hoodoo 

<img src="img/logo.png" align="right"
     alt="Hoodoo logo" width="150" height="150">

An example Docker Compose that spins up a Postgres primary database along with a single 
read replica. There are directions below to seed the primary and verify replication. 

## Postgres replication example

There are two files of importance in this repository: `docker-compose.yml` which defines two
Postgres instances and a large text file (`employees_data.sql.bz2`) containing both a schema and associated data. The Docker image used in this example comes from [Bitnami](https://hub.docker.com/r/bitnami/postgresql), which makes setting up 
database replication incredibly easy.  

## Directions to spin-up Hoodoo

1. First, clone this repository and then change directories into `hoodoo`. 

2. Next, run `docker compose up -d`, which will start two containers from the Bitnami Postgres image. One container is named `postgres-primary` and exposes port `5432` and the other `postgres-replica`. It's exposed on port `5434`. The latter container depends on the primary instance starting up successfully. Both database instances have a database named `hoodoo`.

3. You can shell into either database via the `psql` command (you may need to [install this CLI for accessing Postgres](https://www.timescale.com/blog/how-to-install-psql-on-mac-ubuntu-debian-windows/)). To access the primary, type:
```
$ psql postgresql://postgres:hoodoo@localhost:5432/hoodoo
``` 
and to access the read replica, type:
 
```
$ psql postgresql://postgres:hoodoo@localhost:5434/hoodoo
```

Note the only difference is the port. 

4. Decompress the `employees_data.sql.bz2` file by running:
```
$ bunzip2 employees_data.sql.bz2
```
Note, you may need to install [bzip2](https://en.wikipedia.org/wiki/Bzip2). 

5. Seed the primary with the `employees_data.sql` file
```
$ psql postgresql://postgres:hoodoo@localhost:5432/hoodoo < employees_data.sql

```

6. Login to the primary to verify everything worked: 
```
$ psql postgresql://postgres:hoodoo@localhost:5432/hoodoo
```
And in the shell, run the following query: 
```
select count(*) from employees.employee;
```
The result should be `300024`.

7. Now, login to the read replica and issue the same query. You should see the exact same result. 

```
$ psql postgresql://postgres:hoodoo@localhost:5434/hoodoo
```

Obtain the count of rows in the `employee` table: 
```
select count(*) from employees.employee;
```
And the result should be the same as what you previously saw in the primary: 

```
 count
--------
 300024
(1 row)
```

8. In a Postgres read replica setup, SQL writes flow into the primary and that new data is then automatically replicated to replicas. As their name implies, read replicas are designed for read-only traffic. This design is an example of horizontal scaling -- a database with heavy read-write traffic can be made more efficient by offloading some traffic to other instances. In this case, one or more read replicas can offload read traffic from a database thereby giving it some breathing room. You can see this in action by adding a record to the primary and then verifying it shows up in a replica. 

Shell into the primary (i.e. port `5432`) and add a new employee record: 

```
insert into employees.employee values (99999999999, '1957-03-20', 'Robert', 'Smith', 'M', '2024-01-02');
```

Verify the count of rows in the `employee` table has increased by 1 by running the `count` query again.

```
 count
--------
 300025
(1 row)
```

9. Now shell into the read replica instance on port `5434` and verify that there are also 300,025 rows in the `employee` table. 

10. You can shut both Docker containers down via the `docker compose down` command. 

### References

1. [Employees database for Postgres](https://github.com/h8/employees-database)
1. [Bitnami Docker distribution documentation](https://hub.docker.com/r/bitnami/postgresql)