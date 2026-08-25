# Dual-stack IPv4/IPv6 (IPv4/IPv6 dual-stack)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/services-networking/dual-stack/>
>
> Kubernetes cho phép bạn cấu hình mạng single-stack IPv4, mạng single-stack IPv6,
> hoặc mạng dual-stack với cả hai họ địa chỉ mạng cùng hoạt động. Trang này giải thích cách làm.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 5](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng), bài 13/16 · Kiểm chứng
ở [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md).

Cluster lab là **single-stack IPv4** (Pod CIDR `10.244.0.0/16`, Service CIDR `10.96.0.0/12`),
nên bạn không dựng được dual-stack ở đây. Đọc bài này vì hai lý do rất thực tế: nó giải thích
tại sao Service có trường `clusterIPs` **số nhiều** mà bạn đã thấy trong `kubectl get svc -o
yaml`, và nó cho bạn dấu hiệu để nhận ra một cluster dual-stack khi gặp.

**Phải hiểu ở lần đọc này:**

- Dual-stack = cấp phát **đồng thời** IPv4 và IPv6 cho **Pod và Service**; được bật mặc định từ
  1.21, nhưng vẫn cần nhà cung cấp và **network plugin có hỗ trợ** thì mới dùng được.
- Bốn thành phần phải được cấu hình khớp nhau: kube-apiserver (`--service-cluster-ip-range`),
  kube-controller-manager (`--cluster-cidr` và `--service-cluster-ip-range`), kube-proxy
  (`--cluster-cidr`), kubelet (`--node-ip`, **bắt buộc** với node bare metal không khai cloud
  provider).
- Ba giá trị `.spec.ipFamilyPolicy`: `SingleStack`; `PreferDualStack` — **quay về single-stack**
  nếu cluster không bật dual-stack; `RequireDualStack` — **việc tạo Service qua API sẽ thất bại**
  nếu cluster không bật dual-stack.
- Quan hệ giữa `.spec.clusterIPs` (trường **chính**, dạng mảng) và `.spec.clusterIP` (trường
  **thứ cấp**, tính từ mảng theo phần tử đầu của `.spec.ipFamilies`). `.spec.ipFamilies` chỉ
  thay đổi **có điều kiện**: thêm hoặc bớt họ thứ cấp thì được, **đổi họ chính của một Service
  đang tồn tại thì không**.
- Một ngoại lệ dễ quên: **headless Service không có selector** và không đặt tường minh
  `.spec.ipFamilyPolicy` thì mặc định là **`RequireDualStack`**, khác với mặc định `SingleStack`
  của mọi trường hợp còn lại.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Các kịch bản cấu hình Service dual-stack* — từng ví dụ YAML một | chỉ quan sát được trên cluster đã bật dual-stack | không cần |
