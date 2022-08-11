# k3s-gpu-node-deployment
Non-Gpu node as k3s master, add another node with gpu to the cluster and make it work

# Install gpg key from nvidia
```
distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
    && curl -s -L https://nvidia.github.io/libnvidia-container/gpgkey | sudo apt-key add - \
    && curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```
# Install Drivers. Use the latest drivers!
```
apt-get update && apt-get install -y nvidia-driver-515-server nvidia-container-toolkit nvidia-modprobe
reboot
```

# Check whether GPU recognized. You might have to restart the node after the driver installation to get this working
```
nvidia-smi
```

# Download template from k3d project
```
sudo wget https://k3d.io/v5.4.1/usage/advanced/cuda/config.toml.tmpl -O /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl
```

# Install nvidia plugin. This is optional. You can simply pass in the env variables to pass in GPU access to pods. However this is a nice way to debug whether you have access to the GPUs on contianerd
```
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.12.2/nvidia-device-plugin.yml
```
Try running nvidia plugin and check the logs. If you see this on the correct node then you have to modify the .tmpl file further.
```
nvidia-smi
```
