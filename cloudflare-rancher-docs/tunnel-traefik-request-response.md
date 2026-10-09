## Luồng end-to-end theo 14 bước

Phần này mô tả toàn bộ quá trình: chuẩn bị tunnel, browser gửi request, connector chuyển tiếp vào Kubernetes, Pod xử lý và response quay về.

Dùng request ví dụ:

```text
https://app.hieupn.site/?page=2
```

Các đoạn HTTP được viết theo HTTP/1.1 để dễ nhìn. Browser thực tế có thể dùng HTTP/2 hoặc HTTP/3. Giả sử request được chuyển tới origin, không bị Edge chặn hoặc trả ngay từ cache.

### Bước 1 — §12 chuẩn bị đường đi trước khi có request

Trước khi người dùng truy cập, các cấu hình sau phải tồn tại:

| Bước runbook | Thiết lập | Mục đích |
| --- | --- | --- |
| §9.3 | Traefik và Service `traefik` kiểu ClusterIP | Tạo điểm vào nội bộ của ingress controller |
| §10 | Deployment `web`, Service `web`, Ingress `web` | Tạo ứng dụng và đường định tuyến nội bộ |
| §11 | Domain được Cloudflare quản lý DNS, zone `Active` | Chuẩn bị domain public |
| §12.1 | Tunnel `homelab-k8s` và token | Tạo danh tính tunnel |
| §12.2 | Hai Pod `cloudflared` dùng token | Kết nối cluster với Cloudflare |
| §12.3.1 | Ingress dùng Host `app.hieupn.site` | Khớp hostname public |
| §12.3.3 | Published application route | Gắn hostname với tunnel và origin |

Khi khởi động, `cloudflared` đọc token qua biến môi trường `TUNNEL_TOKEN`, chủ động mở kết nối outbound mã hóa tới Cloudflare bằng QUIC/UDP `7844` hoặc HTTP/2/TCP `7844`, rồi đăng ký connector với tunnel.

Cả hai connector đều có thể Active/Healthy, không có primary/standby cố định.

**Mở tunnel là thiết lập kênh truyền lâu dài.** Đây không phải một HTTP `GET` hay `POST` tới ứng dụng để lấy trang web. Token phục vụ việc tham gia tunnel, không phải payload gửi tới Pod web.

Router/NAT theo dõi kết nối outbound và cho phép dữ liệu thuộc kết nối đó quay về. Vì vậy, Cloudflare không cần mở kết nối inbound mới tới master, worker hoặc router. Lab không cần port-forward hay mở inbound `80`, `443`, `7844` trên router cho đường vào này.

Hai cấu hình chỉ đường đã được chuẩn bị:

| Cấu hình | Ánh xạ |
| --- | --- |
| Published application route | `app.hieupn.site` → tunnel `homelab-k8s` → origin `http://traefik.traefik.svc.cluster.local:80` |
| Ingress `web` | Host `app.hieupn.site` + path Prefix `/` → Service `web:80` |

Token gắn connector với tunnel, không khóa connector vào một cluster cụ thể. Nếu dùng cùng token ở cluster khác, connector đó cũng có thể tham gia cùng tunnel.

### Bước 2 — Browser phân tích URL

Người dùng mở:

```text
https://app.hieupn.site/?page=2
```

Browser đọc:

| Thành phần | Giá trị | Vai trò |
| --- | --- | --- |
| Scheme | `https` | Sử dụng HTTP qua TLS |
| Hostname | `app.hieupn.site` | Tìm IP, kiểm tra certificate và xác định hostname của request |
| Port | `443`, mặc định của HTTPS | Cổng kết nối tới Edge |
| Path | `/` | Đường dẫn yêu cầu |
| Query | `page=2` | Tham số gửi tới ứng dụng |

Browser cần tìm IP, thiết lập kết nối an toàn, rồi gửi request.

### Bước 3 — Browser hỏi public DNS

Browser/hệ điều hành sử dụng DNS resolver để tìm địa chỉ của `app.hieupn.site`. Resolver có thể dùng cache; nếu cần, nó tra cứu hệ thống DNS.

Sau §11.2, Cloudflare là authoritative DNS provider của `hieupn.site`.

