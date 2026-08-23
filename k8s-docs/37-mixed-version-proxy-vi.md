# Proxy phiên bản hỗn hợp (Mixed Version Proxy)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/architecture/mixed-version-proxy/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 1 → nhóm [1c](LO-TRINH-ADMIN.md#1c-vòng-đời-và-cơ-chế-nền-của-object),
bài 6/7 · [Lab 1c](labs/LAB-1C-VONG-DOI-VA-CO-CHE-NEN-CUA-OBJECT.md) phần B5 kiểm chứng vì sao
topology một API server không có đường Mixed Version Proxy để fault-inject.

**Lộ trình đánh dấu bài này là "đọc lướt".** Nó chỉ có ý nghĩa khi cluster có **nhiều API
server chạy các phiên bản Kubernetes khác nhau** — tình huống chỉ xuất hiện giữa chừng một đợt
nâng cấp cluster HA. Cluster lab một control plane của bạn không bao giờ rơi vào cảnh đó.

**Phải hiểu ở lần đọc này — đúng một ý:**

- Khi nâng cấp một cluster HA, trong lúc chuyển tiếp có API server đã biết một API mới còn API
  server khác thì chưa. Tính năng này khiến request rơi vào server cũ được **proxy sang server
  biết API đó**, thay vì trả về `404` gây hiểu lầm.

Biết tên tính năng và biết nó tồn tại là đủ. Đừng đọc kỹ hơn ở giai đoạn 1.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Toàn bộ mục *Bật Peer-aggregated Discovery…* và các cờ `--peer-*` | là cấu hình API server | giai đoạn 8 |
| Truyền tải proxy và xác thực giữa các API server | cần hiểu certificate và aggregation layer | giai đoạn 9 và 14 |
| *Cách hoạt động bên trong* | chi tiết hiện thực | quay lại khi nâng cấp cluster HA thật |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [beta]`

Kubernetes v1.36 bao gồm một tính năng beta cho phép một API Server ủy quyền (proxy)
các yêu cầu tài nguyên đến các API server _ngang hàng_ (_peer_) khác. Tính năng này cũng
cho phép client có được cái nhìn toàn cảnh về các tài nguyên được phục vụ trên toàn bộ
cluster thông qua cơ chế khám phá (discovery). Điều này hữu ích khi trong một cluster có
nhiều API server chạy các phiên bản Kubernetes khác nhau (ví dụ, trong quá trình chuyển
đổi kéo dài sang một bản phát hành Kubernetes mới).

Điều này cho phép quản trị viên cluster cấu hình các cluster có tính sẵn sàng cao
(highly available) có thể được nâng cấp an toàn hơn, bằng cách:

1. đảm bảo rằng các controller dựa vào discovery để hiển thị danh sách toàn diện các
tài nguyên cho những tác vụ quan trọng luôn nhận được cái nhìn hoàn chỉnh về tất cả các
tài nguyên. Chúng ta gọi cơ chế khám phá đầy đủ trên toàn cluster này là
_Peer-aggregated discovery_ (khám phá tổng hợp từ các peer).
1. chuyển hướng các yêu cầu tài nguyên (được thực hiện trong quá trình nâng cấp) đến
đúng kube-apiserver. Việc proxy này giúp người dùng không gặp phải các lỗi 404 Not Found
bất ngờ bắt nguồn từ quá trình nâng cấp. Cơ chế này được gọi là _Mixed Version Proxy_
(proxy phiên bản hỗn hợp).

## Bật Peer-aggregated Discovery và Mixed Version Proxy (Enabling Peer-aggregated Discovery and Mixed Version Proxy)

Đảm bảo rằng [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#UnknownVersionInteroperabilityProxy)
`UnknownVersionInteroperabilityProxy` được bật khi bạn khởi động API Server:

```shell
kube-apiserver \
--feature-gates=UnknownVersionInteroperabilityProxy=true \
# các đối số dòng lệnh bắt buộc cho tính năng này
--peer-ca-file=<path to kube-apiserver CA cert>
--proxy-client-cert-file=<path to aggregator proxy cert>,
--proxy-client-key-file=<path to aggregator proxy key>,
--requestheader-client-ca-file=<path to aggregator CA cert>,
# requestheader-allowed-names có thể để trống để cho phép mọi Common Name
--requestheader-allowed-names=<valid Common Names to verify proxy client cert against>,

