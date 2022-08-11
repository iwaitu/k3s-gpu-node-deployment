# k3s-gpu-node-deployment
Non-Gpu node as k3s master, add another node with gpu to the cluster and make it work

## Install gpg key from nvidia
```
distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
    && curl -s -L https://nvidia.github.io/libnvidia-container/gpgkey | sudo apt-key add - \
    && curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```
## Install Drivers. Use the latest drivers!
```
apt-get update && apt-get install -y nvidia-driver-515-server nvidia-container-toolkit nvidia-modprobe
reboot
```

## Check whether GPU recognized. You might have to restart the node after the driver installation to get this working
```
nvidia-smi
```

## Download template from k3d project
```
sudo wget https://k3d.io/v5.4.1/usage/advanced/cuda/config.toml.tmpl -O /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl
```

## Install nvidia plugin. This is optional. You can simply pass in the env variables to pass in GPU access to pods. However this is a nice way to debug whether you have access to the GPUs on contianerd
```
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.12.2/nvidia-device-plugin.yml
```
## Try running nvidia plugin and check the logs. If you see this on the correct gpu node ,then it is work.
```
2022/08/11 17:22:55 Retreiving plugins.
2022/08/11 17:22:55 Starting GRPC server for 'nvidia.com/gpu'
2022/08/11 17:22:55 Starting to serve 'nvidia.com/gpu' on /var/lib/kubelet/device-plugins/nvidia-gpu.sock
2022/08/11 17:22:55 Registered device plugin for 'nvidia.com/gpu' with Kubelet
```
## but if you see this, you have to modify the .tmpl file further.
```
2022/07/25 18:14:19 Initializing NVML.
2022/07/25 18:14:19 Failed to initialize NVML: could not load NVML library.
2022/07/25 18:14:19 If this is a GPU node, did you set the docker default runtime to `nvidia`
```
1. This used to work previously but with K3S v1.23+ I had issues. You will have to modify /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl add:
```
[plugins.cri.containerd.runtimes.runc.options]
  BinaryName = "/usr/bin/nvidia-container-runtime"
```

2. and modify
```
[plugins.cri.containerd.runtimes.runc]
  runtime_type = "io.containerd.runtime.v1.linux"
```
to
```
[plugins.cri.containerd.runtimes.runc]
  runtime_type = "io.containerd.runc.v2"
```

This should get everything running. Then either use the nvidia plugin to define resources or if you like me and want to share the GPU accross multiple pods then just add these ENV variables to your pod
```
  NVIDIA_VISIBLE_DEVICES: all
  NVIDIA_DRIVER_CAPABILITIES: all
```

# It work for me . Thank you.


