# Slinky and Slurm on OpenShift: A Step-by-Step Tutorial

This tutorial walks you through deploying [Slinky](https://www.schedmd.com/slinky/why-slinky/) (the Slurm operator for Kubernetes) on OpenShift, installing a Slurm cluster with shared storage, and running your first jobs.

Inspired by and credits to:
- [Slinky on OpenShift](https://github.com/redhat-hpc/slinky-on-openshift)
- [Intro to Slinky (Slurm on Kubernetes!)](https://youtu.be/YBPVzde3glw?si=ljOoJSj0aTD7Mk1k)
- [Slurm Introduction (Jobs, Partitions, Nodes, and Concepts)](https://youtu.be/hhIlmi_6E7U?si=fHnznymcJfVUQkvA)

## Prerequisites

- An OpenShift cluster with cluster-admin access
- `oc` CLI installed and configured
- `helm` CLI installed
- An SSH key pair (e.g., `~/.ssh/id_ed25519`)

## Step 1: Install the Slinky Operator

Install the Slurm operator from the OpenShift OperatorHub:

1. Open the OpenShift web console
2. Navigate to **Operators > OperatorHub**
3. Search for **Slurm**
4. Click **Install** and follow the prompts

Once installed, verify the operator is running:

```bash
oc get pods -n slinky
```

You should see output similar to:

```
NAME                                      READY   STATUS    RESTARTS   AGE
slurm-operator-dcff7f56c-5x4gq            1/1     Running   0          16m
slurm-operator-webhook-65995754bf-k4rm4   1/1     Running   0          16m
```

## Step 2: Create the Slurm Namespace

Create a new project for the Slurm deployment and grant the privileged SecurityContextConstraint to the default service account:

```bash
oc adm new-project slurm
oc adm policy add-scc-to-user privileged -n slurm -z default
```

> **Note:** The privileged SCC is used for simplicity. Future versions of Slinky plan to provide a more restrictive SCC.

## Step 3: Create Shared Storage

Before installing Slurm, create a PersistentVolumeClaim for shared home directories. This allows jobs running on different worker nodes to access the same files.

```bash
cat << EOF | oc create -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-home
  namespace: slurm
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: coe-netapp-nas
  volumeMode: Filesystem
EOF
```

> **Note:** Adjust `storageClassName` to match your cluster's shared storage provider (e.g., CephFS, NFS).

## Step 4: Install Slurm with Helm

Deploy Slurm using the Helm chart with CentOS Stream 9 / OpenHPC images and shared storage:

> **Note:** Adjust `rootSshAuthorizedKeys` to match your SSH public key.

```bash
helm install slurm oci://ghcr.io/slinkyproject/charts/slurm --namespace=slurm \
  --version 1.0.1 \
  --set configFiles.gres\\.conf="AutoDetect=nvidia" \
  --set loginsets.slinky.enabled=true \
  --set loginsets.slinky.login.securityContext.privileged=true \
  --set controller.slurmctld.image.repository=quay.io/slinky-on-openshift/slurmctld \
  --set controller.slurmctld.image.tag=25.11.1-centos9-ohpc \
  --set controller.reconfigure.image.repository=quay.io/slinky-on-openshift/slurmctld \
  --set controller.reconfigure.image.tag=25.11.1-centos9-ohpc \
  --set restapi.slurmrestd.image.repository=quay.io/slinky-on-openshift/slurmrestd \
  --set restapi.slurmrestd.image.tag=25.11.1-centos9-ohpc \
  --set accounting.slurmdbd.image.repository=quay.io/slinky-on-openshift/slurmdbd \
  --set accounting.slurmdbd.image.tag=25.11.1-centos9-ohpc \
  --set loginsets.slinky.login.image.repository=quay.io/slinky-on-openshift/login \
  --set loginsets.slinky.login.image.tag=25.11.1-centos9-ohpc \
  --set nodesets.slinky.slurmd.image.repository=quay.io/slinky-on-openshift/slurmd \
  --set nodesets.slinky.slurmd.image.tag=25.11.1-centos9-ohpc \
  --set nodesets.slinky.replicas=3 \
  --set-literal loginsets.slinky.rootSshAuthorizedKeys="$(cat $HOME/.ssh/id_ed25519.pub)" \
  --set-json 'loginsets.slinky.login.volumeMounts=[{"name":"shared-home","mountPath":"/home"}]' \
  --set-json 'loginsets.slinky.podSpec.volumes=[{"name":"shared-home","persistentVolumeClaim":{"claimName":"shared-home"}}]' \
  --set-json 'nodesets.slinky.slurmd.volumeMounts=[{"name":"shared-home","mountPath":"/home"}]' \
  --set-json 'nodesets.slinky.podSpec.volumes=[{"name":"shared-home","persistentVolumeClaim":{"claimName":"shared-home"}}]'
```

> **Tip:** The full list of Helm chart options is available in the [upstream values.yaml](https://github.com/SlinkyProject/slurm-operator/blob/release-1.0/helm/slurm/values.yaml).

## Step 5: Verify the Deployment

Watch the pods come up:

```bash
oc get pods -n slurm
```

Wait until all pods show `Running` status:
(it could take a few minutes)
```
NAME                                 READY   STATUS    RESTARTS      AGE
slurm-controller-0                   3/3     Running   0             5m32s
slurm-login-slinky-559d654f8-dmgpv   1/1     Running   0             5m32s
slurm-restapi-5c568d886b-cdzg7       1/1     Running   0             5m32s
slurm-worker-slinky-0                2/2     Running   0             5m32s
slurm-worker-slinky-1                2/2     Running   3 (50s ago)   5m32s
slurm-worker-slinky-2                2/2     Running   3 (28s ago)   5m32s
```

You can also view all resources in the namespace:

```bash
oc get all -n slurm
```

This shows pods, services, deployments, replica sets, and stateful sets that make up the Slurm cluster.
E.g.,
```
NAME                                     READY   STATUS    RESTARTS        AGE
pod/slurm-controller-0                   3/3     Running   0               7m24s
pod/slurm-login-slinky-559d654f8-dmgpv   1/1     Running   0               7m24s
pod/slurm-restapi-5c568d886b-cdzg7       1/1     Running   0               7m24s
pod/slurm-worker-slinky-0                2/2     Running   0               7m24s
pod/slurm-worker-slinky-1                2/2     Running   3 (2m42s ago)   7m24s
pod/slurm-worker-slinky-2                2/2     Running   3 (2m20s ago)   7m24s

NAME                          TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)        AGE
service/slurm-controller      ClusterIP      172.30.211.252   <none>         6817/TCP       7m27s
service/slurm-login-slinky    LoadBalancer   172.30.158.147   10.32.98.111   22:30864/TCP   7m27s
service/slurm-restapi         ClusterIP      172.30.56.7      <none>         6820/TCP       7m27s
service/slurm-workers-slurm   ClusterIP      None             <none>         6818/TCP       7m27s

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/slurm-login-slinky   1/1     1            1           7m27s
deployment.apps/slurm-restapi        1/1     1            1           7m27s

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/slurm-login-slinky-559d654f8   1         1         1       7m27s
replicaset.apps/slurm-restapi-5c568d886b       1         1         1       7m27s

NAME                                READY   AGE
statefulset.apps/slurm-controller   1/1     7m27s

```


## Step 6: Connect to the Slurm Login Node

SSH into the login pod using `oc exec` as a proxy:

```bash
ssh -o ProxyCommand='oc exec -i -n slurm svc/%h -- socat STDIO TCP:localhost:22' root@slurm-login-slinky
```

> **Note:** Accept the host key fingerprint when prompted. This uses the SSH key you provided during the Helm install.

Alternatively, you can access the Slurm controller directly:

```bash
oc -n slurm exec -it statefulsets/slurm-controller -- bash --login
```

## Step 7: Explore the Cluster with Slurm Commands

Once connected, check the cluster status:

### View node and partition info

```bash
sinfo
```

Output:

```
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
slinky       up   infinite      3   idle slinky-[0-2]
all*         up   infinite      3   idle slinky-[0-2]
```

Key concepts:
- **Partitions** are logical groupings of nodes (similar to queues). The `all` partition (marked with `*`) is the default.
- **Nodes** are the worker pods (`slinky-0`, `slinky-1`, `slinky-2`).
- **State** shows the current status of each node (`idle`, `alloc`, `mix`, `down`).

### Run an interactive command

Use `srun` to run a command on a worker node:

```bash
srun -n 1 -t 1:00 hostname
```

This runs `hostname` on one node with a 1-minute time limit. Output:

```
slinky-1
```

## Step 8: Submit Batch Jobs

### Simple batch job

Submit a background job with `sbatch`:

```bash
sbatch --wrap="sleep 60"
```

Output:

```
Submitted batch job 3
```

### Check the job queue

```bash
squeue
```

Output:

```
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
                 3       all     wrap    slurm  R       0:10      1 slinky-1
```

- **ST** = State (`R` = Running, `PD` = Pending)
- **TIME** = How long the job has been running
- **NODELIST** = Which node(s) the job is running on

### Verify the job is running on the worker

From outside the cluster, you can verify the job process on the worker pod:

```bash
oc -n slurm exec slurm-worker-slinky-1 -- ps -ef
```

You should see the job's processes (e.g., `slurmstepd`, `sleep`).

## Step 9: Write a Batch Script

Create a proper batch script instead of using `--wrap`. Save this as `hello.sh` on the login node:

```bash
cd /home
cat << 'EOF' > hello.sh
#!/bin/bash
#SBATCH --job-name=hello
#SBATCH --partition=all
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --output=hello_%j.out

echo "Hello from $(hostname) at $(date)"
sleep 66
echo "Job completed!"
EOF
```

Submit the script:

```bash
sbatch hello.sh
```

Monitor the job:

```bash
squeue
```

Once completed, check the output file:

```bash
cat hello_<jobid>.out
```

## Step 10: Uninstall

To remove the Slurm deployment:

```bash
helm uninstall slurm -n slurm
```

To remove the operator, use the OpenShift web console:

1. Navigate to **Operators > Installed Operators**
2. Find the Slurm operator
3. Click the menu and select **Uninstall Operator**

## Learn More

- [Slurm Overview](https://slurm.schedmd.com/overview.html)
- [Slurm Quickstart](https://slurm.schedmd.com/quickstart.html)
- [Slurm Documentation](https://slurm.schedmd.com/documentation.html)
- [Slinky Overview](https://www.schedmd.com/slinky/why-slinky/)
- [Slinky Documentation](https://slinky.schedmd.com)
- [Slinky on OpenShift](https://github.com/redhat-hpc/slinky-on-openshift)