| *Traffic chiều đi* — IP masquerading cho IPv6 không định tuyến công khai được | cần địa chỉ IPv6 thật ngoài lab | không cần |
| *Hỗ trợ Windows* (`l2bridge`, overlay VXLAN không hỗ trợ) | lab không có node Windows | giai đoạn 15 |
| Link *Bật mạng dual-stack bằng kubeadm* ở mục *Tiếp theo* | là thao tác dựng cluster | giai đoạn 8, bài [05](05-dual-stack-support-vi.md) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.23 [stable]`

Mạng dual-stack IPv4/IPv6 cho phép cấp phát đồng thời cả địa chỉ IPv4 và IPv6 cho các
Pod và Service.

Mạng dual-stack IPv4/IPv6 được bật theo mặc định cho cluster Kubernetes của bạn kể từ phiên bản
1.21, cho phép gán đồng thời cả địa chỉ IPv4 và IPv6.

## Các tính năng được hỗ trợ (Supported Features)

Dual-stack IPv4/IPv6 trên cluster Kubernetes của bạn cung cấp các tính năng sau:

* Mạng Pod dual-stack (gán một địa chỉ IPv4 và một địa chỉ IPv6 cho mỗi Pod)
* Service hỗ trợ IPv4 và IPv6
* Định tuyến egress của Pod ra ngoài cluster (ví dụ Internet) qua cả giao diện (interface) IPv4 lẫn IPv6

## Điều kiện tiên quyết (Prerequisites)

Cần có các điều kiện tiên quyết sau để sử dụng cluster Kubernetes dual-stack IPv4/IPv6:

* Kubernetes 1.20 trở lên

  Để biết thông tin về việc sử dụng service dual-stack với các phiên bản Kubernetes
  cũ hơn, hãy tham khảo tài liệu của phiên bản Kubernetes đó.

* Nhà cung cấp có hỗ trợ mạng dual-stack (nhà cung cấp cloud hoặc nhà cung cấp khác phải có khả
  năng cung cấp cho các node Kubernetes những giao diện mạng IPv4/IPv6 định tuyến được)
* Một [network plugin](183-network-plugins-vi.md)
  có hỗ trợ mạng dual-stack.

## Cấu hình dual-stack IPv4/IPv6 (Configure IPv4/IPv6 dual-stack) {#configure-ipv4-ipv6-dual-stack}

Để cấu hình dual-stack IPv4/IPv6, hãy đặt các thiết lập gán mạng dual-stack cho cluster:

* kube-apiserver:
  * `--service-cluster-ip-range=<IPv4 CIDR>,<IPv6 CIDR>`
* kube-controller-manager:
  * `--cluster-cidr=<IPv4 CIDR>,<IPv6 CIDR>`
  * `--service-cluster-ip-range=<IPv4 CIDR>,<IPv6 CIDR>`
  * `--node-cidr-mask-size-ipv4|--node-cidr-mask-size-ipv6` mặc định là /24 cho IPv4 và /64 cho IPv6
* kube-proxy:
  * `--cluster-cidr=<IPv4 CIDR>,<IPv6 CIDR>`
* kubelet:
  * `--node-ip=<IPv4 IP>,<IPv6 IP>`
    * Tùy chọn này là bắt buộc đối với các node dual-stack bare metal (các node không định
      nghĩa nhà cung cấp cloud bằng flag `--cloud-provider`). Nếu bạn đang dùng một nhà cung
      cấp cloud và muốn ghi đè các IP node do nhà cung cấp cloud chọn, hãy đặt tùy chọn
      `--node-ip`.
    * (Các nhà cung cấp cloud tích hợp sẵn kiểu cũ (legacy built-in) không hỗ trợ `--node-ip`
      dual-stack.)

> **Ghi chú:**
> Một ví dụ về IPv4 CIDR: `10.244.0.0/16` (dù bạn sẽ cung cấp dải địa chỉ của riêng mình)
>
> Một ví dụ về IPv6 CIDR: `fdXY:IJKL:MNOP:15::/64` (ví dụ này minh họa định dạng chứ không phải
> là một địa chỉ hợp lệ — xem [RFC 4193](https://tools.ietf.org/html/rfc4193))

## Service (Services)

Bạn có thể tạo các Service có thể dùng IPv4, IPv6, hoặc cả hai.

Họ địa chỉ (address family) của một Service mặc định là họ địa chỉ của dải cluster IP đầu tiên
dành cho service (được cấu hình qua flag `--service-cluster-ip-range` của kube-apiserver).

Khi định nghĩa một Service, bạn có thể tùy chọn cấu hình nó là dual-stack. Để chỉ định hành vi
mong muốn, bạn đặt trường `.spec.ipFamilyPolicy` thành một trong các giá trị sau:

* `SingleStack`: Service single-stack. Control plane cấp phát một cluster IP cho Service,
  sử dụng dải cluster IP đầu tiên dành cho service đã được cấu hình.
* `PreferDualStack`: Cấp phát cả cluster IP IPv4 lẫn IPv6 cho Service khi dual-stack được bật.
  Nếu dual-stack không được bật hoặc không được hỗ trợ, nó quay về (fall back) hành vi single-stack.
* `RequireDualStack`: Cấp phát `.spec.clusterIPs` của Service từ cả dải địa chỉ IPv4 lẫn IPv6
  khi dual-stack được bật. Nếu dual-stack không được bật hoặc không được hỗ trợ, việc tạo
  object Service qua API sẽ thất bại.
  * Chọn `.spec.clusterIP` từ danh sách `.spec.clusterIPs` dựa trên họ địa chỉ của phần tử
    đầu tiên trong mảng `.spec.ipFamilies`.

Nếu bạn muốn xác định họ IP nào được dùng cho single-stack hoặc xác định thứ tự của các họ IP
cho dual-stack, bạn có thể chọn các họ địa chỉ bằng cách đặt một trường tùy chọn,
`.spec.ipFamilies`, trên Service.

> **Ghi chú:**
> Trường `.spec.ipFamilies` chỉ có thể thay đổi có điều kiện (conditionally mutable): bạn có thể
> thêm hoặc bớt họ địa chỉ IP thứ cấp, nhưng không thể thay đổi họ địa chỉ IP chính của một
> Service đang tồn tại.

Bạn có thể đặt `.spec.ipFamilies` thành bất kỳ giá trị mảng nào sau đây:

- `["IPv4"]`
- `["IPv6"]`
- `["IPv4","IPv6"]` (dual stack)
- `["IPv6","IPv4"]` (dual stack)

Họ đầu tiên bạn liệt kê được dùng cho trường `.spec.clusterIP` kiểu cũ (legacy).

### Các kịch bản cấu hình Service dual-stack (Dual-stack Service configuration scenarios)

Các ví dụ sau minh họa hành vi của nhiều kịch bản cấu hình Service dual-stack khác nhau.

#### Các tùy chọn dual-stack trên Service mới (Dual-stack options on new Services)

1. Đặc tả Service này không định nghĩa tường minh `.spec.ipFamilyPolicy`. Khi bạn tạo Service
   này, Kubernetes gán một cluster IP cho Service từ dải `service-cluster-ip-range` đầu tiên
   đã cấu hình và đặt `.spec.ipFamilyPolicy` thành `SingleStack`. ([Service không có
   selector](82-service-vi.md#services-without-selectors) và
   [headless Service](82-service-vi.md#headless-services) có selector
   cũng sẽ hành xử theo cùng cách này.)

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: my-service
     labels:
       app.kubernetes.io/name: MyApp
   spec:
     selector:
       app.kubernetes.io/name: MyApp
     ports:
       - protocol: TCP
         port: 80
   ```

