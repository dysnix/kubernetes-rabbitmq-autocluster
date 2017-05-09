# RabbitMQ cluster with Kubernetes Backend

There are several fairly easy steps to build your own cluster from scratch.  

While RabbitMQ can be clusterized, you still need to perform a series of operations to make this cluster work, also RabbitMQ does not have backend support out of the box. However, some developers already solved this by making a plugin which automatically builds RabbitMQ within the given infrastructure using some mechanisms (aws, k8s, dns, etc.) as a backends. Natively, this plugin does not support K8S integration, so we have to add it on the fly.  

## Building image  

It is OK to use official [RabbitMQ docker image](https://hub.docker.com/_/rabbitmq/) and manually add [autocluster plugin](https://github.com/rabbitmq/rabbitmq-autocluster) on build phase. Here's an example from Kubestack RabbitMQ:

```
FROM rabbitmq:latest
ENV RABBITMQ_USE_LONGNAME=true \
    AUTOCLUSTER_LOG_LEVEL=debug \
    AUTOCLUSTER_CLEANUP=true \
    CLEANUP_INTERVAL=60 \
    CLEANUP_WARN_ONLY=false \
    AUTOCLUSTER_TYPE=k8s \
    LANG=en_US.UTF-8
ADD plugins/*.ez /usr/lib/rabbitmq/lib/rabbitmq_server-3.6.8/plugins/
RUN rabbitmq-plugins enable --offline autocluster
```

You can also add whatever plugins you want by adding .ez files to {root}/plugins/ directory and they will be added to the image on build. Don't also forget to activate the plugin by running 

```
rabbitmq-plugins enable --offline plugin_name
```

On the build end as shown above.

Please pay attention to some vital Environmental Variables listed in the Dockerfile. You can either specify them during the build or later, during the YAML files compilation. Leaving them in Dockerfile will 'hardcode' these Variables into the image, but they later can be overwritten by YAML definitions on Kubernetes setup.

```
RABBITMQ_USE_LONGNAME=true — needs to be 'true', k8s backend depends on this option
AUTOCLUSTER_LOG_LEVEL=debug — better set to 'debug', because otherwise autocluster won't show any useful info about itself
AUTOCLUSTER_CLEANUP=true — this is a *must* and it *must* be set to true. otherwise your cluster will still storage info about dead / offline nodes marking them as working, this will result in traffic being sent to dead nodes
CLEANUP_INTERVAL=60 — how often do we clean the cluster from dead / offline nodes
CLEANUP_WARN_ONLY=false — if set to 'false', it really cleans up the cluster from dead / offline nodes
AUTOCLUSTER_TYPE=k8s — describes the backend used to control the cluster
LANG=en_US.UTF-8 — if not defined, kubectl logs will show no valid ASCII symbols on output
```

Full list of autocluster EnvVars can be found [here](https://github.com/aweber/rabbitmq-autocluster/wiki), and a full list of RabbitMQ-related EnvVars can be found [here](https://www.rabbitmq.com/configure.html#define-environment-variables).  
Your installation can be fully controlled by defining these during build or inside YAML files.

## Building plugin

Currently, a pre-built plugin is copied into the image during the build. You can you official pre-built plugin downloaded from https://github.com/aweber/rabbitmq-autocluster/releases but natively, autocluster doesn't support k8s backend, and adding it can be tricky.
We had to compile our own plugin from source. Building it requires Make 4.2 and Erlang to be installed on machine where it is being build. Here's an example:

```
apt-get install gcc make4 erlang erlang-devel -y
mkdir -p /usr/src/rabbitmq-autocluster; cd /usr/src/rabbitmq-autocluster
git clone https://github.com/aweber/rabbitmq-autocluster
make
make dist
```

The output file will be placed in plugins/autocluster.ez and is ready to be used with installation.

## Launching Installation

When image is already built and pushed to the registry and plugin is inside and activated, you're ready to launch your installation in Kubernetes. Currently, our image is built from [this source](https://git.arilot.com/containers/rabbitmq-cluster/) with Gitlab-Runner and all YAML files needed to run the installation are [here](https://git.arilot.com/arilot/kubernetes/tree/master/rabbitmq-autocluster) .

So, we don't have to do anything with the image, it is already in the registry, we only need YAML files to run from.

Clone repo:

```
git clone https://github.com/kuberstack/kubernetes-rabbitmq-autocluster.git
cd kubernetes-rabbitmq-autocluster
```

You need to be 100% sure that Service is up and running before launching deployment. So, lets create erlang.cookie secret first, it is used by nodes to communicate with each other:

```
echo $(openssl rand -base64 32) > erlang.cookie
kubectl -n $namespace create secret generic erlang.cookie --from-file=erlang.cookie
kubectl -n $namespace create -f manifestos/rabbitmq-service.yaml
```

After all services verified as up and running (you can check with kubectl get pods / kubectl get svc), run Deployment:

```
kubectl -n $namespace create -f manifestos/rabbitmq-deploy.yaml
```

Once everything is up you can check cluster status by running:

```
FIRST_POD=$(kubectl get pods -n $namespace -l 'app=rabbitmq' -o jsonpath='{.items[0].metadata.name }')
kubectl -n $namespace exec -ti $FIRST_POD rabbitmqctl cluster_status
```

This should return something like this showing at least N (where N = # of replicas in your Deployment YAML file) nodes joined cluster:

```
Cluster status of node 'rabbit@100.96.24.31' ...
[{nodes,[{disc,['rabbit@100.96.16.106','rabbit@100.96.24.31',
                'rabbit@100.96.9.117']}]},
 {running_nodes,['rabbit@100.96.9.117','rabbit@100.96.16.106',
                 'rabbit@100.96.24.31']},
 {cluster_name,<<"rabbit@rabbitmq-654759146-r01n3">>},
 {partitions,[]},
 {alarms,[{'rabbit@100.96.9.117',[]},
          {'rabbit@100.96.16.106',[]},
          {'rabbit@100.96.24.31',[]}]}]
```
		  
## Common Issues

```
— =ERROR REPORT==== 16-Mar-2017::18:46:38 ===…autocluster: Unsupported backend: k8s.
```

check your autocluster plugin and whether it is built with K8S support

```
— =INFO REPORT==== 20-Mar-2017::18:44:10 ===
autocluster: Failed to get nodes from k8s - 404

=ERROR REPORT==== 20-Mar-2017::18:44:10 ===
autocluster: Could not fetch node list from autocluster_k8s: "404".
```

check if your Service is up and running, also accessible. also double-check all exposed network ports inside YAML file.

```
=ERROR REPORT==== 20-Mar-2017::18:03:19 ===
autocluster: Can not communicate with cluster nodes.

=ERROR REPORT==== 20-Mar-2017::18:03:20 ===
** Connection attempt from disallowed node 'rabbit@100.96.11.146' **
```

check your erlang.cookie and whether it is defined or not, and data accessible from secret.