Với Full Setup của runbook, khi lưu Published application route ở §12.3.3, Cloudflare tạo record dạng:

| Type | Name | Target | Proxy status |
| --- | --- | --- | --- |
| CNAME | `app` | `<TUNNEL-UUID>.cfargotunnel.com` | Proxied |

DNS public trả một hoặc nhiều Cloudflare Edge IPv4/IPv6. Nó không trả ClusterIP, Pod IP, worker IP hay IP router lab.

**DNS chỉ trả địa chỉ. HTTP request lấy trang web không đi qua DNS server.**

Trong `nslookup`, `Server: 127.0.0.53` là DNS stub resolver cục bộ trên Ubuntu. Các địa chỉ dưới `Non-authoritative answer` mới là kết quả phân giải hostname.

### Bước 4 — Browser mở HTTPS tới Cloudflare Edge

Browser kết nối tới Edge trên port `443` và thực hiện TLS handshake.

Edge trình certificate cho `app.hieupn.site`. Browser kiểm tra hostname, hiệu lực và chuỗi tin cậy của certificate. Khi handshake thành công, browser gửi HTTP request trong kênh TLS.

Biểu diễn dễ đọc:

```http
GET /?page=2 HTTP/1.1
Host: app.hieupn.site
Accept: text/html

```

| Dữ liệu | Giá trị |
| --- | --- |
| Method | `GET` |
| Path | `/` |
| Query parameter | `page=2` |
| Host | `app.hieupn.site` |
| Body | Rỗng |

**Host bắt nguồn từ URL người dùng mở.** Browser tạo `Host` trong HTTP/1.1; HTTP/2 và HTTP/3 dùng thông tin tương ứng trong `:authority`.

Ingress không tạo Host header. Nó chỉ khai báo giá trị Host cần so khớp ở bước 10.

Cloudflare Edge kết thúc TLS public để đọc HTTP request và có thể áp dụng WAF, cache hoặc các rule được cấu hình. Browser chưa kết nối tới mạng lab.

### Bước 5 — Cloudflare chọn Published application route

Edge dùng hostname của request để tìm ánh xạ đã cấu hình:

| Hostname | Tunnel | Origin |
| --- | --- | --- |
| `app.hieupn.site` | `homelab-k8s` | `http://traefik.traefik.svc.cluster.local:80` |

Ở §12.3.3:

- Subdomain `app` + Domain `hieupn.site` xác định hostname.
- Việc thêm route **trong tunnel `homelab-k8s`** xác định tunnel nhận request.
- Service URL xác định origin mà connector phải gọi.

**Route có vai trò ở cả trước và sau tunnel:** Edge dùng ánh xạ hostname → tunnel; connector dùng cấu hình origin để gọi đích nội bộ.

Cloudflare không dùng tên Service Traefik để nhận diện cluster. Nó biết connector thuộc tunnel nào qua quá trình đăng ký bằng token ở §12.2.

Public DNS chỉ đưa browser tới Edge. Published application route mới nối request đó với tunnel và origin.

### Bước 6 — Cloudflare chọn connector Healthy và truyền request xuống tunnel

Cloudflare chọn một connector Healthy của tunnel, giả sử replica 1, rồi truyền request qua kết nối đã mở ở bước 1.

Một request bình thường được gửi xuống một connector, không nhân đôi tới cả hai replica. Connector còn lại cung cấp khả năng tiếp tục phục vụ khi một connector mất kết nối; request đang xử lý vẫn có thể bị ảnh hưởng khi connector chết.

**Dữ liệu đi qua tunnel là gì?**

Ở tầng mạng, đó là các byte được đóng gói và mã hóa theo giao thức tunnel. Ở tầng ứng dụng, dữ liệu biểu diễn:

| Thành phần request | Ví dụ hiện tại |
| --- | --- |
| Method | `GET` |
| URL/path và query | `/?page=2` |
| Hostname | `app.hieupn.site` |
| Headers | `Accept: text/html` và các header khác |
| Body | Rỗng |

Toàn bộ request không bắt buộc là JSON. JSON chỉ là một định dạng có thể dùng cho body.

**Dữ liệu xuống tunnel vẫn là request của người dùng**, dù nó đi ngược chiều mở kết nối:

