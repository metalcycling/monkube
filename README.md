# Monkube

A point-in-time view of what a Kubernetes cluster is running and where there is room left
to run more. One invocation replaces a handful of `kubectl get` / `kubectl describe` calls,
issues them concurrently, and prints the results colorized.

There are two independent views, usable separately or together:

- `-r/--resources` — `kubectl get` tables for any set of resources across any set of
  namespaces, with sorting, label filtering and line caps.
- `-nd/--nodes` — per-node cpu / gpu / memory allocation, plus the reasons a node cannot
  accept more work.

## Install

A single script; it needs `kubectl` on your `$PATH` and a context to talk to (the current
one, or any other via `-c/--context`).

```bash
chmod u+x monkube    # then drop it somewhere on your $PATH
pip install orjson   # optional, faster JSON parsing
```

## Usage

```bash
# Pods and services in two namespaces, 5 lines each
monkube -r pods,services -n default,kube-system -l 5

# Newest pods first, restricted by label, only those still Pending
monkube -r pods -s -t "app=trainer" -st Pending

# Allocation for every GPU node
monkube -nd full -go

# Names, ages and status only
monkube -nd reduced

# Just the nodes something is wrong with
monkube -nd reduced -uo

# Two specific nodes
monkube -tn worker-01,worker-02

# Any of the above against another cluster, without switching contexts
monkube -c staging-cluster -nd reduced -uo
```

Without `-c`, the current context is used; `-c` applies to every query the run makes and
leaves `kubectl config current-context` untouched, so two terminals can watch two clusters.
An unknown context is rejected up front, listing the ones that exist.

Without `-n`, the namespace of the context in effect is used (or `default` if it has none).
Given none of `-r`, `-nd` or `-tn`, the help text is printed.

## Options

| Flag | Description |
| --- | --- |
| `-r`, `--resources` | Comma-separated resources to list (`pods,services,…`) |
| `-n`, `--namespaces` | Comma-separated namespaces; defaults to that of the context in effect |
| `-l`, `--num_lines` | Cap the lines printed per resource |
| `-s`, `--sort` | Newest first, by creation timestamp |
| `-t`, `--labels` | Label selector, e.g. `k1=v1,k2='v2\|v3'` (`\|` matches any of) |
| `-st`, `--status` | Keep only pods in this phase; ignored for non-pod resources |
| `-nd`, `--nodes` | Node view: `full` (allocation) or `reduced` (name / age / status) |
| `-go`, `--gpu_only` | Only nodes advertising a GPU resource |
| `-co`, `--cpu_only` | Only nodes that do not |
| `-uo`, `--unhealthy_only` | Only nodes carrying a marker below: cordoned, blocking taint, not `Ready`, or unhealthy GPUs |
| `-ln`, `--num_nodes` | Cap the number of nodes printed |
| `-tn`, `--target_nodes` | Restrict to these nodes; without `-nd`, queries just them |
| `-c`, `--context` | Kubeconfig context to query; defaults to the current one |
| `-j`, `--jobs` | Concurrent `kubectl` workers (default 16) |

## Reading the node view

`-nd full` reports `requested / allocatable (percent)` per node:

```
NAME: worker-01
    cpu:  38348.00 / 223800.00 ( 17.13%)
    gpu:      7.00 /      8.00 ( 87.50%)
    mem:   1083.63 /   1999.55 ( 54.19%)
```

- **Requested, not consumed.** The numerator sums the resource *requests* of the
  containers of every `Running` or `Pending` pod on the node. It answers "what will the
  scheduler still fit here?", not "how busy is this hardware?" — a process pinning a GPU
  at 100% without requesting it contributes nothing.
- **Units.** cpu in millicores, memory in Gi, gpu in devices, keeping fractional values.
  Both `nvidia.com/gpu` and `amd.com/gpu` are recognized, and a node counts as a GPU node
  when it advertises either key at all, even at zero.

Idle is not the same as available, so nodes are annotated with whatever blocks them:

| Marker | Meaning |
| --- | --- |
| `[cordoned]` | `spec.unschedulable` is set; the redundant cordon taint is not repeated |
| `[NotReady]`, `[Ready=Unknown]` | The node's `Ready` condition |
| `[key=value:Effect]` | One per `NoSchedule` / `NoExecute` taint; `PreferNoSchedule` is omitted, as it does not block |
| `[N unhealthy]` | GPUs the node reports but that cannot be scheduled onto |

That last marker exists because kubelet publishes device-plugin resources with capacity =
every device the plugin reported and allocatable = only those it currently considers
healthy. A node whose plugin has withdrawn GPUs therefore reports a *smaller but idle* GPU
count, which is otherwise indistinguishable from a healthy node of that size:

```
NAME: worker-02  [cordoned]
    gpu:      0.00 /      8.00 (  0.00%)

NAME: worker-03
    gpu:      0.00 /      4.00 (  0.00%)  [4 unhealthy]
```

Neither node can take an 8-GPU pod, though both look free without the markers.
`-nd reduced` carries the same annotations in a `STATUS` column, and `-uo/--unhealthy_only`
drops every node that carries no marker at all, leaving just the ones to look into.

## Performance

Independent `kubectl` queries run concurrently on a shared thread pool sized by
`-j/--jobs`, and JSON is handed to [`orjson`](https://github.com/ijl/orjson) as raw bytes.
`orjson` is optional — without it the standard-library `json` module is used instead.

The one non-obvious trick is in `-nd full`, which needs every pod in the cluster: it issues
one request per namespace in parallel rather than a single `--all-namespaces` call. The
apiserver is bound by per-stream throughput, so splitting the stream is several times
faster on a large cluster (~19s → ~6s at 253 nodes / 9300 pods).

Each run closes with the time spent querying the cluster versus parsing the responses.
