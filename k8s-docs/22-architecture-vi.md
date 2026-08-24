# Kiến trúc cluster (Cluster Architecture)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/architecture/>
>
> Các khái niệm kiến trúc đằng sau Kubernetes.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 1 → nhóm [1a](00-ALO-TRINH-ADMIN.md#1a-kiến-trúc-và-mô-hình-điều-khiển),
bài 3/8 · Kiểm chứng ở [Lab 1a](labs/LAB-1A-KIEN-TRUC-VA-MO-HINH-DIEU-KHIEN.md) phần B2 và B3.

Đây là bài dài nhất nhóm 1a. Khoảng một phần ba nội dung nói về control plane HA — thứ thuộc
giai đoạn 8. **Đừng cố hiểu hết trong một lần đọc.**

**Phải hiểu ở lần đọc này:**

- Phân biệt **control plane** (lớp chức năng) với **control-plane node** (cái máy) — mục
  *Phân biệt control plane và control-plane node*. Đây là chỗ dễ nhầm nhất.
- Cơ chế **"một cửa"**: scheduler không gọi kubelet, kubelet không chờ ai ra lệnh; mọi phối
  hợp đều là ghi và đọc qua API server. Đây là ý cốt lõi nhất của cả nhóm 1a.
- Vì sao control-plane node dựng bằng kubeadm vẫn cần kubelet và container runtime, và vì sao
  điều đó **không** biến chúng thành control plane component.
- Vai trò của từng thành phần: `kube-apiserver`, `etcd`, `kube-scheduler`,
  `kube-controller-manager`, `kubelet`, `kube-proxy`, container runtime.
- Chỉ API server nói chuyện trực tiếp với etcd.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Stacked etcd vs external etcd | là quyết định topology khi dựng HA | giai đoạn 8 |
| *Nhiều bản kube-apiserver sau một load balancer*, VIP, kube-vip, leader election | cần hiểu quy trình dựng HA trước | giai đoạn 8 |
| Mục so sánh với Service `type: LoadBalancer` | cần biết Service là gì | giai đoạn 5 |
| Static Pod (chi tiết) | bài riêng, chỉ cần hiểu tối thiểu ở đây | giai đoạn 3, bài [58](58-static-pods-vi.md) |
| Addon: dashboard, monitoring, logging | chủ đề observability | giai đoạn 11 |
| *Các biến thể kiến trúc* | so sánh cách triển khai | giai đoạn 8, 12, 14 |

Từ mục HA, lần này chỉ cần mang đi một ý: **API server không giữ trạng thái nên nhân bản
thoải mái, còn etcd thì không** — đó là lý do etcd quyết định giới hạn quy mô cluster.

---

Một cluster Kubernetes bao gồm một control plane cùng với một tập các máy worker, gọi là các node,
chuyên chạy các ứng dụng đóng gói trong container (containerized applications). Mỗi cluster cần
ít nhất một worker node để chạy các Pod.

Các worker node lưu trữ (host) các Pod — những thành phần cấu thành workload của ứng dụng.
Control plane quản lý các worker node và các Pod trong cluster. Trong môi trường
production, control plane thường chạy trên nhiều máy tính và cluster
thường chạy nhiều node, nhằm cung cấp khả năng chịu lỗi (fault-tolerance) và tính sẵn sàng cao (high availability).

Tài liệu này phác thảo các thành phần bạn cần có để có một cluster Kubernetes hoàn chỉnh và hoạt động được.

![Control plane (kube-apiserver, etcd, kube-controller-manager, kube-scheduler) và một số node. Mỗi node chạy một kubelet và kube-proxy.](https://kubernetes.io/images/docs/kubernetes-cluster-architecture.svg)

*Hình 1. Các thành phần của cluster Kubernetes.*

> **Về kiến trúc này (About this architecture)**
>
> Sơ đồ trong Hình 1 trình bày một kiến trúc tham chiếu ví dụ cho một cluster Kubernetes.
> Cách phân bố thực tế của các thành phần có thể khác nhau tùy theo cách thiết lập và yêu cầu
> cụ thể của từng cluster.
>
> Trong sơ đồ, mỗi node đều chạy thành phần [`kube-proxy`](#kube-proxy). Bạn cần một
> thành phần network proxy trên mỗi node để đảm bảo API Service và các hành vi liên quan
> khả dụng trên mạng của cluster. Tuy nhiên, một số network plugin tự cung cấp
> hiện thực proxy riêng của bên thứ ba. Khi bạn dùng loại network plugin đó,
> node không cần chạy `kube-proxy`.

## Các thành phần của control plane (Control plane components) {#control-plane-components}

Các thành phần của control plane đưa ra những quyết định toàn cục về cluster (ví dụ: lập lịch — scheduling),
đồng thời phát hiện và phản ứng với các sự kiện của cluster (ví dụ: khởi động một
pod mới khi trường `replicas` của một Deployment chưa được thỏa mãn).

Các thành phần control plane có thể chạy trên bất kỳ máy nào trong cluster. Tuy nhiên, để đơn giản,
các script cài đặt thường khởi động tất cả các thành phần control plane trên cùng một máy,
và không chạy container của người dùng trên máy này.
Xem [Tạo cluster có tính sẵn sàng cao với kubeadm](./08-high-availability-vi.md)
để có ví dụ về một thiết lập control plane chạy trên nhiều máy.

### Phân biệt control plane và control-plane node {#control-plane-vs-control-plane-node}

Hai thuật ngữ này mô tả hai mức khác nhau, không phải hai bộ phận ngang hàng của cluster:

- **Control plane** là lớp chức năng, gồm tập hợp các thành phần phần mềm điều khiển cluster:
  API server, etcd, scheduler và controller manager.
- **Control-plane node** là thuật ngữ triển khai, chỉ một máy vật lý hoặc VM được chọn để
  chạy một hay nhiều thành phần control plane. Với kubeadm, máy này còn được đăng ký và
  đánh dấu là một Node có vai trò control plane.

Có thể liên hệ với cơ sở dữ liệu:

```text
Database
    = phần mềm và chức năng cơ sở dữ liệu

Database server
    = máy được chọn để chạy phần mềm cơ sở dữ liệu
```

Tương ứng trong Kubernetes:

```text
Control plane
    = tập hợp thành phần điều khiển cluster

Control-plane node
    = máy được chọn để chạy những thành phần đó
```

Vì vậy, nói “control plane chạy trên ba máy” nghĩa là các thành phần hoặc các instance của
control plane được bố trí trên ba control-plane node. Nó không có nghĩa control plane là một
máy duy nhất được chia thành ba phần.

#### Cách kubeadm bố trí control plane

Về mặt kiến trúc tổng quát, các thành phần control plane có thể được phân bố theo nhiều cách.
Tuy nhiên, khi dựng control plane có tính sẵn sàng cao bằng kubeadm, mỗi control-plane node
thường chạy một bộ gồm:

```text
kube-apiserver
kube-scheduler
kube-controller-manager
```

Nếu dùng mô hình **stacked etcd**, mỗi control-plane node còn chạy một etcd member. Ví dụ:

```text
cp1: API server + scheduler + controller manager + etcd member 1
cp2: API server + scheduler + controller manager + etcd member 2
cp3: API server + scheduler + controller manager + etcd member 3
```

Các API server có thể cùng nhận request qua một load balancer. Các scheduler và controller
manager dùng leader election để một instance giữ vai trò leader tại một thời điểm. Các etcd
member hợp thành một etcd cluster.

Nếu dùng **external etcd**, các control-plane node vẫn chạy API server, scheduler và controller
manager, còn etcd chạy trên một nhóm máy riêng:

```text
cp1, cp2, cp3       : API server + scheduler + controller manager
etcd1, etcd2, etcd3 : external etcd cluster
```

Máy external-etcd không bắt buộc là Kubernetes Node. Vì vậy, chúng không bắt buộc phải có
kubelet hoặc kube-proxy; container runtime cũng chỉ cần nếu cách triển khai etcd thực sự chạy
etcd trong container.

#### Vì sao control-plane node của kubeadm có kubelet và container runtime?

Trong cách triển khai của kubeadm, các thành phần control plane được chạy dưới dạng
**static Pod**. Ở đây chỉ cần hiểu tối thiểu: static Pod là Pod do kubelet trên một máy cụ thể
khởi động và quản lý trực tiếp, không cần scheduler chọn node cho nó. Cơ chế này được giải
thích đầy đủ ở bài [Pod tĩnh](./58-static-pods-vi.md).

Kubeadm ghi manifest của các thành phần control plane vào:

```text
/etc/kubernetes/manifests/
├── etcd.yaml                       # chỉ khi dùng local/stacked etcd
├── kube-apiserver.yaml
├── kube-controller-manager.yaml
└── kube-scheduler.yaml
```

Kubelet trên control-plane node theo dõi thư mục này. Khi thấy manifest, kubelet yêu cầu
container runtime chạy các container tương ứng. Trong cluster lab dùng `containerd`, đường đi
có thể hình dung như sau:

```text
kubeadm ghi manifest
        ↓
/etc/kubernetes/manifests/
        ↓
kubelet trên k8s-master đọc manifest
        ↓
kubelet yêu cầu containerd chạy container
        ↓
API server / etcd / scheduler / controller manager
```

Do đó, các control-plane node do kubeadm quản lý cần có **kubelet** và **container runtime**.
Kubeadm mặc định triển khai `kube-proxy`; còn CNI được người quản trị cài sau khi khởi tạo
control plane và thường cũng chạy thành phần cần thiết trên các node này. Tuy nhiên, cần phân biệt:

| Thành phần | Trạng thái trên control-plane node dùng kubeadm |
|---|---|
| `kubelet` | Cần cho cách kubeadm vận hành các static Pod control plane |
| Container runtime | Cần để thực thi container của các thành phần control plane |
| `kube-proxy` | Kubeadm mặc định thường triển khai; có thể được thay thế hoặc loại bỏ bởi giải pháp mạng khác |
| CNI | Thường hiện diện để cung cấp mạng Pod; cách triển khai phụ thuộc network plugin |

Việc **kubelet, container runtime, kube-proxy và CNI có mặt trên control-plane node không làm
chúng trở thành control-plane component**. Chúng vẫn là thành phần node, runtime hoặc networking;
chúng chỉ cùng chạy trên một máy đang host các thành phần control plane. Tương tự, việc một
database server chạy thêm agent giám sát không biến agent đó thành thành phần của database.

Tóm lại:

> Khi dùng kubeadm dựng control plane HA, mỗi control-plane node thường chạy một bộ gồm API
> server, scheduler và controller manager; nếu dùng stacked etcd thì mỗi node còn chạy một
> etcd member. Những control-plane node này cần kubelet và container runtime để kubelet đọc
> các manifest trong `/etc/kubernetes/manifests/` và yêu cầu runtime như containerd chạy các
> static Pod control plane. Kube-proxy và CNI thường có mặt trên các node đó nhưng không phải
> control-plane component; kube-proxy cũng có thể được thay thế. Nếu etcd được tách ra các máy
> riêng thì các máy external-etcd không bắt buộc là Kubernetes Node và không cần kubelet hoặc
> kube-proxy.

### kube-apiserver

API server là một thành phần của control plane Kubernetes, chịu trách nhiệm cung cấp (expose) API của Kubernetes.
API server là mặt tiền (front end) của control plane Kubernetes.

Hiện thực chính của API server Kubernetes là [kube-apiserver](https://kubernetes.io/docs/reference/generated/kube-apiserver/).
kube-apiserver được thiết kế để mở rộng theo chiều ngang (scale horizontally) — nghĩa là nó mở rộng bằng cách triển khai thêm nhiều instance.
Bạn có thể chạy nhiều instance của kube-apiserver và cân bằng lưu lượng giữa các instance đó.

#### Hình dung: cơ chế "một cửa" {#co-che-mot-cua}

> **Ghi chú:** Phần này không có trong trang gốc. Đây là một cách ví von được thêm vào
> để dễ hình dung vai trò của kube-apiserver trong việc phối hợp giữa các thành phần.

Hãy hình dung cluster như một cơ quan hành chính làm việc theo cơ chế một cửa.

Bạn muốn xin giấy phép, bạn không đi thẳng vào phòng của từng cán bộ. Bạn nộp hồ sơ ở một
quầy tiếp nhận duy nhất. Cán bộ các phòng ban cũng không cầm hồ sơ chạy sang phòng nhau —
họ đọc và cập nhật hồ sơ trong cùng một hệ thống chung.

kube-apiserver chính là quầy tiếp nhận đó. Điều dễ hiểu nhầm nhất nằm ở chỗ này:
các thành phần trong Kubernetes không hề "gọi điện" cho nhau.

- [kube-scheduler](#kube-scheduler) quyết định Pod chạy trên node nào — nhưng nó không báo cho
  kubelet. Nó chỉ ghi vào hồ sơ (qua apiserver): "Pod này thuộc node A".
- [kubelet](#kubelet) trên node A cũng không chờ ai ra lệnh. Nó liên tục theo dõi hồ sơ
  (qua apiserver) và tự thấy: "Ơ, có Pod được gán cho tôi" → tự kéo image về chạy.

Hai thành phần đó phối hợp với nhau xong một việc mà chưa từng nói chuyện trực tiếp với nhau
một lần nào — tất cả giao tiếp đều là ghi vào hồ sơ và đọc hồ sơ, qua đúng một cửa.

Vì sao thiết kế như vậy? Vì chỉ cần gác một cửa là gác được tất cả: kiểm tra danh tính
(authentication), kiểm tra quyền (authorization), áp chính sách (admission control) — làm một lần,
một chỗ. Và vì các thành phần không phụ thuộc trực tiếp vào nhau, một thành phần chết đi sống lại
không làm gãy chuỗi — nó chỉ cần đọc lại hồ sơ là biết mình đang ở đâu.

#### Nhiều bản kube-apiserver sau một load balancer {#nhieu-ban-apiserver}

> **Ghi chú:** Phần này không có trong trang gốc. Nội dung làm rõ ý "mở rộng theo chiều ngang"
> ở trên, và phân biệt với Service `type: LoadBalancer` — hai thứ hay bị lẫn vì trùng tên.

##### 1. Kiến trúc được tổ chức như thế nào

Một control plane HA điển hình trên on-prem (3 node, dựng bằng kubeadm):

```
        kubectl (admin)      kubelet (mọi worker)      CI/CD, app gọi API
                └──────────────────┼──────────────────────┘
                                   ▼
                    https://192.168.100.200:6443   ◀── VIP: một IP "ảo" đại diện
                       (HAProxy + keepalived,          cho cả control plane
                        hoặc kube-vip)
                ┌──────────────────┼──────────────────┐
                ▼                  ▼                  ▼
         ┌────────────┐    ┌────────────┐    ┌────────────┐
         │    cp1     │    │    cp2     │    │    cp3     │
         │ apiserver  │    │ apiserver  │    │ apiserver  │  ← cả 3 cùng chạy,
         │ ctrl-mgr   │    │ ctrl-mgr   │    │ ctrl-mgr   │    cùng nhận request
         │ scheduler  │    │ scheduler  │    │ scheduler  │
         │   etcd ────┼────┼── etcd ────┼────┼── etcd     │  ← Raft quorum 3
         └────────────┘    └────────────┘    └────────────┘
```

Các điểm tổ chức quan trọng:

- **Mọi client chỉ biết một địa chỉ duy nhất** — cái VIP. Trong kubeconfig của admin,
  trong cấu hình kubelet của từng worker, đều ghi `server: https://192.168.100.200:6443`.
  Không ai trỏ thẳng vào IP của cp1/cp2/cp3. Với kubeadm, địa chỉ này được chốt ngay
  lúc dựng: `kubeadm init --control-plane-endpoint "192.168.100.200:6443"`.
- **Load balancer phải có TRƯỚC khi cluster ra đời** — vì địa chỉ VIP được nướng vào
  chứng chỉ TLS và cấu hình của mọi thành phần từ lệnh init. Trên on-prem, nó thường là
  cặp HAProxy + keepalived (VIP trôi giữa 2 máy LB), hoặc gọn hơn là **kube-vip** chạy
  static pod ngay trên chính các control-plane node — VIP trôi giữa cp1/cp2/cp3,
  không cần máy riêng. Xem [Tạo cluster có tính sẵn sàng cao với kubeadm](./08-high-availability-vi.md).
- **Ba apiserver chạy active-active** — cả ba cùng phục vụ đồng thời, LB chia request
  cho cả ba. Không có "bản chính, bản dự phòng". Làm được vậy chính vì stateless:
  request nào rơi vào bản nào cũng tra cùng một etcd.

Chi tiết tinh tế đáng biết: cùng nằm trên 3 node đó nhưng **ba loại thành phần dùng
ba mô hình HA khác nhau**:

| Thành phần | Mô hình HA | Vì sao |
|---|---|---|
| apiserver | **Active-active** — cả 3 cùng làm việc | Stateless, không cần phối hợp gì với nhau |
| etcd | **Quorum Raft** — cả 3 giữ dữ liệu, 1 leader ghi | Stateful, phải giữ nhất quán |
| controller-manager, scheduler | **Leader election** — cả 3 chạy nhưng chỉ 1 bản hoạt động, 2 bản kia đứng chờ | Logic phải là "một người quyết" — hai scheduler cùng gán Pod sẽ giẫm chân nhau |

##### 2. "Scale được" nghĩa là gì

Tải của apiserver là **tải xử lý request**: hàng nghìn kubelet giữ kết nối watch
thường trực, controllers liên tục đọc/ghi, CI/CD bắn kubectl, mỗi request đều phải qua
xác thực (authentication) → phân quyền (authorization) — và với request ghi thì thêm cả
admission control (tốn CPU thật). Cluster càng lớn, lượng này càng phình.

"Scale ngang" nghĩa là: **tải tăng thì thêm bản apiserver, LB tự chia bớt sang** —
3 bản chịu được lượng kết nối lớn hơn hẳn 1 bản. Điều khiến việc này *rẻ và an toàn* là
các bản apiserver **không cần phối hợp với nhau để phục vụ request**: không chia vùng
dữ liệu, không bầu leader, không đồng bộ trạng thái cho nhau. Thêm một bản = chạy thêm
một process trỏ vào cùng etcd, thêm vào backend của LB. Hết.

Nhưng có một trần quan trọng: scale apiserver **không** scale được cả cluster vô hạn,
vì mọi bản đều dồn về **một etcd quorum** — mà etcd không scale ngang được cho ghi
(thêm member chỉ thêm bản sao, không thêm sức ghi). Đến quy mô đủ lớn, nghẽn chuyển
từ apiserver xuống etcd.

Quy luật chung (trùng với nguyên tắc của hệ thống log tập trung): **lớp stateless nhân
thoải mái** (Logstash, apiserver); **lớp stateful là nơi quyết định giới hạn**
(OpenSearch, etcd) — nhân có kỷ luật bằng quorum/replica.

##### 3. Liên quan gì đến Service `type: LoadBalancer`?

**Không liên quan về cơ chế — chỉ trùng chữ "load balancer".** Đây là hai thứ ở hai
tầng khác nhau; phân biệt được chúng là hiểu đúng ranh giới control plane / data plane:

| | LB trước apiserver | Service [`type: LoadBalancer`](./82-service-vi.md#loadbalancer) |
|---|---|---|
| Phục vụ ai | **Control plane** — traffic điều khiển cluster (port 6443) | **Ứng dụng** — traffic của người dùng vào app (port 80/443/...) |
| Đích cuối | Các bản kube-apiserver | Các **Pod** của ứng dụng |
| Ai tạo ra | Người dựng hạ tầng, **trước khi** cluster tồn tại | Kubernetes tạo **khi** `kubectl apply` một Service |
| Kubernetes có biết nó không | Không — nó nằm ngoài, dưới cluster | Có — nó là một object trong cluster |
| Trên cloud | NLB do người dùng/cloud team dựng | Cloud tự cấp (cloud-controller-manager gọi API cloud) |
| Trên on-prem | HAProxy + keepalived, hoặc kube-vip | Phải tự cài **MetalLB** hoặc kube-vip — không có gì cấp sẵn |

Quan hệ nhân quả duy nhất giữa chúng: cái LB thứ nhất phải sống thì cái thứ hai mới
tạo được — vì `kubectl apply` một Service cũng chỉ là một request... đi qua apiserver.

Lý do hai thứ này hay bị lẫn trên on-prem: **kube-vip làm được cả hai vai** — vừa giữ
VIP cho control plane, vừa cấp IP cho Service LoadBalancer. Cùng một phần mềm, hai
chức năng độc lập; bật vai nào là lựa chọn cấu hình.

Chuỗi đầy đủ khi nhìn cả hai tầng cùng lúc:

```
Admin ──▶ VIP:6443 ──▶ apiserver ──▶ etcd            (tầng điều khiển: "hãy chạy app này")
User  ──▶ LB của Service ──▶ Pod của app             (tầng dữ liệu: dùng app đang chạy)
```

> 💡 Thực hành: lab k8s 1 control-plane có thể nâng lên HA bằng kube-vip (không cần
> thêm VM cho LB) — khi đó kubeconfig trỏ VIP thay vì IP node, và có thể tắt một
> control-plane node để kiểm chứng kubectl vẫn chạy — đúng kiểu "bài phá" của lab log.

### etcd

Kho lưu trữ key-value nhất quán (consistent) và có tính sẵn sàng cao, được dùng làm nơi lưu trữ nền (backing store) cho toàn bộ dữ liệu cluster của Kubernetes.

Nếu cluster Kubernetes của bạn dùng etcd làm backing store, hãy đảm bảo bạn có kế hoạch
[sao lưu (back up)](197-configure-upgrade-etcd-vi.md#backing-up-an-etcd-cluster)
cho dữ liệu này.

Bạn có thể tìm thông tin chuyên sâu về etcd trong [tài liệu](https://etcd.io/docs/) chính thức.

### kube-scheduler

Thành phần control plane theo dõi các Pod mới được tạo mà chưa được gán
node, và chọn một node để chúng chạy trên đó.

Các yếu tố được cân nhắc trong quyết định lập lịch bao gồm:
yêu cầu tài nguyên (resource) của từng Pod và của tập hợp Pod, các ràng buộc về phần cứng/phần mềm/chính sách (policy),
các đặc tả affinity và anti-affinity, tính cục bộ của dữ liệu (data locality),
sự ảnh hưởng lẫn nhau giữa các workload, và các thời hạn (deadline).

### kube-controller-manager

Thành phần control plane chạy các tiến trình controller.

Về mặt logic, mỗi controller là một tiến trình riêng biệt, nhưng để giảm độ phức tạp,
tất cả chúng được biên dịch vào một binary duy nhất và chạy trong một tiến trình duy nhất.

Có nhiều loại controller khác nhau. Một vài ví dụ trong số đó là:

- Node controller: Chịu trách nhiệm phát hiện và phản ứng khi các node gặp sự cố (down).
- Job controller: Theo dõi các đối tượng Job đại diện cho các tác vụ chạy một lần (one-off task), sau đó tạo các Pod để chạy các tác vụ đó cho đến khi hoàn thành.
- EndpointSlice controller: Tạo và cập nhật các đối tượng EndpointSlice (để cung cấp mối liên kết giữa Service và Pod).
- ServiceAccount controller: Tạo các ServiceAccount mặc định cho những namespace mới.

Danh sách trên chưa phải là đầy đủ.

### cloud-controller-manager

Một thành phần control plane của Kubernetes nhúng logic điều khiển đặc thù cho từng nền tảng cloud.
Cloud controller manager cho phép bạn kết nối cluster của mình với API của nhà cung cấp cloud (cloud provider),
đồng thời tách các thành phần tương tác với nền tảng cloud đó khỏi
các thành phần chỉ tương tác với cluster của bạn.

cloud-controller-manager chỉ chạy những controller đặc thù cho nhà cung cấp cloud của bạn.
Nếu bạn chạy Kubernetes trên hạ tầng của riêng mình (on-premises), hoặc trong môi trường học tập ngay trên
PC cá nhân, cluster sẽ không có cloud controller manager.

Tương tự kube-controller-manager, cloud-controller-manager kết hợp nhiều vòng lặp điều khiển (control loop)
độc lập về mặt logic vào một binary duy nhất mà bạn chạy như một tiến trình duy nhất. Bạn có thể mở rộng
theo chiều ngang (chạy nhiều hơn một bản sao) để cải thiện hiệu năng hoặc tăng khả năng chịu lỗi.

Các controller sau có thể có phụ thuộc vào nhà cung cấp cloud:

- Node controller: Kiểm tra với nhà cung cấp cloud để xác định xem một node đã bị
  xóa trên cloud hay chưa sau khi node đó ngừng phản hồi
- Route controller: Thiết lập các route trong hạ tầng cloud bên dưới
- Service controller: Tạo, cập nhật và xóa các bộ cân bằng tải (load balancer) của nhà cung cấp cloud

---

## Các thành phần của node (Node components) {#node-components}

Các thành phần node chạy trên mọi node, duy trì các pod đang chạy và cung cấp môi trường runtime của Kubernetes.

### kubelet

Một agent chạy trên mỗi node trong cluster. Nó đảm bảo rằng các container đang chạy trong một Pod.

[kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/) nhận một tập các PodSpec
được cung cấp qua nhiều cơ chế khác nhau và đảm bảo các container được mô tả trong những
PodSpec đó đang chạy và khỏe mạnh (healthy). kubelet không quản lý các container không do
Kubernetes tạo ra.

### kube-proxy (tùy chọn) {#kube-proxy}

kube-proxy là một network proxy chạy trên mỗi node trong cluster của bạn,
hiện thực một phần của khái niệm Service trong Kubernetes.

[kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)
duy trì các network rule trên các node. Những network rule này cho phép các phiên mạng (network session)
từ bên trong hoặc bên ngoài cluster giao tiếp với các Pod của bạn.

kube-proxy sử dụng tầng lọc gói tin (packet filtering) của hệ điều hành nếu tầng này tồn tại
và khả dụng. Nếu không, kube-proxy sẽ tự mình chuyển tiếp lưu lượng.

Nếu bạn dùng một [network plugin](#network-plugins) tự hiện thực việc chuyển tiếp gói tin cho các Service
và cung cấp hành vi tương đương kube-proxy, thì bạn không cần chạy
kube-proxy trên các node trong cluster của mình.

### Container runtime

Một thành phần nền tảng giúp Kubernetes chạy các container một cách hiệu quả.
Nó chịu trách nhiệm quản lý việc thực thi và vòng đời của các container trong môi trường Kubernetes.

Kubernetes hỗ trợ các container runtime như
containerd, CRI-O,
và bất kỳ hiện thực nào khác của [Kubernetes CRI (Container Runtime Interface)](https://github.com/kubernetes/community/blob/main/contributors/devel/sig-node/container-runtime-interface.md).

## Addons

Addons sử dụng các tài nguyên Kubernetes (DaemonSet,
Deployment, v.v.) để hiện thực các tính năng của cluster.
Vì chúng cung cấp các tính năng ở cấp cluster, những tài nguyên thuộc namespace của
addons nằm trong namespace `kube-system`.

Một số addons chọn lọc được mô tả bên dưới; để xem danh sách mở rộng các addons hiện có,
vui lòng xem [Addons](165-addons-vi.md).

### DNS

Mặc dù các addons khác không thực sự bắt buộc, mọi cluster Kubernetes đều nên có
[DNS cho cluster (cluster DNS)](./10-dns-pod-service-vi.md), vì nhiều ví dụ phụ thuộc vào nó.

Cluster DNS là một DNS server, hoạt động bổ sung cho các DNS server khác trong môi trường của bạn,
chuyên phục vụ các bản ghi DNS cho các service của Kubernetes.

Các container do Kubernetes khởi động sẽ tự động đưa DNS server này vào danh sách tìm kiếm DNS của chúng.

### Web UI (Dashboard)

[Dashboard](371-web-ui-dashboard-vi.md) là một giao diện web
đa dụng cho các cluster Kubernetes. Nó cho phép người dùng quản lý và khắc phục sự cố (troubleshoot)
cho các ứng dụng đang chạy trong cluster, cũng như cho chính cluster đó.

### Giám sát tài nguyên container (Container resource monitoring) {#container-resource-monitoring}

[Giám sát tài nguyên container](312-resource-usage-monitoring-vi.md)
ghi lại các số liệu (metrics) chuỗi thời gian tổng quát về các container vào một cơ sở dữ liệu trung tâm, và cung cấp giao diện để duyệt dữ liệu đó.

### Ghi log cấp cluster (Cluster-level Logging) {#cluster-level-logging}

Cơ chế [ghi log cấp cluster (cluster-level logging)](158-logging-vi.md) chịu trách nhiệm
lưu log của container vào một kho log trung tâm có giao diện tìm kiếm/duyệt.

### Network plugins {#network-plugins}

[Network plugins](183-network-plugins-vi.md)
là các thành phần phần mềm hiện thực đặc tả container network interface (CNI).
Chúng chịu trách nhiệm cấp phát địa chỉ IP cho các pod và cho phép các pod
giao tiếp với nhau bên trong cluster.

## Các biến thể kiến trúc (Architecture variations)

Mặc dù các thành phần cốt lõi của Kubernetes luôn nhất quán, cách chúng được triển khai và
quản lý có thể khác nhau. Hiểu rõ các biến thể này là điều then chốt để thiết kế và vận hành
các cluster Kubernetes đáp ứng những nhu cầu vận hành cụ thể.

### Các phương án triển khai control plane (Control plane deployment options)

Các thành phần control plane có thể được triển khai theo một số cách:

**Triển khai truyền thống (Traditional deployment)**
: Các thành phần control plane chạy trực tiếp trên các máy hoặc VM chuyên dụng, thường được quản lý dưới dạng các service của systemd.

**Static Pod**
: Các thành phần control plane được triển khai dưới dạng static Pod, do kubelet trên các node cụ thể quản lý.
  Đây là cách tiếp cận phổ biến được các công cụ như kubeadm sử dụng.

**Tự lưu trữ (Self-hosted)**
: Control plane chạy dưới dạng các Pod ngay bên trong chính cluster Kubernetes, được quản lý bởi các Deployment
  và StatefulSet hoặc các nguyên thủy (primitive) khác của Kubernetes.

**Dịch vụ Kubernetes được quản lý (Managed Kubernetes services)**
: Các nhà cung cấp cloud thường trừu tượng hóa control plane, quản lý các thành phần của nó như một phần trong gói dịch vụ mà họ cung cấp.

### Cân nhắc về vị trí đặt workload (Workload placement considerations)

Vị trí đặt các workload, bao gồm cả các thành phần control plane, có thể thay đổi tùy theo quy mô cluster,
yêu cầu hiệu năng và chính sách vận hành:

- Trong các cluster nhỏ hoặc cluster phát triển (development), các thành phần control plane và workload của người dùng có thể chạy trên cùng các node.
- Các cluster production lớn thường dành riêng một số node cụ thể cho các thành phần control plane,
  tách biệt chúng khỏi workload của người dùng.
- Một số tổ chức chạy các add-on quan trọng hoặc các công cụ giám sát trên những node control plane.

### Công cụ quản lý cluster (Cluster management tools)

Các công cụ như kubeadm, kops và Kubespray đưa ra những cách tiếp cận khác nhau để triển khai và quản lý cluster,
mỗi công cụ có phương pháp bố trí và quản lý thành phần riêng.

### Tùy biến và khả năng mở rộng (Customization and extensibility)

Kiến trúc Kubernetes cho phép tùy biến ở mức độ đáng kể:

- Có thể triển khai các scheduler tùy chỉnh để hoạt động song song với scheduler mặc định của Kubernetes hoặc thay thế hoàn toàn nó.
- Có thể mở rộng API server bằng CustomResourceDefinition và API Aggregation.
- Các nhà cung cấp cloud có thể tích hợp sâu với Kubernetes thông qua cloud-controller-manager.

Tính linh hoạt của kiến trúc Kubernetes cho phép các tổ chức điều chỉnh cluster theo những nhu cầu cụ thể,
cân bằng các yếu tố như độ phức tạp vận hành, hiệu năng và chi phí quản lý.

## Tiếp theo (What's next)

Tìm hiểu thêm về các chủ đề sau:

- [Các Node](./23-nodes-vi.md) và
  [giao tiếp giữa chúng](./24-control-plane-node-communication-vi.md)
  với control plane.
- Các [controller](./25-controllers-vi.md) của Kubernetes.
- [Thu gom rác (Garbage collection)](36-garbage-collection-vi.md) đối với các đối tượng của cluster.
- [kube-scheduler](137-kube-scheduler-vi.md) — scheduler mặc định của Kubernetes.
- [Tài liệu](https://etcd.io/docs/) chính thức của etcd.
- Một số [container runtime](./00-container-runtimes-vi.md) trong Kubernetes.
- Tích hợp với các nhà cung cấp cloud bằng [cloud-controller-manager](34-cloud-controller-vi.md).
- Các lệnh [kubectl](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 1:

1. Nói "control plane chạy trên ba máy" nghĩa là gì? Có phải control plane là một máy bị chia
   làm ba phần không?
2. Scheduler quyết định một Pod sẽ chạy trên node A. Nó báo cho kubelet của node A bằng cách
   nào? Hai thành phần đó có gọi trực tiếp cho nhau không?
3. Thành phần nào đọc và ghi etcd trực tiếp? `kubelet` có kết nối tới etcd không?
4. Trên control-plane node dựng bằng kubeadm có `kubelet`, containerd và `kube-proxy`. Vì sao
   ba thứ đó vẫn không phải control plane component?
5. Vì sao thêm bản `kube-apiserver` thì dễ, còn thêm sức ghi cho etcd thì không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Nghĩa là **các thành phần (hoặc instance) của control plane được bố trí trên ba
   control-plane node**. Control plane là **lớp chức năng** — tập phần mềm điều khiển cluster;
   control-plane node là **cái máy** được chọn để chạy chúng. Đúng như "database" so với
   "database server". Không phải một máy bị chia làm ba.
2. **Nó không báo.** Scheduler chỉ ghi vào object qua API server rằng Pod này thuộc node A.
   Kubelet trên node A liên tục theo dõi API server, tự thấy có Pod được gán cho mình rồi kéo
   image về chạy. Hai thành phần **chưa từng gọi trực tiếp cho nhau** — tất cả là ghi hồ sơ và
   đọc hồ sơ qua đúng một cửa.
3. Chỉ **kube-apiserver**. `kubelet` **không** kết nối tới etcd; nó chỉ nói chuyện với API
   server. Bạn đã tự chứng minh điều này ở Lab 1a phần B3, khi chỉ manifest
   `kube-apiserver.yaml` chứa `--etcd-servers`.
4. Vì phân loại theo **vai trò**, không theo máy. `kubelet` là agent của node, containerd là
   runtime, `kube-proxy` là networking. Chúng có mặt ở đó vì kubeadm chạy control plane dưới
   dạng **static Pod**, mà static Pod thì cần kubelet đọc manifest trong
   `/etc/kubernetes/manifests/` và cần runtime thực thi container. Giống việc một database
   server chạy thêm agent giám sát không biến agent đó thành thành phần của database.
5. `kube-apiserver` **không giữ trạng thái**: các bản không chia vùng dữ liệu, không bầu
   leader, không đồng bộ với nhau. Thêm một bản chỉ là chạy thêm một process trỏ vào cùng etcd
   rồi thêm vào backend của load balancer. etcd thì **có trạng thái** và giữ nhất quán bằng
   quorum Raft — thêm member chỉ thêm bản sao chứ **không thêm sức ghi**. Vì vậy đến quy mô đủ
   lớn, nghẽn chuyển từ apiserver xuống etcd.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