| Góc nhìn | Ý nghĩa |
| --- | --- |
| Router/NAT | Traffic thuộc kết nối đã được connector mở outbound |
| HTTP | Request của browser được chuyển tới ứng dụng |

Không nên gọi request này là “HTTP response cho việc mở outbound”.

### Bước 7 — Tunnel kết thúc tại Pod `cloudflared`

Connector nhận dữ liệu từ tunnel, đọc thông tin request và dựng HTTP request trong chương trình để proxy tới origin.

Ví dụ trong triển khai QUIC, method, Host và headers được đọc từ metadata, còn body được đọc qua stream. Đây là cách biểu diễn nội bộ, không phải yêu cầu truyền toàn bộ request dưới dạng JSON.

Request logic mà connector cần chuyển tiếp là:

```http
GET /?page=2 HTTP/1.1
Host: app.hieupn.site
Accept: text/html

```

Connector đọc origin đã cấu hình:

```text
http://traefik.traefik.svc.cluster.local:80
```

| Phần origin URL | Chỉ dẫn |
| --- | --- |
| `http://` | Gọi origin bằng HTTP |
| `traefik.traefik.svc.cluster.local` | Tìm IP của Service nội bộ |
| `:80` | Kết nối tới port 80 |

Tunnel của runbook là remotely-managed: origin được cấu hình tại §12.3.3 và cung cấp cho connector. Manifest §12.2 chỉ cần token để tham gia tunnel, không cần ghi địa chỉ Traefik.

Edge không trực tiếp mở kết nối tới Service nội bộ. **`cloudflared` là bên gọi origin.**

### Bước 8 — `cloudflared` dùng CoreDNS tìm Service Traefik

Connector phân giải:

```text
traefik.traefik.svc.cluster.local
```

| Phần | Ý nghĩa |
| --- | --- |
| `traefik` đầu tiên | Tên Service |
| `traefik` thứ hai | Namespace |
| `svc.cluster.local` | Miền DNS Service trong cluster |

CoreDNS trả ClusterIP của đúng Service này. Nó không tự chọn một Service khác.

Các bước tạo ra hành vi đó:

| Bước runbook | Vai trò |
| --- | --- |
| §6 | `kubeadm init` triển khai CoreDNS |
| §6.1 | Cài Flannel để hoàn thiện mạng Pod |
| §8.4 | Kiểm tra DNS nội bộ |
| §9.3 | Tạo Service `traefik`, namespace `traefik`, kiểu ClusterIP |
| §12.2 | Pod connector dùng DNS policy mặc định `ClusterFirst` |
| §12.3.3 | Service URL quyết định hostname connector hỏi |

Không cần thêm record hay entry `hosts` thủ công cho Service Traefik. DNS Kubernetes cung cấp ánh xạ theo tài nguyên Service.

Giả sử CoreDNS trả ClusterIP minh họa `10.96.123.45`. Port `80` lấy từ Service URL, không nằm trong câu trả lời DNS.

DNS có thể được cache, nên không phải mỗi request đều phát sinh truy vấn mới.

### Bước 9 — Connector gọi Service Traefik bằng HTTP port 80

Sau khi có IP, connector mở kết nối TCP tới:

```text
10.96.123.45:80
```

Hoặc tái sử dụng kết nối origin đang có.

Kubernetes Service dataplane đưa kết nối tới một Traefik Pod Ready. Trong chart của runbook, Service port `80` ánh xạ tới container port `8000` của Traefik.

Connector gửi HTTP request trên kết nối đó:

```http
GET /?page=2 HTTP/1.1
Host: app.hieupn.site
Accept: text/html

```

Các header thực tế có thể được Cloudflare hoặc proxy bổ sung, loại bỏ hoặc điều chỉnh.

**Đích kết nối và Host là hai thông tin khác nhau:**

| Thông tin | Dùng để làm gì? |
| --- | --- |
| Tên `traefik.traefik.svc.cluster.local` | Tìm IP Service |
| ClusterIP + port `80` | Đưa kết nối tới Traefik |
| `Host: app.hieupn.site` | Cho Traefik biết ứng dụng cần chọn |

Có thể mô phỏng riêng chặng này bằng test nội bộ:

