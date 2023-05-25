Setting up a local kubernetes cluster on macos,
and executing distributed sql queries over iceberg s3.

Dependencies
- socket_vmnet
- hyperkit
- kubectl
- minikube
- trino cli

^ Above socket_vmnet and hyperkit are used on macos to make the linux vm experience as expected (container on macos are vm within vm, need more management/forwarding..etc)

Summary:
There are four main "deployables" organized into four namespaces.
You basically just need to "kubectl create -f" them all in order. No need to add any namespace arguments, as the yamls create namespaces themselves.
Minio should be deployed before everything.
Postgres should be deployed before Hive. Hive deployment runs an init job to initialize the postgres db schema.
Hive should be deployed before Trino. Trino+Iceberg requires some metadata to be stored on a hive metastore.

Bootstrap
- `minikube start --network socket_vmnet --cpus='4' --memory='24GB' --disk-size='250GB'`
- `minikube tunnel` (leave this running to access services via clusterip via loadbalancer)
- `kubectl create -f minio.yaml`
- `kubectl get services -n minio`
- Navigate to the external ip of the minio service, on port 9000  (http://10.100.138.239:9090/buckets)
- Login with the credentials (admin/adminpassword123)
- Create a bucket named "test", no need to modify any other options. Note, this is the only "manual" step. We could create a job to do this using aws cli, but this serves as a good checkpoint to verify configuration.
- `kubectl create -f postgres.yaml`
- `kubectl get pods -n postgres`
- Wait for postgres to be up
- `kubectl create -f hive.yaml`
- `kubectl create -f trino.yaml`
- Trino is started with 1 coordinator and 2 workers

Usage
- `kubectl get services -n trino`
- Note the external ip for trino from above
- `trino --server http://<external ip of trino>:8080 --catalog iceberg --debug`
- This is a sql repl

Some sql commands to try:
- show schemas from iceberg;
- `create schema iceberg.demo with (location = 's3a://test/demo');`
- `create table iceberg.demo.one(country varchar, state varchar, population bigint) with (format = 'PARQUET');`
- `insert into demo.one values('usa','delaware',2000);`
- `insert into demo.one values('usa','new york',1000);`
- `select * from demo.one;`
- `delete from demo.one where state='new york';`

You are also able to access the "hive" catalog that is used by the standalone hive metastore. Trino is configured to access this same catalog.

You can scale the trino worker pool via:
- `kubectl scale statefulset trino-worker -n trino --replicas=4`
- `kubectl scale statefulset trino-worker -n trino --replicas=2`

Useful kubectl:
- `watch -n 1 kubectl get services --all-namespaces`
- `watch -n 1 kubectl get pods -n trino`

Links
- https://trino.io/docs/current/connector/iceberg.html#metadata-tables