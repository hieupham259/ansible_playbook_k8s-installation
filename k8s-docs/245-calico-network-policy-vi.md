# Sử dụng Calico cho NetworkPolicy (Use Calico for NetworkPolicy)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/network-policy-provider/calico-network-policy/

Trang này trình bày một vài cách nhanh để tạo một cluster Calico trên Kubernetes.

## Trước khi bạn bắt đầu (Before you begin)

Hãy quyết định xem bạn muốn triển khai một cluster
[trên cloud](#creating-a-calico-cluster-with-google-kubernetes-engine-gke) hay
[cục bộ](#creating-a-local-calico-cluster-with-kubeadm).

## Tạo một cluster Calico với Google Kubernetes Engine (GKE) (Creating a Calico cluster with Google Kubernetes Engine (GKE)) {#creating-a-calico-cluster-with-google-kubernetes-engine-gke}

**Điều kiện tiên quyết**: [gcloud](https://cloud.google.com/sdk/docs/quickstarts).

1.  Để khởi tạo một cluster GKE với Calico, hãy thêm cờ (flag) `--enable-network-policy`.

    **Cú pháp**
    ```shell
    gcloud container clusters create [CLUSTER_NAME] --enable-network-policy
    ```

    **Ví dụ**
    ```shell
    gcloud container clusters create my-calico-cluster --enable-network-policy
    ```

1.  Để xác minh việc triển khai, hãy dùng lệnh sau.

    ```shell
    kubectl get pods --namespace=kube-system
    ```

    Các pod của Calico bắt đầu bằng `calico`. Hãy kiểm tra để chắc chắn rằng
    mỗi pod đều có trạng thái `Running`.

## Tạo một cluster Calico cục bộ với kubeadm (Creating a local Calico cluster with kubeadm) {#creating-a-local-calico-cluster-with-kubeadm}

Để có một cluster Calico cục bộ trên một máy (single-host) trong mười lăm phút
bằng kubeadm, hãy tham khảo
[Calico Quickstart](https://projectcalico.docs.tigera.io/getting-started/kubernetes/).

## Tiếp theo (What's next)

Khi cluster của bạn đã chạy, bạn có thể làm theo bài
[Khai báo Network Policy](201-declare-network-policy-vi.md) để thử nghiệm
NetworkPolicy của Kubernetes.
