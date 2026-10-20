SETUP (required before the first 'docker compose up')
-----------------------------------------------------
    cp .env.example .env

Edit .env and set DORIS_ROOT_PASSWORD. The compose file reads it with
${DORIS_ROOT_PASSWORD:?...} and refuses to start without it. .env is git-ignored.

FE ports 8030 and 9030 are bound to 127.0.0.1 only.
The BE port 8040 is bound to 127.0.0.1 only. Containers reach it over doris_net by service name.

ROOT PASSWORD
-------------
The apache/doris image does NOT apply DORIS_ROOT_PASSWORD on its own. After the
FE is up, set it once from a MySQL client on the host (verify the syntax for your
Doris version):

    mysql -h 127.0.0.1 -P 9030 -uroot
    SET PASSWORD FOR 'root' = PASSWORD('<the value of DORIS_ROOT_PASSWORD>');

MISSING FILES
-------------
The compose file mounts config directories that are NOT in this repository.
Nothing below has been generated.

1. ./data/doris/fe/conf must contain the FE config files (fe.conf and the rest of
   the FE conf directory) for apache/doris:1.2.2-fe-x86_64. Without them the FE
   container has an empty conf directory and will not start correctly. Copy them
   out of the image, for example:
       docker create --name doris-fe-tmp apache/doris:1.2.2-fe-x86_64
       docker cp doris-fe-tmp:/opt/apache-doris/fe/conf ./data/doris/fe/
       docker rm doris-fe-tmp

2. ./data/doris/be/conf must contain the BE config files (be.conf and the rest of
   the BE conf directory) for apache/doris:1.2.2-be-x86_64. Same method, using the
   BE image and /opt/apache-doris/be/conf.

3. ./data/doris/be/script (optional, mounted as docker-entrypoint-initdb.d) is not
   in this repository. Create it only if you need init scripts.

The ./data tree is created by Docker on first run, so these directories exist but
are empty until populated.

Apache Doris is a high-performance, real-time analytical database based on MPP architecture, known for its extreme speed and ease of use. 
It is a next-generation real-time data warehouse that can handle large amounts of data quickly and easily.

Here are some of the key features of Apache Doris:
- It is a columnar storage engine, which encodes, compresses, and reads data by column. This makes it very efficient for analytical queries.
- It uses a cost-based optimizer (CBO) to figure out the most efficient execution plan for complicated big queries.
- It has a fully vectorized execution engine so it can reduce virtual function calls and cache misses.
- It is MPP-based (Massively Parallel Processing) so it can give full play to the user's machines and cores.
- In Doris, query execution is data-driven, which means whether a query gets executed is determined by whether its relevant data is ready, and this enables more efficient use of CPUs.
- Doris also provides BitmapIndex to speed up data queries.
- Doris currently adopts a complete column storage structure and provides rich indexes to deal with different query scenarios, laying a solid foundation for Doris's efficient writing and query performance.
