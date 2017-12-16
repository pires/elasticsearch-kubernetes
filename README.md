# kubernetes-elasticsearch-cluster
Elasticsearch (6.1.0) cluster on top of Kubernetes made easy.

### Table of Contents

* [(Very) Important Notes](#important-notes)
* [Pre-Requisites](#pre-requisites)
* [Build container image (optional)](#build-images)
* [Test](#test)
  * [Deploy](#deploy)
  * [Access the service](#access-the-service)
* [Pod anti-affinity](#pod-anti-affinity)
* [Availability](#availability)
* [Deploy with Helm](#helm)
* [Install plug-ins](#plugins)
* [Clean-up with Curator](#curator)
* [Kibana](#kibana)
* [FAQ](#faq)
* [Troubleshooting](#troubleshooting)

## Abstract

Elasticsearch best-practices recommend to separate nodes in three roles:
* `Master` nodes - intended for clustering management only, no data, no HTTP API
* `Client` nodes - intended for client usage, no data, with HTTP API
* `Data` nodes - intended for storing and indexing data, no HTTP API

Given this, I'm going to demonstrate how to provision a production grade scenario consisting of 3 master, 2 client and 2 data nodes.

<a id="important-notes">

## (Very) Important notes

* Elasticsearch pods need for an init-container to run in privileged mode, so it can set some VM options. For that to happen, the `kubelet` should be running with args `--allow-privileged`, otherwise
the init-container will fail to run.

* By default, `ES_JAVA_OPTS` is set to `-Xms256m -Xmx256m`. This is a *very low* value but many users, i.e. `minikube` users, were having issues with pods getting killed because hosts were out of memory.
One can change this in the deployment descriptors available in this repository.

* As of the moment, Kubernetes pod descriptors use an `emptyDir` for storing data in each data node container. This is meant to be for the sake of simplicity and should be adapted according to one's storage needs.

* The [stateful](stateful) directory contains an example which deploys the data pods as a `StatefulSet`. These use a `volumeClaimTemplates` to provision persistent storage for each pod.

<a id="pre-requisites">

## Pre-requisites

* Kubernetes cluster with **alpha features enabled** (tested with v1.7.2 on top of [Vagrant + CoreOS](https://github.com/pires/kubernetes-vagrant-coreos-cluster)), thas's because curator
 is a CronJob object which comes from batch/v2alpha1, to enable it, just add
 `--runtime-config=batch/v2alpha1=true` into your kube-apiserver options.
* `kubectl` configured to access the cluster master API Server

<a id="build-images">

## Build images (optional)

Providing one's own version of [the images automatically built from this repository](https://github.com/pires/docker-elasticsearch-kubernetes) will not be supported. This is an *optional* step. One has been warned.

## Test

### Deploy

```
kubectl create -f es-discovery-svc.yaml
kubectl create -f es-svc.yaml
kubectl create -f es-master.yaml
kubectl rollout status -f es-master.yaml
kubectl create -f es-client.yaml
kubectl create -f es-data.yaml
kubectl rollout status -f es-client.yaml
kubectl rollout status -f es-data.yaml
```

Check one of the Elasticsearch master nodes logs:
```
$ kubectl get svc,deployment,pods -l component=elasticsearch
NAME                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
svc/elasticsearch             ClusterIP   10.100.35.143    <none>        9200/TCP   6m
svc/elasticsearch-discovery   ClusterIP   10.100.247.154   <none>        9300/TCP   6m
svc/kubernetes                ClusterIP   10.100.0.1       <none>        443/TCP    18m

NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/es-client   2         2         2            2           1m
deploy/es-data     2         2         2            2           1m
deploy/es-master   3         3         3            3           6m

NAME                            READY     STATUS    RESTARTS   AGE
po/es-client-864b45c866-68f45   1/1       Running   0          1m
po/es-client-864b45c866-wmkdb   1/1       Running   0          1m
po/es-data-8659fdbbcc-8wgr4     1/1       Running   0          1m
po/es-data-8659fdbbcc-gvxnj     1/1       Running   0          1m
po/es-master-7c86c468df-7mp8w   1/1       Running   0          6m
po/es-master-7c86c468df-ds5tb   1/1       Running   0          6m
po/es-master-7c86c468df-wkhxv   1/1       Running   0          6m
```

```
$ kubectl logs po/es-master-7c86c468df-7mp8w
[2017-12-16T15:40:41,573][INFO ][o.e.n.Node               ] [es-master-7c86c468df-7mp8w] initializing ...
[2017-12-16T15:40:41,801][INFO ][o.e.e.NodeEnvironment    ] [es-master-7c86c468df-7mp8w] using [1] data paths, mounts [[/data (/dev/sda9)]], net usable_space [13.7gb], net total_space [15.5gb], types [ext4]
[2017-12-16T15:40:41,801][INFO ][o.e.e.NodeEnvironment    ] [es-master-7c86c468df-7mp8w] heap size [247.5mb], compressed ordinary object pointers [true]
[2017-12-16T15:40:41,803][INFO ][o.e.n.Node               ] [es-master-7c86c468df-7mp8w] node name [es-master-7c86c468df-7mp8w], node ID [sSTfWZuCQTyEDMSW-ZCVRQ]
[2017-12-16T15:40:41,804][INFO ][o.e.n.Node               ] [es-master-7c86c468df-7mp8w] version[6.1.0], pid[1], build[c0c1ba0/2017-12-12T12:32:54.550Z], OS[Linux/4.14.4-coreos/amd64], JVM[Oracle Corporation/OpenJDK 64-Bit Server VM/1.8.0_151/25.151-b12]
[2017-12-16T15:40:41,805][INFO ][o.e.n.Node               ] [es-master-7c86c468df-7mp8w] JVM arguments [-XX:+UseConcMarkSweepGC, -XX:CMSInitiatingOccupancyFraction=75, -XX:+UseCMSInitiatingOccupancyOnly, -XX:+DisableExplicitGC, -XX:+AlwaysPreTouch, -Xss1m, -Djava.awt.headless=true, -Dfile.encoding=UTF-8, -Djna.nosys=true, -Djdk.io.permissionsUseCanonicalPath=true, -Dio.netty.noUnsafe=true, -Dio.netty.noKeySetOptimization=true, -Dlog4j.shutdownHookEnabled=false, -Dlog4j2.disable.jmx=true, -Dlog4j.skipJansi=true, -XX:+HeapDumpOnOutOfMemoryError, -Xms256m, -Xmx256m, -Des.path.home=/elasticsearch, -Des.path.conf=/elasticsearch/config]
[2017-12-16T15:40:43,970][INFO ][o.e.p.PluginsService     ] [es-master-7c86c468df-7mp8w] loaded module [aggs-matrix-stats]
[2017-12-16T15:40:43,970][INFO ][o.e.p.PluginsService     ] [es-master-7c86c468df-7mp8w] loaded module [analysis-common]
[2017-12-16T15:40:43,970][INFO ][o.e.p.PluginsService     ] [es-master-7c86c468df-7mp8w] loaded module [ingest-common]
[2017-12-16T15:40:43,970][INFO ][o.e.p.PluginsService     ] [es-master-7c86c468df-7mp8w] loaded module [lang-expression]
[2017-12-16T15:40:43,970][INFO ][o.e.p.PluginsService     ] [es-master-7c86c468df-7mp8w] loaded module [lang-mustache]
[2017-12-16T15:40:43,970][INFO ][o.e.p.PluginsService     ] [es-master-7c86c468df-7mp8w] loaded module [lang-painless]
[2017-12-16T15:40:43,970][INFO ][o.e.p.PluginsService     ] [es-master-7c86c468df-7mp8w] loaded module [mapper-extras]
[2017-12-16T15:40:43,971][INFO ][o.e.p.PluginsService     ] [es-master-7c86c468df-7mp8w] loaded module [parent-join]
[2017-12-16T15:40:43,971][INFO ][o.e.p.PluginsService     ] [es-master-7c86c468df-7mp8w] loaded module [percolator]
[2017-12-16T15:40:43,971][INFO ][o.e.p.PluginsService     ] [es-master-7c86c468df-7mp8w] loaded module [reindex]
[2017-12-16T15:40:43,971][INFO ][o.e.p.PluginsService     ] [es-master-7c86c468df-7mp8w] loaded module [repository-url]
[2017-12-16T15:40:43,971][INFO ][o.e.p.PluginsService     ] [es-master-7c86c468df-7mp8w] loaded module [transport-netty4]
[2017-12-16T15:40:43,971][INFO ][o.e.p.PluginsService     ] [es-master-7c86c468df-7mp8w] loaded module [tribe]
[2017-12-16T15:40:43,973][INFO ][o.e.p.PluginsService     ] [es-master-7c86c468df-7mp8w] no plugins loaded
[2017-12-16T15:40:48,571][INFO ][o.e.d.DiscoveryModule    ] [es-master-7c86c468df-7mp8w] using discovery type [zen]
[2017-12-16T15:40:49,733][INFO ][o.e.n.Node               ] [es-master-7c86c468df-7mp8w] initialized
[2017-12-16T15:40:49,734][INFO ][o.e.n.Node               ] [es-master-7c86c468df-7mp8w] starting ...
[2017-12-16T15:40:50,110][INFO ][o.e.t.TransportService   ] [es-master-7c86c468df-7mp8w] publish_address {10.244.32.2:9300}, bound_addresses {10.244.32.2:9300}
[2017-12-16T15:40:50,134][INFO ][o.e.b.BootstrapChecks    ] [es-master-7c86c468df-7mp8w] bound or publishing to a non-loopback or non-link-local address, enforcing bootstrap checks
[2017-12-16T15:40:54,261][INFO ][o.e.c.s.ClusterApplierService] [es-master-7c86c468df-7mp8w] detected_master {es-master-7c86c468df-ds5tb}{GUKwA7U7SQOfLYQaS22ENw}{DwjBzy7FSyeHs_g95auQwA}{10.244.63.3}{10.244.63.3:9300}, added {{es-master-7c86c468df-ds5tb}{GUKwA7U7SQOfLYQaS22ENw}{DwjBzy7FSyeHs_g95auQwA}{10.244.63.3}{10.244.63.3:9300},}, reason: apply cluster state (from master [master {es-master-7c86c468df-ds5tb}{GUKwA7U7SQOfLYQaS22ENw}{DwjBzy7FSyeHs_g95auQwA}{10.244.63.3}{10.244.63.3:9300} committed version [1]])
[2017-12-16T15:40:54,265][INFO ][o.e.n.Node               ] [es-master-7c86c468df-7mp8w] started
[2017-12-16T15:40:58,440][INFO ][o.e.c.s.ClusterApplierService] [es-master-7c86c468df-7mp8w] added {{es-master-7c86c468df-wkhxv}{y5hQrG7xSBKYvf8psV1KWw}{36PVrHWWTZu7kvWANnKoAg}{10.244.92.2}{10.244.92.2:9300},}, reason: apply cluster state (from master [master {es-master-7c86c468df-ds5tb}{GUKwA7U7SQOfLYQaS22ENw}{DwjBzy7FSyeHs_g95auQwA}{10.244.63.3}{10.244.63.3:9300} committed version [3]])
[2017-12-16T15:45:32,798][INFO ][o.e.c.s.ClusterApplierService] [es-master-7c86c468df-7mp8w] added {{es-client-864b45c866-68f45}{0mcUtbXwS9CYWr8bt32FvQ}{MlvGJVChRRqFfJhtePxFxw}{10.244.32.3}{10.244.32.3:9300},}, reason: apply cluster state (from master [master {es-master-7c86c468df-ds5tb}{GUKwA7U7SQOfLYQaS22ENw}{DwjBzy7FSyeHs_g95auQwA}{10.244.63.3}{10.244.63.3:9300} committed version [4]])
[2017-12-16T15:45:35,078][INFO ][o.e.c.s.ClusterApplierService] [es-master-7c86c468df-7mp8w] added {{es-data-8659fdbbcc-gvxnj}{4LSzWE3SQHi3zqrINCB7vw}{ZB_qpIRFQOOldEtQzJqRxQ}{10.244.63.4}{10.244.63.4:9300},}, reason: apply cluster state (from master [master {es-master-7c86c468df-ds5tb}{GUKwA7U7SQOfLYQaS22ENw}{DwjBzy7FSyeHs_g95auQwA}{10.244.63.3}{10.244.63.3:9300} committed version [5]])
[2017-12-16T15:45:43,445][INFO ][o.e.c.s.ClusterApplierService] [es-master-7c86c468df-7mp8w] added {{es-client-864b45c866-wmkdb}{OerRk897Sk2nhq3cWJQIvA}{wDZQRyZkRKGQL_N3kuZ2wQ}{10.244.92.3}{10.244.92.3:9300},}, reason: apply cluster state (from master [master {es-master-7c86c468df-ds5tb}{GUKwA7U7SQOfLYQaS22ENw}{DwjBzy7FSyeHs_g95auQwA}{10.244.63.3}{10.244.63.3:9300} committed version [6]])
[2017-12-16T15:45:44,987][INFO ][o.e.c.s.ClusterApplierService] [es-master-7c86c468df-7mp8w] added {{es-data-8659fdbbcc-8wgr4}{vBCPvM7CT0O-h-BonsgrGA}{QMMMKonRQ86O76b9JJkfRA}{10.244.92.4}{10.244.92.4:9300},}, reason: apply cluster state (from master [master {es-master-7c86c468df-ds5tb}{GUKwA7U7SQOfLYQaS22ENw}{DwjBzy7FSyeHs_g95auQwA}{10.244.63.3}{10.244.63.3:9300} committed version [7]])
```

As we can assert, the cluster is up and running. Easy, wasn't it?

### Access the service

*Don't forget* that services in Kubernetes are only acessible from containers in the cluster. For different behavior one should [configure the creation of an external load-balancer](http://kubernetes.io/v1.1/docs/user-guide/services.html#type-loadbalancer). While it's supported within this example service descriptor, its usage is out of scope of this document, for now.

*Note:* if you are using one of the cloud providers which support external load balancers, setting the type field to "LoadBalancer" will provision a load balancer for your Service. You can uncomment the field in [es-svc.yaml](https://github.com/pires/kubernetes-elasticsearch-cluster/blob/master/es-svc.yaml).
```
$ kubectl get svc elasticsearch
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
elasticsearch   ClusterIP   10.100.35.143   <none>        9200/TCP   8m
```

From any host on the Kubernetes cluster (that's running `kube-proxy` or similar), run:

```
$ curl http://10.100.35.143:9200
```

One should see something similar to the following:

```json
{
  "name" : "es-client-864b45c866-68f45",
  "cluster_name" : "myesdb",
  "cluster_uuid" : "hMCMXG7hQSOpZ6BNu-sO_Q",
  "version" : {
    "number" : "6.1.0",
    "build_hash" : "c0c1ba0",
    "build_date" : "2017-12-12T12:32:54.550Z",
    "build_snapshot" : false,
    "lucene_version" : "7.1.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

Or if one wants to see cluster information:

```
$ curl http://10.100.35.143:9200/_cluster/health?pretty
```

One should see something similar to the following:

```json
{
  "cluster_name" : "myesdb",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 7,
  "number_of_data_nodes" : 2,
  "active_primary_shards" : 0,
  "active_shards" : 0,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```
<a id="pod-anti-affinity">

## Pod anti-affinity

One of the main advantages of running Elasticsearch on top of Kubernetes is how resilient the cluster becomes, particularly during
node restarts. However if all data pods are scheduled onto the same node(s), this advantage decreases significantly and may even
result in no data pods being available.

It is then **highly recommended**, in the context of the solution described in this repository, that one adopts [pod anti-affinity](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#inter-pod-affinity-and-anti-affinity-beta-feature)
in order to guarantee that two data pods will never run on the same node.

Here's an example:
```yaml
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: role
              operator: In
              values:
              - data
          topologyKey: kubernetes.io/hostname
  containers:
  - (...)
```

<a id="availability">

## Availability

If one wants to ensure that no more than `n` Elasticsearch nodes will be unavailable at a time, one can optionally (change and) apply the following manifests:
```
kubectl create -f es-master-pdb.yaml
kubectl create -f es-data-pdb.yaml
```

**Note:** This is an advanced subject and one should only put it in practice if one understands clearly what it means both in the Kubernetes and Elasticsearch contexts. For more information, please consult [Pod Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions).

<a id="helm">

## Deploy with Helm

[Helm](https://github.com/kubernetes/helm) charts for a basic (non-stateful) ElasticSearch deployment are maintained at https://github.com/clockworksoul/helm-elasticsearch. With Helm properly installed and configured, standing up a complete cluster is almost trivial:

```
$ git clone https://github.com/clockworksoul/helm-elasticsearch.git
$ helm install helm-elasticsearch
```

<a id="plugins">

## Install plug-ins

The image used in this repo is very minimalist. However, one can install additional plug-ins at will by simply specifying the `ES_PLUGINS_INSTALL` environment variable in the desired pod descriptors. For instance, to install [Google Cloud Storage](https://www.elastic.co/guide/en/elasticsearch/plugins/current/repository-gcs.html) and [S3](https://www.elastic.co/guide/en/elasticsearch/plugins/current/repository-s3.html) plug-ins it would be like follows:
```yaml
- name: "ES_PLUGINS_INSTALL"
  value: "repository-gcs,repository-s3"
```

**Note:** The X-Pack plugin does not currently work with the `quay.io/pires/docker-elasticsearch-kubernetes` image. See Issue #102

<a id="curator">

## Clean-up with Curator

Additionally, one can run a [CronJob](http://kubernetes.io/docs/user-guide/cron-jobs/) that will periodically run [Curator](https://github.com/elastic/curator) to clean up indices (or do other actions on the Elasticsearch cluster).

```
kubectl create -f es-curator-config.yaml
```

Kubernetes 1.7:
```
kubectl create -f es-curator_v2alpha1.yaml
```

Kubernetes 1.8:
```
kubectl create -f es-curator_v1beta1.yaml
```

Please, confirm the job has been created.

```
$ kubectl get cronjobs
NAME      SCHEDULE    SUSPEND   ACTIVE    LAST-SCHEDULE
curator   1 0 * * *   False     0         <none>
```

The job is configured to run once a day at _1 minute past midnight and delete indices that are older than 3 days_.

**Notes**

- One can change the schedule by editing the cron notation in `es-curator.yaml`.
- One can change the action (e.g. delete older than 3 days) by editing the `es-curator-config.yaml`.
- The definition of the `action_file.yaml` is quite self-explaining for simple set-ups. For more advanced configuration options, please consult the [Curator Documentation](https://www.elastic.co/guide/en/elasticsearch/client/curator/current/index.html).

If one wants to remove the curator job, just run:

```
kubectl delete cronjob curator
kubectl delete configmap curator-config
```

Various parameters of the cluster, including replica count and memory allocations, can be adjusted by editing the `helm-elasticsearch/values.yaml` file. For information about Helm, please consult the [complete Helm documentation](https://github.com/kubernetes/helm/blob/master/docs/index.md).

<a id="kibana>

## Kibana

**ATTENTION**: This is community supported so it most probably is out-of-date.

Additionally, one can also add Kibana to the mix. In order to do so, one must use a container image of Kibana without x-pack,
as it's not supported by the Elasticsearch container images used in this repository.

An image is already provided but one can build their own like follows:

```
FROM docker.elastic.co/kibana/kibana:5.5.1
RUN bin/kibana-plugin remove x-pack
```

If ones does provide their own image, one must make sure to alter the following files before deploying:

```
kubectl create -f kibana.yaml
kubectl create -f kibana-svc.yaml
```

Kibana will be available through service `kibana`, and one will be able to access it from within the cluster or
proxy it through the Kubernetes API Server, as follows:

```
https://<API_SERVER_URL>/api/v1/proxy/namespaces/default/services/kibana/proxy
```

One can also create an Ingress to expose the service publicly or simply use the service nodeport.
In the case one proceeds to do so, one must change the environment variable `SERVER_BASEPATH` to the match their environment.

## FAQ

### Why does `NUMBER_OF_MASTERS` differ from number of master-replicas?
The default value for this environment variable is 2, meaning a cluster will need a minimum of 2 master nodes to operate. If a cluster has 3 masters and one dies, the cluster still works. Minimum master nodes are usually `n/2 + 1`, where `n` is the number of master nodes in a cluster. If a cluster has 5 master nodes, one should have a minimum of 3, less than that and the cluster _stops_. If one scales the number of masters, make sure to update the minimum number of master nodes through the Elasticsearch API as setting environment variable will only work on cluster setup. More info: https://www.elastic.co/guide/en/elasticsearch/guide/1.x/_important_configuration_changes.html#_minimum_master_nodes


### How can I customize `elasticsearch.yaml`?
Read a different config file by settings env var `path.conf=/path/to/my/config/`. Another option would be to build one's own image from  [this repository](https://github.com/pires/docker-elasticsearch-kubernetes)

## Troubleshooting

### No up-and-running site-local

One of the errors one may come across when running the setup is the following error:
```
[2016-11-29T01:28:36,515][WARN ][o.e.b.ElasticsearchUncaughtExceptionHandler] [] uncaught exception in thread [main]
org.elasticsearch.bootstrap.StartupException: java.lang.IllegalArgumentException: No up-and-running site-local (private) addresses found, got [name:lo (lo), name:eth0 (eth0)]
	at org.elasticsearch.bootstrap.Elasticsearch.init(Elasticsearch.java:116) ~[elasticsearch-5.0.1.jar:5.0.1]
	at org.elasticsearch.bootstrap.Elasticsearch.execute(Elasticsearch.java:103) ~[elasticsearch-5.0.1.jar:5.0.1]
	at org.elasticsearch.cli.SettingCommand.execute(SettingCommand.java:54) ~[elasticsearch-5.0.1.jar:5.0.1]
	at org.elasticsearch.cli.Command.mainWithoutErrorHandling(Command.java:96) ~[elasticsearch-5.0.1.jar:5.0.1]
	at org.elasticsearch.cli.Command.main(Command.java:62) ~[elasticsearch-5.0.1.jar:5.0.1]
	at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:80) ~[elasticsearch-5.0.1.jar:5.0.1]
	at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:73) ~[elasticsearch-5.0.1.jar:5.0.1]
Caused by: java.lang.IllegalArgumentException: No up-and-running site-local (private) addresses found, got [name:lo (lo), name:eth0 (eth0)]
	at org.elasticsearch.common.network.NetworkUtils.getSiteLocalAddresses(NetworkUtils.java:187) ~[elasticsearch-5.0.1.jar:5.0.1]
	at org.elasticsearch.common.network.NetworkService.resolveInternal(NetworkService.java:246) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.common.network.NetworkService.resolveInetAddresses(NetworkService.java:220) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.common.network.NetworkService.resolveBindHostAddresses(NetworkService.java:130) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.transport.TcpTransport.bindServer(TcpTransport.java:575) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.transport.netty4.Netty4Transport.doStart(Netty4Transport.java:182) ~[?:?]
 	at org.elasticsearch.common.component.AbstractLifecycleComponent.start(AbstractLifecycleComponent.java:68) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.transport.TransportService.doStart(TransportService.java:182) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.common.component.AbstractLifecycleComponent.start(AbstractLifecycleComponent.java:68) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.node.Node.start(Node.java:525) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.bootstrap.Bootstrap.start(Bootstrap.java:211) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.bootstrap.Bootstrap.init(Bootstrap.java:288) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.bootstrap.Elasticsearch.init(Elasticsearch.java:112) ~[elasticsearch-5.0.1.jar:5.0.1]
 	... 6 more
[2016-11-29T01:28:37,448][INFO ][o.e.n.Node               ] [kIEYQSE] stopping ...
[2016-11-29T01:28:37,451][INFO ][o.e.n.Node               ] [kIEYQSE] stopped
[2016-11-29T01:28:37,452][INFO ][o.e.n.Node               ] [kIEYQSE] closing ...
[2016-11-29T01:28:37,464][INFO ][o.e.n.Node               ] [kIEYQSE] closed
```

This is related to how the container binds to network ports (defaults to ``_local_``). It will need to match the actual node network interface name, which depends on what OS and infrastructure provider one uses. For instance, if the primary interface on the node is `p1p1` then that is the value that needs to be set for the `NETWORK_HOST` environment variable.
Please see [the documentation](https://github.com/pires/docker-elasticsearch#environment-variables) for reference of options.

In order to workaround this, set `NETWORK_HOST` environment variable in the pod descriptors as follows:
```yaml
- name: "NETWORK_HOST"
  value: "_eth0_" #_p1p1_ if interface name is p1p1, _ens4_ if interface name is ens4, and so on.
```

### (IPv6) org.elasticsearch.bootstrap.StartupException: BindTransportException

Intermittent failures occur when the local network interface has both IPv4 and IPv6 addresses, and Elasticsearch tries to bind to the IPv6 address first.
If the IPv4 address is chosen first, Elasticsearch starts correctly.

In order to workaround this, set `NETWORK_HOST` environment variable in the pod descriptors as follows:
```yaml
- name: "NETWORK_HOST"
  value: "_eth0:ipv4_" #_p1p1:ipv4_ if interface name is p1p1, _ens4:ipv4_ if interface name is ens4, and so on.
```
