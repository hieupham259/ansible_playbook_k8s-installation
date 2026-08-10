# Weave Net cho NetworkPolicy (Weave Net for NetworkPolicy)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/network-policy-provider/weave-network-policy/

Trang này hướng dẫn cách sử dụng Weave Net cho NetworkPolicy.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes. Hãy làm theo
[hướng dẫn bắt đầu với kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) để dựng (bootstrap) một cluster.

## Cài đặt addon Weave Net (Install the Weave Net addon)

Làm theo hướng dẫn [Tích hợp Kubernetes qua Addon](https://github.com/weaveworks/weave/blob/master/site/kubernetes/kube-addon.md#-installation).

Addon Weave Net cho Kubernetes đi kèm một
[Network Policy Controller](https://github.com/weaveworks/weave/blob/master/site/kubernetes/kube-addon.md#network-policy)
tự động giám sát Kubernetes để phát hiện mọi annotation NetworkPolicy trên tất cả các
namespace và cấu hình các quy tắc `iptables` nhằm cho phép hoặc chặn lưu lượng theo đúng chỉ dẫn của các policy.

## Kiểm tra việc cài đặt (Test the installation)

Xác minh rằng weave hoạt động.

Nhập lệnh sau:

```shell
kubectl get pods -n kube-system -o wide
```

Kết quả xuất ra tương tự như sau:

```
NAME                                    READY     STATUS    RESTARTS   AGE       IP              NODE
weave-net-1t1qg                         2/2       Running   0          9d        192.168.2.10    worknode3
weave-net-231d7                         2/2       Running   1          7d        10.2.0.17       worknodegpu
weave-net-7nmwt                         2/2       Running   3          9d        192.168.2.131   masternode
weave-net-pmw8w                         2/2       Running   0          9d        192.168.2.216   worknode2
```

Mỗi Node có một Pod weave, và tất cả các Pod đều ở trạng thái `Running` và `2/2 READY`. (`2/2` nghĩa là mỗi Pod có `weave` và `weave-npc`.)

## Tiếp theo (What's next)

Sau khi đã cài đặt addon Weave Net, bạn có thể làm theo bài
[Khai báo Network Policy](201-declare-network-policy-vi.md)
để thử nghiệm NetworkPolicy của Kubernetes. Nếu bạn có bất kỳ câu hỏi nào, hãy liên hệ với chúng tôi tại
[#weave-community trên Slack hoặc Weave User Group](https://github.com/weaveworks/weave#getting-help).
