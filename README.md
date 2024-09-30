# platform-infrastructure

## install rke2

References:
- https://docs.rke2.io/install/quickstart

```sh
# server node installation

curl -sfL https://get.rke2.io | sudo sh -

sudo systemctl enable rke2-server.service
sudo systemctl start rke2-server.service

journalctl -u rke2-server -f

# agent (worker) node installation

curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_TYPE="agent" sh -

sudo systemctl enable rke2-agent.service

sudo mkdir -p /etc/rancher/rke2/
sudo vi /etc/rancher/rke2/config.yaml
# >>>
# server: https://desktop-ubuntu:9345
# token: <taken from server node's '/var/lib/rancher/rke2/server/node-token'>
# node-label:
#   - nvidia.com/gpu.present=true
# node-taint:
#   - nvidia.com/gpu:NoSchedule

sudo systemctl start rke2-agent.service
```

## install nvidia gpu operator
```sh
# for rke2
kubectl apply -f ./k8s-core/NVIDIA/gpu-operator-values.yaml
kubectl delete -f ./k8s-core/NVIDIA/gpu-operator-values.yaml

# or for other clusters
helm upgrade -i gpu-operator gpu-operator \
    --namespace=gpu-operator \
    --create-namespace \
    --repo=https://helm.ngc.nvidia.com/nvidia \
    --version=v24.6.2
helm delete gpu-operator -n gpu-operator
```

## or, install nvidia device plugin
```sh
sudo nvidia-ctk runtime configure --runtime=containerd
# or,
sudo nvidia-ctk runtime configure --runtime=containerd --runtime-path=/run/k3s/containerd/containerd.sock

helm upgrade -i nvidia-device-plugin nvidia-device-plugin \
    --namespace=nvidia-device-plugin \
    --create-namespace \
    --repo=https://nvidia.github.io/k8s-device-plugin \
    --version=0.16.2

helm delete nvidia-device-plugin -n nvidia-device-plugin
```

## test cuda
```sh
kubectl run nbody-gpu-benchmark \
  --image=nvcr.io/nvidia/k8s/cuda-sample:nbody \
  --overrides='
{
  "apiVersion": "v1",
  "spec": {
    "restartPolicy": "OnFailure",
    "runtimeClassName": "nvidia",
    "nodeSelector": { "nvidia.com/gpu.present": "true" },
    "tolerations": [
      {
        "key": "nvidia.com/gpu",
        "operator": "Exists",
        "effect": "NoSchedule"
      }
    ],
    "containers": [
      {
        "name": "nbody-gpu-benchmark",
        "image": "nvcr.io/nvidia/k8s/cuda-sample:nbody",
        "args": [
          "nbody",
          "-gpu",
          "-benchmark"
        ],
        "env": [
          {
            "name": "NVIDIA_VISIBLE_DEVICES",
            "value": "all"
          },
          {
            "name": "NVIDIA_DRIVER_CAPABILITIES",
            "value": "all"
          }
        ],
        "resources": {
          "limits": {
            "nvidia.com/gpu": "1"
          }
        }
      }
    ]
  }
}'
```
