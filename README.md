# Monkube: Monitoring Kubernetes

This command-line utility lets you monitor different resources in different namespaces with ease

**SUGGESTION**: To make it easier to work with `monkube`, make the utility an executable (e.g., `$ chmod u+x monkube`) and add it to your `$PATH`.

## Usage

Check the `help` option of Monkube to find the list of options

```bash
$ monkube --help
usage: Monkube [-h] [-r RESOURCES] [-n NAMESPACES] [-l NUM_LINES] [-s] [-t LABELS] [-nd {full,reduced}] [-ln NUM_NODES]

Monitoring of kubernetes resources

options:
  -h, --help            show this help message and exit
  -r RESOURCES, --resources RESOURCES
                        Comma-separated list of resources to be monitored
  -n NAMESPACES, --namespaces NAMESPACES
                        Comma-separated list of namespaces to be monitored
  -l NUM_LINES, --num_lines NUM_LINES
                        Number of lines to print for each resource
  -s, --sort            Sort resources starting with the most recently used
  -t LABELS, --labels LABELS
                        Get subset of resources matching the specified comma-separated list of labels (e.g., "--labels key-1=value-1,key-2=value-2"
  -nd {full,reduced}, --nodes {full,reduced}
                        Show allocation and utilization of nodes
  -ln NUM_NODES, --num_nodes NUM_NODES
                        Show data for the first 'num_nodes'
```

## Example

Here is an example of how to use the utility

```bash
$ monkube --resources pods,services --namespaces default,kube-system --num_lines 5 --nodes full --num_nodes 3
------------------------------------- PODS: ------------------------------------

NAMESPACE: 'default'
No resources found in default namespace.

NAMESPACE: 'kube-system'
NAME                                                 READY   STATUS    RESTARTS      AGE
event-exporter-gke-5b8bcb44f7-rfc78                  2/2     Running   0             24d
filestore-node-4lvts                                 3/3     Running   0             7h
filestore-node-4njxs                                 3/3     Running   0             15d
filestore-node-6gzt2                                 3/3     Running   0             20d
filestore-node-8xqzh                                 3/3     Running   0             26d

----------------------------------- SERVICES: ----------------------------------

NAMESPACE: 'default'
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.56.0.1    <none>        443/TCP   134d

NAMESPACE: 'kube-system'
NAME             TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
kube-dns         ClusterIP   10.56.0.10    <none>        53/UDP,53/TCP   134d
metrics-server   ClusterIP   10.56.10.80   <none>        443/TCP         134d

-------------------------------- CLUSTER NODES: --------------------------------

NAME: gke-vantroid-default-pool-06aeef3b-46q9
    cpu:   911.00 /   940.00 ( 96.91%)
    gpu:     0.00 /     0.00 (  0.00%)
    mem:     1.33 /     2.84 ( 46.90%)

NAME: gke-vantroid-default-pool-06aeef3b-6748
    cpu:   893.00 /   940.00 ( 95.00%)
    gpu:     0.00 /     0.00 (  0.00%)
    mem:     1.21 /     2.84 ( 42.56%)

NAME: gke-vantroid-default-pool-06aeef3b-bgbz
    cpu:   891.00 /   940.00 ( 94.79%)
    gpu:     0.00 /     0.00 (  0.00%)
    mem:     1.12 /     2.84 ( 39.56%)

--------------------------------------------------------------------------------

   ETCD time: 1.59 s
Parsing time: 0.01 s
```
