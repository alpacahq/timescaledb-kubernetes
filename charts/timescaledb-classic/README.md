# TimescaleDB Helm Chart

This directory contains a Kubernetes chart to deploy a five node [TimescaleDB](https://github.com/timescale/timescaledb/) cluster using a [Docker Image](https://github.com/timescale/timescaledb-docker-ha) and a StatefulSet.

## Prerequisites Details
* Kubernetes 1.5+
* PV support on the underlying infrastructure

## StatefulSet Details
* https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/

## StatefulSet Caveats
* https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#limitations

## Todo
* Make namespace configurable

## Chart Details
This chart will do the following:

* Implement a HA scalable TimescaleDB using a Kubernetes StatefulSet.

## Installing the Chart

To install the chart with the release name `my-release`:

```console
$ helm install --name my-release .
```

To install the chart with randomly generated passwords:

```console
$ helm install --name my-release . \
  --set credentials.superuser="$(< /dev/urandom tr -dc _A-Z-a-z-0-9 | head -c32)",credentials.admin="$(< /dev/urandom tr -dc _A-Z-a-z-0-9 | head -c32)",credentials.standby="$(< /dev/urandom tr -dc _A-Z-a-z-0-9 | head -c32)"
```

## Connecting to PostgreSQL

Your access point is a cluster IP. In order to access it spin up another pod:

```console
$ kubectl run -i --tty --rm psql --image=postgres --restart=Never -- bash -il
```

Then, from inside the pod, connect to PostgreSQL:

```console
$ psql -U admin -h my-release-timescaledb.default.svc.cluster.local postgres
<admin password from values.yaml>
postgres=>
```

## Configuration

The following table lists the configurable parameters of the TimescaleDB chart and their default values.

|       Parameter                   |           Description                       |                         Default                     |
|-----------------------------------|---------------------------------------------|-----------------------------------------------------|
| `nameOverride`                    | Override the name of the chart              | `timescaledb`                                       |
| `fullnameOverride`                | Override the fullname of the chart          | `nil`                                               |
| `replicaCount`                    | Amount of pods to spawn                     | `3`                                                 |
| `image.repository`                | The image to pull                           | `timescaledev/timescaledb-ha`                       |
| `image.tag`                       | The version of the image to pull            | `78603166-pg11`                                     |
| `image.pullPolicy`                | The pull policy                             | `IfNotPresent`                                      |
| `credentials.superuser`           | Password of the superuser                   | `tea`                                               |
| `credentials.admin`               | Password of the admin                       | `cola`                                              |
| `credentials.standby`             | Password of the replication user            | `pinacolada`                                        |
| `kubernetes.dcs.enable`           | Using Kubernetes as DCS                     | `true`                                              |
| `kubernetes.configmaps.enable`    | Using Kubernetes configmaps instead of endpoints | `false`                                        |
| `env`                             | Extra custom environment variables          | `{}`                                                |
| `resources`                       | Any resources you wish to assign to the pod | `{}`                                                |
| `nodeSelector`                    | Node label to use for scheduling            | `{}`                                                |
| `tolerations`                     | List of node taints to tolerate             | `[]`                                                |
| `affinityTemplate`                | A template string to use to generate the affinity settings | Anti-affinity preferred on hostname  |
| `affinity`                        | Affinity settings. Overrides `affinityTemplate` if set. | `{}`                                    |
| `schedulerName`                   | Alternate scheduler name                    | `nil`                                               |
| `persistentVolume.accessModes`    | Persistent Volume access modes              | `[ReadWriteOnce]`                                   |
| `persistentVolume.annotations`    | Annotations for Persistent Volume Claim`    | `{}`                                                |
| `persistentVolume.mountPath`      | Persistent Volume mount root path           | `/home/postgres/pgdata`                             |
| `persistentVolume.size`           | Persistent Volume size                      | `2Gi`                                               |
| `persistentVolume.storageClass`   | Persistent Volume Storage Class             | `volume.alpha.kubernetes.io/storage-class: default` |
| `persistentVolume.subPath`        | Subdirectory of Persistent Volume to mount  | `""`                                                |
| `rbac.create`                     | Create required role and rolebindings       | `true`                                              |
| `serviceAccount.create`           | If true, create a new service account	      | `true`                                              |
| `serviceAccount.name`             | Service account to be used. If not set and `serviceAccount.create` is `true`, a name is generated using the fullname template | `nil` |

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`.

Alternatively, a YAML file that specifies the values for the parameters can be provided while installing the chart. For example,

```console
$ helm install --name my-release -f values.yaml .
```

> **Tip**: You can use the default [values.yaml](values.yaml)

## Cleanup

To remove the spawned pods you can run a simple `helm delete <release-name>`.

Helm will however preserve created persistent volume claims,
to also remove them execute the commands below.

```console
$ release=<release-name>
$ helm delete $release
$ kubectl delete pvc -l release=$release
```

## Internals
TimescaleDB is built on top of PostgreSQL. To ensure HA, [Patroni](https://github.com/zalando/patroni)
is used and for Backup & Recovery [pgBackRest](https://github.com/pgbackrest/pgbackrest) is included.

Patroni is responsible for electing a PostgreSQL master pod by leveraging the
DCS of your choice. After election it adds a `spilo-role=master` label to the
elected master and set the label to `spilo-role=replica` for all replicas.
Simultaneously it will update the `<release-name>-timescaledb` endpoint to let the
service route traffic to the elected master.

```console
$ kubectl get pods -l spilo-role -L spilo-role
NAME                       READY   STATUS    RESTARTS   AGE     SPILO-ROLE
my-release-timescaledb-0   1/1     Running   0          5m1s    master
my-release-timescaledb-1   1/1     Running   0          4m20s   replica
my-release-timescaledb-2   1/1     Running   0          3m34s   replica
my-release-timescaledb-3   1/1     Running   0          84s     replica
my-release-timescaledb-4   1/1     Running   0          44s     replica
```