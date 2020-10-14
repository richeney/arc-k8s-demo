# Demo Instructions

Demo flow roughly follows <https://docs.microsoft.com/azure/azure-arc/kubernetes/connect-cluster>.

## Pre-req info

* Using the richeney@azurecitadel.com userid into the VS subscription.
* Microsoft.Kubernetes and Microsoft.KubernetesConfiguration are registered.
* The CLI has the connectedk8s and k8sconfiguration extensions.

[Kind](https://kind.sigs.k8s.io/) has been installed in WSL.

The <https://github.com/azure/arc-k8s-demo> repo has been forked as <https://github.com/richeney/arc-k8s-demo>.

## Demo

1. Start kind

    Start up kind in WSL:

    ```bash
    sudo service docker start
    GO111MODULE="on" go get sigs.k8s.io/kind@v0.9.0
    kind create cluster
    ```

    kind will be installed to `~/go/bin` - added to the PATH

1. Docker error

    If you get "docker: Error response from daemon: cgroups: cannot find cgroup mount destination: unknown."

    ```bash
    sudo mkdir /sys/fs/cgroup/systemd
    sudo mount -t cgroup -o none,name=systemd cgroup /sys/fs/cgroup/systemd
    ```

    Then rerun step 1.

3. Create the cluster

    ```bash
    kind create cluster
    ```

    Expected output:

    ```text
    Creating cluster "kind" ...
     ✓ Ensuring node image (kindest/node:v1.19.1) 🖼
     ✓ Preparing nodes 📦
     ✓ Writing configuration 📜
     ✓ Starting control-plane 🕹️
     ✓ Installing CNI 🔌
     ✓ Installing StorageClass 💾
    Set kubectl context to "kind-kind"
    You can now use your cluster with:

    kubectl cluster-info --context kind-kind

    Thanks for using kind! 😊
    ```

1. Check cluster

    ```bash
    kubectl cluster-info --context kind-kind
    ```

1. Check the node

    ```bash
    kubectl get nodes
    ```

    ```text
    NAME                 STATUS   ROLES    AGE   VERSION
    kind-control-plane   Ready    master   79m   v1.19.1
    ```

1. Connect the cluster

    Make sure the resource group is empty in the portal before starting.

    ```bash
    az connectedk8s connect --name AzureArcTest01 --resource-group AzureArcTest --tags kubernetes=kind platform=wsl
    ```

    This takes a few minutes to create the resource and the azure-arc namespace using Helm v3.

1. Show the azure-arc namespace

    ```bash
    kubectl -n azure-arc get deployments,pods
    ```

    Expected output:

    ```text
    NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/cluster-metadata-operator   1/1     1            1           8m13s
    deployment.apps/clusteridentityoperator     1/1     1            1           8m13s
    deployment.apps/config-agent                1/1     1            1           8m13s
    deployment.apps/controller-manager          1/1     1            1           8m13s
    deployment.apps/flux-logs-agent             1/1     1            1           8m13s
    deployment.apps/metrics-agent               1/1     1            1           8m13s
    deployment.apps/resource-sync-agent         1/1     1            1           8m13s

    NAME                                             READY   STATUS    RESTARTS   AGE
    pod/cluster-metadata-operator-6699fbcd6c-ws2j4   2/2     Running   0          8m13s
    pod/clusteridentityoperator-7b56768b44-j46js     3/3     Running   0          8m13s
    pod/config-agent-75fd98c7d8-gz7qz                3/3     Running   0          8m13s
    pod/controller-manager-85947c8c84-cgj6g          3/3     Running   0          8m13s
    pod/flux-logs-agent-677845c757-tvzmw             2/2     Running   0          8m13s
    pod/metrics-agent-599c898cdf-5kjr6               2/2     Running   0          8m13s
    pod/resource-sync-agent-696f479b64-m9w96         3/3     Running   0          8m13s
    ```

1. Show the portal

    * tagging etc.
    * configs
        * cluster-config
        * namespace
    * policy

1. Use the arc-k8s-demo repo for cluster-config

    | Namespaces | cluster-config, team-a, team-b |
    | Deployments | cluster-config/azure-vote |
    | ConfigMap | team-a/endpoints |

    ```bash
    az k8sconfiguration create \
        --name cluster-config \
        --cluster-name AzureArcTest01 --resource-group AzureArcTest \
        --operator-instance-name cluster-config --operator-namespace cluster-config \
        --repository-url https://github.com/richeney/arc-k8s-demo \
        --scope cluster --cluster-type connectedClusters --enable-helm-operator \
        --operator-params='--git-email richeney@microsoft.com --git-poll-interval 1m'
    ```

1. Check Settings / Configurations

    Allows multiples. Can also use private Git repos and paths within repos.

1. Show namespaces

    ```bash
    kubectl get namespaces --show-labels
    ```

    Example output:

    ```text
    NAME                 STATUS   AGE    LABELS
    azure-arc            Active   54m    admission.policy.azure.com/ignore=true,control-plane=true
    cluster-config       Active   30m    <none>
    default              Active   137m   <none>
    itops                Active   29m    fluxcd.io/sync-gc-mark=sha256.s0hhEcxGyWcRUZRCAoFaLzqsWY9rYEp1oDTiC8OCdu8,name=itops
    kube-node-lease      Active   137m   <none>
    kube-public          Active   137m   <none>
    kube-system          Active   137m   <none>
    local-path-storage   Active   137m   <none>
    team-a               Active   29m    fluxcd.io/sync-gc-mark=sha256.AX5heFItzwbiMQg5wirmTBd2Il9RwMOATFLUsNphW3Y,name=team-a
    team-b               Active   29m    fluxcd.io/sync-gc-mark=sha256.l_GBFLpspcnDHRHp_jkMcVVXMXu50L_pvWyy7G-H-PI,name=team-b
    ```

1. Show the deployments in the cluster-config namespace

    ```bash
    kubectl -n cluster-config get deploy -o wide | cut -c1-132
    ```

    Example output:

    ```text
    NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS           IMAGES
                               SELECTOR
    cluster-config                                     1/1     1            1           32m   flux                 mcr.microsoft.com/oss/fluxcd/flux:1.20.0          instanceName=cluster-config,name=flux
    cluster-config-helm-cluster-config-helm-operator   0/1     1            0           21m   flux-helm-operator   docker.io/fluxcd/helm-operator:1.0.0-rc4          app=helm-operator,release=cluster-config-helm-cluster-config
    memcached                                          1/1     1            1           32m   memcached            mcr.microsoft.com/oss/memcached/memcached:1.6.6   name=memcached
    ```

    This is the perfect width when CTRL-0 on Surface Book at 200%, or 1080p with CTRL+= x3.

1. Helm demo

    **YOU ARE HERE**

    ```bash
    export RESOURCE_GROUP=AzureArcTest
    export CLUSTER_NAME=AzureArcTest01
    ```