1. Đặc tả Service này định nghĩa tường minh `PreferDualStack` trong `.spec.ipFamilyPolicy`. Khi
   bạn tạo Service này trên một cluster dual-stack, Kubernetes gán cả địa chỉ IPv4 lẫn IPv6
   cho service. Control plane cập nhật `.spec` của Service để ghi lại các địa chỉ IP đã gán.
   Trường `.spec.clusterIPs` là trường chính, chứa cả hai địa chỉ IP đã gán; `.spec.clusterIP`
   là trường thứ cấp với giá trị được tính từ `.spec.clusterIPs`.

   * Đối với trường `.spec.clusterIP`, control plane ghi lại địa chỉ IP thuộc cùng họ địa chỉ
     với dải cluster IP đầu tiên dành cho service.
   * Trên một cluster single-stack, cả hai trường `.spec.clusterIPs` và `.spec.clusterIP` đều
     chỉ liệt kê một địa chỉ.
   * Trên một cluster đã bật dual-stack, việc chỉ định `RequireDualStack` trong
     `.spec.ipFamilyPolicy` hành xử giống như `PreferDualStack`.

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: my-service
     labels:
       app.kubernetes.io/name: MyApp
   spec:
     ipFamilyPolicy: PreferDualStack
     selector:
       app.kubernetes.io/name: MyApp
     ports:
       - protocol: TCP
         port: 80
   ```

1. Đặc tả Service này định nghĩa tường minh `IPv6` và `IPv4` trong `.spec.ipFamilies` đồng thời
   định nghĩa `PreferDualStack` trong `.spec.ipFamilyPolicy`. Khi Kubernetes gán một địa chỉ
   IPv6 và một địa chỉ IPv4 trong `.spec.clusterIPs`, `.spec.clusterIP` được đặt thành địa chỉ
   IPv6 vì đó là phần tử đầu tiên trong mảng `.spec.clusterIPs`, ghi đè giá trị mặc định.

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: my-service
     labels:
       app.kubernetes.io/name: MyApp
   spec:
     ipFamilyPolicy: PreferDualStack
     ipFamilies:
     - IPv6
     - IPv4
     selector:
       app.kubernetes.io/name: MyApp
     ports:
       - protocol: TCP
         port: 80
   ```

#### Hành vi mặc định của dual-stack trên Service đang tồn tại (Dual-stack defaults on existing Services)

Các ví dụ sau minh họa hành vi mặc định khi dual-stack vừa được bật trên một cluster đã có sẵn
các Service. (Nâng cấp một cluster hiện có lên 1.21 trở lên sẽ bật dual-stack.)

