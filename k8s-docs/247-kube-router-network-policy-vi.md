# Sử dụng Kube-router cho NetworkPolicy (Use Kube-router for NetworkPolicy)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/network-policy-provider/kube-router-network-policy/

Trang này hướng dẫn cách sử dụng [Kube-router](https://github.com/cloudnativelabs/kube-router) cho NetworkPolicy.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes đang chạy. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng bất kỳ trình cài đặt cluster nào như Kops, Bootkube, Kubeadm, v.v.

## Cài đặt addon Kube-router (Installing Kube-router addon)

Addon Kube-router đi kèm một Network Policy Controller theo dõi (watch) Kubernetes API server để phát hiện mọi thay đổi về NetworkPolicy và pod, rồi cấu hình các quy tắc iptables và ipset nhằm cho phép hoặc chặn lưu lượng theo đúng chỉ dẫn của các policy. Vui lòng làm theo hướng dẫn [thử Kube-router với các trình cài đặt cluster](https://www.kube-router.io/docs/user-guide/#try-kube-router-with-cluster-installers) để cài đặt addon Kube-router.

## Tiếp theo (What's next)

Sau khi đã cài đặt addon Kube-router, bạn có thể làm theo bài [Khai báo Network Policy](201-declare-network-policy-vi.md) để thử nghiệm NetworkPolicy của Kubernetes.
