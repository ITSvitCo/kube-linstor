# Kube-Linstor

Containerized Linstor Storage easy to run in your Kubernetes cluster.

## Images


| Image                    | Build Status                                                                      |
|:-------------------------|:----------------------------------------------------------------------------------|
| **[linstor-controller]** | [![linstor-controller-status]](https://hub.docker.com/r/kvaps/linstor-controller) |
| **[linstor-satellite]**  | [![linstor-satellite-status]](https://hub.docker.com/r/kvaps/linstor-satellite)   |
| **[linstor-stunnel]**    | [![linstor-stunnel-status]](https://hub.docker.com/r/kvaps/linstor-stunnel)       |

[linstor-controller]: dockerfiles/linstor-controller/Dockerfile
[linstor-controller-status]: https://img.shields.io/docker/cloud/build/kvaps/linstor-controller.svg
[linstor-satellite]: dockerfiles/linstor-controller/Dockerfile
[linstor-satellite-status]: https://img.shields.io/docker/cloud/build/kvaps/linstor-satellite.svg
[linstor-stunnel]: dockerfiles/linstor-stunnel/Dockerfile
[linstor-stunnel-status]: https://img.shields.io/docker/cloud/build/kvaps/linstor-stunnel.svg

## Requirements

* Working Kubernetes cluster.
* DRBD9 kernel module installed on each sattelite node.
* PostgeSQL database / etcd or any other backing store for redundancy.

## Limitations

* Containerized Linstor satellites tested only on Ubuntu and Debian systems, but should work on CentOS systems as well.

## QuckStart

Linstor consists of several components:

* **Linstor-controller** - Controller is main control point for Linstor, it provides API for clients and communicates with satellites for creating and monitor DRBD-devices.
* **Linstor-satellite** - Satellites run on every node, they listen and perform controller tasks. They operates directly with LVM and ZFS subsystems.
* **Linstor-csi** - CSI driver provides compatibility level for adding Linstor support for Kubernetes.

#### Preparation

[Install Helm](https://helm.sh/docs/intro/) and clone this repository, then cd into it.

> **_NOTE:_**  
> Commands below provided for Helm v3 but Helm v2 is also supported.  
> You can use `helm template` instead of `helm install`, this is also working as well.

Create `linstor` namespace.
```
kubectl create ns linstor
```

#### Database

* Install [stolon](https://github.com/helm/charts/tree/master/stable/stolon) chart:

  ```bash
  helm repo add stable https://kubernetes-charts.storage.googleapis.com
  helm install linstor-db stable/stolon --namespace linstor -f examples/linstor-db.yaml
  ```

  > **_NOTE:_**  
  > In case of update your stolon add `--set job.autoCreateCluster=false` flag to not reinitialisate your cluster.

* Create Persistent Volumes:
  ```bash
  helm install \
    --set node=node1,path=/var/lib/linstor-db \
    data-linstor-db-stolon-keeper-0 \
    helm/pv-hostpath --namespace linstor

  helm install \
    --set node=node2,path=/var/lib/linstor-db \
    data-linstor-db-stolon-keeper-1 \
    helm/pv-hostpath --namespace linstor

  helm install \
    --set node=node3,path=/var/lib/linstor-db \
    data-linstor-db-stolon-keeper-2 \
    helm/pv-hostpath --namespace linstor
  ```

  Parameters `name` and `namespace` **must match** the PVC's name and namespace of your database, `node` should match exact node name.

  Check your PVC/PV list after creation, if everything right, they should obtain **Bound** status.

* Connect to database:
  ```bash
  kubectl exec -ti -n linstor linstor-db-stolon-keeper-0 bash
  PGPASSWORD=$(cat $STKEEPER_PG_SU_PASSWORDFILE) psql -h linstor-db-stolon-proxy -U stolon postgres
  ```

* Create user and database for linstor:
  ```bash
  CREATE DATABASE linstor;
  CREATE USER linstor WITH PASSWORD 'hackme';
  GRANT ALL PRIVILEGES ON DATABASE linstor TO linstor;
  ```

#### Linstor

* Install kube-linstor chart:

  ```bash
  helm install linstor helm/kube-linstor --namespace linstor -f examples/linstor-db.yaml
  ```

## Usage

You can get interactive linstor shell by simple exec into **linstor-controller** container:

```bash
kubectl exec -ti -n linstor linstor-linstor-controller-0 -- linstor
```

Refer to [official linstor documentation](https://docs.linbit.com/linbit-docs/) for define nodes and create new resources.

#### SSL notes

This chart enables SSL encryption for control-plane by default. It does not affect the DRBD performance but makes your LINSTOR setup more secure.

Any way, do not forget to specify `--communicates-type SSL` option during node creation, example:

```bash
linstor node create alpha 1.2.3.4 --communication-type SSL
```

If you want to have external access, you need to download certificates for linstor client:

```bash
kubectl get secrets -name linstor-linstor-client-tls \
  -o go-template='{{ index .data "tls.key" | base64decode }}{{ index .data "tls.crt" | base64decode }}
```

Then follow [official linstor documentation](https://www.linbit.com/drbd-user-guide/users-guide-linstor/#s-rest-api-https-restricted-client) to configure client.

#### Enable SSL

If you're switching your setup from PLAIN to SSL, this simple command will reconfigure all your nodes:

```bash
linstor n l | awk '/(PLAIN)/ { print "linstor n i m -p 3367 --communication-type SSL " $2 " default" }' | sh -ex
```

## Licenses

* **[This project](LICENSE)** under **Apache License**
* **[linstor-server]**, **[drbd]** and **[drbd-utils]** is **GPL** licensed by LINBIT
* **[linstor-csi]** under **Apache License** by LINBIT

[linstor-server]: https://github.com/LINBIT/linstor-server/blob/master/COPYING
[drbd]: https://github.com/LINBIT/drbd-9.0/blob/master/COPY
[drbd-utils]: https://github.com/LINBIT/drbd-utils/blob/master/COPYING
[linstor-csi]: https://github.com/piraeusdatastore/linstor-csi/blob/master/LICENSE
