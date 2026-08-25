# Mở rộng dải IP của Service (Extend Service IP Ranges)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/network/extend-service-ip-ranges/>
>
> Tài liệu này trình bày cách mở rộng dải Service IP hiện có đã được gán cho một cluster.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 21 — DNS, CNI và kube-proxy](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy),
bài 12/14 · Phần II không có lab: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) và tự chấm bằng **Checkpoint** ở cuối
[mục giai đoạn 21](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy). Bài này **làm được
trên cluster lab**: `kubectl get servicecidr`, `kubectl get ipaddress`, thêm một ServiceCIDR mới
rồi xóa nó đi — thao tác cộng thêm, hoàn tác được.

Bài này và bài [394](394-reconfigure-default-service-ip-ranges-vi.md) ngay sau nó là một **cặp dễ
nhầm**. Ranh giới cần nhớ từ bây giờ: bài này **thêm** dải mới cho cluster đang chạy, không đụng
tới kube-apiserver, và hoàn tác được; bài 394 **thay** dải mặc định — phải sửa cờ của
kube-apiserver và đánh số lại Service. Nó nối tiếp bài [88](88-cluster-ip-allocation-vi.md) đã đọc
ở mạch chính.

**Phải hiểu ở lần đọc này:**

- Đối tượng `ServiceCIDR` mang tên phổ biến **`kubernetes`** do kube-apiserver tạo, phản ánh giá
  trị cờ `--service-cluster-ip-range`. `kubectl get servicecidr` là chỗ đọc dải Service hiện hành
  của cluster, và Service `kubernetes` lấy **địa chỉ đầu tiên** trong dải đó làm ClusterIP.
- Mỗi ClusterIP đã cấp có một đối tượng **`IPAddress`** tương ứng, trỏ ngược về Service qua cột
  `PARENTREF`. Đó là cách control plane theo dõi từng địa chỉ đã dùng.
- Cách mở rộng ở mục *Thêm một ServiceCIDR mới*: **tạo thêm một đối tượng ServiceCIDR**, không
  sửa đối tượng cũ và **không** cấu hình lại kube-apiserver. Bài nói rõ trước đây mở rộng dải
  Service là thao tác gây gián đoạn và thậm chí có thể mất dữ liệu.
- Số địa chỉ thật sự dùng được: với `/28` là `2^(32-28) - 2 = 14`, mà `kubernetes.default` luôn
  chiếm một, nên chỉ tạo thêm được 13 Service. Hết dải thì apiserver trả
  `failed to allocate a serviceIP: range is full`.
- Cơ chế bảo vệ ở mục *Xóa một ServiceCIDR*: finalizer
  `networking.k8s.io/service-cidr-finalizer` giữ object lại ở trạng thái chờ xóa khi vẫn còn
  IPAddress phụ thuộc — nhìn `status.conditions` thấy `Ready=False`, `reason: OrphanIPAddress`.
  Finalizer chỉ được gỡ khi không còn IPAddress nào thuộc subnet đó, hoặc có subnet khác chứa
  chúng.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Trước khi bạn bắt đầu* — minikube và các playground | lộ trình không dùng minikube hay cluster dùng chung | cluster VM ba node của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) là môi trường thực hành duy nhất |
