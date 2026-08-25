# Sử dụng Cilium cho NetworkPolicy (Use Cilium for NetworkPolicy)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/network-policy-provider/cilium-network-policy/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 21 — DNS, CNI và kube-proxy](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy)
— **trang con của bài 7/14**, mục [Cài đặt một Network Policy
Provider](243-network-policy-provider-vi.md); nó không có dòng riêng trong lộ trình. Phần II không
có lab: thực hành thẳng trên cluster VM của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) và tự chấm
bằng **Checkpoint** ở cuối [mục giai đoạn
21](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy).

Đây là trang provider **dài nhất** nhóm, nhưng cluster lab **đã chạy Calico** từ
[Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) nên chỉ bài
[245](245-calico-network-policy-vi.md) khớp cluster của bạn. **Không cài Cilium** vào cluster lab:
đổi CNI sẽ phá [chuỗi snapshot](labs/README.md#3-chuỗi-snapshot) mà các lab sau dựa vào. Đọc để
biết Cilium khác các provider kia ở đâu, không để làm theo.

**Phải hiểu ở lần đọc này:**

- Đặc trưng phân biệt Cilium, nằm ở mục *Tìm hiểu các thành phần của Cilium* và ở cuối mục
  minikube: một Pod `cilium` chạy **trên mỗi node** và áp đặt network policy bằng **Linux BPF**;
  và Getting Started Guide dạy áp đặt cả policy **L3/L4** (IP + port) lẫn policy **L7**, ví dụ
  HTTP. Hai provider dùng `iptables` trong cùng nhóm — [kube-router](247-kube-router-network-policy-vi.md)
  và [Weave Net](249-weave-network-policy-vi.md) — không nói tới tầng L7.
- Cilium là provider duy nhất trong nhóm cần **một CLI riêng ngoài `kubectl`**: bài tải
  `cilium-linux-amd64.tar.gz` vào `/usr/local/bin` rồi dùng `cilium install` để cài và
  `cilium status` để xem trạng thái triển khai.
- `cilium install` **tự phát hiện cấu hình cluster** rồi tạo ra một tập thành phần mà bài liệt kê
  đích danh: một CA trong Secret `cilium-ca` cùng certificate cho Hubble, các ServiceAccount, các
  ClusterRole, một ConfigMap, một **DaemonSet Agent** và một **Deployment Operator**. Nhìn danh
  sách đó là thấy một CNI hiện đại không chỉ là một binary thả lên node.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Triển khai Cilium trên Minikube để kiểm thử cơ bản* — `minikube start --network-plugin=cni`, `curl -LO` tải CLI, `sudo tar xzvfC`, `cilium install` | lộ trình cấm minikube và cấm cluster dùng chung; đây lại là quy trình cài một CNI thật | **không chạy trên cluster lab** — cluster đã dùng Calico từ [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) và đổi CNI phá [chuỗi snapshot](labs/README.md#3-chuỗi-snapshot) |
| Mục *Triển khai Cilium cho môi trường production* | chỉ là con trỏ sang tài liệu cài đặt của chính dự án Cilium | không có trong lộ trình; mở khi bạn phải tiếp quản một cluster chạy Cilium |
| Hubble và các certificate của nó trong danh sách thành phần | Hubble là lớp quan sát riêng của Cilium, không phải cơ chế của Kubernetes | không có trong lộ trình; phần quan sát của lộ trình đi bằng metrics-server và log ở [Lab 11a](labs/LAB-11A-OBSERVABILITY.md) |
| Mục *Trước khi bạn bắt đầu* — minikube và các playground | lộ trình cấm minikube, kind và cluster dùng chung | cluster VM ba node của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) |

---

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

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 21:

1. Bài nói Cilium áp đặt network policy **bằng cơ chế gì** của kernel, và Getting Started Guide
   dạy áp đặt policy ở **những tầng nào**? Điểm nào trong hai câu trả lời đó phân biệt Cilium với
   [kube-router](247-kube-router-network-policy-vi.md) và [Weave Net](249-weave-network-policy-vi.md)?
2. **Câu bẫy.** `cilium install` chỉ là một lệnh, nên dễ nghĩ nó chỉ thả xuống một DaemonSet. Bài
   liệt kê đủ những gì lệnh đó tạo ra — kể lại, và chỉ ra thứ nào chứng tỏ Cilium còn cần một
   thành phần **không** chạy theo kiểu mỗi-node-một-Pod.
3. Giả sử Cilium đang chạy trên cluster ba node của bạn. Theo đúng câu của bài, lệnh
   `kubectl get pods --namespace=kube-system -l k8s-app=cilium` sẽ trả về bao nhiêu Pod và vì sao?
   Và vì sao bạn **không** chạy `cilium install` lên `lab-k8s-master` sau khi đọc xong trang này?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Bằng **Linux BPF** — bài nói thẳng: một Pod `cilium` chạy trên mỗi node và áp đặt network
   policy lên lưu lượng đi đến/đi từ các Pod trên node đó bằng Linux BPF. Về tầng policy, Getting
   Started Guide giải thích cách áp đặt **cả policy L3/L4 (địa chỉ IP + port) lẫn policy L7, ví dụ
   HTTP**. Chính **tầng L7** là điểm phân biệt: hai trang kia chỉ nói tới việc cấu hình quy tắc
   `iptables` (kube-router thêm `ipset`) để cho phép hoặc chặn lưu lượng, **không** đề cập tới
   HTTP hay bất kỳ tầng ứng dụng nào.
2. Bài liệt kê: **CA trong Secret `cilium-ca` cùng các certificate cho Hubble**, các
   **ServiceAccount**, các **ClusterRole**, một **ConfigMap**, một **DaemonSet Agent** và một
   **Deployment Operator**. Thứ phá vỡ hình dung "chỉ một DaemonSet" là **Deployment Operator** —
   Deployment nghĩa là một thành phần chạy tập trung, không phải một bản sao trên mỗi node như
   DaemonSet Agent. Đáng chú ý nữa: lệnh này **tự phát hiện cấu hình cluster** rồi mới quyết định
   cài gì, chứ không áp một manifest cố định.
3. **Ba Pod — một trên mỗi node**, vì bài nói "Một Pod `cilium` chạy trên mỗi node trong cluster
   của bạn"; đó là hệ quả trực tiếp của việc agent được triển khai dạng DaemonSet. Còn lý do không
   chạy `cilium install`: cluster lab **đã có một provider thực thi NetworkPolicy** là Calico từ
   [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md); cài Cilium là **đổi CNI**, và điều đó phá
   [chuỗi snapshot](labs/README.md#3-chuỗi-snapshot) mà các lab sau dựa vào. Ở giai đoạn 21, trang
   này đọc để **biết lựa chọn**, không phải để thi công.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
