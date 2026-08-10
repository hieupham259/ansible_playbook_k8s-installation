# Romana cho NetworkPolicy (Romana for NetworkPolicy)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/network-policy-provider/romana-network-policy/

Trang này hướng dẫn cách sử dụng Romana cho NetworkPolicy.

## Trước khi bạn bắt đầu (Before you begin)

Hoàn thành các bước 1, 2 và 3 trong [hướng dẫn bắt đầu với kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/).

## Cài đặt Romana với kubeadm (Installing Romana with kubeadm)

Làm theo [hướng dẫn cài đặt dạng container hóa](https://github.com/romana/romana/tree/master/containerize) dành cho kubeadm.

## Áp dụng network policy (Applying network policies)

Để áp dụng network policy, hãy dùng một trong các cách sau:

* [Romana network policies](https://github.com/romana/romana/wiki/Romana-policies).
    * [Ví dụ về Romana network policy](https://github.com/romana/core/blob/master/doc/policy.md).
* NetworkPolicy API.

## Tiếp theo (What's next)

Sau khi đã cài đặt Romana, bạn có thể làm theo bài [Khai báo Network Policy](201-declare-network-policy-vi.md) để thử nghiệm NetworkPolicy của Kubernetes.
