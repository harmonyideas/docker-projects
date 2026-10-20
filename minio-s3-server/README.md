SETUP (required before the first 'docker compose up')
-----------------------------------------------------
    cp .env.example .env

Edit .env and set MINIO_ROOT_USER, MINIO_ROOT_PASSWORD and GRAFANA_ADMIN_PASSWORD.
The compose file reads them with ${VAR:?...} and refuses to start without them.
.env is git-ignored; never commit it.

Published ports are bound to 127.0.0.1 only (MinIO API 9000, console 9001,
Grafana 3000, Prometheus 9090). Put a reverse proxy in front of them for remote access.

MISSING FILES
-------------
The compose file mounts two files that are NOT in this repository. The stack
will not start correctly until they exist:

1. ./nginx.conf (mounted into the nginx load balancer)
   Must be an nginx config with an upstream of minio1..minio4 on port 9000
   for the S3 API and minio1..minio4 on port 9001 for the console, listening on
   9000 and 9001 inside the container.

2. ./prometheus.yml (mounted into the prometheus container)
   Must scrape MinIO at minio1:9000 (or the nginx service) on /minio/v2/metrics/cluster.
   MINIO_PROMETHEUS_AUTH_TYPE is no longer set to 'public', so MinIO's default (jwt)
   applies and Prometheus must send a bearer token. Generate one with
   'mc admin prometheus generate <alias>' and add it as bearer_token in the scrape job.

Grafana is started with the admin password from .env. Grafana provisioning and
dashboards are not included.

MinIO is a High Performance Object Storage released under GNU Affero General Public License v3.0. It is API compatible with Amazon S3 cloud storage service. Use MinIO to build high performance infrastructure for machine learning, analytics and application data workloads.

This README [MiniIO QuickStart Guide](https://github.com/minio/minio/blob/master/README.md) provides quickstart instructions on running MinIO on bare metal hardware, including container-based installations. For Kubernetes environments, use the [MinIO Kubernetes Operator](https://github.com/minio/operator/blob/master/README.md).


This docker-compose.yml file includes prometheus and grafana for reporting and analytical data.

*TODO*
- Integrage encryption with KMS and Vault (HashiCorp)
