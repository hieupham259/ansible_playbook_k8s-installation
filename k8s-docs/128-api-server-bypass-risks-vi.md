# Rủi ro vượt qua Kubernetes API Server (Kubernetes API Server Bypass Risks)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/api-server-bypass-risks/>
>
> Thông tin về kiến trúc bảo mật liên quan đến API server và các thành phần khác

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), bài 13/18 · Kiểm chứng ở Lab 9b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài này lật ngược ba bài bạn đã đọc. Bài [119](119-controlling-access-vi.md) xây một luồng bốn
chặng chặt chẽ; bài [24](24-control-plane-node-communication-vi.md) nói mọi truy cập API đều kết
thúc tại API server; bài [58](58-static-pods-vi.md) giới thiệu static Pod như cơ chế bình thường
của kubeadm. Bài này chỉ ra **bốn đường đi vòng** qua luồng đó — và static Pod là một trong số
đó. Bài ngắn, đọc hết.

**Phải hiểu ở lần đọc này:**

- Điểm chung của cả bốn đường: chúng **không chịu admission control** và **không được audit
  logging của Kubernetes ghi lại**. Đó là lý do chúng nguy hiểm hơn một request thường, kể cả
  request có đặc quyền cao, đi qua API server.
- **Static Pod**: kubelet nạp và **trực tiếp quản lý** manifest trong thư mục hoặc URL được chỉ
  định; **API server không quản lý** chúng. Kẻ ghi được vào vị trí đó sửa hoặc thêm được static
  Pod. Static Pod bị hạn chế truy cập object khác trong API — ví dụ **không mount được Secret
  của cluster** — nhưng vẫn dùng được `hostPath` mount từ node.
- Hai hệ quả tinh vi của static Pod: nếu kẻ tấn công dùng **tên namespace không hợp lệ**, Pod
  **không hiện trong Kubernetes API** và chỉ phát hiện được bằng công cụ chạy trên chính máy
  chủ đó; và nếu static Pod **không vượt qua được admission control**, kubelet không đăng ký nó
  với API server **nhưng Pod vẫn chạy trên node**.
- **API của kubelet** (thường mở trên TCP 10250 của worker): cho phép lộ thông tin Pod, log, và
  **thực thi lệnh trong mọi container đang chạy trên node**. Vì một số endpoint chạy Websocket
  qua HTTP `GET`, **quyền get trên `nodes/proxy` không phải quyền chỉ đọc**. Mặc định cấu hình
  kubelet cho phép **truy cập ẩn danh**; chế độ đó **không xác nhận với control plane** rằng bên
  gọi đã được cấp quyền, nên phải chuyển sang xác thực webhook hoặc certificate.
- **API của etcd** (TCP 2379): client duy nhất cần truy cập là API server và công cụ backup. Vì
  etcd **chưa có mô hình cấp quyền được chấp nhận rộng rãi**, **bất kỳ certificate nào do CA mà
  etcd tin tưởng phát hành đều có toàn quyền đọc ghi** — kể cả certificate vốn chỉ dùng để
  health check. **Socket của container runtime** cũng vậy: ai truy cập được socket thì khởi
  chạy hoặc tương tác được với container trên node đó.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Cấu hình xác thực kubelet ở chế độ webhook hoặc certificate | là thao tác sửa cấu hình kubelet | giai đoạn 8, bài [04](04-kubelet-integration-vi.md) |
| *Các quyền chi tiết* thay cho quyền bao trùm `nodes/proxy` | tra cứu khi thiết kế RBAC cho dịch vụ giám sát | giai đoạn 11 |
| Kiểm toán tập trung mọi truy cập vào thư mục static Pod manifest | cần audit backend | CP7 audit/encryption |
| Kiểm soát private key của etcd, mutual TLS, CA riêng cho etcd | thuộc vòng đời chứng chỉ và vận hành etcd | CP3 chứng chỉ và CP4 etcd |

---

Kubernetes API server là điểm vào (entry point) chính của cluster đối với các bên bên ngoài
(người dùng và dịch vụ) tương tác với cluster.

Trong vai trò này, API server có một số cơ chế kiểm soát bảo mật quan trọng được tích hợp
sẵn, chẳng hạn như ghi log kiểm toán (audit logging) và admission controller. Tuy nhiên,
vẫn có những cách thay đổi cấu hình hoặc nội dung của cluster mà vượt qua (bypass) được
các cơ chế kiểm soát này.

Trang này mô tả những cách mà các cơ chế kiểm soát bảo mật tích hợp trong Kubernetes API
server có thể bị vượt qua, để những người vận hành cluster và kiến trúc sư bảo mật có thể
đảm bảo rằng các đường vòng này được hạn chế một cách thích hợp.

