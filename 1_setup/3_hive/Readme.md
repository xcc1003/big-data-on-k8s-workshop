# Hive Metastore standalone with Postgresql backend

This chart will install a standalone hive metastore backed by a postgresql
database (bitnami chart).
The configuration includes the drivers to connect to s3 storage. This means
you can use this hive metastore for example in combination with trino to
access data stored on s3 buckets.

#### Steps to do

1. Build and Push Docker Image
2. Configure and Install Helm Chart

## Docker Image

The Hive standalone Metastore image (`docker/Dockerfile.hive`) is completely selfe made and needs to be build and pushed to a repository that can be refered from Kubernetes.

to build the image run one of the following commands

```
# regular build & push
docker build -t thinkportgmbh/workshops:hive-metastore -f Dockerfile.hive .
docker push  thinkportgmbh/workshops:hive-metastore


# crossbuild on Mac Book with M1 Chip
docker buildx build --push --platform linux/amd64,linux/arm64 --tag thinkportgmbh/workshops:hive-metastore  -f Dockerfile.hive .
```

## Helm Chart

### Set values

first edit the `values.yaml` file and set the values accordingly for

- s3 (endpoint and secrets)
- hive image
- user and pwd for postgres backend

### Install Chart

first create a new namespace

```consol
create namespace hive
```

then run helm

```consol
create namespace hive

helm upgrade --install -f values.yaml  hive-metastore -n hive .

```

Deploy additional Ingress

### Uninstall Chart

in case of issues uninstall the chart and the create pvc for the database

```
helm list -n hive
helm delete hive-metastore -n hive

kubectl delete pvc data-hive-metastore-postgresql-0
```

### Configuration

The values.yaml of this chart is sectioned in different topics. You can
also use the command line to set the parameters (take a look on the [helm readme](https://helm.sh/docs/intro/using_helm/)).

#### S3 Configuration

| Parameter                   | Default value | Description                                                                                                                                                |
| --------------------------- | ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| s3.accessKey                | dummy         | The S3 access key to connect to your s3 server.                                                                                                            |
| s3.secretKey                | dummy         | The S3 secret key to connect to your s3 server.                                                                                                            |
| s3.endpoint                 | dummy         | The url of your S3 server.                                                                                                                                 |
| s3.sslEnabled               | false         | Should ssl be used when establishing the connection?                                                                                                       |
| s3.existingSecret           | Not used      | Name of the secret that stores your s3 configuration. The secret values of the keys supplied in secretKeyNames will override the corresponding s3 entries. |
| s3.existingSecretNamespace  | Not used      | Namespace of the existing secret.                                                                                                                          |
| s3.secretKeyNames.accessKey | Not used      | Name of the key that has a value for the access key.                                                                                                       |
| s3.secretKeyNames.secretKey | Not used      | Name of the key that has a value for the secret key.                                                                                                       |
| s3.secretKeyNames.endpoint  | Not used      | Name of the key that has a value for the server url.                                                                                                       |

#### Postgresql Configuration

| Parameter                     | Default value      | Description                                                                                                                                                                                                                                                         |
| ----------------------------- | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| deployPostgresql.enabled      | true               | Deploys a postgresql server as backend in the target namespace. When set to false postgresql.existingInstance must be used with a correct combination of the postgresql.\* parameters. An existing secret generated by a bitnami postgresql chart can be used, too. |
| deployPostgresql.createSecret | true               | Will create a secret for the postgresql server. Must be used with postgresql.exisitingSecret. When used with postgresql.Password the supplied password will be used. Otherwise a random password will be generated.                                                 |
| postgresql.postgresqlPassword | Not used           | Password of the "postgres" user (used by hive metastore to connect to the database).                                                                                                                                                                                |
| postgresql.postgresqlUsername | Not used           | Name of the sql user ("postgres") that is used by hive metastore to connect to the database).                                                                                                                                                                       |
| postgresql.postgresqlDatabase | metastore          | Name of the database that is used by hive metastore when connecting the postgresql server).                                                                                                                                                                         |
| postgresql.existingSecret     | metastore-postgres | Namespace of a existing bitnami postgresql secret / the name of the secret that will be created if deployPostgresql.createSecret is set to true.                                                                                                                    |
| postgresql.exisitingInstance  | Not used           | This is not an official bitnami postgres parameter. It is only used when deployPostgresql.enabled is set to false.                                                                                                                                                  |
| Any other bitnami parameter.  | -                  | You can use any of the bitnami postgresql parameters. But please not that this is not tested or supported and can lead to unpredictable results.                                                                                                                    |

##### Default Parameters

| Parameter          | Default value | Description                                                                                       |
| ------------------ | ------------- | ------------------------------------------------------------------------------------------------- |
| hiveConfigAsSecret | true          | The metastore-site.xml file will be saved as secret. If set to false a configmap will be created. |

While some other default configuration parameters can be changed (default section in
values.yaml), it is only recommended for good reasons. Be sure that
you know what you are doing and that you are able to debug unexpected
behaviour.

#### Secrets

The following secrets can be used / are generated:

- postgresql database credentials (existing/generated)
- s3 server credentials (exisiting)
- hive metastore configuration (metastore-site.xml) (generated)

## More Info

- [PostgreSQL Bitnami Chart](https://github.com/bitnami/charts/tree/master/bitnami/postgresql)
- [Hive Metastore](https://cwiki.apache.org/confluence/display/hive/design#Design-Metastore)

## Typical output on success

** Please be patient while the chart is being deployed **

PostgreSQL can be accessed via port 5432 on the following DNS names from within your cluster:

    hive-metastore-postgresql.hive.svc.cluster.local:5432 - Read/Write connection

To get the password for "postgres" run:

    export POSTGRES_ADMIN_PASSWORD=$(kubectl get secret --namespace hive hive-metastore-postgresql -o jsonpath="{.data.postgresql-postgres-password}" | base64 --decode)

To connect to your database run the following command:

    kubectl run hive-metastore-postgresql-client --rm --tty -i --restart='Never' --namespace hive --image docker.io/bitnami/postgresql:11.14.0-debian-10-r28 --env="PGPASSWORD=$POSTGRES_PASSWORD"

--command -- psql --host hive-metastore-postgresql -U train@thinkport -d metastore -p 5432

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace hive svc/hive-metastore-postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U train@thinkport -d metastore -p 5432

Metastore Chart Information

**_ Please wait - the deployment will take some time to be ready _**

Get the hive metastore URL by running these commands:

export POD_NAME=$(kubectl get pods --namespace hive -l "app.kubernetes.io/name=hive-metastore,app.kubernetes.io/instance=hive-metastore" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace hive $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:9083 to use your application"
  kubectl --namespace hive port-forward $POD_NAME 9083:$CONTAINER_PORT

## Typical Issues

- image cannot be started --> Image was build on mac m1 without crossbuild
- issues authenticating on posgres --> use secret enabled on upgrade and new secret added in kubernetes but not in the hive xml configuration file - uninstall and install again or use password and username