```bash
ING_IP=$(kubectl -n traefik get svc traefik \
  -o jsonpath='{.spec.clusterIP}')

curl -v \
  -H 'Host: app.hieupn.site' \
  "http://$ING_IP/?page=2"
```

Lệnh này bỏ qua DNS public, Edge và tunnel. Connector thực tế dùng HTTP client nội bộ, không chạy `curl`.

ClusterIP là địa chỉ ảo ổn định, không phải server riêng. Khi Pod Traefik đổi IP, endpoint của Service được cập nhật.

### Bước 10 — Traefik dùng Host và path để chọn ứng dụng

Traefik đã theo dõi Kubernetes API và nạp rule Ingress:

```yaml
rules:
  - host: app.hieupn.site
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: web
              port:
                number: 80
```

Request được đối chiếu:

| Dữ liệu request | Rule | Kết quả |
| --- | --- | --- |
| Host `app.hieupn.site` | `host: app.hieupn.site` | Khớp |
| Path `/` | Prefix `/` | Khớp |
| Query `page=2` | Rule này không chọn backend theo query | Tiếp tục chuyển tới backend |

**Service URL không phải path của request.** Traefik không so toàn bộ URL `http://traefik…:80` với `path: /`.

Ingress là cấu hình, không phải một hop mạng. Traefik là chương trình thực hiện việc so khớp và chuyển tiếp.

Trong lab không override Host, hostname public được giữ khi connector gọi origin. §12.3.3 có tùy chọn đặt tường minh **HTTP Host Header = `app.hieupn.site`**.

Nếu override Host thành `traefik.traefik.svc.cluster.local`, request không khớp rule trên. Kết nối vẫn dùng HTTP port 80; thay Host không biến HTTP thành HTTPS.

Nếu Ingress vẫn dùng hostname tạm `app.example.com`, rule cũng không khớp và Traefik thường trả `404` khi không có router khác phù hợp.

### Bước 11 — Backend Service `web` dẫn tới một Pod web

Ingress chỉ định backend logic là Service `web:80`.

Service `web` chọn nhóm Pod có label `app: web`. Kubernetes cập nhật EndpointSlice theo các endpoint và trạng thái readiness; Traefik dùng thông tin backend để chọn một Pod phù hợp.

Tùy cấu hình, Traefik có thể gọi qua Service ClusterIP hoặc trực tiếp tới Pod endpoint.

Traefik tạo request tới backend, mang method, path/query, headers và body cần chuyển tiếp. Các proxy có thể điều chỉnh header hoặc cách truyền HTTP. Trong cấu hình đang xét, không có rule đổi method hay rewrite path.

**Pod web mới là nơi xử lý request và tạo nội dung.**

Với nginx demo, body có thể hiển thị:

```text
Server address: <Pod-IP>:80
Server name: web-<replicaset-hash>-<pod-suffix>
```

Query `page=2` được gửi tới ứng dụng nhưng không tự tạo chức năng phân trang. Nhiều request có thể được xử lý bởi các Pod khác nhau.

**Nếu request có JSON body**

Giả sử thay app demo bằng API hỗ trợ `/api/orders`, request có thể là:

```http
POST /api/orders?dry_run=true HTTP/1.1
Host: app.hieupn.site
Content-Type: application/json

{"product_id":123,"quantity":2}
```

Tunnel và proxy chuyển tiếp:

| Thành phần | Giá trị |
| --- | --- |
| Method | `POST` |
| Path | `/api/orders` |
| Query | `dry_run=true` |
| Body | JSON chứa `product_id` và `quantity` |

JSON chỉ là body của request. Ứng dụng API mới quyết định xử lý và tạo response; nginx demo hiện tại không tự có API tạo đơn hàng.

### Bước 12 — Response quay ngược về browser

Pod tạo response gồm status, headers và body. Ví dụ minh họa:

```http
HTTP/1.1 200 OK
Content-Type: text/plain

Server address: ...
Server name: web-...
...
```

Response đi qua các chặng:

| Chặng | Dữ liệu và cách truyền |
| --- | --- |
| Pod web → Traefik | HTTP response của backend |
| Traefik → `cloudflared` | HTTP response trên kết nối origin |
| `cloudflared` → Edge | Status, headers và body được truyền qua stream tunnel tương ứng |
| Edge → Browser | HTTP response được gửi trong phiên HTTPS của browser |