## Static Pod (Static Pods) {#static-pods}

Kubelet trên mỗi node sẽ nạp và trực tiếp quản lý mọi manifest được lưu trong một thư mục
được chỉ định hoặc được tải về từ một URL cụ thể, dưới dạng
[*static Pod*](./58-static-pods-vi.md) trong cluster của bạn. API server không quản lý các
static Pod này. Kẻ tấn công có quyền ghi vào vị trí đó có thể sửa đổi cấu hình của các
static Pod được nạp từ nguồn này, hoặc có thể đưa vào các static Pod mới.

Static Pod bị hạn chế truy cập các đối tượng khác trong Kubernetes API. Ví dụ, bạn không thể
cấu hình một static Pod để mount một Secret từ cluster. Tuy nhiên, các Pod này vẫn có thể
thực hiện những hành động nhạy cảm về bảo mật khác, chẳng hạn như dùng `hostPath` mount từ
node bên dưới.

Theo mặc định, kubelet tạo một mirror pod để static Pod hiển thị được trong Kubernetes API.
Tuy nhiên, nếu kẻ tấn công dùng một tên namespace không hợp lệ khi tạo Pod, Pod đó sẽ không
hiển thị trong Kubernetes API và chỉ có thể bị phát hiện bởi các công cụ có quyền truy cập
vào (các) máy chủ bị ảnh hưởng.

