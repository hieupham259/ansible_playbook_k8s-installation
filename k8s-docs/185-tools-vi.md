# Cài đặt công cụ (Install Tools)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/tools/>
>
> Thiết lập các công cụ Kubernetes trên máy tính của bạn.

> **Ghi chú:** Xem trang [Môi trường học tập (Learning environment)](https://kubernetes.io/docs/setup/learning-environment/)
> để thiết lập một môi trường thực hành.

## kubectl

Công cụ dòng lệnh của Kubernetes, [kubectl](https://kubernetes.io/docs/reference/kubectl/kubectl/),
cho phép bạn chạy các lệnh đối với các cluster Kubernetes.
Bạn có thể dùng kubectl để triển khai ứng dụng, xem xét (inspect) và quản lý tài nguyên của cluster,
và xem log. Để biết thêm thông tin, bao gồm danh sách đầy đủ các thao tác của kubectl, xem
[tài liệu tham khảo `kubectl`](https://kubernetes.io/docs/reference/kubectl/).

kubectl có thể cài đặt được trên nhiều nền tảng Linux khác nhau, macOS và Windows.
Tìm hệ điều hành mà bạn dùng bên dưới.

- [Cài đặt kubectl trên Linux](186-install-kubectl-linux-vi.md)
- [Cài đặt kubectl trên macOS](187-install-kubectl-macos-vi.md)
- [Cài đặt kubectl trên Windows](188-install-kubectl-windows-vi.md)

## kind

[`kind`](https://kind.sigs.k8s.io/) cho phép bạn chạy Kubernetes trên máy tính
cục bộ của mình. Công cụ này yêu cầu bạn đã cài đặt
[Docker](https://www.docker.com/) hoặc [Podman](https://podman.io/).

Trang [Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/) của kind
chỉ cho bạn những gì cần làm để bắt đầu chạy được với kind.

[Xem hướng dẫn Quick Start của kind](https://kind.sigs.k8s.io/docs/user/quick-start/)

## minikube

Giống như `kind`, [`minikube`](https://minikube.sigs.k8s.io/) là một công cụ cho phép bạn chạy Kubernetes
cục bộ. `minikube` chạy một cluster Kubernetes cục bộ tất-cả-trong-một (all-in-one) hoặc nhiều node
trên máy tính cá nhân của bạn (bao gồm PC chạy Windows, macOS và Linux) để bạn có thể dùng thử
Kubernetes, hoặc dùng cho công việc phát triển hằng ngày.

Bạn có thể làm theo hướng dẫn chính thức
[Get Started!](https://minikube.sigs.k8s.io/docs/start/) nếu mối quan tâm của bạn
là cài đặt được công cụ này.

[Xem hướng dẫn Get Started! của minikube](https://minikube.sigs.k8s.io/docs/start/)

Sau khi `minikube` đã hoạt động, bạn có thể dùng nó để
[chạy một ứng dụng mẫu](https://kubernetes.io/docs/tutorials/hello-minikube/).

## kubeadm

Bạn có thể dùng công cụ kubeadm để tạo và quản lý các cluster Kubernetes.
Nó thực hiện các hành động cần thiết để dựng lên một cluster tối thiểu khả dụng (minimum viable) và an toàn,
theo cách thân thiện với người dùng.

[Cài đặt kubeadm](01-install-kubeadm-vi.md) chỉ cho bạn cách cài đặt kubeadm.
Sau khi cài đặt, bạn có thể dùng nó để [tạo một cluster](02-create-cluster-kubeadm-vi.md).

[Xem hướng dẫn cài đặt kubeadm](01-install-kubeadm-vi.md)