| Hai ví dụ `ValidatingAdmissionPolicy` ở mục *Chính sách về Service CIDR* và biểu thức CEL của chúng | là công cụ quản trị tập trung, không phải cơ chế cấp phát IP mà bài đang dạy | chặng admission đã học ở [giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), bài [119](119-controlling-access-vi.md) và [173](173-admission-webhooks-vi.md); riêng cú pháp CEL nằm ngoài phạm vi lộ trình |
| Ví dụ CIDR IPv6 `2001:db8::/64` trong danh sách `allowed` | cluster lab chạy thuần IPv4 | bài [395](395-validate-dual-stack-vi.md) — bài 14/14 của giai đoạn 21 |
| Feature gate `MultiCIDRServiceAllocator` và mốc phiên bản của tính năng | cách bật/tắt feature gate đã học rồi; ở đây chỉ cần biết tính năng đang bật mặc định | bài [196](196-configure-feature-gates-vi.md) — [giai đoạn 20](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.33 [stable]` (được bật mặc định)

Tài liệu này trình bày cách mở rộng dải Service IP hiện có đã được gán cho một cluster.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản v1.29 hoặc mới hơn. Để kiểm tra phiên bản, hãy
chạy `kubectl version`.

> **Ghi chú:** Bạn vẫn có thể dùng tính năng này trên phiên bản cũ hơn, nhưng nó chỉ đạt trạng
> thái GA và được hỗ trợ chính thức kể từ v1.33.

## Mở rộng dải IP của Service (Extend Service IP Ranges)

Những cluster Kubernetes có kube-apiserver đã bật
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`MultiCIDRServiceAllocator` và có nhóm API `networking.k8s.io/v1` đang hoạt động sẽ tạo một
đối tượng ServiceCIDR mang cái tên phổ biến (well-known) là `kubernetes`; đối tượng đó chỉ
định một dải địa chỉ IP dựa trên giá trị của tham số dòng lệnh `--service-cluster-ip-range`
truyền cho kube-apiserver.

```sh
kubectl get servicecidr
```

```
NAME         CIDRS          AGE
kubernetes   10.96.0.0/28   17d
```

Service phổ biến tên `kubernetes` — thứ phơi bày endpoint của kube-apiserver ra cho các Pod —
sẽ tính ra địa chỉ IP đầu tiên trong dải ServiceCIDR mặc định và dùng địa chỉ IP đó làm cluster
IP của mình.

```sh
kubectl get service kubernetes
```

```
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   17d
```

Trong trường hợp này, Service mặc định dùng ClusterIP 10.96.0.1, và địa chỉ đó có một đối
tượng IPAddress tương ứng.

```sh
kubectl get ipaddress 10.96.0.1
```

```
NAME        PARENTREF
10.96.0.1   services/default/kubernetes
```

Các ServiceCIDR được bảo vệ bằng finalizer, nhằm tránh để lại các ClusterIP của Service ở
trạng thái mồ côi (orphan); finalizer chỉ được gỡ bỏ khi có một subnet khác chứa những
IPAddress đang tồn tại, hoặc khi không còn IPAddress nào thuộc subnet đó.

## Tăng số địa chỉ IP khả dụng cho Service (Extend the number of available IPs for Services)

Có những tình huống người dùng cần tăng số địa chỉ khả dụng cho Service. Trước đây, việc mở
rộng dải Service là một thao tác gây gián đoạn (disruptive) và thậm chí có thể dẫn tới mất dữ
liệu. Với tính năng mới này, người dùng chỉ cần thêm một ServiceCIDR mới là đã tăng được số
địa chỉ khả dụng.

### Thêm một ServiceCIDR mới (Adding a new ServiceCIDR)

Trên một cluster dùng dải 10.96.0.0/28 cho Service, chỉ có 2^(32-28) - 2 = 14 địa chỉ IP khả
dụng. Service `kubernetes.default` luôn luôn được tạo; với ví dụ này, bạn chỉ còn 13 Service
có thể tạo thêm.

```sh
for i in $(seq 1 13); do kubectl create service clusterip "test-$i" --tcp 80 -o json | jq -r .spec.clusterIP; done
```

```
10.96.0.11
10.96.0.5
10.96.0.12
10.96.0.13
10.96.0.14
10.96.0.2
10.96.0.3
10.96.0.4
10.96.0.6
10.96.0.7
10.96.0.8
10.96.0.9
error: failed to create ClusterIP service: Internal error occurred: failed to allocate a serviceIP: range is full
```

Bạn có thể tăng số địa chỉ IP khả dụng cho Service bằng cách tạo một ServiceCIDR mới để mở
rộng hoặc bổ sung thêm dải địa chỉ IP.

```sh
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: ServiceCIDR
metadata:
  name: newcidr1
spec:
  cidrs:
  - 10.96.0.0/24
EOF
```

```
servicecidr.networking.k8s.io/newcidr1 created
```

và việc này cho phép bạn tạo các Service mới với ClusterIP được lấy từ dải mới đó.

```sh
for i in $(seq 13 16); do kubectl create service clusterip "test-$i" --tcp 80 -o json | jq -r .spec.clusterIP; done
```

```
10.96.0.48
10.96.0.200
10.96.0.121
10.96.0.144
```

### Xóa một ServiceCIDR (Deleting a ServiceCIDR)

Bạn không thể xóa một ServiceCIDR nếu vẫn còn các IPAddress phụ thuộc vào ServiceCIDR đó.

```sh
kubectl delete servicecidr newcidr1
```

```
servicecidr.networking.k8s.io "newcidr1" deleted
```

Kubernetes dùng một finalizer trên ServiceCIDR để theo dõi quan hệ phụ thuộc này.

```sh
kubectl get servicecidr newcidr1 -o yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: ServiceCIDR
metadata:
  creationTimestamp: "2023-10-12T15:11:07Z"
  deletionGracePeriodSeconds: 0
  deletionTimestamp: "2023-10-12T15:12:45Z"
  finalizers:
  - networking.k8s.io/service-cidr-finalizer
  name: newcidr1
  resourceVersion: "1133"
  uid: 5ffd8afe-c78f-4e60-ae76-cec448a8af40
spec:
  cidrs:
  - 10.96.0.0/24
status:
  conditions:
  - lastTransitionTime: "2023-10-12T15:12:45Z"
    message: There are still IPAddresses referencing the ServiceCIDR, please remove
      them or create a new ServiceCIDR
    reason: OrphanIPAddress
    status: "False"
    type: Ready
```

Bằng cách xóa những Service đang giữ các địa chỉ IP chặn việc xóa ServiceCIDR

```sh
for i in $(seq 13 16); do kubectl delete service "test-$i" ; done
```

```
service "test-13" deleted
service "test-14" deleted
service "test-15" deleted
service "test-16" deleted
```

control plane sẽ nhận thấy việc xóa này. Sau đó control plane gỡ finalizer của nó đi, nhờ vậy
ServiceCIDR đang chờ xóa mới thực sự bị xóa.

```sh
kubectl get servicecidr newcidr1
```

```
Error from server (NotFound): servicecidrs.networking.k8s.io "newcidr1" not found
```

## Chính sách về Service CIDR trong Kubernetes (Kubernetes Service CIDR Policies)

Quản trị viên cluster có thể triển khai các chính sách (policy) để kiểm soát việc tạo và sửa
đổi tài nguyên ServiceCIDR bên trong cluster. Điều này cho phép quản lý tập trung những dải
địa chỉ IP được dùng cho Service, đồng thời giúp ngăn các cấu hình ngoài ý muốn hoặc xung đột
lẫn nhau. Kubernetes cung cấp những cơ chế như Validating Admission Policy để áp đặt các quy
tắc đó.

### Ngăn việc tạo/cập nhật ServiceCIDR trái phép bằng Validating Admission Policy (Preventing Unauthorized ServiceCIDR Creation/Update using Validating Admission Policy)

Sẽ có những tình huống mà quản trị viên cluster muốn giới hạn các dải được phép dùng, hoặc
muốn chặn hoàn toàn mọi thay đổi đối với dải Service IP của cluster.

> **Ghi chú:** ServiceCIDR mặc định tên "kubernetes" được kube-apiserver tạo ra để bảo đảm
> tính nhất quán trong cluster và là thứ bắt buộc phải có để cluster hoạt động, nên nó luôn
> luôn phải được cho phép. Bạn có thể bảo đảm `ValidatingAdmissionPolicy` của mình không ràng
> buộc ServiceCIDR mặc định bằng cách thêm mệnh đề:
>
> ```yaml
>   matchConditions:
>   - name: 'exclude-default-servicecidr'
>     expression: "object.metadata.name != 'kubernetes'"
> ```
>
> giống như trong các ví dụ dưới đây.

#### Giới hạn dải Service CIDR trong một số dải cụ thể (Restrict Service CIDR ranges to some specific ranges)

Dưới đây là ví dụ về một `ValidatingAdmissionPolicy` chỉ cho phép tạo ServiceCIDR nếu chúng là
dải con của các dải nằm trong danh sách `allowed`. (Chẳng hạn, chính sách ví dụ này sẽ cho phép
một ServiceCIDR có `cidrs: ['10.96.1.0/24']` hoặc
`cidrs: ['2001:db8:0:0:ffff::/80', '10.96.0.0/20']`, nhưng sẽ không cho phép ServiceCIDR có
`cidrs: ['172.20.0.0/16']`.) Bạn có thể sao chép chính sách này và đổi giá trị của `allowed`
sang giá trị phù hợp với cluster của bạn.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: "servicecidrs.default"
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups:   ["networking.k8s.io"]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["servicecidrs"]
  matchConditions:
  - name: 'exclude-default-servicecidr'
    expression: "object.metadata.name != 'kubernetes'"
  variables:
  - name: allowed
    expression: "['10.96.0.0/16','2001:db8::/64']"
  validations:
  - expression: "object.spec.cidrs.all(newCIDR, variables.allowed.exists(allowedCIDR, cidr(allowedCIDR).containsCIDR(newCIDR)))"
  # Với mọi CIDR (newCIDR) được liệt kê trong spec.cidrs của đối tượng ServiceCIDR được gửi lên,
  # kiểm tra xem trong danh sách `allowed` của VAP có tồn tại ít nhất một CIDR (allowedCIDR)
  # chứa trọn vẹn newCIDR hay không.
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "servicecidrs-binding"
spec:
  policyName: "servicecidrs.default"
  validationActions: [Deny,Audit]
```

Hãy tham khảo [tài liệu về CEL](https://kubernetes.io/docs/reference/using-api/cel/) để tìm
hiểu thêm về CEL nếu bạn muốn tự viết `expression` kiểm tra của riêng mình.

#### Chặn mọi hoạt động sử dụng API ServiceCIDR (Restrict any usage of the ServiceCIDR API)

Ví dụ sau minh họa cách dùng một `ValidatingAdmissionPolicy` cùng binding của nó để chặn việc
tạo bất kỳ dải Service CIDR mới nào, ngoại trừ ServiceCIDR mặc định tên "kubernetes":

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: "servicecidrs.deny"
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups:   ["networking.k8s.io"]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["servicecidrs"]
  validations:
  - expression: "object.metadata.name == 'kubernetes'"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "servicecidrs-deny-binding"
spec:
  policyName: "servicecidrs.deny"
  validationActions: [Deny,Audit]
```

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 21:

1. Đối tượng `ServiceCIDR` tên `kubernetes` từ đâu mà có, và Service `kubernetes` chọn địa chỉ nào
   trong dải đó làm ClusterIP của mình?
2. Cluster lab dùng Service CIDR mặc định của kubeadm là `10.96.0.0/12`. Chạy
   `kubectl get servicecidr` trên `lab-k8s-master`, bạn kỳ vọng thấy tên gì và dải nào? Service
   `kubernetes` trong namespace `default` sẽ mang ClusterIP nào?
3. **Câu bẫy.** Cluster của bạn hết địa chỉ Service. Để có thêm chỗ, bạn có phải sửa cờ
   `--service-cluster-ip-range` của kube-apiserver rồi khởi động lại nó không?
4. Bạn `kubectl delete servicecidr newcidr1`, lệnh in ra "deleted", nhưng object vẫn còn khi
   `kubectl get`. Chuyện gì đang giữ nó lại, nhìn vào trường nào để biết chắc, và làm gì để nó
   thật sự biến mất?
5. Một cluster dùng dải `/28` cho Service thì tạo thêm được bao nhiêu Service, và thông báo lỗi
   khi hết dải là gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Nó do **kube-apiserver tạo ra**, mang tên phổ biến `kubernetes`, và chỉ định dải IP **dựa trên
   giá trị cờ `--service-cluster-ip-range`** truyền cho apiserver. Service `kubernetes` — thứ phơi
   bày endpoint của apiserver cho các Pod — **lấy địa chỉ IP đầu tiên trong dải** đó.
2. `kubectl get servicecidr` cho một dòng **`kubernetes`** với `CIDRS` là **`10.96.0.0/12`**.
   Service `kubernetes` mang ClusterIP **`10.96.0.1`** — địa chỉ đầu tiên của dải. Đây cũng chính
   là địa chỉ mà gate DNS trong cluster của Lab 00 kiểm chứng.
3. **Không.** Đó đúng là cách cũ, và bài nói rõ cách cũ **gây gián đoạn, thậm chí có thể mất dữ
   liệu**. Cách của bài này là **tạo thêm một đối tượng `ServiceCIDR` mới** — cluster nhận ngay
   dải bổ sung, kube-apiserver không phải đổi cờ, không phải khởi động lại. Sửa cờ apiserver là
   việc của bài [394](394-reconfigure-default-service-ip-ranges-vi.md), và đó là chuyện khác hẳn:
   **thay** dải mặc định chứ không **thêm** dải.
4. **Finalizer `networking.k8s.io/service-cidr-finalizer`** giữ nó lại, vì vẫn còn `IPAddress`
   tham chiếu tới ServiceCIDR đó. Nhìn `status.conditions`: `type: Ready`, `status: "False"`,
   `reason: OrphanIPAddress`, kèm `deletionTimestamp` đã được đặt. Xóa những Service đang giữ các
   địa chỉ đó thì control plane **tự gỡ finalizer** và object mới thực sự bị xóa.
5. `2^(32-28) - 2 = 14` địa chỉ, trừ đi `kubernetes.default` luôn tồn tại nên còn **13 Service**
   tạo thêm được. Khi hết, lệnh tạo Service trả về
   **`failed to allocate a serviceIP: range is full`**.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