Nếu một static Pod không vượt qua được admission control, kubelet sẽ không đăng ký Pod đó
với API server. Tuy nhiên, Pod vẫn chạy trên node. Để biết thêm thông tin, tham khảo
[kubeadm issue #1541](https://github.com/kubernetes/kubeadm/issues/1541#issuecomment-487331701).

### Biện pháp giảm thiểu (Mitigations) {#static-pods-mitigations}

- Chỉ [bật chức năng static Pod manifest của kubelet](293-static-pod-tasks-vi.md#static-pod-creation)
  nếu node thật sự cần.
- Nếu một node dùng chức năng static Pod, hãy hạn chế quyền truy cập filesystem vào thư mục
  chứa static Pod manifest hoặc URL, chỉ cho những người dùng cần quyền truy cập đó.
- Hạn chế quyền truy cập vào các tham số và file cấu hình của kubelet để ngăn kẻ tấn công
  thiết lập đường dẫn hoặc URL cho static Pod.
- Thường xuyên kiểm toán và báo cáo tập trung mọi truy cập vào các thư mục hoặc vị trí lưu
  trữ web chứa static Pod manifest và file cấu hình kubelet.

## API của kubelet (The kubelet API) {#kubelet-api}

Kubelet cung cấp một HTTP API, thường được mở trên cổng TCP 10250 của các worker node trong
cluster. API này cũng có thể được mở trên các node control plane tùy vào bản phân phối
Kubernetes đang dùng. Truy cập trực tiếp vào API này cho phép lộ thông tin về các pod đang
chạy trên node, log của các pod đó, và thực thi lệnh trong mọi container đang chạy trên
node.

Một số endpoint trong đó hỗ trợ giao thức Websocket thông qua các HTTP request `GET`, vốn
được cấp quyền bằng verb **get**. Điều này có nghĩa là quyền **get** trên `nodes/proxy`
không phải là quyền chỉ-đọc, và nó cấp quyền truy cập tới các endpoint có thể được dùng để
thực thi lệnh trong bất kỳ container nào đang chạy trên node.

Khi người dùng của cluster Kubernetes có quyền truy cập RBAC tới các sub-resource của đối
tượng `Node`, quyền truy cập đó đóng vai trò là sự cấp quyền để tương tác với API của
kubelet. Mức truy cập chính xác phụ thuộc vào sub-resource nào đã được cấp quyền, như được
mô tả chi tiết tại
[cấp quyền cho kubelet (kubelet authorization)](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/#kubelet-authorization).

Truy cập trực tiếp vào API của kubelet không chịu sự kiểm soát của admission control và
không được ghi lại bởi cơ chế audit logging của Kubernetes. Kẻ tấn công có quyền truy cập
trực tiếp vào API này có thể vượt qua các cơ chế kiểm soát vốn dùng để phát hiện hoặc ngăn
chặn một số hành động nhất định.

API của kubelet có thể được cấu hình để xác thực (authenticate) các request theo nhiều cách.
Theo mặc định, cấu hình kubelet cho phép truy cập ẩn danh (anonymous). Hầu hết các nhà cung
cấp Kubernetes thay đổi mặc định này để dùng xác thực bằng webhook và certificate. Điều đó
cho phép control plane đảm bảo rằng bên gọi đã được cấp quyền truy cập tài nguyên API
`nodes` hoặc các sub-resource của nó. Chế độ truy cập ẩn danh mặc định không thực hiện sự
xác nhận này với control plane.

### Biện pháp giảm thiểu (Mitigations)

- Hạn chế quyền truy cập các sub-resource của đối tượng API `nodes` bằng các cơ chế như
  [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/). Chỉ cấp quyền truy
  cập này khi thật sự cần, chẳng hạn cho các dịch vụ giám sát (monitoring).
- Tránh cấp quyền bao trùm `nodes/proxy`, ngay cả khi chỉ với verb **get**. Thay vào đó,
  hãy cấp [các quyền chi tiết (granular permissions)](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/#fine-grained-authorization).
- Hạn chế truy cập vào cổng của kubelet. Chỉ cho phép các dải địa chỉ IP được chỉ định và
  đáng tin cậy truy cập cổng này.
- Đảm bảo rằng [xác thực kubelet (kubelet authentication)](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/#kubelet-authentication)
  được đặt ở chế độ webhook hoặc certificate.
- Đảm bảo rằng cổng kubelet "chỉ-đọc" không cần xác thực không được bật trên cluster.

## API của etcd (The etcd API)

Các cluster Kubernetes dùng etcd làm kho lưu trữ dữ liệu (datastore). Dịch vụ `etcd` lắng
nghe trên cổng TCP 2379. Các client duy nhất cần quyền truy cập là Kubernetes API server và
bất kỳ công cụ sao lưu (backup) nào mà bạn sử dụng. Truy cập trực tiếp vào API này cho phép
lộ hoặc sửa đổi bất kỳ dữ liệu nào được lưu trong cluster.

Quyền truy cập API của etcd thường được quản lý bằng xác thực certificate phía client
(client certificate authentication). Bất kỳ certificate nào do một certificate authority mà
etcd tin tưởng phát hành đều cho phép truy cập đầy đủ vào dữ liệu lưu bên trong etcd.

Truy cập trực tiếp vào etcd không chịu sự kiểm soát của admission control trong Kubernetes
và không được ghi lại bởi cơ chế audit logging của Kubernetes. Kẻ tấn công có quyền đọc
private key của etcd client certificate mà API server dùng (hoặc có thể tạo một client
certificate mới được tin tưởng) có thể giành được quyền quản trị cluster (cluster admin)
bằng cách truy cập các secret của cluster hoặc sửa đổi các quy tắc truy cập. Ngay cả khi
không nâng đặc quyền RBAC trong Kubernetes, kẻ tấn công có khả năng sửa đổi etcd vẫn có thể
lấy ra bất kỳ đối tượng API nào hoặc tạo các workload mới bên trong cluster.

Nhiều nhà cung cấp Kubernetes cấu hình etcd dùng mutual TLS (cả client và server đều xác
minh certificate của nhau để xác thực). Hiện chưa có một cách hiện thực nào được chấp nhận
rộng rãi cho việc cấp quyền (authorization) đối với API của etcd, mặc dù tính năng này có
tồn tại. Vì không có mô hình cấp quyền, bất kỳ certificate nào có quyền client tới etcd đều
có thể được dùng để giành toàn quyền truy cập etcd. Thông thường, các etcd client
certificate vốn chỉ dùng cho việc kiểm tra tình trạng (health checking) cũng có thể cấp
toàn quyền đọc và ghi.

### Biện pháp giảm thiểu (Mitigations) {#etcd-api-mitigations}

- Đảm bảo rằng certificate authority mà etcd tin tưởng chỉ được dùng cho mục đích xác thực
  với dịch vụ đó.
- Kiểm soát quyền truy cập private key của certificate máy chủ etcd, cũng như client
  certificate và key của API server.
- Cân nhắc hạn chế truy cập cổng etcd ở tầng mạng, chỉ cho phép truy cập từ các dải địa chỉ
  IP được chỉ định và đáng tin cậy.

## Socket của container runtime (Container runtime socket) {#runtime-socket}

Trên mỗi node của một cluster Kubernetes, quyền tương tác với các container được kiểm soát
bởi container runtime (hoặc nhiều runtime, nếu bạn cấu hình nhiều hơn một). Thông thường,
container runtime mở một Unix socket mà kubelet có thể truy cập. Kẻ tấn công có quyền truy
cập socket này có thể khởi chạy các container mới hoặc tương tác với các container đang
chạy.

Ở cấp độ cluster, tác động của quyền truy cập này phụ thuộc vào việc các container chạy
trên node bị xâm phạm có quyền truy cập Secret hoặc dữ liệu bí mật khác mà kẻ tấn công có
thể lợi dụng để leo thang đặc quyền sang các worker node khác hoặc sang các thành phần của
control plane hay không.

### Biện pháp giảm thiểu (Mitigations) {#runtime-socket-mitigations}

- Đảm bảo rằng bạn kiểm soát chặt chẽ quyền truy cập filesystem đối với các socket của
  container runtime. Khi có thể, hạn chế quyền truy cập này cho người dùng `root`.
- Cô lập kubelet khỏi các thành phần khác đang chạy trên node, bằng các cơ chế như Linux
  kernel namespace.
- Đảm bảo rằng bạn hạn chế hoặc cấm việc dùng [`hostPath` mount](./91-volumes-vi.md#hostpath)
  bao gồm socket của container runtime, dù là mount trực tiếp hay mount một thư mục cha.
  Ngoài ra, các `hostPath` mount phải được đặt là chỉ-đọc (read-only) để giảm thiểu rủi ro
  kẻ tấn công vượt qua các hạn chế về thư mục.
- Hạn chế người dùng truy cập vào các node, và đặc biệt hạn chế quyền truy cập superuser
  vào các node.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 9:

1. Bài liệt kê bốn đường vượt qua Kubernetes API server. Kể đủ bốn, và nói điểm chung khiến
   chúng nguy hiểm.
2. Trên cluster lab, control plane chạy dưới dạng static Pod trong `/etc/kubernetes/manifests`
   của `k8s-master` (bài [58](58-static-pods-vi.md)). Vì sao quyền ghi vào thư mục đó tương
   đương quyền quản trị cluster?
3. "Một static Pod bị admission controller từ chối thì nó không chạy" — đúng hay sai, và điều
   gì thực sự xảy ra?
4. Bài [24](24-control-plane-node-communication-vi.md) đã chỉ ra điểm yếu ở chiều API server →
   kubelet. Bài này bổ sung mối nguy nào từ phía một client bất kỳ, và vì sao nó đáng lo hơn
   một request có đặc quyền cao đi qua API server?
5. Một etcd client certificate "chỉ dùng để kiểm tra tình trạng (health check)" có phải là
   credential ít rủi ro không? Vì sao?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Static Pod**, **API của kubelet**, **API của etcd**, và **socket của container runtime**.
   Điểm chung: chúng cho phép thay đổi cấu hình hoặc nội dung của cluster mà **vượt qua các cơ
   chế kiểm soát bảo mật tích hợp trong API server** — cụ thể là **không chịu admission control**
   và **không được audit logging của Kubernetes ghi lại**. Kẻ tấn công đi đường này không để lại
   dấu vết trong audit log của cluster.
2. Vì **kubelet nạp và trực tiếp quản lý mọi manifest trong thư mục đó dưới dạng static Pod, và
   API server không quản lý chúng**. Kẻ có quyền ghi vào đó **sửa được cấu hình các static Pod
   đang có hoặc đưa vào static Pod mới**, và Pod mới đó vẫn dùng được `hostPath` mount từ node
   bên dưới. Trên `k8s-master`, chính các thành phần control plane nằm ở đây — nên quyền ghi
   thư mục này là quyền định đoạt control plane, không cần một dòng RBAC nào.
3. **Sai.** Bài nói rõ: nếu một static Pod không vượt qua được admission control, **kubelet sẽ
   không đăng ký Pod đó với API server** — nhưng **Pod vẫn chạy trên node**. Trực giác "admission
   chặn được nên an toàn" sai vì admission control chỉ đứng trên đường vào API server, còn static
   Pod thì không đi qua đường đó. Tệ hơn: Pod đang chạy mà `kubectl get pods` không thấy.
4. Mối nguy là **truy cập trực tiếp vào API của kubelet** trên cổng 10250, cho phép **thực thi
   lệnh trong mọi container đang chạy trên node**, lấy log và thông tin Pod. Đáng lo hơn vì
   **truy cập trực tiếp vào API của kubelet không chịu admission control và không được audit
   logging ghi lại** — một quản trị viên có đặc quyền cao đi qua API server thì mọi hành động
   vẫn nằm trong audit log, còn đường này thì không.
5. **Không, nó là credential toàn quyền.** Vì **chưa có cách hiện thực nào được chấp nhận rộng
   rãi cho việc cấp quyền đối với API của etcd**, nên **bất kỳ certificate nào do certificate
   authority mà etcd tin tưởng phát hành đều cho phép truy cập đầy đủ vào dữ liệu trong etcd**.
   Bài nói thẳng: các etcd client certificate vốn chỉ dùng cho health checking **cũng có thể cấp
   toàn quyền đọc và ghi**. Ai sửa được etcd thì lấy ra được mọi object API, tạo được workload
   mới, hoặc đổi quy tắc truy cập — mà không cần nâng đặc quyền RBAC nào.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
