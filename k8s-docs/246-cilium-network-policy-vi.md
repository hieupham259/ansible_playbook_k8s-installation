# Sử dụng Cilium cho NetworkPolicy (Use Cilium for NetworkPolicy)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/network-policy-provider/cilium-network-policy/

Trang này hướng dẫn cách sử dụng Cilium cho NetworkPolicy.

Để tìm hiểu thông tin nền về Cilium, hãy đọc
[Introduction to Cilium](https://docs.cilium.io/en/stable/overview/intro).

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, nhập `kubectl version`.

## Triển khai Cilium trên Minikube để kiểm thử cơ bản (Deploying Cilium on Minikube for Basic Testing)

Để làm quen với Cilium một cách dễ dàng, bạn có thể làm theo
[Cilium Kubernetes Getting Started Guide](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/)
để thực hiện một cài đặt DaemonSet cơ bản của Cilium trên minikube.

Để khởi động minikube — yêu cầu phiên bản v1.5.2 trở lên — hãy chạy với các
tham số sau:

```shell
minikube version
```

```
minikube version: v1.5.2
```

```shell
minikube start --network-plugin=cni
```

Với minikube, bạn có thể cài đặt Cilium bằng công cụ CLI của nó. Để làm vậy, trước tiên
hãy tải phiên bản mới nhất của CLI bằng lệnh sau:

```shell
curl -LO https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz
```

Sau đó giải nén file vừa tải về vào thư mục `/usr/local/bin` của bạn bằng lệnh sau:

```shell
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
rm cilium-linux-amd64.tar.gz
```

Sau khi chạy các lệnh trên, bạn có thể cài đặt Cilium bằng lệnh sau:

```shell
cilium install
```

Cilium sau đó sẽ tự động phát hiện cấu hình cluster, rồi tạo và cài đặt
các thành phần phù hợp để việc cài đặt thành công.
Các thành phần đó gồm:

- Certificate Authority (CA) trong Secret `cilium-ca` và các certificate cho Hubble
  (lớp quan sát — observability — của Cilium).
- Các service account.
- Các cluster role.
- ConfigMap.
- DaemonSet Agent và một Deployment Operator.

Sau khi cài đặt, bạn có thể xem trạng thái tổng thể của việc triển khai Cilium
bằng lệnh `cilium status`.
Xem output mong đợi của lệnh `status`
[tại đây](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#validate-the-installation).

Phần còn lại của Getting Started Guide giải thích cách áp đặt (enforce) cả các chính sách
bảo mật L3/L4 (tức là địa chỉ IP + port) lẫn các chính sách bảo mật L7 (ví dụ: HTTP)
bằng một ứng dụng ví dụ.

## Triển khai Cilium cho môi trường production (Deploying Cilium for Production Use)

Để có hướng dẫn chi tiết về việc triển khai Cilium cho production, xem:
[Cilium Kubernetes Installation Guide](https://docs.cilium.io/en/stable/network/kubernetes/concepts/)
Tài liệu này bao gồm các yêu cầu chi tiết, hướng dẫn và các file DaemonSet
mẫu cho production.

## Tìm hiểu các thành phần của Cilium (Understanding Cilium components)

Việc triển khai một cluster với Cilium sẽ thêm các Pod vào namespace `kube-system`. Để xem
danh sách các Pod này, hãy chạy:

```shell
kubectl get pods --namespace=kube-system -l k8s-app=cilium
```

Bạn sẽ thấy một danh sách Pod tương tự như sau:

```console
NAME           READY   STATUS    RESTARTS   AGE
cilium-kkdhz   1/1     Running   0          3m23s
...
```

Một Pod `cilium` chạy trên mỗi node trong cluster của bạn và áp đặt network policy
lên lưu lượng đi đến/đi từ các Pod trên node đó bằng Linux BPF.

## Tiếp theo (What's next)

Khi cluster của bạn đã chạy, bạn có thể làm theo bài
[Khai báo Network Policy](201-declare-network-policy-vi.md)
để thử nghiệm NetworkPolicy của Kubernetes với Cilium.
Chúc bạn vui, và nếu có câu hỏi, hãy liên hệ với chúng tôi qua
[Cilium Slack Channel](https://slack.cilium.io/).