Response có thể được chuyển tiếp theo từng phần, không cần chờ toàn bộ body hoàn tất.

**Không có một kết nối mạng duy nhất chạy xuyên từ browser đến Pod.** Các proxy sử dụng kết nối riêng ở từng chặng và liên kết request với response tương ứng.

| Chặng | Bảo mật trong lab |
| --- | --- |
| Browser ↔ Edge | HTTPS |
| Edge ↔ `cloudflared` | Tunnel mã hóa |
| `cloudflared` ↔ Traefik | HTTP nội bộ |
| Traefik ↔ Pod web | HTTP nội bộ |

Router/NAT không cần nhận kết nối inbound mới. Public DNS và CoreDNS không tham gia chuyển response.

Cloudflare có thể thêm `cf-ray` và điều chỉnh response headers. Nội dung trang trong ví dụ vẫn do Pod web tạo ra.

### Bước 13 — §13 kiểm tra từng lớp của chuỗi

| Kiểm tra | Chứng minh được | Chưa tự chứng minh được |
| --- | --- | --- |
| §13.1: `nslookup app.hieupn.site` | Public DNS trả Edge IP | Tunnel và origin hoạt động |
| §12.3.1: gọi ClusterIP Traefik với Host thật | Chuỗi nội bộ Traefik → backend trả nội dung | Đường public qua Cloudflare |
| §13.2: `curl -I https://app.hieupn.site` | Đường public trả headers | Body đúng hoặc origin vừa được gọi nếu có cache |
| §13.3: `curl -sS` hoặc browser | Đường public trả nội dung mong đợi | Origin hiện tại khỏe nếu nội dung được trả từ cache |

`curl -I` gửi method **HEAD**; response cho HEAD không có body. `curl -sS` không chỉ định method khác sẽ gửi GET.

`server: cloudflare` và `cf-ray` cho biết response đi qua Edge, nhưng tự chúng không chứng minh request vừa đi xuống tunnel. Khi cần kiểm tra origin hiện tại, kết hợp test nội bộ, log hoặc kiểm tra/bypass cache phù hợp.

`Server address` và `Server name` trong body demo là thông tin Pod đã tạo nội dung, không phải Edge hay Traefik.

### Bước 14 — Phân biệt bốn lớp chọn đường

| Lớp | Câu hỏi được trả lời | Ánh xạ trong lab |
| --- | --- | --- |
| Public DNS | Browser phải kết nối tới đâu? | `app.hieupn.site` → Edge IP |
| Published application route | Hostname thuộc tunnel và origin nào? | `app.hieupn.site` → `homelab-k8s` → Service Traefik |
| Traefik Ingress rule | Request thuộc ứng dụng nào? | Host/path → Service `web:80` |
| Service/backend endpoints | Pod nào nhận request? | Backend `web` → một Pod phù hợp |

Phân loại vai trò:

| Nhóm | Thành phần | Nhiệm vụ |
| --- | --- | --- |
| Tìm địa chỉ | Public DNS, CoreDNS | Trả IP cần kết nối |
| Nhận/chuyển tiếp HTTP | Cloudflare Edge, `cloudflared`, Traefik | Proxy request và response |
| Cấu hình chỉ đường | Published application route, Ingress | Khai báo tunnel/origin và Host/path → backend |
| Địa chỉ và chuyển tiếp mạng | Service, ClusterIP, EndpointSlice | Cung cấp địa chỉ ổn định và thông tin backend |
| Xử lý ứng dụng | Pod web | Tạo response |

Trong nhóm mạng, Service khai báo cách truy cập backend, ClusterIP là địa chỉ ảo, EndpointSlice lưu thông tin endpoint. Dataplane thực hiện chuyển tiếp khi sử dụng Service IP; các object đó không phải ba chương trình proxy HTTP.

Người đọc cần theo được ba thông tin xuyên suốt:

- **Kết nối đi tới đâu:** Edge IP, rồi ClusterIP Traefik, rồi Pod backend.
- **Request mang gì:** method, path/query, hostname, headers và body.
- **Ai quyết định bước tiếp theo:** DNS, cấu hình tunnel, rule Ingress và lựa chọn backend.
