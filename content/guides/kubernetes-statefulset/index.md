---
title: "Running in Kubernetes"
layout: docs
date: 2021-03-07T21:25:53Z
menu:
  docs:
    parent: "guides"
weight: 350
---


This guide will get you running Litestream as a sidecar container in a Kubernetes StatefulSet, alongside your existing application using SQLite. This means that as long as your application is running, so will Litestream. This guide assumes you already have an application using SQLite running in Kubernetes, but a full set of example YAML manifests will be provided.


## Prerequisites

This guide assumes you have read the [_Getting Started_](/getting-started)
tutorial already. Please read that to understand the basic operation of Litestream.

It also assumes you already have a Kubernetes [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) and that your SQLite database is stored on a [Volume](https://kubernetes.io/docs/concepts/storage/volumes/). However the YAML files for a full working example are available [on GitHub](https://github.com/cablespaghetti/litestream.io/tree/develop/content/guides/kubernetes-statefulset).

You will also need [Helm](https://helm.sh/) installed on your local machine to deploy MinIO.

### Setting up MinIO

We'll use an instance of [MinIO](https://min.io/) running in our cluster, deployed using [minio-operator](https://github.com/minio/operator) for this example, using the operator's [Helm Chart](https://github.com/minio/operator/tree/master/helm/minio-operator)

1. Add the Helm chart repository
```bash
helm repo add minio https://operator.min.io/
```

2. Install the operator to your cluster
```bash
helm install --namespace minio-operator --create-namespace --generate-name minio/minio-operator
```

3. Create the MinIO cluster with default configuration in your current namespace
```bash
kubectl apply -f https://raw.githubusercontent.com/minio/operator/master/examples/tenant-lite.yaml
```

{{< alert icon="!" text="This creates 4 MinIO pods with 100GB storage each. You may wish to adjust this." >}}

4. Wait for the minio pods to come up in your cluster (this might take a few minutes), then port forward to make minio available on your local machine with
```bash
kubectl port-forward svc/minio 4443:443
```

5. Open a web browser to <a href="https://localhost:4443/" target="_blank">https://localhost:4443/</a>, accept the certificate warning and enter the default credentials:

```
Username: minio
Password: minio123
```

6. Click the "+" button in the lower right-hand corner and then click the
_"Create Bucket"_ icon. Name your bucket, `"mybkt"`.


## Kubernetes Secret for Credentials

You must provide Litestream with the credentials for your bucket somehow. Whilst we're using MinIO here with default credentials, this will usually be sensitive credentials. The best way built into Kubernetes is to use [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/).

To create a secret you can run a kubectl command like:

```sh
kubectl create secret generic litestream --from-literal=AWS_ACCESS_KEY_ID="minio" --from-literal=AWS_SECRET_ACCESS_KEY="minio123"
```


## Kubernetes ConfigMap for Litestream configuration

When running as a sidecar container, we'll configure Litestream using a
[configuration file](/reference/config) and store it in a Kubernetes [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/).

Litestream monitors one or more _databases_ and each of those databases
replicates to one or more _replicas_. First, we'll create a basic configuration
file; save it as `litestream.yml`.

This assumes your application stores its SQLite database in the root of your Persistent Volume and it is called `db.db`.

```yaml
addr: ":9090"
dbs:
  - path: /db/db.db
    replicas:
      - url: s3://mybkt.minio:443/db.db
```

This configuration specifies that we want want Litestream to monitor our
`db.db` database in a directory called `db` (where we have mounted the Persistent Volume) and continuously replicate it to our `mybkt` bucket on our minio cluster at https://minio. We've also enabled the metrics endpoint which will come in useful if you want to monitor LiteStream with Prometheus.

After changing our configuration, we'll need to save it as a ConfigMap:

```sh
kubectl create configmap litestream --from-file=litestream.yml
```


## Adding Litestream to our StatefulSet

Now that we've got our ConfigMap and Secret set up for Litestream to use, we need to add an additional container to the Pod created by our StatefulSet. Due to the limitations of SQLite and Litestream, your StatefulSet should be set up to only create a single Pod.


### Volumes

There are two volumes Litestream is going to need; one for the configuration file and one for your SQLite database.

Set up the volume for your Pod like this:

```yaml
volumes:
  - name: litestream
    configMap:
      name: litestream
```

You should also have a volumeClaimTemplate for the Persistent Volume storing your SQLite database in your StatefulSet like this:

```yaml
volumeClaimTemplates:
- metadata:
    name: database
  spec:
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 100Mi
```


### The Litestream Container

Now everything else is set up you can add an extra entry to the `containers` section of your StatefulSet configuration like this:

```yaml
- name: litestream
  image: litestream/litestream:0.3.3
  volumeMounts:
  - name: database
    mountPath: /db
  - name: litestream
    mountPath: /etc/litestream.yml
    subPath: litestream.yml
  env:
  - name: AWS_ACCESS_KEY_ID
    valueFrom:
      secretKeyRef:
        name: litestream
        key: AWS_ACCESS_KEY_ID
  - name: AWS_SECRET_ACCESS_KEY
    valueFrom:
      secretKeyRef:
        name: litestream
        key: AWS_SECRET_ACCESS_KEY
  args:
  - replicate
  ports:
  - name: metrics
    containerPort: 9090
```

This configuration:

* Pulls the Litestream container image from Docker Hub
* Mounts the database Persistent Volume to `/db` in the container
* Mounts the `litestream.yml` file in your ConfigMap to `/etc/litestream.yml` in the container
* Gets your AWS credentials from the Secret and exposes them to Litestream as environment variables
* Runs the Litestream [replicate](/reference/replicate/) command
* Exposes the metrics endpoint on port 9090

After applying this configuration you should be able to see Litestream start up by tailing the logs:

```sh
kubectl logs -f -c litestream myapp-0
```


## Further reading

You now have a production-ready replication setup using SQLite and Litestream.
Please see the [Reference](/reference) section for more configuration options.
