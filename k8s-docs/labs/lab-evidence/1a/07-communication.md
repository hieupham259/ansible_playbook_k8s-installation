# LAB 1A — B7: Giao tiếp giữa Node và Control Plane

```text
kubectl ------HTTPS:6443------> kube-apiserver
kubelet ------HTTPS:6443------> kube-apiserver
kube-apiserver --HTTPS:10250--> kubelet
```

Trong control plane mặc định, API server là thành phần nói trực tiếp với etcd; Node và Pod
không kết nối trực tiếp tới etcd. Lab không triển khai Konnectivity service; đây chỉ là một lựa
chọn tunnel/proxy cho đường API server tới node.
