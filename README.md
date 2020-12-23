# Keycloak K3s Deployment

This repository contains instructions and manifest files for deploying Keycloak
on a K3s Kubernetes cluster.

[Keycloak] is a widely used production Identity Provider (IdP) and Identity
Broker and natively supports authentication mechanisms such as SAML and OpenID
Connect.

[K3s] is a lightweight, certified Kubernetes distribution, for production
workloads from Rancher Labs.

## Keycloak Container Images

The JBoss Keycloak images provided on Docker Hub ([jboss/keycloak]) only support
the `linux/amd64` platform. My K3s cluster is running on Raspberry Pi and
requires `arm64` support. I have built and provided Keycloak images for `arm64`
which can be found on Docker Hub, see [sleighzy/keycloak] for these.

## Keycloak Data Storage

This repository provides two options for storing configuration and user data.

- the default `H2` embedded database (file-system backed)
- PostgreSQL database

### H2 embedded database

When using the H2 embedded database the `300-pvc.yaml` file should be applied.
This creates a persistent volume claim using the K3s [local path provisioner]
storage class. When deploying into a multi-node cluster with the default K3s
local path provisioner configuration, and not a shared NFS or other mechanism,
the storage will be created on that node. If the Keycloak pod is scheduled onto
a different node later then this storage directory will not exist and it will
not be able to find the Keycloak configuration or user data. Using pod affinity
can enforce that the container is always scheduled onto the same node. See
[External Hard Drive for Persistent Storage] for more information on my K3s
configuration and deployment manifest example for achieving this.

### PostgreSQL database

The manifest files in this repository use PostgreSQL as the persistent databaase
as this is what I use in my cluster.

The below command can be run to port-forward to the Postgres database running
within the K3s cluster. Your database may be installed elsewhere so use whatever
process you currently follow for database administration.

```sh
$ kubectl port-forward -n postgres svc/postgres 5432:5432
Forwarding from 127.0.0.1:5432 -> 5432
Forwarding from [::1]:5432 -> 5432
```

The below commands will connect to Postgres and create a new database for the
`keycloak` user to store the Keycloak configuration and user data. Keycloak will
generate the schema when it starts up for the first time and connects to this
database. The value for the encrypted password needs to be remembered as this
will be base64 encoded and added as a Kubernetes secret.

```sh
$ psql --host localhost --user postgres

postgres=# create database keycloak;
CREATE DATABASE
postgres=# create user keycloak with encrypted password 'xxxxxxxxx';
CREATE ROLE
postgres=# grant all privileges on database keycloak to keycloak;
GRANT

postgres=# \q
```

## Deploy Keycloak

Create the namespace to deploy the resources into.

```sh
kubectl create namespace keycloak
```

Create passwords and add as base64 encoded secrets in the `200-secrets.yaml`
file and then apply it.

The `my-postgres-password` value should match the password that was provided as
the "encrypted password" when the Postgres `keycloak` user was created. This is
only needed when integrating with Postgres.

```sh
$ echo -n 'my-keycloak-password' | base64
bXkta2V5Y2xvYWstcGFzc3dvcmQ=
$ echo -n 'my-postgres-password' | base64
bXktcG9zdGdyZXMtcGFzc3dvcmQ=
```

### Using H2 Embedded Database

The configuration and manifests in this repository use the Postgres database by
default. To use with the H2 embedded database instead the `POSTGRES_X` variables
in the `100-config.yaml` file and associated references in the
`600-deployment.yaml` file will need to be removed. In addition to this the two
sections for the `keycloak-data` persistent volume and volume mount in the
`600-deployment.yaml` need to be uncommented before applying the file.

### Start Deployment

```sh
$ kubectl apply -f 600-deployment.yaml

$ kubectl get events -n keycloak -w
LAST SEEN   TYPE     REASON                 OBJECT                               MESSAGE
24m         Normal    ScalingReplicaSet   deployment/keycloak              Scaled up replica set keycloak-5cb8fc668b to 1
24m         Normal    SuccessfulCreate    replicaset/keycloak-5cb8fc668b   Created pod: keycloak-5cb8fc668b-jmblh
24m         Normal    Scheduled           pod/keycloak-5cb8fc668b-jmblh    Successfully assigned keycloak/keycloak-5cb8fc668b-jmblh to k3s-1
24m         Normal    Pulling             pod/keycloak-5cb8fc668b-jmblh    Pulling image "sleighzy/keycloak:12.0.1-arm64"
24m         Normal    Pulled              pod/keycloak-5cb8fc668b-jmblh    Successfully pulled image "sleighzy/keycloak:12.0.1-arm64"
24m         Normal    Created             pod/keycloak-5cb8fc668b-jmblh    Created container keycloak
24m         Normal    Started             pod/keycloak-5cb8fc668b-jmblh    Started container keycloak
```

Follow the Keycloak logs to ensure that everything has started as expected.

```sh
$ kubectl logs -f -n keycloak -l app=keycloak

...
22:28:02,008 INFO  [org.jboss.as] (Controller Boot Thread) WFLYSRV0060: Http management interface listening on http://127.0.0.1:9990/management
22:28:02,008 INFO  [org.jboss.as] (Controller Boot Thread) WFLYSRV0051: Admin console listening on http://127.0.0.1:9990
```

Run the below command to forward port `32080` on the K3s node to the Keycloak
service.

```sh
kubectl port-forward -n keycloak --address 0.0.0.0 service/keycloak 32080:http
```

You should now be able to access Keycloak in your browser by navigating to:

<http://k3s-node-ip-address:32080>

### Ingress Route

My K3s cluster uses [Traefik v2] as the Kubernetes ingress controller and the
`IngressRoute` resource it provides. You can just as easily use the standard
Kubernetes `Ingress` resource, or other such ingress controller annotations or
configuration methods.

The `700-ingressroute.yaml` file specifies the usage of TLS and associated
certificates. You should always use HTTPS when accessing resources, especially
over the internet. You can update that file as necessary to work with your
deployment, e.g. using a different Traefik endpoint and removing the `tls`
section.

[external hard drive for persistent storage]:
  https://github.com/sleighzy/raspberry-pi-k3s-homelab/blob/main/k3s.md#external-hard-drive-for-persistent-storage
[jboss/keycloak]: https://hub.docker.com/r/jboss/keycloak
[keycloak]: https://www.keycloak.org/
[local path provisioner]: https://rancher.com/docs/k3s/latest/en/storage/
[sleighzy/keycloak]: https://hub.docker.com/repository/docker/sleighzy/keycloak
[traefik v2]: https://traefik.io/traefik/
