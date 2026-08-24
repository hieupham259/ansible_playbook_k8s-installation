# Dùng một KMS provider để mã hóa dữ liệu (Using a KMS provider for data encryption)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/kms-provider/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** tài liệu tra cứu thuộc nhánh `/docs/tasks/`
([Checkpoint tiếp nối](00-ALO-TRINH-ADMIN.md#checkpoint-tiếp-nối--nhánh-docstasks), mục
[CP7 — Audit và mã hóa dữ liệu](00-ALO-TRINH-ADMIN.md#cp7--audit-và-mã-hóa-dữ-liệu), bài 4/6),
nối tiếp bài [208 — Encrypting Confidential Data at Rest](208-encrypt-data-vi.md): ở đó bạn đã
thấy giới hạn của key cục bộ (key nằm ngay trên host), bài này là lời giải — đưa key mã hóa
key (KEK) ra một KMS bên ngoài.

Cluster lab không có dịch vụ KMS bên ngoài, nên lần đọc này là đọc hiểu cơ chế, không thực
hành trên `k8s-master`. Trọng tâm là kiến trúc envelope encryption và các hệ quả vận hành
(xoay KEK, health check), không phải viết plugin.

**Phải hiểu ở lần đọc này:**

- Sơ đồ envelope encryption: dữ liệu trong etcd được mã hóa bằng DEK sinh cục bộ; DEK (hoặc
  seed sinh DEK ở KMS v2) được mã hóa bằng KEK nằm trong KMS từ xa — KMS từ xa không bao giờ
  chạm vào bản thân dữ liệu Secret.
- Chuỗi giao tiếp: `kube-apiserver` → (gRPC qua UNIX domain socket) → KMS plugin chạy **cùng
  host** với control plane → (giao thức riêng của KMS) → KMS từ xa; mọi credential nói chuyện
  với KMS do plugin tự quản lý.
- Cách khai báo provider `kms` trong `EncryptionConfiguration` và khác biệt v1/v2: v2 đặt
  `apiVersion: v2`, không có `cachesize` (DEK được cache vô thời hạn sau khi unwrap); v1 đã
  deprecated từ v1.28, tắt mặc định từ v1.29, muốn dùng phải bật `--feature-gates=KMSv1=true`.
- Vai trò của `key_id`: giá trị trả về từ `Status` là chuẩn mực (authoritative); `key_id` đổi
  nghĩa là KEK từ xa đã xoay; plugin không được tái sử dụng `key_id` cũ; việc xoay không tức
  thời (API server poll `Status` khoảng mỗi phút và giữ trạng thái hợp lệ cuối chừng 3 phút),
  và Kubernetes khuyến nghị xoay KEK ít nhất mỗi 90 ngày.
- Quy trình đưa KMS vào dùng thật: viết `EncryptionConfiguration` với `kms` đứng đầu → trỏ
  `--encryption-provider-config` → restart API server → verify tiền tố `k8s:enc:kms:v2:`
  bằng `etcdctl` → ép ghi lại toàn bộ Secret bằng
  `kubectl get secrets --all-namespaces -o json | kubectl replace -f -`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| "Phát triển một gRPC server cho KMS plugin" và các ghi chú chi tiết về `EncryptRequest`/`DecryptRequest`, ngưỡng độ trễ, field `annotations` | dành cho người viết plugin, không phải người vận hành; admin chỉ chọn và triển khai plugin có sẵn | chỉ cần khi bạn phát triển plugin riêng — ngoài phạm vi lộ trình admin |
| Chi tiết KMS v1 (`cachesize`, message `v1beta1`) | v1 đã deprecated và tắt mặc định; chỉ gặp khi vận hành cluster cũ hơn v1.29 | khi phải bảo trì cluster legacy còn dùng KMS v1 |
| Bảng health check endpoint (Individual vs Single Healthcheck) | phụ thuộc tổ hợp v1/v2 và auto-reload mà lab không có | khi cấu hình giám sát `kube-apiserver` trong môi trường có KMS thật, cùng kiến thức auto-reload của bài [208](208-encrypt-data-vi.md) |

---

Trang này chỉ ra cách cấu hình một Key Management Service (KMS) provider và plugin để bật mã
hóa dữ liệu secret. Trong Kubernetes 1.36 có hai phiên bản mã hóa at-rest bằng KMS.
Bạn nên dùng KMS v2 nếu khả thi, vì KMS v1 đã bị deprecated (từ Kubernetes v1.28) và bị tắt
mặc định (từ Kubernetes v1.29). KMS v2 có đặc tính hiệu năng tốt hơn đáng kể so với KMS v1.

> **Thận trọng:** Tài liệu này dành cho bản hiện thực đã phổ biến rộng rãi (generally
> available) của KMS v2 (và cho bản hiện thực version 1 đã deprecated). Nếu bạn đang dùng
> bất kỳ thành phần control plane nào cũ hơn Kubernetes v1.29, hãy xem trang tương ứng trong
> tài liệu của phiên bản Kubernetes mà cluster của bạn đang chạy. Các bản phát hành Kubernetes
> trước đó có hành vi khác, có thể liên quan tới an toàn thông tin.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một Kubernetes cluster, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Phiên bản Kubernetes mà bạn cần phụ thuộc vào việc bạn đã chọn phiên bản KMS API nào.
Kubernetes khuyến nghị dùng KMS v2.

- Nếu bạn chọn KMS API v1 để hỗ trợ các cluster trước phiên bản v1.27, hoặc nếu bạn có một
  KMS plugin cũ (legacy) chỉ hỗ trợ KMS v1, thì mọi phiên bản Kubernetes còn được hỗ trợ đều
  dùng được. API này đã bị deprecated kể từ Kubernetes v1.28. Kubernetes không khuyến nghị
  dùng API này.

Để kiểm tra phiên bản, nhập `kubectl version`.

### KMS v1

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.28 [deprecated]`

* Yêu cầu Kubernetes phiên bản 1.10.0 trở lên

* Với phiên bản 1.29 trở lên, bản hiện thực v1 của KMS bị tắt mặc định.
  Để bật tính năng này, đặt `--feature-gates=KMSv1=true` rồi mới cấu hình một KMS v1 provider.

* Cluster của bạn phải dùng etcd v3 trở lên

### KMS v2

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.29 [stable]`

* Cluster của bạn phải dùng etcd v3 trở lên

## Mã hóa KMS và key mã hóa theo từng object (KMS encryption and per-object encryption keys)

KMS encryption provider dùng sơ đồ mã hóa phong bì (envelope encryption) để mã hóa dữ liệu
trong etcd. Dữ liệu được mã hóa bằng một key mã hóa dữ liệu (data encryption key — DEK).
Các DEK được mã hóa bằng một key mã hóa key (key encryption key — KEK) được lưu trữ và quản
lý trong một KMS từ xa.

Nếu bạn dùng bản hiện thực v1 (đã deprecated) của KMS, một DEK mới được sinh ra cho mỗi lần
mã hóa.

Với KMS v2, một DEK mới được sinh ra **cho mỗi lần mã hóa**: API server dùng một
_hàm dẫn xuất key_ (key derivation function) để sinh các key mã hóa dữ liệu dùng một lần từ
một seed bí mật kết hợp với một ít dữ liệu ngẫu nhiên. Seed được xoay mỗi khi KEK được xoay
(xem mục _Hiểu về key_id và việc xoay key_ bên dưới để biết thêm chi tiết).

KMS provider dùng gRPC để giao tiếp với một KMS plugin cụ thể qua UNIX domain socket.
KMS plugin — được hiện thực dưới dạng một gRPC server và được triển khai trên cùng (các) host
với control plane của Kubernetes — chịu trách nhiệm cho toàn bộ giao tiếp với KMS từ xa.

## Cấu hình KMS provider (Configuring the KMS provider)

Để cấu hình một KMS provider trên API server, thêm một provider kiểu `kms` vào mảng
`providers` trong file cấu hình mã hóa và đặt các thuộc tính sau:

### KMS v1 {#configuring-the-kms-provider-kms-v1}

* `apiVersion`: Phiên bản API của KMS provider. Để trống giá trị này hoặc đặt là `v1`.
* `name`: Tên hiển thị của KMS plugin. Không thể thay đổi sau khi đã đặt.
* `endpoint`: Địa chỉ lắng nghe của gRPC server (KMS plugin). Endpoint là một UNIX domain
  socket.
* `cachesize`: Số lượng key mã hóa dữ liệu (DEK) được cache ở dạng rõ (in the clear). Khi đã
  được cache, DEK có thể được dùng mà không cần gọi thêm tới KMS; trong khi các DEK không
  được cache thì cần một lời gọi tới KMS để unwrap.
* `timeout`: `kube-apiserver` chờ kms-plugin phản hồi trong bao lâu trước khi trả về lỗi
  (mặc định là 3 giây).

### KMS v2 {#configuring-the-kms-provider-kms-v2}

* `apiVersion`: Phiên bản API của KMS provider. Đặt giá trị này là `v2`.
* `name`: Tên hiển thị của KMS plugin. Không thể thay đổi sau khi đã đặt.
* `endpoint`: Địa chỉ lắng nghe của gRPC server (KMS plugin). Endpoint là một UNIX domain
  socket.
* `timeout`: `kube-apiserver` chờ kms-plugin phản hồi trong bao lâu trước khi trả về lỗi
  (mặc định là 3 giây).

KMS v2 không hỗ trợ thuộc tính `cachesize`. Tất cả các key mã hóa dữ liệu (DEK) sẽ được cache
ở dạng rõ sau khi server đã unwrap chúng thông qua một lời gọi tới KMS. Một khi đã được cache,
các DEK có thể được dùng để giải mã vô thời hạn mà không cần gọi tới KMS.

Xem [Hiểu về cấu hình mã hóa at rest](208-encrypt-data-vi.md).

## Xây dựng một KMS plugin (Implementing a KMS plugin)

Để có một KMS plugin, bạn có thể phát triển một plugin gRPC server mới hoặc bật một KMS plugin
đã được nhà cung cấp cloud (cloud provider) của bạn cung cấp sẵn. Sau đó bạn tích hợp plugin
với KMS từ xa và triển khai nó trên control plane của Kubernetes.

### Bật KMS được cloud provider của bạn hỗ trợ (Enabling the KMS supported by your cloud provider)

Tham khảo tài liệu của cloud provider để biết hướng dẫn bật KMS plugin đặc thù của cloud
provider đó.

### Phát triển một gRPC server cho KMS plugin (Developing a KMS plugin gRPC server)

Bạn có thể phát triển một gRPC server cho KMS plugin bằng file stub có sẵn cho Go. Với các
ngôn ngữ khác, bạn dùng một file proto để tạo file stub mà bạn có thể dùng để phát triển mã
nguồn của gRPC server.

#### KMS v1 {#developing-a-kms-plugin-gRPC-server-kms-v1}

* Dùng Go: Dùng các hàm và cấu trúc dữ liệu trong file stub:
  [api.pb.go](https://github.com/kubernetes/kms/blob/release-1.36/apis/v1beta1/api.pb.go)
  để phát triển mã nguồn gRPC server

* Dùng ngôn ngữ khác Go: Dùng trình biên dịch protoc với file proto:
  [api.proto](https://github.com/kubernetes/kms/blob/release-1.36/apis/v1beta1/api.proto)
  để sinh file stub cho ngôn ngữ cụ thể đó

#### KMS v2 {#developing-a-kms-plugin-gRPC-server-kms-v2}

* Dùng Go: Một [thư viện](https://github.com/kubernetes/kms/blob/release-1.36/pkg/service/interface.go)
  mức cao được cung cấp để quá trình này dễ dàng hơn. Các bản hiện thực mức thấp có thể dùng
  các hàm và cấu trúc dữ liệu trong file stub:
  [api.pb.go](https://github.com/kubernetes/kms/blob/release-1.36/apis/v2/api.pb.go)
  để phát triển mã nguồn gRPC server

* Dùng ngôn ngữ khác Go: Dùng trình biên dịch protoc với file proto:
  [api.proto](https://github.com/kubernetes/kms/blob/release-1.36/apis/v2/api.proto)
  để sinh file stub cho ngôn ngữ cụ thể đó

Sau đó dùng các hàm và cấu trúc dữ liệu trong file stub để phát triển mã nguồn server.

#### Ghi chú (Notes)

##### KMS v1 {#developing-a-kms-plugin-gRPC-server-notes-kms-v1}

* Phiên bản kms plugin: `v1beta1`

  Khi phản hồi lời gọi thủ tục (procedure call) Version, một KMS plugin tương thích phải trả
  về `v1beta1` trong `VersionResponse.version`.

* Phiên bản message: `v1beta1`

  Mọi message từ KMS provider đều có field version đặt là `v1beta1`.

* Giao thức: UNIX domain socket (`unix`)

  Plugin được hiện thực dưới dạng một gRPC server lắng nghe trên UNIX domain socket. Bản
  triển khai plugin phải tạo một file trên filesystem để chạy kết nối gRPC unix domain
  socket. API server (gRPC client) được cấu hình với endpoint unix domain socket của KMS
  provider (gRPC server) để giao tiếp với nó. Có thể dùng abstract Linux socket bằng cách
  bắt đầu endpoint với `/@`, tức là `unix:///@foo`. Phải cẩn trọng khi dùng loại socket này
  vì chúng không có khái niệm ACL (khác với socket dựa trên file truyền thống). Tuy nhiên,
  chúng chịu sự chi phối của Linux networking namespace, nên chỉ các container trong cùng
  một pod truy cập được, trừ khi dùng host networking.

##### KMS v2 {#developing-a-kms-plugin-gRPC-server-notes-kms-v2}

* Phiên bản KMS plugin: `v2`

  Khi phản hồi lời gọi thủ tục từ xa `Status`, một KMS plugin tương thích phải trả về phiên
  bản tương thích KMS của nó trong `StatusResponse.version`. Phản hồi status đó cũng phải
  bao gồm "ok" trong `StatusResponse.healthz` và một `key_id` (ID của KEK trong KMS từ xa)
  trong `StatusResponse.key_id`. Dự án Kubernetes khuyến nghị bạn làm cho plugin của mình
  tương thích với KMS API `v2` ổn định. Kubernetes 1.36 cũng hỗ trợ API `v2beta1`
  cho KMS; các bản phát hành Kubernetes tương lai nhiều khả năng sẽ tiếp tục hỗ trợ phiên
  bản beta đó.

  API server poll lời gọi thủ tục `Status` xấp xỉ mỗi phút một lần khi mọi thứ khỏe mạnh,
  và mỗi 10 giây một lần khi plugin không khỏe mạnh. Các plugin phải chú ý tối ưu lời gọi
  này vì nó sẽ chịu tải liên tục.

* Mã hóa (Encryption)

  Lời gọi thủ tục `EncryptRequest` cung cấp bản rõ (plaintext) và một UID phục vụ mục đích
  ghi log. Phản hồi phải bao gồm bản mã (ciphertext), `key_id` của KEK đã dùng, và — tùy
  chọn — bất kỳ metadata nào mà KMS plugin cần để hỗ trợ các lời gọi `DecryptRequest` trong
  tương lai (qua field `annotations`). Plugin phải bảo đảm rằng mọi bản rõ khác nhau đều cho
  ra một phản hồi `(ciphertext, key_id, annotations)` khác nhau.

  Nếu plugin trả về một map `annotations` khác rỗng, mọi key của map phải là tên miền đầy đủ
  (fully qualified domain name) như `example.com`. Một ví dụ về trường hợp dùng `annotation`
  là `{"kms.example.io/remote-kms-auditid":"<audit ID mà KMS từ xa sử dụng>"}`

  API server không thực hiện lời gọi thủ tục `EncryptRequest` với tần suất cao. Dù vậy, các
  bản hiện thực plugin vẫn nên hướng tới việc giữ độ trễ của mỗi request dưới 100 mili giây.

* Giải mã (Decryption)

  Lời gọi thủ tục `DecryptRequest` cung cấp bộ `(ciphertext, key_id, annotations)` từ
  `EncryptRequest` và một UID phục vụ mục đích ghi log. Đúng như kỳ vọng, nó là phép đảo
  ngược của lời gọi `EncryptRequest`. Plugin phải xác minh rằng `key_id` là một giá trị mà
  chúng hiểu — chúng không được cố giải mã dữ liệu trừ khi chắc chắn rằng dữ liệu đó do
  chính chúng mã hóa vào một thời điểm trước đó.

  API server có thể thực hiện hàng nghìn lời gọi thủ tục `DecryptRequest` khi khởi động để
  lấp đầy watch cache của nó. Do đó các bản hiện thực plugin phải thực hiện các lời gọi này
  nhanh nhất có thể, và nên hướng tới việc giữ độ trễ của mỗi request dưới 10 mili giây.

* Hiểu về `key_id` và việc xoay key (Understanding `key_id` and Key Rotation)

  `key_id` là tên công khai, không bí mật, của KEK trong KMS từ xa hiện đang được dùng. Nó
  có thể bị ghi log trong quá trình hoạt động bình thường của API server, do đó không được
  chứa bất kỳ dữ liệu riêng tư nào. Các bản hiện thực plugin được khuyến khích dùng một giá
  trị hash để tránh rò rỉ bất kỳ dữ liệu nào. Các metric của KMS v2 chú ý hash giá trị này
  trước khi phơi bày nó qua endpoint `/metrics`.

  API server coi `key_id` trả về từ lời gọi thủ tục `Status` là chuẩn mực (authoritative).
  Do đó, một thay đổi ở giá trị này báo hiệu cho API server rằng KEK từ xa đã thay đổi, và
  dữ liệu được mã hóa bằng KEK cũ cần được đánh dấu là cũ (stale) khi một thao tác ghi
  không đổi nội dung (no-op write) được thực hiện (như mô tả bên dưới). Nếu một lời gọi thủ
  tục `EncryptRequest` trả về một `key_id` khác với `Status`, phản hồi đó bị vứt bỏ và
  plugin bị coi là không khỏe mạnh. Vì vậy các bản hiện thực phải bảo đảm `key_id` trả về
  từ `Status` giống với `key_id` trả về bởi `EncryptRequest`. Hơn nữa, plugin phải bảo đảm
  `key_id` ổn định và không nhảy qua lại giữa các giá trị (tức là trong lúc xoay KEK từ xa).

  Plugin không được tái sử dụng các `key_id`, kể cả trong tình huống một KEK từ xa từng dùng
  trước đây được đưa vào dùng lại. Ví dụ, nếu một plugin từng dùng `key_id=A`, chuyển sang
  `key_id=B`, rồi quay lại `key_id=A` — thay vì báo cáo `key_id=A`, plugin nên báo cáo một
  giá trị dẫn xuất như `key_id=A_001` hoặc dùng một giá trị mới như `key_id=C`.

  Vì API server poll `Status` khoảng mỗi phút một lần, việc xoay `key_id` không diễn ra tức
  thời. Hơn nữa, API server sẽ tiếp tục dùng trạng thái hợp lệ cuối cùng trong khoảng ba
  phút. Do đó, nếu người dùng muốn tiếp cận việc di trú lưu trữ (storage migration) một cách
  thụ động (tức là bằng cách chờ đợi), họ phải lên lịch việc di trú diễn ra sau `3 + N + M`
  phút kể từ khi KEK từ xa được xoay (`N` là thời gian plugin cần để quan sát thấy `key_id`
  thay đổi và `M` là khoảng đệm mong muốn để các thay đổi cấu hình được xử lý — khuyến nghị
  `M` tối thiểu là năm phút). Lưu ý rằng không cần restart API server để thực hiện xoay KEK.

  > **Thận trọng:** Vì bạn không kiểm soát được số lần ghi thực hiện với DEK, dự án
  > Kubernetes khuyến nghị xoay KEK ít nhất mỗi 90 ngày.

* Giao thức: UNIX domain socket (`unix`)

  Plugin được hiện thực dưới dạng một gRPC server lắng nghe trên UNIX domain socket. Bản
  triển khai plugin phải tạo một file trên filesystem để chạy kết nối gRPC unix domain
  socket. API server (gRPC client) được cấu hình với endpoint unix domain socket của KMS
  provider (gRPC server) để giao tiếp với nó. Có thể dùng abstract Linux socket bằng cách
  bắt đầu endpoint với `/@`, tức là `unix:///@foo`. Phải cẩn trọng khi dùng loại socket này
  vì chúng không có khái niệm ACL (khác với socket dựa trên file truyền thống). Tuy nhiên,
  chúng chịu sự chi phối của Linux networking namespace, nên chỉ các container trong cùng
  một pod truy cập được, trừ khi dùng host networking.

### Tích hợp KMS plugin với KMS từ xa (Integrating a KMS plugin with the remote KMS)

KMS plugin có thể giao tiếp với KMS từ xa bằng bất kỳ giao thức nào mà KMS đó hỗ trợ. Mọi
dữ liệu cấu hình, bao gồm cả credential xác thực mà KMS plugin dùng để giao tiếp với KMS từ
xa, đều do KMS plugin tự lưu trữ và quản lý một cách độc lập. KMS plugin có thể mã hóa bản
mã kèm thêm metadata bổ sung có thể cần đến trước khi gửi nó tới KMS để giải mã (KMS v2 làm
quá trình này dễ hơn bằng cách cung cấp field `annotations` chuyên dụng).

### Triển khai KMS plugin (Deploying the KMS plugin)

Bảo đảm rằng KMS plugin chạy trên cùng (các) host với (các) Kubernetes API server.

## Mã hóa dữ liệu của bạn bằng KMS provider (Encrypting your data with the KMS provider)

Để mã hóa dữ liệu:

1. Tạo một file `EncryptionConfiguration` mới dùng các thuộc tính phù hợp cho provider `kms`
   để mã hóa các resource như Secret và ConfigMap. Nếu bạn muốn mã hóa một extension API được
   định nghĩa trong một CustomResourceDefinition, cluster của bạn phải đang chạy Kubernetes
   v1.26 trở lên.

1. Đặt flag `--encryption-provider-config` trên kube-apiserver trỏ tới vị trí của file cấu
   hình.

1. Đối số boolean `--encryption-provider-config-automatic-reload` quyết định file được đặt
   bởi `--encryption-provider-config` có được
   [tự động nạp lại](208-encrypt-data-vi.md#configure-automatic-reloading)
   khi nội dung trên đĩa thay đổi hay không.

1. Restart API server của bạn.

### KMS v1 {#encrypting-your-data-with-the-kms-provider-kms-v1}

   ```yaml
   apiVersion: apiserver.config.k8s.io/v1
   kind: EncryptionConfiguration
   resources:
     - resources:
         - secrets
         - configmaps
         - pandas.awesome.bears.example
       providers:
         - kms:
             name: myKmsPluginFoo
             endpoint: unix:///tmp/socketfile-foo.sock
             cachesize: 100
             timeout: 3s
         - kms:
             name: myKmsPluginBar
             endpoint: unix:///tmp/socketfile-bar.sock
             cachesize: 100
             timeout: 3s
   ```

### KMS v2 {#encrypting-your-data-with-the-kms-provider-kms-v2}

   ```yaml
   apiVersion: apiserver.config.k8s.io/v1
   kind: EncryptionConfiguration
   resources:
     - resources:
         - secrets
         - configmaps
         - pandas.awesome.bears.example
       providers:
         - kms:
             apiVersion: v2
             name: myKmsPluginFoo
             endpoint: unix:///tmp/socketfile-foo.sock
             timeout: 3s
         - kms:
             apiVersion: v2
             name: myKmsPluginBar
             endpoint: unix:///tmp/socketfile-bar.sock
             timeout: 3s
   ```

Đặt `--encryption-provider-config-automatic-reload` thành `true` sẽ gộp mọi health check về
một endpoint health check duy nhất. Các health check riêng lẻ chỉ khả dụng khi đang dùng các
KMS v1 provider và cấu hình mã hóa không được tự động nạp lại.

Bảng sau tóm tắt các endpoint health check cho từng phiên bản KMS:

| Cấu hình KMS | Không có Automatic Reload | Có Automatic Reload |
| ------------------ | ------------------------ | --------------------- |
| Chỉ KMS v1         | Healthcheck riêng lẻ     | Healthcheck duy nhất  |
| Chỉ KMS v2         | Healthcheck duy nhất     | Healthcheck duy nhất  |
| Cả KMS v1 và v2    | Healthcheck riêng lẻ     | Healthcheck duy nhất  |
| Không dùng KMS     | Không có                 | Healthcheck duy nhất  |

`Healthcheck duy nhất` (Single Healthcheck) nghĩa là endpoint health check duy nhất là
`/healthz/kms-providers`.

`Healthcheck riêng lẻ` (Individual Healthchecks) nghĩa là mỗi KMS plugin có một endpoint
health check gắn liền, dựa trên vị trí của nó trong cấu hình mã hóa: `/healthz/kms-provider-0`,
`/healthz/kms-provider-1`, v.v.

Các đường dẫn endpoint healthcheck này được viết cứng (hard coded) và do server sinh
ra/kiểm soát. Chỉ số (index) của các healthcheck riêng lẻ tương ứng với thứ tự mà cấu hình
mã hóa KMS được xử lý.

Cho tới khi các bước được định nghĩa trong
[Bảo đảm mọi secret đều được mã hóa](#ensuring-all-secrets-are-encrypted) được thực hiện,
danh sách `providers` nên kết thúc bằng provider `identity: {}` để cho phép đọc dữ liệu chưa
mã hóa. Một khi mọi resource đã được mã hóa, nên gỡ bỏ provider `identity` để ngăn API server
chấp nhận dữ liệu chưa mã hóa.

Để biết chi tiết về định dạng `EncryptionConfiguration`, hãy xem
[tài liệu tham chiếu API mã hóa của API server](https://kubernetes.io/docs/reference/config-api/apiserver-config.v1/).

## Xác minh dữ liệu đã được mã hóa (Verifying that the data is encrypted)

Khi mã hóa at rest được cấu hình đúng, các resource được mã hóa khi ghi. Sau khi restart
`kube-apiserver`, bất kỳ Secret mới tạo hoặc mới cập nhật nào — hay các loại resource khác
được cấu hình trong `EncryptionConfiguration` — sẽ được mã hóa khi lưu trữ. Để xác minh, bạn
có thể dùng chương trình dòng lệnh `etcdctl` để truy xuất nội dung dữ liệu secret của bạn.

1. Tạo một secret mới tên `secret1` trong namespace `default`:

   ```shell
   kubectl create secret generic secret1 -n default --from-literal=mykey=mydata
   ```

1. Dùng dòng lệnh `etcdctl`, đọc secret đó ra từ etcd:

   ```shell
   ETCDCTL_API=3 etcdctl get /kubernetes.io/secrets/default/secret1 [...] | hexdump -C
   ```

   trong đó `[...]` chứa các đối số bổ sung để kết nối tới etcd server.

1. Xác minh rằng secret được lưu trữ có tiền tố `k8s:enc:kms:v1:` với KMS v1 hoặc tiền tố
   `k8s:enc:kms:v2:` với KMS v2 — điều này cho thấy provider `kms` đã mã hóa dữ liệu kết quả.

1. Xác minh rằng secret được giải mã đúng khi truy xuất qua API:

   ```shell
   kubectl describe secret secret1 -n default
   ```

   Secret phải chứa `mykey: mydata`

## Bảo đảm mọi secret đều được mã hóa (Ensuring all secrets are encrypted) {#ensuring-all-secrets-are-encrypted}

Khi mã hóa at rest được cấu hình đúng, các resource được mã hóa khi ghi. Vì vậy chúng ta có
thể thực hiện một thao tác cập nhật tại chỗ không đổi nội dung (in-place no-op update) để
bảo đảm dữ liệu được mã hóa.

Lệnh sau đọc mọi secret rồi cập nhật lại chúng để áp dụng mã hóa phía server. Nếu xảy ra lỗi
do xung đột ghi (conflicting write), hãy thử lại lệnh. Với các cluster lớn hơn, bạn có thể
muốn chia nhỏ các secret theo namespace hoặc viết script cho việc cập nhật.

```shell
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

## Chuyển từ encryption provider cục bộ sang KMS provider (Switching from a local encryption provider to the KMS provider)

Để chuyển từ một encryption provider cục bộ sang provider `kms` và mã hóa lại toàn bộ secret:

1. Thêm provider `kms` làm mục đầu tiên trong file cấu hình như ví dụ sau.

   ```yaml
   apiVersion: apiserver.config.k8s.io/v1
   kind: EncryptionConfiguration
   resources:
     - resources:
         - secrets
       providers:
         - kms:
             apiVersion: v2
             name : myKmsPlugin
             endpoint: unix:///tmp/socketfile.sock
         - aescbc:
             keys:
               - name: key1
                 secret: <BASE 64 ENCODED SECRET>
   ```

1. Restart mọi tiến trình `kube-apiserver`.

1. Chạy lệnh sau để buộc mọi secret được mã hóa lại bằng provider `kms`.

   ```shell
   kubectl get secrets --all-namespaces -o json | kubectl replace -f -
   ```

## Tiếp theo (What's next)

<a id="disabling-encryption-at-rest" />

Nếu bạn không còn muốn dùng mã hóa cho dữ liệu được lưu bền vững trong Kubernetes API, hãy đọc
[giải mã dữ liệu đã được lưu trữ at rest](202-decrypt-data-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc này:

1. **Câu bẫy.** Khi bật KMS v2, mỗi lần bạn tạo hoặc đọc một Secret, `kube-apiserver` có phải
   gọi tới dịch vụ KMS từ xa để mã hóa/giải mã Secret đó không? Vì sao?
2. Trong sơ đồ envelope encryption của bài, thứ gì nằm trong KMS từ xa, thứ gì nằm phía
   Kubernetes? Nếu kẻ tấn công lấy được bản dump etcd (nhưng không có quyền gọi KMS), chúng
   đọc được gì?
3. Trên cluster lab của bạn (control plane là `k8s-master`), nếu muốn dùng một KMS provider
   thì KMS plugin phải được triển khai ở đâu, và `kube-apiserver` giao tiếp với nó bằng cơ
   chế gì? Giá trị nào trong `EncryptionConfiguration` khai báo điểm giao tiếp đó?
4. Plugin từng dùng `key_id=A`, chuyển sang `key_id=B`, rồi KEK cũ được đưa vào dùng lại.
   Plugin có được báo cáo lại `key_id=A` không? Và vì sao sau khi xoay KEK, bạn không nên
   chạy di trú lưu trữ ngay lập tức mà phải chờ?
5. Sau khi cấu hình provider `kms` đứng đầu danh sách và restart API server, vì sao vẫn phải
   chạy `kubectl get secrets --all-namespaces -o json | kubectl replace -f -`, và làm sao
   chứng minh một Secret cụ thể đã thực sự được mã hóa bằng KMS?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không phải mỗi lần.** Đây chính là điểm của envelope encryption: dữ liệu được mã hóa
   bằng DEK sinh **cục bộ** (KMS v2 dẫn xuất DEK dùng một lần từ một seed bí mật); KMS từ xa
   chỉ dùng để wrap/unwrap seed bằng KEK. Sau khi server unwrap, DEK được cache ở dạng rõ và
   dùng để giải mã vô thời hạn mà không cần gọi KMS. Bài cũng nói rõ API server không gọi
   `EncryptRequest` với tần suất cao. Trực giác "mã hóa bằng KMS nghĩa là KMS mã hóa từng
   object" sai — KMS chỉ giữ và vận hành KEK.
2. **KEK nằm trong KMS từ xa; DEK/seed (đã được KEK mã hóa) và dữ liệu (đã được DEK mã hóa)
   nằm phía Kubernetes trong etcd.** Kẻ có bản dump etcd chỉ thấy bản mã với tiền tố
   `k8s:enc:kms:...` và DEK ở dạng đã wrap — không giải mã được vì unwrap DEK đòi hỏi gọi
   được KMS từ xa (mọi credential đó do KMS plugin quản lý, không nằm trong etcd).
3. KMS plugin phải chạy **trên cùng host với `kube-apiserver`**, tức là trên `k8s-master`
   (bài yêu cầu plugin được triển khai trên cùng (các) host với control plane). API server
   là gRPC client nói chuyện với plugin (gRPC server) qua **UNIX domain socket**; điểm giao
   tiếp được khai báo bằng thuộc tính `endpoint` (dạng `unix:///...`) của provider `kms`
   trong `EncryptionConfiguration`.
4. **Không.** Plugin không được tái sử dụng `key_id`, kể cả khi KEK cũ được dùng lại — phải
   báo giá trị dẫn xuất như `A_001` hoặc giá trị mới, vì API server coi thay đổi `key_id` từ
   `Status` là tín hiệu KEK đã xoay và `key_id` nhảy qua lại sẽ làm plugin bị coi là không
   khỏe mạnh. Phải chờ vì việc xoay không tức thời: API server chỉ poll `Status` khoảng mỗi
   phút và còn dùng tiếp trạng thái hợp lệ cuối chừng ba phút, nên di trú thụ động phải lên
   lịch ở mốc `3 + N + M` phút sau khi xoay KEK.
5. Vì mã hóa chỉ áp dụng **khi ghi**: các Secret tạo trước đó vẫn nằm trong etcd ở dạng do
   provider cũ ghi (hoặc dạng rõ). Lệnh no-op replace đọc mọi Secret rồi ghi đè lại, lần ghi
   này đi qua provider `kms` đứng đầu. Chứng minh bằng cách đọc thẳng từ etcd:
   `ETCDCTL_API=3 etcdctl get /kubernetes.io/secrets/<ns>/<tên> ... | hexdump -C` và kiểm tra
   tiền tố `k8s:enc:kms:v2:` (v2) hay `k8s:enc:kms:v1:` (v1), rồi `kubectl describe secret`
   để xác nhận API vẫn giải mã đúng.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
