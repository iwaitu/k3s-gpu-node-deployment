
# k3s-gpu-node-deployment
Non-Gpu node as k3s master, add another node with gpu to the cluster and make it work

# on your new gpu node: 

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
## now install k3s to your new node
```
curl -sfL https://get.k3s.io | sh -
```
or if your server in china
```
curl -sfL https://rancher-mirror.oss-cn-beijing.aliyuncs.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -
```
## after this, you need to get two things for add your new node to k3s
### 1. copy your kubeconfig to your new node in this path /etc/rancher/k3s/k3s.yaml
### 2. login to your k3s master node and type this command to show the token
```
cat /var/lib/rancher/k3s/server/node-token
```
### 3. now you can add your new node to k3s cluster
```
curl -sfL https://get.k3s.io | K3S_URL=https://myserver:6443 K3S_TOKEN=mynodetoken sh -
```

## Download template from k3d project on your k3s master node
```
sudo wget https://k3d.io/v5.4.1/usage/advanced/cuda/config.toml.tmpl -O /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl
```
or use the mirror file in this repos
```
sudo wget https://raw.githubusercontent.com/iwaitu/k3s-gpu-node-deployment/main/config.toml.tmpl /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl
```



# master node

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
### This used to work previously but with K3S v1.23+ I had issues. You will have to modify /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl on youe new gpu node add:
```
[plugins.cri.containerd.runtimes.runc.options]
  BinaryName = "/usr/bin/nvidia-container-runtime"
```

### and modify
```
[plugins.cri.containerd.runtimes.runc]
  runtime_type = "io.containerd.runtime.v1.linux"
```
to
```
[plugins.cri.containerd.runtimes.runc]
  runtime_type = "io.containerd.runc.v2"
```

after all ,you can try to run a gpu pod now:
```
kubectl apply -f https://raw.githubusercontent.com/iwaitu/k3s-gpu-node-deployment/main/gpu.yaml
```
if the pod is deploy success, you can access the shell to exec nvidia-smi, you will see your gpu driver infomation.


This should get everything running. Then either use the nvidia plugin to define resources or if you like me and want to share the GPU accross multiple pods then just add these ENV variables to your pod
```
  NVIDIA_VISIBLE_DEVICES: all
  NVIDIA_DRIVER_CAPABILITIES: all
```

# if your new gpu node can not run any docker image ,maybe you need to install docker-ce on the new gpu node
```
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```
# that's all. I hope this can help you .


