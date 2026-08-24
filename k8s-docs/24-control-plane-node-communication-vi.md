# Giao tiếp giữa Node và Control Plane (Communication between Nodes and the Control Plane)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/>
>
> Tài liệu này liệt kê các đường giao tiếp giữa API server và cluster Kubernetes,
> nhằm giúp bạn tùy chỉnh bản cài đặt để tăng cường bảo mật cấu hình mạng.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 1 → nhóm [1a](00-ALO-TRINH-ADMIN.md#1a-kiến-trúc-và-mô-hình-điều-khiển),
bài 7/8 · Kiểm chứng ở [Lab 1a](labs/LAB-1A-KIEN-TRUC-VA-MO-HINH-DIEU-KHIEN.md) phần B7.

Bài này vốn **không phải bài giới thiệu kiến trúc** — nó là bài hardening bảo mật mạng, viết
cho người sắp cấu hình cluster chạy trên mạng không tin cậy. Vì vậy nó nhắc tới client
certificate, service account token, Pod, Service và kube-proxy, toàn thứ thuộc giai đoạn 3, 5
và 9. Thấy mơ hồ ở những đoạn đó là bình thường; phần cần hiểu lúc này chỉ khoảng 15 dòng.

**Phải hiểu ở lần đọc này:**

- Mô hình **hub-and-spoke**: mọi truy cập API từ node đều kết thúc tại API server. Không có
  thành phần control plane nào khác được thiết kế để phục vụ từ xa.
- Chiều ngược lại có đúng **hai đường**: API server → kubelet, và API server → node/pod/service
  qua chức năng proxy của chính API server.
- Đường API server → kubelet dùng để làm gì: lấy log, attach, port-forward.
- Điểm **bất đối xứng** đáng nhớ: chiều node → control plane mặc định đã bảo mật, còn chiều
  control plane → node thì mặc định không xác minh certificate của kubelet.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Client certificate, kubelet TLS bootstrapping | chưa học xác thực | giai đoạn 9 |
| Pod gọi API bằng service account, Service `kubernetes` | chưa học Pod, Service, ServiceAccount | giai đoạn 3, 5, 9 |
| `--kubelet-certificate-authority`, kubelet authn/authz | là thao tác hardening | giai đoạn 9 |
| *API server đến node, pod và service* | cần biết Pod và Service | giai đoạn 5 |
| *Đường hầm SSH* | đã deprecated | không cần |
| *Dịch vụ Konnectivity* | chỉ cần biết nó tồn tại như một lựa chọn thay thế | không cần |

Bài này bạn sẽ quay lại ở giai đoạn 9 khi học
[119 — Kiểm soát truy cập](119-controlling-access-vi.md) và
[128 — Rủi ro vượt qua API Server](128-api-server-bypass-risks-vi.md). Lúc đó các đoạn đang mơ
hồ mới có chỗ móc vào.

---

Tài liệu này liệt kê các đường giao tiếp (communication path) giữa API server
và cluster Kubernetes.
Mục đích là cho phép người dùng tùy chỉnh bản cài đặt của mình để tăng cường bảo mật (harden) cấu hình mạng,
sao cho cluster có thể chạy trên một mạng không tin cậy (untrusted network) — hoặc trên các địa chỉ IP
hoàn toàn công khai của một nhà cung cấp đám mây (cloud provider).

## Node đến Control Plane (Node to Control Plane)

Kubernetes dùng mẫu API kiểu "trục và nan hoa" (hub-and-spoke). Mọi truy cập API từ các node (hoặc các pod chạy trên chúng)
đều kết thúc tại API server. Không có thành phần control plane nào khác được thiết kế để cung cấp
dịch vụ từ xa. API server được cấu hình để lắng nghe các kết nối từ xa trên một port HTTPS
bảo mật (thường là 443), với một hoặc nhiều hình thức
[xác thực](https://kubernetes.io/docs/reference/access-authn-authz/authentication/) (authentication) phía client được bật.
Bạn nên bật một hoặc nhiều hình thức [ủy quyền](https://kubernetes.io/docs/reference/access-authn-authz/authorization/) (authorization),
đặc biệt khi cho phép [request ẩn danh](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#anonymous-requests) (anonymous requests)
hoặc [token của service account](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#service-account-tokens).

Các node nên được cấp phát (provision) certificate gốc công khai (public root certificate) của cluster,
để chúng có thể kết nối an toàn tới API server cùng với thông tin xác thực client (client credentials) hợp lệ.
Một cách làm tốt là cung cấp thông tin xác thực cho kubelet dưới dạng client certificate. Xem
[kubelet TLS bootstrapping](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/)
để biết cách cấp phát tự động client certificate cho kubelet.

Các Pod muốn kết nối tới API server có thể thực hiện điều đó một cách an toàn bằng cách tận dụng service account —
Kubernetes sẽ tự động đưa (inject) certificate gốc công khai và một bearer token hợp lệ
vào pod khi pod được khởi tạo.
Service `kubernetes` (trong namespace `default`) được cấu hình với một địa chỉ IP ảo, và địa chỉ này được
chuyển hướng (thông qua `kube-proxy`) tới endpoint HTTPS trên API server.

Các thành phần control plane cũng giao tiếp với API server qua port bảo mật này.

Kết quả là, chế độ hoạt động mặc định cho các kết nối từ node và các pod chạy trên
node tới control plane đã được bảo mật sẵn, và có thể chạy trên các mạng không tin cậy và/hoặc
mạng công cộng.

## Control plane đến node (Control plane to node)

Có hai đường giao tiếp chính từ control plane (API server) tới các node.
Đường thứ nhất là từ API server tới tiến trình kubelet chạy trên mỗi node trong cluster.
Đường thứ hai là từ API server tới bất kỳ node, pod hay service nào, thông qua chức năng _proxy_
của API server.

### API server đến kubelet (API server to kubelet)

Các kết nối từ API server tới kubelet được dùng để:

* Lấy log của các pod.
* Attach (thường thông qua `kubectl`) vào các pod đang chạy.
* Cung cấp chức năng port-forwarding của kubelet.

Các kết nối này kết thúc tại endpoint HTTPS của kubelet. Theo mặc định, API server không
xác minh serving certificate của kubelet, điều này khiến kết nối dễ bị tấn công xen giữa (man-in-the-middle)
và **không an toàn** khi chạy trên các mạng không tin cậy và/hoặc mạng công cộng.

Để xác minh kết nối này, hãy dùng flag `--kubelet-certificate-authority` để cung cấp cho API
server một bộ certificate gốc (root certificate bundle) dùng để xác minh serving certificate của kubelet.

Nếu điều đó không khả thi, hãy dùng [đường hầm SSH](#ssh-tunnels) giữa API server và kubelet khi cần,
để tránh kết nối qua mạng không tin cậy hoặc mạng công cộng.

Cuối cùng, nên bật [xác thực và/hoặc ủy quyền cho kubelet](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/)
để bảo vệ API của kubelet.

### API server đến node, pod và service (API server to nodes, pods, and services)

Các kết nối từ API server tới một node, pod hay service mặc định là kết nối HTTP thường,
do đó chúng không được xác thực và cũng không được mã hóa. Chúng có thể chạy qua kết nối HTTPS
bảo mật bằng cách thêm tiền tố `https:` vào tên node, pod hay service trong URL của API, nhưng khi đó chúng
sẽ không kiểm tra certificate mà endpoint HTTPS cung cấp, cũng không gửi kèm thông tin xác thực của client. Vì vậy,
mặc dù kết nối được mã hóa, nó không mang lại bất kỳ bảo đảm nào về tính toàn vẹn (integrity). Các kết nối này
**hiện không an toàn** để chạy trên các mạng không tin cậy hoặc mạng công cộng.

### Đường hầm SSH (SSH tunnels) {#ssh-tunnels}

Kubernetes hỗ trợ [đường hầm SSH](https://www.ssh.com/academy/ssh/tunneling) (SSH tunnel) để bảo vệ các đường giao tiếp
từ control plane tới node. Trong cấu hình này, API server khởi tạo một đường hầm SSH tới từng node trong cluster
(kết nối tới SSH server đang lắng nghe trên port 22) và chuyển toàn bộ lưu lượng (traffic) hướng tới kubelet, node, pod hay
service đi qua đường hầm đó.
Đường hầm này bảo đảm lưu lượng không bị lộ ra bên ngoài mạng mà các node đang chạy trong đó.

> **Ghi chú:**
> Đường hầm SSH hiện đã bị loại bỏ dần (deprecated), vì vậy bạn không nên chọn dùng chúng trừ khi bạn thực sự
> biết mình đang làm gì. [Dịch vụ Konnectivity](#konnectivity-service) là giải pháp thay thế cho
> kênh giao tiếp này.

### Dịch vụ Konnectivity (Konnectivity service) {#konnectivity-service}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.18 [beta]`

Là giải pháp thay thế cho đường hầm SSH, dịch vụ Konnectivity cung cấp proxy ở tầng TCP cho
giao tiếp từ control plane tới cluster. Dịch vụ Konnectivity gồm hai phần:
Konnectivity server trong mạng của control plane và các Konnectivity agent trong mạng của các node.
Các Konnectivity agent khởi tạo kết nối tới Konnectivity server và duy trì các kết nối
mạng đó.
Sau khi bật dịch vụ Konnectivity, toàn bộ lưu lượng từ control plane tới các node sẽ đi qua các
kết nối này.

Hãy làm theo [tác vụ thiết lập dịch vụ Konnectivity](https://kubernetes.io/docs/tasks/extend-kubernetes/setup-konnectivity/) để thiết lập
dịch vụ Konnectivity trong cluster của bạn.

## Tiếp theo (What's next)

* Đọc về [các thành phần control plane của Kubernetes](22-architecture-vi.md#control-plane-components)
* Tìm hiểu thêm về [mô hình Hub and Spoke](https://book.kubebuilder.io/multiversion-tutorial/conversion-concepts.html#hubs-spokes-and-other-wheel-metaphors)
* Tìm hiểu cách [bảo mật một cluster](256-securing-a-cluster-vi.md)
* Tìm hiểu thêm về [Kubernetes API](21-kubernetes-api-vi.md)
* [Thiết lập dịch vụ Konnectivity](https://kubernetes.io/docs/tasks/extend-kubernetes/setup-konnectivity/)
* [Dùng Port Forwarding để truy cập ứng dụng trong cluster](366-port-forward-vi.md)
* Tìm hiểu cách [lấy log của các Pod](300-debug-running-pod-vi.md#examine-pod-logs), [dùng kubectl port-forward](366-port-forward-vi.md#forward-a-local-port-to-a-port-on-the-pod)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 1:

1. Kubelet muốn báo trạng thái node thì nói chuyện với thành phần nào? Nó có nói trực tiếp với
   scheduler hay etcd không?
2. Khi bạn chạy `kubectl logs`, request đi qua những chặng nào?
3. Trong hai chiều giao tiếp đó, chiều nào mặc định đã bảo mật, chiều nào là điểm yếu?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Chỉ với **API server**. **Không** nói trực tiếp với scheduler hay etcd. Đây là mô hình
   hub-and-spoke: mọi truy cập API từ node đều kết thúc tại API server, và không có thành phần
   control plane nào khác được thiết kế để cung cấp dịch vụ từ xa.
2. `kubectl` → **API server** (HTTPS, thường là 6443) → **kubelet** trên node đang chạy Pod đó
   (endpoint HTTPS của kubelet) → log đi ngược lại theo cùng đường. Client **không** mở kết nối
   trực tiếp tới kubelet. Cùng đường này được dùng cho attach và port-forward.
3. Chiều **node → control plane đã bảo mật mặc định**: HTTPS trên port bảo mật, node được cấp
   certificate gốc của cluster và thông tin xác thực client, nên chạy được trên cả mạng không
   tin cậy. Chiều **control plane → node là điểm yếu**: mặc định API server **không xác minh
   serving certificate của kubelet**, nên kết nối dễ bị tấn công xen giữa. Khắc phục bằng
   `--kubelet-certificate-authority` — thuộc phần hardening ở giai đoạn 9.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