1. Khi dual-stack được bật trên một cluster, các Service đang tồn tại (dù là `IPv4` hay `IPv6`)
   được control plane cấu hình để đặt `.spec.ipFamilyPolicy` thành `SingleStack` và đặt
   `.spec.ipFamilies` thành họ địa chỉ của Service hiện có. Cluster IP hiện có của Service sẽ
   được lưu trong `.spec.clusterIPs`.

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: my-service
     labels:
       app.kubernetes.io/name: MyApp
   spec:
     selector:
       app.kubernetes.io/name: MyApp
     ports:
       - protocol: TCP
         port: 80
   ```

   Bạn có thể xác nhận hành vi này bằng cách dùng kubectl để kiểm tra một service đang tồn tại.

   ```shell
   kubectl get svc my-service -o yaml
   ```

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     labels:
       app.kubernetes.io/name: MyApp
     name: my-service
   spec:
     clusterIP: 10.0.197.123
     clusterIPs:
     - 10.0.197.123
     ipFamilies:
     - IPv4
     ipFamilyPolicy: SingleStack
     ports:
     - port: 80
       protocol: TCP
       targetPort: 80
     selector:
       app.kubernetes.io/name: MyApp
     type: ClusterIP
   status:
     loadBalancer: {}
   ```

1. Khi dual-stack được bật trên một cluster, các
   [headless Service](82-service-vi.md#headless-services) có selector
   đang tồn tại được control plane cấu hình để đặt `.spec.ipFamilyPolicy` thành `SingleStack`
   và đặt `.spec.ipFamilies` thành họ địa chỉ của dải cluster IP đầu tiên dành cho service
   (được cấu hình qua flag `--service-cluster-ip-range` của kube-apiserver) mặc dù
   `.spec.clusterIP` được đặt là `None`.

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: my-service
     labels:
       app.kubernetes.io/name: MyApp
   spec:
     selector:
       app.kubernetes.io/name: MyApp
     ports:
       - protocol: TCP
         port: 80
   ```

   Bạn có thể xác nhận hành vi này bằng cách dùng kubectl để kiểm tra một headless service có
   selector đang tồn tại.

   ```shell
   kubectl get svc my-service -o yaml
   ```

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     labels:
       app.kubernetes.io/name: MyApp
     name: my-service
   spec:
     clusterIP: None
     clusterIPs:
     - None
     ipFamilies:
     - IPv4
     ipFamilyPolicy: SingleStack
     ports:
     - port: 80
       protocol: TCP
       targetPort: 80
     selector:
       app.kubernetes.io/name: MyApp
   ```

#### Chuyển đổi Service giữa single-stack và dual-stack (Switching Services between single-stack and dual-stack)

Các Service có thể được thay đổi từ single-stack sang dual-stack và từ dual-stack về single-stack.

1. Để thay đổi một Service từ single-stack sang dual-stack, hãy đổi `.spec.ipFamilyPolicy` từ
   `SingleStack` thành `PreferDualStack` hoặc `RequireDualStack` tùy nhu cầu. Khi bạn thay đổi
   Service này từ single-stack sang dual-stack, Kubernetes gán họ địa chỉ còn thiếu để Service
   giờ đây có cả địa chỉ IPv4 lẫn IPv6.

   Chỉnh sửa đặc tả Service, cập nhật `.spec.ipFamilyPolicy` từ `SingleStack` thành `PreferDualStack`.

   Trước:

   ```yaml
   spec:
     ipFamilyPolicy: SingleStack
   ```

   Sau:

   ```yaml
   spec:
     ipFamilyPolicy: PreferDualStack
   ```

1. Để thay đổi một Service từ dual-stack về single-stack, hãy đổi `.spec.ipFamilyPolicy` từ
   `PreferDualStack` hoặc `RequireDualStack` thành `SingleStack`. Khi bạn thay đổi Service này
   từ dual-stack về single-stack, Kubernetes chỉ giữ lại phần tử đầu tiên trong mảng
   `.spec.clusterIPs`, đặt `.spec.clusterIP` thành địa chỉ IP đó và đặt `.spec.ipFamilies`
   thành họ địa chỉ của `.spec.clusterIPs`.

### Headless Service không có selector (Headless Services without selector)

Đối với [Headless Service không có selector](82-service-vi.md#without-selectors)
và không đặt tường minh `.spec.ipFamilyPolicy`, trường `.spec.ipFamilyPolicy` mặc định là
`RequireDualStack`.

### Service kiểu LoadBalancer (Service type LoadBalancer)

Để cấp phát (provision) một bộ cân bằng tải (load balancer) dual-stack cho Service của bạn:

* Đặt trường `.spec.type` thành `LoadBalancer`
* Đặt trường `.spec.ipFamilyPolicy` thành `PreferDualStack` hoặc `RequireDualStack`

> **Ghi chú:**
> Để dùng Service kiểu `LoadBalancer` dual-stack, nhà cung cấp cloud của bạn phải hỗ trợ
> bộ cân bằng tải IPv4 và IPv6.

## Traffic chiều đi (Egress traffic)

Nếu bạn muốn bật traffic chiều đi (egress) để tiếp cận các đích ngoài cluster (ví dụ Internet
công cộng) từ một Pod dùng địa chỉ IPv6 không định tuyến công khai được, bạn cần cho phép Pod
đó sử dụng một địa chỉ IPv6 được định tuyến công khai thông qua một cơ chế như proxy trong suốt
(transparent proxying) hoặc giả trang IP (IP masquerading). Dự án
[ip-masq-agent](https://github.com/kubernetes-sigs/ip-masq-agent) hỗ trợ IP masquerading trên
các cluster dual-stack.

> **Ghi chú:**
> Hãy bảo đảm nhà cung cấp CNI của bạn hỗ trợ IPv6.

## Hỗ trợ Windows (Windows support) {#windows-support}

Kubernetes trên Windows không hỗ trợ mạng single-stack "chỉ IPv6" (IPv6-only). Tuy nhiên,
mạng dual-stack IPv4/IPv6 cho các pod và node với service một họ địa chỉ (single-family)
thì được hỗ trợ.

Bạn có thể dùng mạng dual-stack IPv4/IPv6 với các mạng `l2bridge`.

> **Ghi chú:**
> Các mạng Overlay (VXLAN) trên Windows **không** hỗ trợ mạng dual-stack.

Bạn có thể đọc thêm về các chế độ mạng khác nhau cho Windows trong chủ đề
[Mạng trên Windows](89-windows-networking-vi.md#network-modes).

## Tiếp theo (What's next)

* [Kiểm chứng mạng dual-stack IPv4/IPv6](395-validate-dual-stack-vi.md)
* [Bật mạng dual-stack bằng kubeadm](./05-dual-stack-support-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 5:

1. Cluster lab chỉ cấu hình một dải Service IPv4 (`10.96.0.0/12`). Bạn tạo một Service với
   `ipFamilyPolicy: RequireDualStack` thì kết quả ra sao? Còn `PreferDualStack`?
2. Trên một cluster single-stack, `.spec.clusterIPs` của một Service chứa mấy phần tử, và
   `.spec.clusterIP` có bị bỏ trống không?
3. Một headless Service **không có selector** và không đặt `.spec.ipFamilyPolicy` thì mặc định
   là gì? So với một Service thường trong cùng tình huống?
4. Bạn muốn đổi họ IP **chính** của một Service đang chạy từ IPv4 sang IPv6. Làm được không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. `RequireDualStack`: **việc tạo object Service qua API sẽ thất bại** — bài nói rõ nếu
   dual-stack không được bật hoặc không được hỗ trợ thì API từ chối. `PreferDualStack`: Service
   **vẫn tạo được**, nó **quay về hành vi single-stack** và nhận một cluster IP IPv4. Đây là
   khác biệt then chốt giữa "ưu tiên" và "bắt buộc".
2. **Một phần tử**, và `.spec.clusterIP` **không** bị bỏ trống. Bài nói: trên một cluster
   single-stack, cả `.spec.clusterIPs` lẫn `.spec.clusterIP` **đều chỉ liệt kê một địa chỉ**.
   Cần nhớ đúng vai: `clusterIPs` là trường chính, `clusterIP` là trường thứ cấp có giá trị được
   tính từ `clusterIPs` theo phần tử đầu của `.spec.ipFamilies`.
3. Mặc định của nó là **`RequireDualStack`** — đây là ngoại lệ duy nhất trong bài. Mọi trường
   hợp còn lại, kể cả Service thường và headless Service **có** selector, được control plane đặt
   `.spec.ipFamilyPolicy` thành **`SingleStack`** khi bạn không khai tường minh. Trực giác "mặc
   định thì luôn là SingleStack" sẽ dẫn bạn đi sai đúng ở chỗ này.
4. **Không.** `.spec.ipFamilies` chỉ **thay đổi có điều kiện**: bạn có thể thêm hoặc bớt **họ
   địa chỉ IP thứ cấp**, nhưng **không thể thay đổi họ địa chỉ IP chính của một Service đang tồn
   tại**. Muốn đổi họ chính thì phải tạo Service mới.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
