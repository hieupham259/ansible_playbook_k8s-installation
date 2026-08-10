# Sử dụng Antrea cho NetworkPolicy (Use Antrea for NetworkPolicy)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/network-policy-provider/antrea-network-policy/

Trang này hướng dẫn cách cài đặt và sử dụng plugin CNI Antrea trên Kubernetes.
Để tìm hiểu thông tin nền về Project Antrea, hãy đọc
[Introduction to Antrea](https://antrea.io/docs/).

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes. Hãy làm theo
[hướng dẫn bắt đầu với kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/)
để dựng một cluster.

## Triển khai Antrea với kubeadm (Deploying Antrea with kubeadm)

Hãy làm theo hướng dẫn
[Getting Started](https://github.com/vmware-tanzu/antrea/blob/main/docs/getting-started.md)
để triển khai Antrea cho kubeadm.

## Tiếp theo (What's next)

Khi cluster của bạn đã chạy, bạn có thể làm theo bài
[Khai báo Network Policy](201-declare-network-policy-vi.md) để thử nghiệm
NetworkPolicy của Kubernetes.
