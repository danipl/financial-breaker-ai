> **Important:** This step is critical for enabling GPU access in your kubernetes and Jenkins agent. The configuration
> is highly dependent on your specific hardware and software setup. Don't expect this guide to work seamlessly in all
> environments, and investigate and adapt the steps according to your particular hardware and software configuration.

Host Preparation (Drivers & Toolkit)
====================================

For training Large Language Models (LLMs) or Small Language Models (SLMs), the Kubernetes cluster requires at least one
node equipped with LLM-training resources, primarily NVIDIA GPUs. Additionally, these GPUs must be accessible from pods
running in Kubernetes, particularly those executing compute-intensive training workloads.
The [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html#operator-install-guide)
is the Kubernetes operator that enables this functionality.

## Configure K3s (The "Hybrid" Template)

This section configures K3s's containerd runtime to support NVIDIA GPUs by creating a custom configuration template. The
process involves copying K3s's auto-generated containerd configuration and extending it with the NVIDIA container
runtime, allowing Kubernetes pods to access GPU resources for compute-intensive workloads like LLM/SLM training.

K3s uses a minimal containerd configuration that doesn't include GPU runtime support by default. Simply installing the
NVIDIA drivers and container toolkit is not enough—containerd must be explicitly configured to recognize and use the
NVIDIA runtime. This "hybrid" approach is critical because it preserves K3s's default networking (CNI) and core
functionality while adding GPU capabilities, ensuring both standard Kubernetes workloads and GPU-accelerated pods can
coexist seamlessly in the cluster.

We needed a custom `config.toml.tmpl` that preserves K3s default networking (CNI) while adding the NVIDIA runtime.

```bash
# 1. Copy the working auto-generated config to a template
sudo cp /var/lib/rancher/k3s/agent/etc/containerd/config.toml \
        /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl

# 2. Append the NVIDIA runtime configuration to the template
sudo bash -c 'cat <<EOF >> /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl

[plugins.cri.containerd.runtimes.nvidia]
  privileged_without_host_devices = false
  runtime_engine = ""
  runtime_root = ""
  runtime_type = "io.containerd.runc.v2"
  [plugins.cri.containerd.runtimes.nvidia.options]
    BinaryName = "/usr/bin/nvidia-container-runtime"
EOF'

# 3. Restart K3s to apply changes
sudo systemctl restart k3s
```

## Docker Check

Before proceeding with Kubernetes-specific deployments, it is essential to verify that the host's NVIDIA Container
Toolkit is correctly installed and functional.

Running the following command confirms that Docker can successfully interface with the GPU drivers and pass the hardware
through to a container.

> docker run --rm --gpus all nvidia/cuda:12.6.2-cudnn-runtime-ubuntu22.04 nvidia-smi

A successful check will display the nvidia-smi table showing your GPU model and driver version.