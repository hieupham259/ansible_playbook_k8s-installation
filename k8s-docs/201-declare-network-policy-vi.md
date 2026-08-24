# Khai báo Network Policy (Declare Network Policy)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** tài liệu tra cứu thuộc nhánh `/docs/tasks/`
([Checkpoint tiếp nối](00-ALO-TRINH-ADMIN.md#checkpoint-tiếp-nối--nhánh-docstasks), mục
[CP6 — DNS, CNI và kube-proxy](00-ALO-TRINH-ADMIN.md#cp6--dns-cni-và-kube-proxy)), nối tiếp bài
[84 — Network Policy](84-network-policies-vi.md).

Lưu ý cho cluster lab: snapshot `01-cluster-ready` của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md)
chạy Flannel — CNI **không** hỗ trợ NetworkPolicy. Các bước trong bài chỉ cho kết quả đúng
sau khi cluster đã chuyển sang CNI hỗ trợ NetworkPolicy (snapshot `02-net-ready` của Lab 5b).

**Phải hiểu ở lần đọc này:**

- Trình tự kiểm chứng một NetworkPolicy: dựng đích (Deployment `nginx` + Service `nginx`) →
  đo đường cơ sở bằng busybox (`wget` thành công) → áp policy → chứng minh Pod thiếu label
  bị chặn → chứng minh Pod mang label `access=true` đi qua được.
- Policy `access-nginx` chọn Pod đích bằng `spec.podSelector` (`app: nginx` — label do
  Deployment tự gán cho Pod) và chỉ cho phép ingress từ các Pod khớp `podSelector` nguồn
  `access: "true"`.
- Điều kiện tiên quyết quan trọng nhất: network provider (CNI) phải hỗ trợ NetworkPolicy.
  Tạo được object NetworkPolicy không có nghĩa là nó có hiệu lực.
- Triệu chứng khi bị policy chặn: `wget` hết thời gian chờ (`download timed out`), không
  phải lỗi từ chối kết nối.
- `podSelector` rỗng chọn **tất cả** Pod trong namespace.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Danh sách sáu network provider trong phần điều kiện tiên quyết | chỉ cần một CNI hỗ trợ NetworkPolicy; việc chọn và cài đặt CNI đó thuộc lab đổi CNI | mục Network Policy Providers của [CP6](00-ALO-TRINH-ADMIN.md#cp6--dns-cni-và-kube-proxy) và Lab 5b |

---

Tài liệu này giúp bạn bắt đầu sử dụng [NetworkPolicy API](84-network-policies-vi.md) của
Kubernetes để khai báo các network policy chi phối cách các Pod giao tiếp với nhau.

> **Ghi chú:** Mục này đề cập đến các sản phẩm hoặc dự án bên thứ ba cung cấp chức năng mà
> Kubernetes cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những sản phẩm
> hoặc dự án đó. Để thêm một sản phẩm hoặc dự án vào danh sách này, hãy đọc
> [hướng dẫn về nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content)
> trước khi gửi thay đổi.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Phiên bản Kubernetes server của bạn phải bằng hoặc mới hơn v1.8. Để kiểm tra phiên bản, nhập
`kubectl version`.

Hãy chắc chắn rằng bạn đã cấu hình một network provider có hỗ trợ network policy. Có nhiều
network provider hỗ trợ NetworkPolicy, bao gồm:

* [Antrea](244-antrea-network-policy-vi.md)
* [Calico](245-calico-network-policy-vi.md)
* [Cilium](246-cilium-network-policy-vi.md)
* [Kube-router](247-kube-router-network-policy-vi.md)
* [Romana](248-romana-network-policy-vi.md)
* [Weave Net](249-weave-network-policy-vi.md)

## Tạo một Deployment `nginx` và expose nó qua một Service (Create an `nginx` deployment and expose it via a service)

Để thấy network policy của Kubernetes hoạt động thế nào, hãy bắt đầu bằng việc tạo một
Deployment `nginx`.

```console
kubectl create deployment nginx --image=nginx
```

```none
deployment.apps/nginx created
```

Expose Deployment này thông qua một Service có tên `nginx`.

```console
kubectl expose deployment nginx --port=80
```

```none
service/nginx exposed
```

Các lệnh trên tạo một Deployment với một Pod nginx và expose Deployment đó qua một Service
tên là `nginx`. Pod và Deployment `nginx` nằm trong namespace `default`.

```console
kubectl get svc,pod
```

```none
NAME                        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
service/kubernetes          10.100.0.1    <none>        443/TCP    46m
service/nginx               10.100.0.16   <none>        80/TCP     33s

NAME                        READY         STATUS        RESTARTS   AGE
pod/nginx-701339712-e0qfq   1/1           Running       0          35s
```

## Kiểm tra Service bằng cách truy cập từ một Pod khác (Test the service by accessing it from another Pod)

Bạn sẽ có thể truy cập Service `nginx` mới từ các Pod khác. Để truy cập Service `nginx` từ
một Pod khác trong namespace `default`, hãy khởi động một container busybox:

```console
kubectl run busybox --rm -ti --image=busybox -- /bin/sh
```

Trong shell vừa mở, chạy lệnh sau:

```shell
wget --spider --timeout=1 nginx
```

```none
Connecting to nginx (10.100.0.16:80)
remote file exists
```

## Giới hạn truy cập vào Service `nginx` (Limit access to the `nginx` service)

Để giới hạn truy cập vào Service `nginx` sao cho chỉ những Pod mang label `access: true` mới
có thể truy vấn nó, hãy tạo một object NetworkPolicy như sau:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-nginx
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
  - from:
    - podSelector:
        matchLabels:
          access: "true"
```

Tên của một object NetworkPolicy phải là một
[tên DNS subdomain](17-names-vi.md#dns-subdomain-names) hợp lệ.

> **Ghi chú:** NetworkPolicy bao gồm một `podSelector` để chọn nhóm Pod mà policy áp dụng.
> Bạn có thể thấy policy này chọn các Pod mang label `app=nginx`. Label này đã được tự động
> thêm vào Pod trong Deployment `nginx`. Một `podSelector` rỗng sẽ chọn tất cả Pod trong
> namespace.

## Gán policy cho Service (Assign the policy to the service)

Dùng kubectl để tạo một NetworkPolicy từ file `nginx-policy.yaml` ở trên:

```console
kubectl apply -f https://k8s.io/examples/service/networking/nginx-policy.yaml
```

```none
networkpolicy.networking.k8s.io/access-nginx created
```

## Kiểm tra truy cập Service khi chưa định nghĩa label access (Test access to the service when access label is not defined)

Khi bạn thử truy cập Service `nginx` từ một Pod không có đúng label, request sẽ hết thời
gian chờ (time out):

```console
kubectl run busybox --rm -ti --image=busybox -- /bin/sh
```

Trong shell vừa mở, chạy lệnh:

```shell
wget --spider --timeout=1 nginx
```

```none
Connecting to nginx (10.100.0.16:80)
wget: download timed out
```

## Định nghĩa label access và kiểm tra lại (Define access label and test again)

Bạn có thể tạo một Pod với đúng label để thấy rằng request được cho phép:

```console
kubectl run busybox --rm -ti --labels="access=true" --image=busybox -- /bin/sh
```

Trong shell vừa mở, chạy lệnh:

```shell
wget --spider --timeout=1 nginx
```

```none
Connecting to nginx (10.100.0.16:80)
remote file exists
```

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc này:

1. Kể lại trình tự các bước mà bài dùng để chứng minh NetworkPolicy có hiệu lực. Vì sao bước
   đo đường cơ sở (truy cập thành công **trước khi** áp policy) lại quan trọng?
2. **Câu bẫy.** Cluster lab của bạn đang ở snapshot chạy Flannel. Bạn apply
   `nginx-policy.yaml` và kubectl báo `networkpolicy.networking.k8s.io/access-nginx created`.
   Pod busybox **không** có label `access=true` còn truy cập được `nginx` không? Vì sao?
3. Trong policy `access-nginx`, selector nào chọn Pod đích và selector nào chọn nguồn được
   phép? Label `app=nginx` từ đâu mà có?
4. Một NetworkPolicy trong namespace `default` có `spec.podSelector: {}` (rỗng) thì áp dụng
   cho những Pod nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Dựng đích → đo đường cơ sở → áp policy → kiểm chứng cả hai chiều.** Cụ thể: tạo
   Deployment `nginx` và expose qua Service `nginx`; từ busybox `wget` thành công (đường cơ
   sở); apply `access-nginx`; `wget` từ Pod không có label bị `download timed out`; `wget`
   từ Pod chạy với `--labels="access=true"` lại thành công. Bước đường cơ sở quan trọng vì
   nếu không đo trước, khi thấy timeout bạn không phân biệt được "policy đang chặn" với
   "Service/DNS/CNI vốn đã hỏng từ đầu".
2. **Vẫn truy cập được.** `kubectl apply` chỉ ghi object vào API — việc thực thi policy là
   của network provider. Bài nhấn mạnh điều kiện tiên quyết là phải cấu hình một network
   provider **hỗ trợ** NetworkPolicy, và Flannel không nằm trong danh sách đó; policy tồn
   tại trong API nhưng không có hiệu lực. Trực giác "tạo object thành công nghĩa là nó đang
   hoạt động" sai ở chỗ NetworkPolicy chỉ là bản khai báo ý định.
3. `spec.podSelector` (`matchLabels: app: nginx`) chọn nhóm Pod **đích** mà policy áp lên;
   `ingress[].from[].podSelector` (`access: "true"`) chọn **nguồn** được phép gửi traffic
   vào. Label `app=nginx` được tự động thêm vào Pod bởi Deployment `nginx` khi tạo bằng
   `kubectl create deployment` — bạn không tự gán nó.
4. **Tất cả Pod trong namespace `default`.** Theo Ghi chú của bài, một `podSelector` rỗng
   chọn mọi Pod trong namespace chứa policy.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
