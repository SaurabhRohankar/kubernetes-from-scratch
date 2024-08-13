# Kubernetes From Scratch: Building Kubelet

## Table of Contents
- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Step 1: Install Container Runtime Interface (CRI)](#step-1-install-container-runtime-interface-cri)
- [Step 2: Installing and Configuring Kubelet](#step-2-installing-and-configuring-kubelet)
- [Step 3: Creating a Test Pod](#step-3-creating-a-test-pod)
- [Step 4: Installing CNI Plugins and Resolving Network Errors](#step-4-installing-cni-plugins-and-resolving-network-errors)
- [Step 5: Verifying Kubelet Operation](#step-5-verifying-kubelet-operation)
- [Troubleshooting](#troubleshooting)
- [Additional Resources](#additional-resources)
- [Conclusion](#conclusion)

## Introduction

In this guide, we'll set up kubelet, the primary component in Kubernetes from scratch. We'll configure the kubelet to run in standalone mode. 
### What is Kubelet?

- Kubelet ensures that pods are running and their current state matches their desired state.
- It runs on a worker node in kubernetes cluster.
- Acts as a watchman, monitoring all pods running on the node.
- In a full-fledged K8s cluster, it takes commands from kube-apiserver. For this part, we'll configure it to watch a specific directory on our worker node for pods since we don't have the kube-apiserver yet.

## Prerequisites

Before beginning, ensure you have:
- A Linux environment (this guide uses Arch Linux)
- Root or sudo access
- Basic understanding of Kubernetes concepts
- Go programming language installed (for CNI plugins)
- jq installed

## Step 1: Install Container Runtime Interface (CRI)

Kubernetes is a Container orchestration platform so it needs containers and for the containers to run we need some sort of runtime known as Container Runtime Interface (CRI). 
There are many runtimes available in the market like docker, cri-o, containerd, etc.  We'll use containerd as our CRI.
### Why Containerd and not Docker?

Kubernetes removed dockershim support in version 1.24. Using Docker as a runtime now requires an additional tool (cri-dockerd) to make it OCI compliant, which adds overhead. Containerd offers a more streamlined solution.

Install containerd using your system's package manager.

For Arch Linux:

```bash
sudo pacman -S containerd
sudo systemctl start containerd
sudo systemctl enable containerd
```

## Step 2: Installing and Configuring Kubelet

1. Download the kubelet binary:

```bash
wget https://dl.k8s.io/v1.30.3/bin/linux/amd64/kubelet
chmod +x kubelet
sudo mv kubelet /usr/local/bin
```

2. Let's try to run the kubelet now and see what happens.

```bash
sudo kubelet
```

You might see error similar to this

```
I0728 08:51:21.995280     848 server.go:484] "Kubelet version" kubeletVersion="v1.30.3"
I0728 08:51:21.995391     848 server.go:486] "Golang settings" GOGC="" GOMAXPROCS="" GOTRACEBACK=""
I0728 08:51:21.995530     848 server.go:647] "Standalone mode, no API client"
I0728 08:51:22.004845     848 server.go:535] "No api server defined - no events will be sent to API server"
I0728 08:51:22.005085     848 server.go:742] "--cgroups-per-qos enabled, but --cgroup-root was not specified.  defaulting to /"
E0728 08:51:22.005202     848 run.go:74] "command failed" err="failed to run Kubelet: running with swap on is not supported, please disable swap! or set --fail-swap-on flag to false. /proc/swaps contained: [Filename\t\t\t\tType\t\tSize\t\tUsed\t\tPriority /dev/zram0                              partition\t988668\t\t0\t\t100]"
```


 **Swap Enabled Error**: 
   Kubernetes, by default, doesn't support running with swap enabled for performance and consistency reasons. You have two options to resolve this:
   - Disable swap on your system (recommended for production):
     ```bash
     sudo swapoff -a
     ```
   - Allow kubelet to run with swap enabled (for testing/development):
     We'll handle this in our configuration file.

Also we can see in the output that out kubelet is running in standalone mode so we also need to give it a static path. So let's create a config file for this which we can pass to the kubelet.

### Creating a Kubelet Configuration File

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
enableServer: false
staticPodPath: /home/saura/workspaces/kubernetes/kubernetes_from_scratch/kubelet-static-pod
readOnlyPort: 10250
failSwapOn: false
podCIDR: 10.1.0.0/24
authentication:
  anonymous:
    enabled: true
  webhook:
    enabled: false
authorization:
  mode: AlwaysAllow
containerRuntimeEndpoint: unix:///run/containerd/containerd.sock
```

This configuration:
- Sets `failSwapOn` to `false`, allowing kubelet to run with swap enabled
- Specifies a `staticPodPath` for kubelet to watch
- Configures basic authentication and authorization settings
- Sets the container runtime endpoint for containerd

3. Create the static pod directory:

> **Note:** Create a directory in your own workspace

```bash
mkdir -p $HOME/workspaces/kubernetes/kubernetes_from_scratch/kubelet-static-pod
```

4. Now let's run kubelet with new configuration:

```bash
sudo kubelet --config=/home/saura/workspaces/kubernetes/kubernetes_from_scratch/config/kubeletConfig.yaml
```

This should start kubelet without the previous errors. If you encounter any new issues, refer to the [Troubleshooting](#troubleshooting) section or consult the official Kubernetes documentation.

## Step 3: Creating a Test Pod

Create a YAML file for a simple nginx pod in the static pod directory:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-test-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

Now let's check if pod has been launched successfully or not. Since we don't have a kubectl install we'll need to curl the kubelet api directly.

Our kubelet is running on port 10250 which we had provided in kubeConfig.yaml file.

```
curl -s -k http://localhost:10250/pods | jq .
```

> **Note:** jq is a command line tool used for parsing and manipulating the json and it's pretty handy so install it if not already

We'll see that pod has been launched but it's is pending state. And if we check the kubelet logs we can clearly see the error as following.

```error
E0728 10:29:08.799347    1321 pod_workers.go:1298] "Error syncing pod, skipping" err="network is not ready: container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized" pod="default/nginx-test-pod" podUID="48d45a7696548b0097b95366bed2c2c2
```

Let's understand and resolve this issue
## Step 4: Installing CNI Plugins and Resolving Network Errors

The above error indicates that the Container Network Interface (CNI) plugins are not set up correctly. So what is CNI?

### Understanding CNI Plugins

CNI plugins provide fundamental networking capabilities for your pods, including:
- Pod-to-pod communication
- IP address assignment
- Network policy enforcement

### Installing CNI Plugins

> Install Go (if not already installed)

1. Clone the CNI plugins repository:

```bash
git clone https://github.com/containernetworking/plugins.git
cd plugins
```

2. Build the plugins:

```bash
./build_linux.sh
```

3. Create the necessary directories and copy the plugins binaries:

```bash
sudo mkdir -p /opt/cni/bin
sudo cp bin/* /opt/cni/bin/
```

### Configuring CNI Plugins

Next, we need to configure which CNI plugins kubelet and in turn containerd will use:

1. Create the configuration directory:

```bash
sudo mkdir -p /etc/cni/net.d
```

2. Create a configuration file for the bridge plugin:

```bash
sudo tee /etc/cni/net.d/10-mynet.conf <<EOF
{
    "cniVersion": "0.2.0",
    "name": "mynet",
    "type": "bridge",
    "bridge": "cni0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "subnet": "10.1.0.0/24",
        "routes": [
            { "dst": "0.0.0.0/0" }
        ]
    }
}
EOF
```

3. Create a configuration file for the loopback plugin:

```bash
sudo tee /etc/cni/net.d/99-loopback.conf <<EOF
{
    "cniVersion": "0.2.0",
    "name": "lo",
    "type": "loopback"
}
EOF
```

After setting up the CNI plugins, let's restart the containerd and rerun our kubelet.

```bash
sudo systemctl restart containerd
sudo kubelet --config=/home/saura/workspaces/kubernetes/kubernetes_from_scratch/config/kubeletConfig.yaml
```
## Step 5: Verifying Kubelet Operation

Now that we've addressed the initial errors, let's verify that kubelet is operating correctly.

### Checking Pod Status

Use the following command to check the status of your pods:

```bash
curl -s http://localhost:10250/pods | jq .items[].status.phase
```

If everything is set up correctly, you should see:

```
"Running"
```

### Verifying Containers in Containerd

To verify that the containers are running in containerd:

```bash
sudo ctr -n k8s.io c ls
```

You should see output similar to this:

```
CONTAINER                                                           IMAGE                             RUNTIME
1a90491255d09b8d153e505fd3ac903c467559644964a00c756510a41b3ad439    registry.k8s.io/pause:3.8         io.containerd.runc.v2
cfd7fe0abafc2be30e5266ef24088e5f44e655422fbb95752bf34abd1ff17254    docker.io/library/nginx:latest    io.containerd.runc.v2
```

### Testing Kubelet's Auto-healing Capability

To test kubelet's ability to automatically restart failed containers:

1. Find the container ID of your nginx container from the output of the previous command.

2. Manually stop the container:

```bash
sudo ctr -n k8s.io tasks kill <container-id>
sudo ctr -n k8s.io containers rm <container-id>
```

3. Check the pod status again after a few seconds:

```bash
curl -s http://localhost:10250/pods | jq .items[].status.containerStatuses[].state
```

You should see that a new container has been started automatically:

```json
{
  "running": {
    "startedAt": "2024-07-30T03:11:27Z"
  }
}
```

This demonstrates kubelet's auto-healing capability, ensuring that the desired state of your pods is maintained.


## Troubleshooting

If you encounter any issues during this process, here are some common problems and their solutions:

1. **CNI plugin not initialized**: Ensure that you've correctly installed and configured the CNI plugins as described above.

2. **Container runtime not ready**: Check if containerd is running with `systemctl status containerd`. If it's not running, start it with `sudo systemctl start containerd`.

3. **Permission denied errors**: Ensure you're running the commands with sudo or as the root user.

4. **Network connectivity issues**: Check your firewall settings and ensure that the necessary ports are open for kubelet and container networking.

5. **Swap-related errors**: If you encounter swap-related errors and don't want to disable swap, ensure that `failSwapOn: false` is set in your kubelet configuration.

If you continue to face issues, consult the [Kubernetes troubleshooting guide](https://kubernetes.io/docs/tasks/debug-application-cluster/troubleshooting/) or seek help from the Kubernetes community.

## Additional Resources

- [Kubernetes Official Documentation](https://kubernetes.io/docs/concepts/overview/components/#kubelet)
- [containerd Documentation](https://containerd.io/docs/)
- [CNI Specification](https://github.com/containernetworking/cni/blob/master/SPEC.md)
- [Kubelet Configuration](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)
## Conclusion

We've successfully set up a standalone kubelet, which required:

1. CRI (Containerd)
2. CNI Plugins
3. Kubelet Binary

This completes the first component of our Kubernetes cluster. In the next step, we'll explore kube-apiserver and configure kubelet to listen to it instead of watching a static directory.