# các cờ (flag) tùy chọn cho tính năng này
--peer-advertise-ip=`IP of this kube-apiserver that should be used by peers to proxy requests`
--peer-advertise-port=`port of this kube-apiserver that should be used by peers to proxy requests`

# …và các cờ khác như thường lệ
```

### Truyền tải proxy và xác thực giữa các API server (Proxy transport and authentication between API servers) {#transport-and-authn}

* kube-apiserver nguồn tái sử dụng
  [các cờ xác thực client hiện có của API server](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-aggregation-layer/#kubernetes-apiserver-client-authentication)
  `--proxy-client-cert-file` và `--proxy-client-key-file` để trình ra danh tính của mình,
  danh tính này sẽ được peer của nó (kube-apiserver đích) xác minh. API server đích
  xác minh kết nối peer đó dựa trên cấu hình bạn chỉ định thông qua đối số dòng lệnh
  `--requestheader-client-ca-file`.

* Để xác thực chứng chỉ phục vụ (serving certificate) của server đích, bạn phải cấu hình
  một bundle certificate authority bằng cách chỉ định đối số dòng lệnh `--peer-ca-file`
  cho API server **nguồn**.

### Cấu hình kết nối tới peer API server (Configuration for peer API server connectivity)

Để thiết lập vị trí mạng của một kube-apiserver mà các peer sẽ dùng để proxy các yêu cầu,
hãy sử dụng các đối số dòng lệnh `--peer-advertise-ip` và `--peer-advertise-port` cho
kube-apiserver, hoặc chỉ định các trường này trong file cấu hình API server.
Nếu các cờ này không được chỉ định, các peer sẽ dùng giá trị từ đối số dòng lệnh
`--advertise-address` hoặc `--bind-address` của kube-apiserver.
Nếu các giá trị đó cũng không được thiết lập, giao diện mạng (interface) mặc định của
máy chủ sẽ được sử dụng.

## Peer-aggregated discovery

Khi bạn bật tính năng này, các yêu cầu discovery sẽ mặc định tự động được phục vụ bằng
một tài liệu discovery toàn diện (liệt kê tất cả các tài nguyên được phục vụ bởi bất kỳ
apiserver nào trong cluster).

Nếu bạn muốn yêu cầu một tài liệu discovery không tổng hợp từ các peer
(non peer-aggregated), bạn có thể biểu thị điều đó bằng cách thêm Accept header sau vào
yêu cầu discovery:

```
application/json;g=apidiscovery.k8s.io;v=v2;as=APIGroupDiscoveryList;profile=nopeer
```

> **Ghi chú:**
> Peer-aggregated discovery chỉ được hỗ trợ cho các yêu cầu
> [Aggregated Discovery](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#aggregated-discovery)
> tới endpoint `/apis`, và không áp dụng cho các yêu cầu
> [Unaggregated (Legacy) Discovery](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#unaggregated-discovery).

## Cơ chế proxy phiên bản hỗn hợp (Mixed version proxying)

Khi bạn bật cơ chế proxy phiên bản hỗn hợp,
[tầng tổng hợp (aggregation layer)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/)
sẽ nạp một filter đặc biệt thực hiện những việc sau:

* Khi một yêu cầu tài nguyên đến một API server không thể phục vụ API đó
  (hoặc vì server này ở phiên bản có trước khi API được giới thiệu, hoặc API bị tắt trên
  API server đó), API server sẽ cố gắng gửi yêu cầu đến một peer API server có thể phục
  vụ API được yêu cầu. Nó thực hiện điều này bằng cách nhận diện các API group / phiên
  bản / tài nguyên mà server cục bộ không nhận ra, và cố gắng proxy các yêu cầu đó tới
  một peer API server có khả năng xử lý yêu cầu.
* Nếu peer API server không phản hồi, API server _nguồn_ sẽ trả về lỗi 503
  ("Service Unavailable").

### Cách hoạt động bên trong (How it works under the hood)

Khi một API Server nhận được một yêu cầu tài nguyên, trước tiên nó kiểm tra xem những
API server nào có thể phục vụ tài nguyên được yêu cầu. Việc kiểm tra này được thực hiện
bằng tài liệu discovery non peer-aggregated (không tổng hợp từ các peer).

* Nếu tài nguyên có trong tài liệu discovery non peer-aggregated lấy từ API server đã
  nhận yêu cầu (ví dụ, `GET /api/v1/pods/some-pod`), yêu cầu sẽ được xử lý cục bộ.

* Nếu tài nguyên trong một yêu cầu (ví dụ, `GET /apis/resource.k8s.io/v1beta1/resourceclaims`)
  không được tìm thấy trong tài liệu discovery non peer-aggregated lấy từ API server
  đang cố xử lý yêu cầu (gọi là _API server xử lý_ — _handling API server_), nhiều khả
  năng vì API `resource.k8s.io/v1beta1` được giới thiệu trong một phiên bản Kubernetes
  mới hơn trong khi _API server xử lý_ đang chạy phiên bản cũ hơn không hỗ trợ API đó,
  thì _API server xử lý_ sẽ truy xuất các peer API server có phục vụ API group / phiên
  bản / tài nguyên liên quan (`resource.k8s.io/v1beta1/resourceclaims` trong trường hợp
  này) bằng cách kiểm tra tài liệu discovery non peer-aggregated từ tất cả các peer API
  server. Sau đó _API server xử lý_ proxy yêu cầu tới một trong những peer
  kube-apiserver phù hợp có biết về tài nguyên được yêu cầu.

* Nếu không có peer nào được biết đến cho API group / phiên bản / tài nguyên đó, API
  server xử lý sẽ chuyển yêu cầu vào chuỗi handler của chính nó, và cuối cùng chuỗi này
  sẽ trả về phản hồi 404 ("Not Found").

* Nếu API server xử lý đã nhận diện và chọn được một peer API server, nhưng peer đó
  không phản hồi (vì các lý do như sự cố kết nối mạng, hoặc tình huống tranh chấp dữ
  liệu (data race) giữa thời điểm yêu cầu được nhận và thời điểm một controller đăng ký
  thông tin của peer vào control plane), thì API server xử lý sẽ trả về lỗi 503
  ("Service Unavailable").

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

1. Tính năng này giải quyết vấn đề gì, và nó chỉ xuất hiện trong hoàn cảnh nào?
2. Cluster lab một control plane của bạn có cần tính năng này không? Vì sao?
3. Không có tính năng này, một client gọi API mà API server đang xử lý chưa biết sẽ nhận được
   mã lỗi nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Nó tránh **lỗi `404 Not Found` bất ngờ trong quá trình nâng cấp**, khi cluster tạm thời có
   nhiều API server chạy các phiên bản Kubernetes khác nhau. Request tới một server chưa biết
   API đó sẽ được proxy sang một peer biết nó.
2. **Không.** Cluster chỉ có một API server nên không bao giờ tồn tại hai phiên bản API server
   cùng lúc. Đây đúng là lý do lộ trình xếp bài này vào diện đọc lướt.
3. **`404` (Not Found)** — API server xử lý chuyển request vào chuỗi handler của chính nó và
   chuỗi này kết thúc bằng 404. Nếu đã chọn được peer nhưng peer không phản hồi thì là **`503`
   (Service Unavailable)**.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
