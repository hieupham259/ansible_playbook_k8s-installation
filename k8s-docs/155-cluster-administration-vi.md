# Quản trị cluster (Cluster Administration)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/cluster-administration/>
>
> Các chi tiết ở mức thấp hơn liên quan đến việc tạo hoặc quản trị một cluster Kubernetes.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 12](LO-TRINH-ADMIN.md#giai-đoạn-12--quản-trị-cluster-nâng-cao), bài 1/8 ·
Kiểm chứng ở Lab 12 (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Đây là **trang mục lục** của cả nhánh quản trị cluster trên kubernetes.io, gần như toàn bộ là
danh sách link. Giá trị của nó không nằm ở nội dung mà ở **cấu trúc**: biết công việc quản trị
được chia thành những nhóm nào, để khi gặp một câu hỏi vận hành bạn biết mở nhánh nào. Đọc như
đọc mục lục, đừng bấm hết link.

**Phải hiểu ở lần đọc này:**

- Bốn nhóm việc của người quản trị theo cách trang này chia: **lập kế hoạch cluster**, **quản lý
  cluster** (node, resource quota), **bảo mật cluster**, và **dịch vụ cluster tùy chọn** (DNS,
  logging/monitoring).
- Bộ câu hỏi phải trả lời **trước** khi chọn distro: thử nghiệm trên máy cá nhân hay HA nhiều
  node, hosted hay tự host, on-premises hay cloud (IaaS), bare metal hay VM, chỉ vận hành hay
  còn phát triển mã nguồn.
- Cảnh báo về distro: **không phải distro nào cũng còn được duy trì tích cực** — chọn cái đã được
  kiểm thử với một phiên bản Kubernetes gần đây.
- Ranh giới cần nhớ: **Kubernetes không hỗ trợ trực tiếp cluster lai (hybrid)** giữa on-premises
  và cloud; cách làm là dựng nhiều cluster.
- Mục *Bảo mật kubelet* được tách riêng khỏi phần bảo mật chung, và nó nối lại đúng ba việc bạn
  đã gặp: [giao tiếp control plane ↔ node](24-control-plane-node-communication-vi.md), TLS
  bootstrapping, và authn/authz cho kubelet.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Generate Certificates* trong *Bảo mật cluster* | quản lý vòng đời certificate là một module riêng, không gói trong một link | CP3 vòng đời chứng chỉ, bài [156](156-certificates-vi.md) |
| *Auditing* | audit policy và backend là chủ đề riêng, cần sửa cấu hình API server | CP7 audit và mã hóa dữ liệu |
| *Using Sysctls in a Kubernetes Cluster* | chỉnh tham số kernel chỉ cần cho workload đặc thù | CP5 cấu hình lại cluster đang chạy |
| *Admission Webhook Good Practices* | thiết kế webhook là việc mở rộng cluster, không phải vận hành | giai đoạn 14 (CRD và Operator) |
| Link *tự động mở rộng node* trong *Quản lý cluster* | có bài riêng ngay trong giai đoạn này | bài [171](171-node-autoscaling-vi.md) |

---

Trang tổng quan về quản trị cluster dành cho bất kỳ ai đang tạo hoặc quản trị một cluster
Kubernetes. Trang này giả định bạn đã có một số hiểu biết về các
[khái niệm](https://kubernetes.io/docs/concepts/) cốt lõi của Kubernetes.

## Lập kế hoạch cho cluster (Planning a cluster)

Xem các hướng dẫn trong mục [Setup](https://kubernetes.io/docs/setup/) để có các ví dụ về
cách lập kế hoạch, dựng và cấu hình cluster Kubernetes. Các giải pháp được liệt kê trong
bài viết này được gọi là *distro*.

> **Ghi chú:**
> Không phải distro nào cũng được duy trì (maintain) một cách tích cực. Hãy chọn những
> distro đã được kiểm thử với một phiên bản Kubernetes gần đây.

Trước khi chọn một hướng dẫn, dưới đây là một số điểm cần cân nhắc:

- Bạn muốn thử nghiệm Kubernetes trên máy tính của mình, hay muốn xây dựng một cluster
  nhiều node có tính sẵn sàng cao (high availability)? Hãy chọn distro phù hợp nhất với
  nhu cầu của bạn.
- Bạn sẽ sử dụng **một cluster Kubernetes được host sẵn (hosted)**, chẳng hạn như
  [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/), hay **tự host
  cluster của riêng mình**?
- Cluster của bạn sẽ đặt **tại chỗ (on-premises)** hay **trên cloud (IaaS)**? Kubernetes
  không hỗ trợ trực tiếp cluster lai (hybrid). Thay vào đó, bạn có thể dựng nhiều cluster.
- **Nếu bạn đang cấu hình Kubernetes on-premises**, hãy cân nhắc xem
  [mô hình mạng (networking model)](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
  nào phù hợp nhất.
- Bạn sẽ chạy Kubernetes trên **phần cứng "bare metal"** hay trên **máy ảo (VM)**?
- Bạn **muốn vận hành một cluster**, hay dự định **tham gia phát triển mã nguồn của dự án
  Kubernetes**? Nếu là trường hợp sau, hãy chọn một distro đang được phát triển tích cực.
  Một số distro chỉ dùng bản phát hành nhị phân (binary release), nhưng lại cung cấp nhiều
  lựa chọn đa dạng hơn.
- Làm quen với các [thành phần (components)](https://kubernetes.io/docs/concepts/overview/components/)
  cần thiết để vận hành một cluster.

## Quản lý cluster (Managing a cluster)

* Tìm hiểu cách [quản lý node](https://kubernetes.io/docs/concepts/architecture/nodes/).
  * Đọc về [tự động mở rộng node (Node autoscaling)](https://kubernetes.io/docs/concepts/cluster-administration/node-autoscaling/).

* Tìm hiểu cách thiết lập và quản lý [resource quota](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
  cho các cluster dùng chung.

## Bảo mật cluster (Securing a cluster)

* [Generate Certificates](https://kubernetes.io/docs/tasks/administer-cluster/certificates/)
  mô tả các bước tạo certificate bằng những bộ công cụ (tool chain) khác nhau.

* [Kubernetes Container Environment](https://kubernetes.io/docs/concepts/containers/container-environment/)
  mô tả môi trường dành cho các container do Kubelet quản lý trên một node Kubernetes.

* [Controlling Access to the Kubernetes API](https://kubernetes.io/docs/concepts/security/controlling-access)
  mô tả cách Kubernetes triển khai kiểm soát truy cập (access control) cho chính API của nó.

* [Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
  giải thích việc xác thực (authentication) trong Kubernetes, bao gồm các tùy chọn xác thực
  khác nhau.

* [Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
  là bước phân quyền (authorization), tách biệt với xác thực, và kiểm soát cách các lời gọi
  HTTP được xử lý.

* [Using Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
  giải thích các plug-in chặn (intercept) các request tới Kubernetes API server sau bước
  xác thực và phân quyền.

* [Admission Webhook Good Practices](https://kubernetes.io/docs/concepts/cluster-administration/admission-webhooks-good-practices/)
  cung cấp các thực hành tốt và những điểm cần cân nhắc khi thiết kế mutating admission
  webhook và validating admission webhook.

* [Using Sysctls in a Kubernetes Cluster](https://kubernetes.io/docs/tasks/administer-cluster/sysctl-cluster/)
  mô tả cho quản trị viên cách sử dụng công cụ dòng lệnh `sysctl` để thiết lập các tham số
  kernel.

* [Auditing](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/) mô tả cách làm
  việc với log kiểm toán (audit log) của Kubernetes.

### Bảo mật kubelet (Securing the kubelet)

* [Giao tiếp giữa Control Plane và Node](./24-control-plane-node-communication-vi.md)
* [TLS bootstrapping](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/)
* [Xác thực/phân quyền cho kubelet](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/)

## Các dịch vụ cluster tùy chọn (Optional Cluster Services)

* [DNS Integration](./10-dns-pod-service-vi.md) mô tả cách phân giải một tên DNS trực tiếp
  thành một service của Kubernetes.

* [Logging and Monitoring Cluster Activity](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
  giải thích cách hoạt động của logging trong Kubernetes và cách triển khai nó.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 12:

1. Cluster lab của bạn là một control plane `k8s-master` cộng hai worker, chạy trên VM tự dựng
   trong nhà. Chiếu vào danh sách cân nhắc ở mục *Lập kế hoạch cho cluster*, bạn đã chọn phương
   án nào ở từng câu hỏi?
2. **Câu bẫy.** Sếp muốn một cluster duy nhất có worker nằm ở phòng máy công ty và worker nằm
   trên cloud, để "vừa tiết kiệm vừa co giãn được". Trang này nói gì về ý tưởng đó?
3. Bạn cần ba thứ sau, mỗi thứ tìm ở nhóm nào của trang này: giới hạn tài nguyên cho một
   namespace dùng chung; cách kubelet tự xin certificate; cách phân giải tên DNS thành Service?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Tự host** (không dùng cluster hosted kiểu GKE), **on-premises** chứ không phải cloud IaaS,
   chạy trên **máy ảo** chứ không phải bare metal, là cluster **nhiều node nhưng không HA** (một
   control plane duy nhất), và mục đích là **vận hành** chứ không phải phát triển mã nguồn dự án
   Kubernetes. Vì on-premises nên câu hỏi về **mô hình mạng** cũng thuộc về bạn — và đó là lựa
   chọn Flannel đã được chốt từ Lab 00.
2. Trang này nói thẳng: **Kubernetes không hỗ trợ trực tiếp cluster lai (hybrid)**. Cách làm đúng
   là **dựng nhiều cluster**. Trực giác sai vì Kubernetes vốn được quảng bá như thứ chạy được ở
   mọi nơi — nhưng "chạy được ở mọi nơi" nghĩa là dựng được cluster ở mọi nơi, không có nghĩa là
   một cluster trải ngang nhiều nơi.
3. Resource quota nằm ở nhóm **Quản lý cluster**; **TLS bootstrapping** nằm ở nhóm **Bảo mật
   kubelet** trong phần *Bảo mật cluster*; **DNS Integration** nằm ở nhóm **Các dịch vụ cluster
   tùy chọn**. Đó chính là công dụng duy nhất nhưng thật sự của một trang mục lục: rút ngắn quãng
   đường từ câu hỏi tới đúng tài liệu.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
