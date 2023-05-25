# Iceberg-trino-k8s-demo
Setting up a local kubernetes cluster on macos,
and executing distributed sql queries over iceberg s3 via trino.

![architecture diagram](./resource/architecture.png)

## Dependencies
- socket_vmnet (`brew install socket_vmnet`)
- hyperkit (`brew install hyperkit`)
- kubectl (`brew install kubernetes-cli`)
- minikube (`brew install minikube`)
- trino cli (`brew install trino`)

^ Above socket_vmnet and hyperkit are used on macos to make the linux vm experience as expected (container on macos are vm within vm, need more management/forwarding..etc)

## Summary

There are four main "deployables" organized into four namespaces.

You basically just need to `kubectl create -f <file>` them all in order. No need to add any namespace arguments, as the yamls create namespaces themselves.

Minio should be deployed before everything.

Postgres should be deployed before Hive. Hive deployment runs an init job to initialize the postgres db schema.

Hive should be deployed before Trino. Trino+Iceberg requires some metadata to be stored on a hive metastore.

As stated on trino:

```
The Iceberg table format manages most metadata in metadata files in the object storage itself.
A small amount of metadata, however, still requires the use of a metastore.
```

## Bootstrap
- `minikube start --network socket_vmnet --cpus='4' --memory='24GB' --disk-size='250GB'`
- `minikube tunnel` (leave this running to access services via clusterip via loadbalancer)
- `kubectl create -f minio.yaml`
- `kubectl get services -n minio`
- Navigate to the external ip of the minio service, on port 9090  (http://10.100.138.239:9090/buckets)
- Login with the credentials (admin/adminpassword123)
- Create a bucket named "test", no need to modify any other options. Note, this is the only "manual" step. We could create a job to do this using aws cli, but this serves as a good checkpoint to verify configuration.
- `kubectl create -f postgres.yaml`
- `kubectl get pods -n postgres`
- Wait for postgres to be up
- `kubectl create -f hive.yaml`
- `kubectl create -f trino.yaml`
- Trino is started with 1 coordinator and 2 workers

## Usage
- `kubectl get services -n trino`
- Note the external ip for trino from above
- `trino --server http://<external ip of trino>:8080 --catalog iceberg --debug`
- This is a sql repl

## Some sql commands to try
- `show schemas from iceberg;`
- `create schema iceberg.demo with (location = 's3a://test/demo');`
- `create table iceberg.demo.one(country varchar, state varchar, population bigint) with (format = 'PARQUET');`
- `show tables from demo;`
- `insert into demo.one values('usa','delaware',2000);`
- `insert into demo.one values('usa','new york',1000);`
- `select * from demo.one;`
- `delete from demo.one where state='new york';`
- `select * from demo."one$snapshots";` This `$snapshots` is accessing iceberg metadata, refer to trino iceberg connector for other iceberg metadata tables.

You are also able to access the "hive" catalog that is used by the standalone hive metastore. Trino is configured to access this same catalog.


## Disclaimer
Minio/s3 and postgres credentials are passed into apps using secrets and environment variables.
Hive/Trino both have configuration files that are injected into the container, these configurations have the credentials templated via environment variable.
In other words, if you need to change minio/postgres credentials, you will need to recreate the 'minio-admin' and 'postgres-admin' secrets.
There are duplicate copies of these secrets in each namespace (secrets are bound to namespaces).
A production environment would have a better secret-to-namespace management strategy.

A minio user/pass can be used as an s3 key/secret.

Iceberg has two version, V1 and V2. V2 is needed for deletion (schema evolution).
This is only supported when iceberg uses a hive metastore for the catalog, jdbc/rest will not work.

I've referenced bits and pieces to get this all hooked up together.
The repo from alexcpn was helpful for figuring out a lot of the configuration, especially the standalone hive metastore.
I've used the image built from alexcpn as it is heavily inspired by the official iceberg-spark configurations, adds some jars we need from trino/postgres.
Checking the [dockerfile](https://github.com/alexcpn/presto_in_kubernetes/blob/main/hive/Dockerfile), I wouldnt do anything different and this image is already publically available.
This image is used for the metastore, and for initializing the postgres db(via the apache schematool).

## Misc

Take a look around through the minio webui, check out the "test" bucket.

Experiment and see what the data/metadata looks like when a schema is created in  `hive` vs `iceberg`, or when the file formats are different `PARQUET/AVRO/ORC`.

Explore the trino webui!
If trino is running with `--debug` you will also get a url link to the job, like - http://10.107.143.42:8080/ui/query.html?20230525_143631_00014_6xyhs

Try to scale the trino workers, and watch as the queries are distributed to them.

You can scale the trino worker pool via:
- `kubectl scale statefulset trino-worker -n trino --replicas=4`
- `kubectl scale statefulset trino-worker -n trino --replicas=2`

Useful kubectl:
- `watch -n 1 kubectl get services --all-namespaces`
- `watch -n 1 kubectl get pods -n trino`


Links
- https://trino.io/docs/current/connector/iceberg.html#metadata-tables
- https://github.com/alexcpn/presto_in_kubernetes