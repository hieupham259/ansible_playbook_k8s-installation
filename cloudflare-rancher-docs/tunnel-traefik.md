# Luồng Cloudflare Tunnel → Kubernetes qua Traefik

> Tài liệu này tách ra từ [§12.3.2 của runbook-k8s-vmware.md](../runbook-k8s-vmware.md#1232-hiểu-đầy-đủ-luồng-cloudflare-tunnel--kubernetes)
> để giữ runbook ở độ dài đọc được. Đây là phần **giải thích cơ chế**, không chứa bước thao tác:
> mọi lệnh, gate `PASS:` và thứ tự triển khai vẫn nằm trong runbook.
>
> Tham chiếu dạng `§12`, `§13`, `§12.3.1` trong bài trỏ về mục cùng số của
> [§12.3.2 của runbook-k8s-vmware.md](../runbook-k8s-vmware.md#1232-hiểu-đầy-đủ-luồng-cloudflare-tunnel--kubernetes). Tên gọi lấy đúng theo lab của runbook:
> hostname `app.hieupn.site`, tunnel `homelab-k8s`, Traefik ở namespace `traefik`, `cloudflared`
> ở namespace `cloudflare`.

Bố cục: hai sơ đồ tổng quan → bản kể lại bằng ngôn ngữ đơn giản → luồng end-to-end theo 14 bước
→ 12 mục đào sâu từng hop.

---

Điểm cốt lõi: **Cloudflare không chủ động mở một kết nối mới từ Internet vào IP của lab**. Chính các Pod `cloudflared` đang nằm trong Kubernetes đã mở kết nối outbound tới Cloudflare và giữ các kết nối đó hoạt động. Khi có request, Cloudflare truyền request xuống chính đường kết nối đã được mở sẵn.

```mermaid
flowchart TB
    subgraph PUBLIC["Internet và Cloudflare"]
        USER["Browser<br/>https://app.hieupn.site"]
        PDNS["Cloudflare public DNS<br/>app.hieupn.site → Edge Anycast IP"]
        EDGE["Cloudflare Edge<br/>TLS · WAF · Tunnel routing"]
        ROUTE["Published application route<br/>app.hieupn.site →<br/>http://traefik.traefik.svc.cluster.local:80"]
    end

    subgraph LAB["Mạng lab sau router/NAT — không mở inbound"]
        subgraph K8S["Kubernetes cluster"]
            CF1["cloudflared connector replica 1<br/>Active / Healthy"]
            CF2["cloudflared connector replica 2<br/>Active / Healthy"]
            DNS["CoreDNS"]
            TSVC["Service traefik<br/>ClusterIP :80"]
            TPOD["Traefik Pod"]
            ING["Ingress web — object cấu hình<br/>Host: app.hieupn.site"]
            WSVC["Service web :80"]
            WPOD1["Pod web replica 1"]
            WPOD2["Pod web replica 2"]
        end
    end

    USER -->|"1. DNS query"| PDNS
    USER -->|"2. HTTPS :443 tới Edge IP"| EDGE
    ROUTE -.->|"Control plane: hostname → tunnel + origin"| EDGE

    CF1 ==>|"A. Outbound mã hóa<br/>QUIC/UDP hoặc HTTP2/TCP :7844"| EDGE
    CF2 ==>|"A. Outbound mã hóa<br/>QUIC/UDP hoặc HTTP2/TCP :7844"| EDGE
    EDGE -.->|"3. Request xuống một connector Healthy<br/>qua connection đã mở"| CF1
    EDGE -.->|"hoặc connector này"| CF2

    CF1 -->|"4a. DNS query origin"| DNS
    CF2 -->|"4a. DNS query origin"| DNS
    DNS -->|"4b. Trả ClusterIP"| CF1
    DNS -->|"4b. Trả ClusterIP"| CF2
    CF1 -->|"5. HTTP :80 nếu replica 1 được chọn"| TSVC
    CF2 -->|"5. HTTP :80 nếu replica 2 được chọn"| TSVC

    TSVC -->|"6. Chọn Traefik endpoint"| TPOD
    ING -.->|"Control plane: Traefik watch và nạp routing rule"| TPOD
    TPOD -->|"7. Host app.hieupn.site khớp rule"| WSVC
    WSVC -->|"8. Chọn backend"| WPOD1
    WSVC -->|"hoặc"| WPOD2

    WPOD1 -.->|"9. Response quay ngược cùng đường"| USER
    WPOD2 -.->|"9. Response quay ngược cùng đường"| USER

    classDef ext fill:#e8eefc,stroke:#2b4c9b,stroke-width:1px,color:#10203f
    classDef k8s fill:#ffffff,stroke:#526070,stroke-width:1px,color:#10203f
    classDef cfg fill:#fdf3e0,stroke:#a8791f,stroke-width:1px,color:#3d2c07
    class USER,PDNS,EDGE ext
    class CF1,CF2,DNS,TSVC,TPOD,WSVC,WPOD1,WPOD2 k8s
    class ROUTE,ING cfg
    style PUBLIC fill:#f1f5ff,stroke:#8094c4,stroke-width:1px,color:#20345f
    style LAB fill:#f4f6f9,stroke:#8a97a6,stroke-width:1px,color:#2b3543
    style K8S fill:#ffffff,stroke:#aab4c0,stroke-width:1px,color:#2b3543
```

Nét đậm `==>` là kết nối outbound do cả hai replica `cloudflared` chủ động mở và giữ; chúng đều Active/Healthy, không có replica primary/standby cố định. Nét đứt từ Edge cho biết mỗi request được đẩy xuống **một** connector Healthy qua connection đã mở sẵn. `Published application route` và `Ingress web` là control-plane/configuration, không phải hop mạng: route ánh xạ hostname public sang tunnel/origin, còn Traefik watch Ingress để nạp rule Host → Service. Response từ Pod web quay ngược qua chính chuỗi Traefik → `cloudflared` → Tunnel → Edge → Browser.

**Trình tự đầy đủ của một request và response:**

```mermaid
%%{init: {"themeVariables": {"noteBkgColor": "#f1f5ff", "noteBorderColor": "#8094c4", "noteTextColor": "#10203f"}}}%%
sequenceDiagram
    participant B as Browser
    participant D as Cloudflare public DNS
    participant E as Cloudflare Edge
    participant C as cloudflared Pod
    participant K as CoreDNS
    participant TS as Service traefik
    participant T as Traefik Pod
    participant WS as Service web
    participant W as Pod web

    Note over C,E: Tunnel mã hóa đã được cloudflared chủ động mở outbound từ trước
    Note over E,C: Published route: app.hieupn.site → homelab-k8s → Traefik:80

    B->>D: app.hieupn.site nằm ở đâu?
    D-->>B: Cloudflare Edge IP
    B->>E: HTTPS :443, hostname app.hieupn.site
    E-->>C: Đẩy request xuống tunnel đang mở

    C->>K: Resolve traefik.traefik.svc.cluster.local
    K-->>C: ClusterIP của Service traefik
    C->>TS: HTTP :80, Host app.hieupn.site
    TS->>T: Chọn Traefik Pod đang Ready

    Note over T: Traefik đã đọc Ingress từ Kubernetes API<br/>Ingress là cấu hình, không phải hop mạng
    T->>WS: Host và path khớp → backend Service web:80
    WS->>W: Chọn một Pod web

    W-->>T: HTTP response
    T-->>C: Response qua kết nối nội bộ
    C-->>E: Response qua tunnel mã hóa
    E-->>B: HTTPS response
```

Sơ đồ tuần tự đọc từ trên xuống dưới. Mũi tên đi sang phải là chiều request vào ứng dụng; các mũi tên nét đứt quay sang trái là chiều response trở về browser. `Cloudflare public DNS` chỉ dẫn browser tới Edge, còn `CoreDNS` chỉ được Pod trong cluster dùng để tìm ClusterIP của Service Traefik.

## Đối chiếu cấu hình: hostname chọn tunnel, Service URL chọn origin

Ở §12.3.3 của runbook, vào **Zero Trust → Networks → Connectors**
(UI mới có thể hiện **Tunnels & Mesh**) → tunnel `homelab-k8s` →
**Published application routes** → **Add route** → **Published application**,
rồi khai báo:

| Trường | Giá trị | Vai trò |
| --- | --- | --- |
| **Subdomain** | `app` (để trống nếu dùng root domain) | Ghép với Domain để xác định hostname public |
| **Domain** | `hieupn.site` | Cùng Subdomain tạo hostname `app.hieupn.site` |
| **Service URL** | `http://traefik.traefik.svc.cluster.local:80` | Đích mà connector `cloudflared` gọi sau khi nhận request từ tunnel |

**Cách hiểu:** tạo route cho `app.hieupn.site` **bên trong tunnel
`homelab-k8s`** là thiết lập lối vào đúng tunnel; Service URL là thiết lập
“vào tunnel rồi thì connector gọi đâu trong cụm”.

Subdomain và Domain xác định **hostname**, còn tunnel đang được chọn khi tạo route
xác định **tunnel nhận request**. Hai trường hostname tự chúng không xác định tunnel.
Service URL cũng không giúp Cloudflare nhận diện cluster: nhiều cluster có thể có
cùng tên Service `traefik.traefik.svc.cluster.local`.

Các bước tiên quyết và bước cấu hình nối với nhau như sau:

| Bước runbook | Thiết lập gì? | Vai trò khi request chạy |
| --- | --- | --- |
| §9.3 | Helm tạo Service `traefik` trong namespace `traefik`, loại `ClusterIP` | Có địa chỉ nội bộ ổn định dẫn tới Pod Traefik |
| §11 | Chuẩn bị domain và chuyển DNS authoritative sang Cloudflare; zone phải `Active` | Có nền DNS public để publish hostname |
| §12.1 | Tạo tunnel `homelab-k8s`, có ID riêng, và lấy token | Xác định danh tính tunnel |
| §12.2 | Deploy `cloudflared` bằng token của tunnel | Cloudflare nhận biết các connection đang mở thuộc tunnel nào |
| §12.3.1 | Đổi Ingress sang Host `app.hieupn.site`, test nội bộ PASS | Traefik có rule khớp hostname public |
| §12.3.3 | Save Published application route trong đúng tunnel và kiểm tra DNS record `Proxied` | Gắn hostname với tunnel, đồng thời khai báo origin connector phải gọi |
| §13 | Kiểm tra DNS, HTTPS headers và body/trình duyệt | Xác nhận đường public hoạt động end-to-end |

Với **Full Setup** của runbook, Cloudflare tự tạo DNS record khi thêm Published
application route. Chỉ coi cấu hình hoàn tất khi route đã lưu đúng, record tồn tại
đúng một lần và connector/tunnel đang **Healthy**.

Token gắn connector với **tunnel**, không khóa connector vào một Kubernetes cluster
cụ thể. Nếu dùng cùng token để chạy `cloudflared` ở cluster khác, connector đó cũng
có thể tham gia cùng tunnel. Khi cần tách đường vào giữa các cluster, dùng tunnel/token
riêng và giữ token bí mật.

### Từ Service URL tới kết nối nội bộ

§12.3.3 là **bước lưu cấu hình**; khi request đến, Edge và connector sử dụng cấu hình
đã lưu. Route có vai trò ở cả hai phía:

1. **Tại Cloudflare:** hostname `app.hieupn.site` được ánh xạ tới tunnel
   `homelab-k8s`; request được truyền xuống một connector Healthy của tunnel đó.
2. **Tại connector:** `cloudflared` dùng Service URL để gọi origin nội bộ.
   Đây là remotely-managed tunnel, nên cấu hình origin nằm trên dashboard;
   manifest `cloudflared.yaml` ở §12.2 chỉ cần token để tham gia tunnel,
   không cần ghi thêm địa chỉ Traefik.
3. **Tại Traefik:** Ingress tiếp tục quyết định hostname/path này thuộc ứng dụng nào.

Service URL chỉ dẫn rõ ba việc:

| Phần URL | Chỉ dẫn cho `cloudflared` |
| --- | --- |
| `http://` | Dùng HTTP để gọi origin |
| `traefik.traefik.svc.cluster.local` | Phân giải tên Service `traefik` trong namespace `traefik` |
| `:80` | Kết nối tới port 80 của địa chỉ đã phân giải |

CoreDNS không chọn ngẫu nhiên một Service. Nó trả ClusterIP của Service có
**đúng tên và namespace được hỏi**. Service đã được tạo ở §9.3; DNS Kubernetes
cung cấp ánh xạ tên Service tương ứng, không cần nhập record hoặc entry `hosts`
riêng cho Traefik. CoreDNS được triển khai khi `kubeadm init` ở §6 và kiểm tra tại
§8.4; Pod `cloudflared` ở §12.2 dùng DNS policy mặc định `ClusterFirst`.

Ví dụ CoreDNS trả `10.96.123.45` (IP minh họa), connector mở connection tới
`10.96.123.45:80`. **Tên DNS được dùng để tìm IP; IP được dùng để mở kết nối.**
DNS có thể được cache và connection có thể được tái sử dụng, nên không phải mỗi
request đều tạo một DNS query và connection mới.

Trong cấu hình lab không override Host, HTTP request gửi qua connection đó vẫn mang
`Host: app.hieupn.site`. Phân biệt:

| Thông tin | Mục đích |
| --- | --- |
| `traefik.traefik.svc.cluster.local` | Tìm IP Service origin |
| ClusterIP Traefik + port 80 | Đích kết nối mạng |
| `Host: app.hieupn.site` và path | Traefik dùng để khớp rule Ingress |

### Phân loại vai trò trong toàn bộ luồng

| Nhóm | Thành phần | Nhiệm vụ |
| --- | --- | --- |
| Tìm địa chỉ | Public DNS, CoreDNS | Trả IP cần kết nối |
| Nhận/chuyển tiếp HTTP | Cloudflare Edge, `cloudflared`, Traefik | Proxy request và response |
| Cấu hình chỉ đường | Published application route, Ingress | Khai báo tunnel/origin và Host/path → backend |
| Địa chỉ và chuyển tiếp mạng | Service, ClusterIP, EndpointSlice | Cung cấp địa chỉ ổn định và thông tin backend |
| Xử lý ứng dụng | Pod web | Tạo response |

Trong nhóm mạng, **Service** khai báo cách truy cập nhóm backend, **ClusterIP** là
địa chỉ ảo, **EndpointSlice** lưu thông tin các endpoint; dataplane Kubernetes thực
hiện chuyển tiếp khi đi qua Service IP. Chúng không phải ba chương trình proxy HTTP.
Traefik cũng có thể gọi trực tiếp Pod endpoint tùy cấu hình.

Public DNS và CoreDNS chỉ tham gia tìm địa chỉ, không chuyển tiếp HTTP.
Published application route và Ingress là cấu hình, không phải hop mạng.
Response từ Pod web quay về qua Traefik → `cloudflared` → tunnel → Edge → browser;
không quay qua hai hệ DNS.

## Cách đọc toàn bộ luồng §12 → §13 theo ngôn ngữ đơn giản

Có thể hình dung Cloudflare Edge là **cổng tiếp nhận công khai**, Tunnel là **đường hầm đã mở sẵn**, Pod `cloudflared` là **đầu đường hầm nằm trong cluster**, Traefik là **lễ tân đọc hostname để chọn ứng dụng**, Service là **địa chỉ ổn định của một nhóm backend**, còn Pod web là **server thật sự xử lý request và tạo response**.

Trước khi người dùng truy cập, §12 đã chuẩn bị năm liên kết:

1. Token gắn hai Pod `cloudflared` với tunnel `homelab-k8s`.
2. Hai Pod chủ động mở và giữ các kết nối outbound mã hóa tới Cloudflare; router/NAT của lab không mở inbound.
3. Public DNS làm cho `app.hieupn.site` được client phân giải tới Cloudflare Edge.
4. Published application route ánh xạ `app.hieupn.site` → tunnel `homelab-k8s` → origin `http://traefik.traefik.svc.cluster.local:80`.
5. Ingress làm cho Traefik nạp rule `Host: app.hieupn.site` → backend `Service web:80`.

Khi người dùng nhập `https://app.hieupn.site/`, URL cung cấp ba dữ kiện: `https` yêu cầu kết nối TLS công khai, `app.hieupn.site` là hostname dùng để chọn route, còn `/` là path gửi tới ứng dụng. Một request hoàn chỉnh đi qua các hop sau:

1. **Browser hỏi public DNS.** DNS trả một hoặc nhiều Cloudflare Anycast IP. Browser không nhận ClusterIP, Pod IP, worker IP hay IP router của lab.
2. **Browser mở HTTPS tới Cloudflare Edge trên port 443.** Cloudflare cung cấp certificate cho `app.hieupn.site`, kết thúc TLS public và nhận request mang hostname logic `Host: app.hieupn.site` (hoặc `:authority` tương đương khi dùng HTTP/2/HTTP/3).
3. **Cloudflare tra Published application route.** Hostname cho biết phải dùng tunnel `homelab-k8s` và origin nội bộ nào. Cloudflare chọn một connector `cloudflared` đang Healthy; một request bình thường không được gửi đồng thời tới cả hai replica.
4. **Cloudflare đẩy request xuống connection tunnel đã tồn tại.** Đây không phải một connection inbound mới tới IP lab. Dữ liệu đi ngược xuống chính session outbound mà Pod `cloudflared` đã chủ động mở qua router/NAT từ trước.
5. **Tunnel kết thúc tại Pod `cloudflared`.** Pod này nhận request rồi đóng vai trò reverse proxy/client để mở một connection nội bộ mới tới origin `traefik.traefik.svc.cluster.local:80`.
6. **`cloudflared` hỏi CoreDNS.** CoreDNS trả ClusterIP của Service `traefik` trong namespace `traefik`; public DNS không tham gia bước này và Cloudflare Edge cũng không thể tự phân giải tên `*.svc.cluster.local`.
7. **`cloudflared` gọi Service Traefik bằng HTTP port 80.** Kubernetes Service dataplane chuyển connection từ ClusterIP tới một Traefik Pod đang Ready. ClusterIP là địa chỉ ảo ổn định, không phải một server riêng.
8. **Traefik chọn đúng ứng dụng.** Traefik đã watch Kubernetes API và nạp rule từ Ingress trước đó. Request không đi xuyên qua object Ingress; Traefik chỉ so sánh `Host: app.hieupn.site` và path `/` với rule đã nạp. Khi khớp, backend logic là `Service web:80`.
9. **Service/backend chọn một Pod web.** Traefik/Kubernetes dùng các endpoint của Service `web` để chọn một Pod phù hợp. Pod web mới là server thực sự tạo nội dung `nginxdemos/hello`.

Response quay về trên các connection tương ứng theo chiều ngược lại:

```text
Pod web
→ Traefik
→ cloudflared
→ tunnel mã hóa
→ Cloudflare Edge
→ kết nối HTTPS
→ Browser
```

Router vẫn không nhận kết nối inbound mới: nó chỉ thấy traffic hai chiều thuộc session outbound đã được `cloudflared` thiết lập. Các chặng bảo mật cũng phải được phân biệt:

```text
Browser ── HTTPS :443 ──> Cloudflare Edge
Cloudflare Edge ══ tunnel mã hóa ══> cloudflared Pod
cloudflared Pod ── HTTP :80 nội bộ ──> Traefik → Pod web
```

Bốn lớp “route” trong chuỗi trả lời bốn câu hỏi khác nhau:

| Lớp | Câu hỏi được trả lời | Kết quả trong lab này |
| --- | --- | --- |
| Public DNS | Browser phải kết nối tới đâu? | Cloudflare Edge IP |
| Published application route | Hostname này thuộc tunnel và origin nào? | `app.hieupn.site` → `homelab-k8s` → Traefik:80 |
| Traefik Ingress rule | Request này thuộc ứng dụng nào? | Host/path khớp → `Service web:80` |
| Service/endpoints | Backend cụ thể nào nhận request? | Một Pod có label/endpoint của `web` |

Vì thế các kết quả nhìn thấy ở §13 thuộc các tầng khác nhau:

- `nslookup app.hieupn.site` trả Cloudflare Edge IP: mới chứng minh public DNS dẫn client tới đúng cổng Cloudflare.
- `curl -I https://app.hieupn.site` trả `HTTP 200`, `server: cloudflare` và `cf-ray`: chứng minh request đã đi qua Edge và chuỗi end-to-end trả được response headers; `server: cloudflare` không có nghĩa Cloudflare tạo nội dung ứng dụng.
- Body/trình duyệt hiện `Server address` và `Server name`: đây là Pod IP và tên Pod web đã xử lý request, không phải địa chỉ Cloudflare hay Traefik.

Tóm lại, request vào đúng Pod nhờ bốn lần lựa chọn liên tiếp: **hostname public → đúng Cloudflare Edge → đúng tunnel/origin → đúng Ingress/Service → đúng Pod backend**. Mười hai mục cuối tài liệu này phân tích sâu từng hop; §13 của runbook kiểm tra lần lượt DNS public, HTTPS headers và body thật của chính chuỗi này.

## Luồng end-to-end theo 14 bước

### Bước 1 — §12 chuẩn bị đường đi trước khi có request

Cloudflare tạo tunnel `homelab-k8s` và sinh token. Deployment trong namespace `cloudflare` dùng token đó để chạy hai Pod `cloudflared`. Cả hai connector đều Active/Healthy và chủ động mở các connection outbound dài hạn tới Cloudflare bằng QUIC/UDP `7844` hoặc HTTP/2/TCP `7844`.

```text
cloudflared Pod → router/NAT → Internet → Cloudflare
```

Cloudflare không mở connection mới vào master, worker hay router. Router chỉ cho phép session outbound do `cloudflared` khởi tạo; sau đó dữ liệu request và response đi hai chiều bên trong session đã tồn tại. Vì vậy lab không cần public IP cho Kubernetes, port-forward hoặc mở inbound `80`, `443`, `7844`.

§12 cũng chuẩn bị hai bảng route:

```text
Published application:
app.hieupn.site → homelab-k8s → http://traefik.traefik.svc.cluster.local:80

Kubernetes Ingress:
Host app.hieupn.site + Path / → Service web:80
```

### Bước 2 — Browser phân tích URL

Khi người dùng nhập:

```text
https://app.hieupn.site/
```

browser tách URL thành:

```text
https://  app.hieupn.site  /
│         │                 │
│         │                 └─ path gửi tới ứng dụng
│         └─ hostname dùng cho DNS, TLS và routing
└─ giao thức HTTPS
```

Browser cần tìm IP cho hostname, thiết lập TLS và gửi request cho path `/`.

### Bước 3 — Browser hỏi Cloudflare public DNS

Browser/hệ điều hành hỏi DNS: “`app.hieupn.site` nằm ở đâu?”. Vì record Tunnel ở trạng thái Proxied, DNS trả một hoặc nhiều Cloudflare Anycast IPv4/IPv6, không trả IP router, master, worker, ClusterIP hay Pod IP của lab.

```text
app.hieupn.site → Cloudflare Edge IP
```

Trong output `nslookup`, dòng `Server: 127.0.0.53` chỉ là DNS stub resolver cục bộ trên Ubuntu. Các địa chỉ trong `Non-authoritative answer` mới là kết quả phân giải hostname.

### Bước 4 — Browser mở HTTPS tới Cloudflare Edge

Browser kết nối tới Cloudflare Edge IP trên port `443`. Cloudflare đưa certificate hợp lệ cho `app.hieupn.site`, kết thúc TLS public rồi nhận request có hostname logic:

```http
GET / HTTP/1.1
Host: app.hieupn.site
```

Với HTTP/2 hoặc HTTP/3, tên trường trên wire có thể là `:authority`, nhưng ý nghĩa vẫn là hostname `app.hieupn.site`. Ở chặng này Cloudflare có thể áp dụng WAF, DDoS protection, cache và các edge rule. Browser chưa hề kết nối tới mạng lab.

### Bước 5 — Cloudflare chọn Published application route

Cloudflare đọc hostname và tìm thấy cấu hình:

```text
app.hieupn.site
→ tunnel homelab-k8s
→ origin http://traefik.traefik.svc.cluster.local:80
```

DNS chỉ đưa browser tới Cloudflare Edge. Published route mới cho Cloudflare biết request này thuộc tunnel nào và connector phải gọi origin nội bộ nào.

### Bước 6 — Cloudflare chọn một connector Healthy và đẩy request xuống tunnel

Cloudflare thấy hai connector của `homelab-k8s` đang Healthy rồi chọn một connector phù hợp cho request. Một request bình thường đi xuống một connector, không được nhân đôi tới cả hai replica.

```text
Không phải: Cloudflare → mở connection inbound mới tới IP lab

Mà là:     cloudflared → đã mở connection outbound tới Cloudflare
            Cloudflare → đẩy request xuống connection đang mở
```

Nếu một Pod `cloudflared` chết, connector còn lại tiếp tục nhận request. Hai replica tạo HA cho tunnel, không phải bộ cân bằng tải giữa các Pod web.

### Bước 7 — Tunnel kết thúc tại Pod `cloudflared`

Điểm cuối của tunnel trong lab là Pod `cloudflared`, không phải Traefik:

```text
Cloudflare Edge ══ tunnel mã hóa ══> cloudflared Pod
```

Sau khi nhận request, `cloudflared` đóng vai trò reverse proxy/client. Nó đọc Service URL của Published route rồi mở một connection HTTP nội bộ mới tới `traefik.traefik.svc.cluster.local:80`. Cloudflare Edge không trực tiếp truy cập địa chỉ `*.svc.cluster.local`.

### Bước 8 — `cloudflared` dùng CoreDNS tìm Service Traefik

Pod `cloudflared` phân giải origin qua CoreDNS của Kubernetes:

```text
traefik.traefik.svc.cluster.local:80
│       │       │   │             │
│       │       │   │             └─ Service port
│       │       │   └─ cluster domain nội bộ
│       │       └─ đối tượng Service
│       └─ namespace traefik
└─ tên Service traefik
```

CoreDNS trả ClusterIP của Service Traefik. Đây là hệ DNS nội bộ thứ hai, độc lập với Cloudflare public DNS.

### Bước 9 — Service Traefik chuyển connection tới Traefik Pod

`cloudflared` gọi:

```text
http://<Traefik-ClusterIP>:80
```

Kubernetes Service dataplane chọn một Traefik Pod đang Ready và chuyển connection tới Pod đó:

```text
cloudflared → Service traefik ClusterIP:80 → Traefik Pod
```

ClusterIP là địa chỉ ảo ổn định đại diện cho các endpoint, không phải một máy chủ riêng. Pod Traefik có thể bị tạo lại và đổi Pod IP mà Service DNS vẫn giữ nguyên.

### Bước 10 — Traefik dùng Host header để chọn đúng ứng dụng

Traefik đã watch Kubernetes API và nạp rule từ Ingress trước khi request đến. Object Ingress là cấu hình control plane, không phải một hop mà packet phải đi xuyên qua.

Hostname bắt nguồn từ URL người dùng nhập và được giữ qua từng chặng:

```text
Browser
  Host: app.hieupn.site
        ↓
Cloudflare Edge
  hostname: app.hieupn.site
        ↓ tunnel
cloudflared
  Host: app.hieupn.site
        ↓ HTTP origin request
Traefik
```

`cloudflared` kết nối mạng tới `http://<Traefik-ClusterIP>:80`, nhưng HTTP request bên trong connection vẫn mang `Host: app.hieupn.site`. ClusterIP/port giúp Kubernetes đưa connection tới Traefik Pod; Host header giúp Traefik chọn đúng Router. Vì Published application không đặt `HTTP Host Header` override nên hostname public không bị đổi thành `traefik.traefik.svc.cluster.local`.

Traefik so sánh hostname/path với rule:

```text
Host app.hieupn.site + Path /
→ backend Service web:80
```

Nếu Ingress còn dùng `app.example.com`, rule không khớp và Traefik thường trả `404`.

### Bước 11 — Service `web` chọn Pod backend

Ingress chỉ định backend logic là `Service web:80`. Traefik/Kubernetes dùng các endpoint của Service để chọn một Pod có label phù hợp, ví dụ `app=web`. Tùy cách Traefik được cấu hình, nó có thể dùng Service dataplane hoặc kết nối trực tiếp tới Pod endpoint đã lấy từ Kubernetes.

```text
Traefik → Service/endpoints web → một Pod web
```

Pod web mới là server thực sự xử lý request và tạo nội dung. Vì thế trang `nginxdemos/hello` có thể hiển thị:

```text
Server address: <Pod-IP>:80
Server name: web-<replicaset-hash>-<pod-suffix>
```

Chạy nhiều request có thể thấy Pod name/IP thay đổi do backend được chọn khác nhau.

### Bước 12 — Response quay ngược về browser

Pod web tạo HTTP response rồi trả về trên các connection tương ứng theo chiều ngược lại:

```text
Pod web
→ Traefik
→ cloudflared
→ tunnel mã hóa
→ Cloudflare Edge
→ kết nối HTTPS
→ Browser
```

Router/NAT không nhận một connection inbound mới. Nó chỉ chuyển traffic response thuộc session outbound mà `cloudflared` đã thiết lập. Ba chặng bảo mật là:

```text
Browser ── HTTPS :443 ──> Cloudflare Edge
Cloudflare Edge ══ tunnel mã hóa ══> cloudflared Pod
cloudflared Pod ── HTTP :80 nội bộ ──> Traefik → Pod web
```

Cloudflare có thể thêm các response header như `server: cloudflare` và `cf-ray`, sau đó gửi response về browser qua HTTPS. Nội dung trang vẫn do Pod web tạo ra.

### Bước 13 — §13 kiểm tra từng lớp của chuỗi

Ba phép kiểm tra có phạm vi khác nhau:

| Kiểm tra | Chứng minh được | Chưa tự chứng minh được |
| --- | --- | --- |
| `nslookup app.hieupn.site` | Public DNS dẫn client tới Cloudflare Edge | Tunnel, Traefik, Ingress và Pod còn hoạt động |
| `curl -I https://app.hieupn.site` | DNS, HTTPS/TLS, Published route, Tunnel và origin trả được headers | Nội dung body hiển thị đúng |
| `curl -sS https://app.hieupn.site` hoặc browser | Body thật đã được một Pod web xử lý và trả về | — |

`HTTP 200`, `server: cloudflare` và `cf-ray` cho biết response đã đi qua Cloudflare Edge. `Server address`/`Server name` trong body là IP và tên Pod web đã phục vụ request, không phải Cloudflare Edge hay Traefik.

### Bước 14 — Phân biệt bốn loại “route”

| Loại route | Câu hỏi nó trả lời | Ánh xạ trong lab |
| --- | --- | --- |
| **Public DNS route** | Browser phải kết nối tới đâu? | `app.hieupn.site` → Cloudflare Edge IP |
| **Published application route** | Hostname này thuộc tunnel và origin nào? | `app.hieupn.site` → `homelab-k8s` → `traefik...:80` |
| **Traefik Ingress route** | Request này thuộc ứng dụng/Service nào? | Host `app.hieupn.site`, path `/` → `Service web:80` |
| **Service/endpoint route** | Pod cụ thể nào nhận request? | `Service web` → một Pod endpoint Ready |

Bốn loại route thực hiện bốn lần lựa chọn nối tiếp, không thay thế nhau:

```text
Hostname public
→ đúng Cloudflare Edge
→ đúng tunnel và origin
→ đúng Ingress/Service
→ đúng Pod backend
```

## 1. `cloudflared` đã nằm bên trong cluster

Deployment ở §12.2 tạo hai Pod `cloudflared` trong namespace `cloudflare`. Vì là Pod Kubernetes, mỗi replica:

- dùng được mạng nội bộ của cluster;
- dùng CoreDNS để phân giải tên `*.svc.cluster.local`;
- truy cập được các Kubernetes Service;
- có egress ra Internet;
- dùng `TUNNEL_TOKEN` để đăng ký với cùng tunnel `homelab-k8s`.

Hai replica không cần hai tunnel riêng. Chúng cùng gắn vào một tunnel UUID để dự phòng: nếu một connector/worker lỗi, replica còn lại tiếp tục phục vụ. Replica `cloudflared` là cơ chế HA cho đường hầm, không phải bộ cân bằng tải backend ứng dụng; Kubernetes Service và Traefik đảm nhiệm việc phân phối request ở phía cluster. Xem [Cloudflare — Kubernetes deployment guide](https://developers.cloudflare.com/tunnel/deployment-guides/kubernetes/).

## 2. Pod chủ động mở đường outbound qua router/NAT

Khi container khởi động, `cloudflared` đọc token và mở các kết nối outbound dài hạn tới Cloudflare. Với `--protocol auto` mặc định, nó có thể dùng QUIC qua UDP `7844` hoặc HTTP/2 qua TCP `7844`. Cloudflare duy trì nhiều kết nối tới ít nhất hai data center để có dự phòng. Xem [Cloudflare Tunnel configuration](https://developers.cloudflare.com/tunnel/configuration/) và [Tunnel with firewall](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/configure-tunnels/tunnel-with-firewall/).

Router chỉ thấy chiều khởi tạo:

```text
cloudflared Pod → router/NAT → Cloudflare
```

NAT/firewall tạo state cho session outbound. Sau khi session tồn tại, dữ liệu đi được hai chiều trong chính session đó. Cloudflare **không** quay lại mở kết nối trực tiếp tới IP riêng của master/worker, nên kiến trúc này không cần:

- IP public cho Kubernetes;
- port-forward trên router;
- mở inbound `80`, `443` hoặc `7844` vào lab;
- công bố IP worker hay ClusterIP ra Internet.

Dashboard `Healthy` với `Replicas = 2` chứng minh hai Pod đã đăng ký và giữ được đường kết nối từ cluster tới Cloudflare.

**Vì sao outbound thay được inbound?** Vì "chiều" của một kết nối chỉ tồn tại ở khoảnh khắc khởi tạo: bắt tay xong thì kênh là **hai chiều như nhau** — bên nào cũng gửi được, bên nào cũng nhận được. Router/NAT vận hành đúng theo nguyên lý đó: nó chặn kết nối *khởi tạo từ ngoài vào*, nhưng ghi state cho session *từ trong gọi ra* và cho toàn bộ dữ liệu trả về của session đó đi qua — nếu không thì việc duyệt web thường ngày đã không hoạt động (browser gọi ra, cả trang web đổ về, router không cần mở port nào). Giống một cuộc gọi điện thoại: ai bấm số không quyết định ai được nói; nối máy rồi thì hai bên bình đẳng. `cloudflared` là bên bấm số, Cloudflare là bên nghe máy; request của người dùng sau đó được Cloudflare "nói" xuống qua cuộc gọi đang mở. Cái giá của mô hình nằm ở mục 6: đường public sống chết theo các kết nối do connector giữ — connector chết hết thì hostname mất đường về origin dù DNS vẫn trỏ đúng.

**`cloudflared` gọi ra khi nào — liên tục hay theo từng request?** Không phải mỗi request một lần gọi. Nó quay số đúng **một lần lúc container khởi động**: đọc token, mở mặc định bốn kết nối dài hạn tới ít nhất hai data center Cloudflare khác nhau, rồi giữ chúng sống bằng keep-alive. Từ đó trở đi nó không "gọi ra" thêm — mọi request được Edge multiplex (ghép kênh) xuống các kết nối sẵn có, nhiều request dùng chung một kết nối. `cloudflared` chỉ quay số lại khi một kết nối đứt (mạng chập chờn, Cloudflare bảo trì data center, Pod restart) — tự động nối lại, không cần ai can thiệp. Vì vậy trạng thái `Healthy` trên dashboard nghĩa là "các kết nối đang sống", không phải "vừa có request thành công".

## 3. Published application là bảng ánh xạ hostname → origin nội bộ

Khai báo ở bước kế tiếp có ý nghĩa:

```text
Public hostname: app.hieupn.site
Tunnel:          homelab-k8s
Service URL:     http://traefik.traefik.svc.cluster.local:80
```

Tức là:

```text
Nếu Cloudflare nhận request cho app.hieupn.site
→ chọn tunnel homelab-k8s
→ gửi request xuống một connector Healthy
→ cloudflared gọi http://traefik.traefik.svc.cluster.local:80
```

Đây là cấu hình của **remotely-managed tunnel**, được lưu ở Cloudflare và áp dụng cho các connector của tunnel. Có thể publish nhiều hostname qua cùng một tunnel, mỗi hostname ánh xạ tới một service nội bộ khác nhau. Xem [Cloudflare — Published application routing](https://developers.cloudflare.com/tunnel/routing/).

## 4. Có hai hệ DNS độc lập

Khi lưu Published application trong Full Setup, Cloudflare tự tạo DNS record public tương đương:

```text
app.hieupn.site → <TUNNEL-UUID>.cfargotunnel.com
```

Record được proxy qua Cloudflare. Người dùng ngoài Internet nhận địa chỉ Anycast của Cloudflare, không nhận ClusterIP, Pod IP, worker IP hoặc IP router của lab.

Ở phía trong cluster, `cloudflared` lại dùng CoreDNS để phân giải origin:

| Hệ DNS               | Tên được phân giải              | Người dùng kết quả                                |
| --------------------- | ------------------------------------- | ------------------------------------------------------ |
| Cloudflare public DNS | `app.hieupn.site`                   | Browser ngoài Internet tìm tới Cloudflare Edge      |
| Kubernetes CoreDNS    | `traefik.traefik.svc.cluster.local` | Pod`cloudflared` tìm ClusterIP của Service Traefik |

Cloudflare Edge không phân giải và không truy cập trực tiếp tên `*.svc.cluster.local`; tên này chỉ có nghĩa sau khi request đã xuống Pod `cloudflared` bên trong cluster. Kubernetes định nghĩa DNS Service theo dạng `<service>.<namespace>.svc.<cluster-domain>`; xem [Kubernetes — DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/).

## 5. Request từ browser tới Cloudflare Edge

Khi người dùng mở:

```text
https://app.hieupn.site
```

trình duyệt thực hiện:

1. Hỏi DNS public và nhận địa chỉ Cloudflare.
2. Mở HTTPS tới Cloudflare Edge trên port `443`.
3. Nhận certificate cho `app.hieupn.site` từ Cloudflare.
4. Gửi request có HTTP Host header `app.hieupn.site`.
5. Cloudflare có thể áp dụng TLS termination, WAF, DDoS protection, cache và các rule ở edge.

Browser kết nối tới Cloudflare, **không** kết nối tới router hay worker của lab.

## 6. Cloudflare gửi request xuống đường hầm đã tồn tại

Cloudflare dùng hostname và DNS/tunnel route để xác định tunnel UUID. Nó chọn một connector đang Healthy rồi multiplex request xuống kết nối mã hóa mà connector đó đã mở từ trước:

```text
Không phải: Cloudflare → mở kết nối mới vào IP lab

Mà là:     cloudflared → mở kết nối outbound tới Cloudflare
            Cloudflare → trả request xuống kết nối đang mở
```

Nếu một Pod `cloudflared` chết, replica còn lại có thể tiếp tục. Nếu không còn connector nào Healthy nhưng DNS record vẫn tồn tại, Cloudflare không tới được origin; DNS record không tự biến mất chỉ vì tunnel dừng. Xem [Cloudflare Tunnel overview](https://developers.cloudflare.com/tunnel/) và [Cloudflare routing](https://developers.cloudflare.com/tunnel/routing/).

## 7. Tunnel kết thúc tại Pod `cloudflared`

Tunnel không kết thúc tại Traefik. Điểm cuối trong lab là Pod `cloudflared`, và Pod này tiếp tục đóng vai trò reverse proxy/client cho origin nội bộ:

```text
Cloudflare Edge ══ tunnel mã hóa ══> cloudflared Pod
cloudflared Pod ── HTTP nội bộ ──> Traefik Service
```

Service URL không phải địa chỉ mà Cloudflare Edge truy cập trực tiếp. Nó là chỉ dẫn để `cloudflared`, sau khi nhận request trong cluster, mở một connection mới tới origin.

## 8. `cloudflared` dùng CoreDNS tìm Service Traefik

Tên origin được tách như sau:

```text
traefik.traefik.svc.cluster.local:80
│       │       │   │             │
│       │       │   │             └─ Service port HTTP
│       │       │   └─ DNS domain nội bộ mặc định
│       │       └─ đối tượng Kubernetes Service
│       └─ namespace
└─ tên Service
```

CoreDNS trả ClusterIP của Service `traefik`. `cloudflared` gọi Service port `80`; cơ chế dataplane của Kubernetes chuyển connection tới một Pod Traefik đang Ready. Dùng Service DNS thay vì Pod IP giúp origin ổn định khi Pod Traefik bị tạo lại hoặc đổi IP.

## 9. Traefik dùng Host header chọn đúng ứng dụng

Request đến Traefik vẫn mang:

```http
Host: app.hieupn.site
```

Ta không đặt **HTTP Host Header override**, nên không yêu cầu `cloudflared` thay hostname gửi tới origin. Traefik so sánh Host header với Ingress đã PASS §12.3.1:

```yaml
spec:
  rules:
    - host: app.hieupn.site
      http:
        paths:
          - path: /
            backend:
              service:
                name: web
                port:
                  number: 80
```

Khi hostname khớp, Traefik chuyển request tới Service `web:80`. Nếu Ingress còn `app.example.com`, Traefik không tìm thấy Router phù hợp và thường trả `404`. Tùy chọn `httpHostHeader` chỉ cần khi muốn chủ động ghi đè Host gửi tới origin; xem [Cloudflare — Origin parameters](https://developers.cloudflare.com/tunnel/advanced/origin-parameters/).

## 10. Service `web` chọn Pod và response quay ngược lại

Service `web` chọn các Pod có label `app=web` rồi chuyển request tới một backend. Pod tạo response có thể chứa:

```text
Server address: <Pod-IP>:80
Server name: web-<replicaset-hash>-<pod-suffix>
```

Response đi ngược đúng chuỗi:

```text
Pod web
→ Service web
→ Traefik
→ cloudflared
→ tunnel mã hóa
→ Cloudflare Edge
→ Browser
```

Chạy request nhiều lần có thể thấy `Server name`/`Server address` thay đổi giữa hai Pod web; việc chọn backend thuộc tầng Kubernetes Service/Traefik, không phải hai replica `cloudflared`.

## 11. Vì sao browser dùng HTTPS nhưng Service URL dùng HTTP?

Hai trường này thuộc hai chặng khác nhau:

```text
Browser ── HTTPS ──> Cloudflare Edge
Cloudflare Edge ══ tunnel mã hóa ══> cloudflared
cloudflared ── HTTP nội bộ ──> Traefik:80
```

Cloudflare xử lý certificate và TLS public ở edge; kết nối Edge ↔ `cloudflared` được tunnel bảo vệ. `Type=HTTP` hoặc scheme `http://` chỉ mô tả chặng cuối từ Pod `cloudflared` tới Traefik bên trong cluster. Nếu threat model yêu cầu mã hóa cả chặng nội bộ, có thể cấu hình HTTPS origin riêng; lab này dùng HTTP để tránh quản lý thêm certificate nội bộ.

## 12. Ba liên kết mà bước Published application tạo ra

```text
1. Public DNS
   app.hieupn.site → Cloudflare

2. Tunnel routing
   app.hieupn.site → homelab-k8s

3. Internal origin
   homelab-k8s → http://traefik.traefik.svc.cluster.local:80
```

Điều đưa traffic từ Cloudflare vào Kubernetes là tổ hợp của **các Pod `cloudflared` nằm trong cluster**, **kết nối outbound dài hạn do chúng mở**, và **route ánh xạ hostname public tới Kubernetes Service nội bộ** — không phải port-forward hay public IP của lab.

---

## Tự kiểm tra

Các tham chiếu §9–§14 bên dưới trỏ tới [runbook-k8s-vmware.md](../runbook-k8s-vmware.md).

Phần này kiểm tra kiến thức Cloudflare Tunnel → Traefik → ứng dụng sau khi hoàn tất
§9–§13. Cách trình bày theo [bài Probes](../k8s-docs/49-probes-vi.md): tự trả lời trước,
sau đó mới mở đáp án. Sáu câu đầu là bộ câu hỏi vòng 1; các câu tiếp theo tổng hợp
những câu hỏi bổ sung đã dùng để làm rõ kiến thức. Không cần nhớ nguyên lệnh,
nhưng phải giải thích được từng cấu hình và phạm vi của từng phép kiểm tra.

Đọc lại khi cần:
[luồng và vai trò thành phần](tunnel-traefik.md),
[request/response theo 14 bước](tunnel-traefik-request-response.md).

### Câu hỏi — vòng 1

1. **Đường đi của request:** khi mở `https://app.hieupn.site/`, request đi từ browser
   đến Pod web và response quay về thế nào? Thành phần nào tìm địa chỉ, chuyển tiếp
   HTTP, chỉ cung cấp cấu hình, chuyển tiếp mạng và xử lý ứng dụng? TLS diễn ra trước
   hay sau khi browser gửi HTTP request?
2. **Tunnel và NAT:** không mở inbound trên router mà vẫn truy cập được app vì sao?
   Ai mở kết nối trước, tới đâu? Cloudflare chuyển request về lab bằng cách nào?
   Vì sao đó không phải kết nối inbound mới, và vì sao không gọi request xuống tunnel
   là “HTTP response cho việc mở outbound”? Các bước setup/verify nằm ở đâu?
3. **Hai hệ DNS:** browser phân giải `app.hieupn.site` nhận IP gì? Connector phân giải
   `traefik.traefik.svc.cluster.local` hỏi ai và nhận IP gì? Vì sao Edge không cần
   tự phân giải tên Service? Bước nào quyết định connector hỏi Service Traefik thay
   vì Service khác?
4. **Publish app:** cluster, Traefik và app đã test nội bộ thành công, Ingress còn
   hostname tạm `app.example.com`, chưa chuẩn bị domain/tunnel. Hãy nêu đúng thứ tự
   thao tác tại Hostinger, Cloudflare và Kubernetes, cùng điều kiện kiểm tra trước
   khi coi publish hoàn tất.
5. **Service URL và Host:** origin là `http://traefik.traefik.svc.cluster.local:80`
   nhưng Ingress khai `host: app.hieupn.site`. Vì sao vẫn khớp? Hostname origin và
   HTTP Host phục vụ nhiệm vụ gì? Nếu override Host thành
   `traefik.traefik.svc.cluster.local`, rule còn khớp không? Override có đổi sang
   HTTPS hoặc đổi port không?
6. **Khoanh vùng lỗi:** `curl -H 'Host: app.hieupn.site' "http://$ING_IP/"` trả đúng
   app nhưng browser báo không tìm thấy hostname public. Test nội bộ chứng minh
   được gì và chưa chứng minh gì? Kiểm tra tiếp bằng lệnh/dashboard nào? Có nên
   sửa Ingress hoặc restart Pod web ngay không?

### Câu hỏi bổ sung

7. Nếu request mang Host `app.hieupn.site` nhưng Ingress vẫn khai
   `app.example.com`, ai phát hiện không khớp, kết quả thường là gì và kiểm tra thế nào?
8. Chưa tạo Published application route nhưng connector đã kết nối thành công:
   tunnel có thể Healthy không? App đã public được chưa?
9. Public DNS và tunnel hoạt động nhưng CoreDNS hỏng: request có thể tới connector
   không? Connector gọi origin bằng tên Service được không? Giả sử không có DNS cache
   hoặc kết nối origin mở sẵn.
10. Vì sao request vào đúng tunnel/connector thay vì cluster khác có cùng tên Service?
    Token có khóa connector vào một cluster cụ thể không? Subdomain/Domain, tunnel
    đang chọn và Service URL mỗi thứ quyết định gì?
11. CoreDNS biết ClusterIP của Service Traefik bằng bước setup nào? Có cần thêm record
    thủ công không? Sau khi DNS trả IP, connector gọi bằng IP hay tên? Ai cung cấp
    origin URL cho connector trong mô hình remotely-managed?
12. Nameserver có ý nghĩa gì? Trước/sau §11.2, ai quản lý DNS và ai giữ việc đăng ký
    domain? Record của app sau publish là gì? Địa chỉ Traefik có nằm trong DNS public
    không? Root domain và `www` có tự phục vụ cùng app không?
13. Dữ liệu đi qua tunnel có bắt buộc là JSON không? Mô tả method, path/query,
    Host, headers, body của GET `/?page=2` và POST có JSON; connector gọi origin
    thế nào và response quay về ra sao? Bước nào thiết lập Host thực tế và bước nào
    chỉ khai báo Host mong đợi?
14. Hai Ingress `app.hieupn.site` → Service `web` và `api.hieupn.site` → Service
    `api` cùng dùng origin Service Traefik. Traefik phân biệt bằng gì? Replica
    connector 1 có bị ghép cố định với Pod web replica 1 không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

### Đáp án 1 — Đường đi và vai trò

Browser dùng resolver để tìm Edge IP, kết nối tới Edge trên port 443, hoàn tất
TLS handshake và kiểm tra certificate rồi mới gửi HTTP request. Edge chọn tunnel
theo hostname, truyền request qua kết nối đã mở tới một connector. Connector
phân giải tên origin bằng CoreDNS, gọi ClusterIP Service Traefik:80; dataplane
chuyển tới Traefik Pod. Traefik khớp Host/path với rule Ingress rồi chọn backend
của Service web; một Pod web xử lý và tạo response. Traefik có thể gọi Pod endpoint
trực tiếp hoặc qua Service IP tùy cấu hình.

Response quay về Pod → Traefik → connector → tunnel → Edge → browser trên các
kết nối tương ứng. Không có một TCP connection duy nhất xuyên từ browser tới Pod.
DNS và object cấu hình không chuyển tiếp HTTP response.

| Nhóm | Thành phần | Nhiệm vụ |
| --- | --- | --- |
| Tìm địa chỉ | Public DNS, CoreDNS | Trả IP cần kết nối |
| Nhận/chuyển tiếp HTTP | Edge, cloudflared, Traefik | Proxy request/response |
| Cấu hình chỉ đường | Published application route, Ingress | Khai báo tunnel/origin và Host/path → backend |
| Địa chỉ và chuyển tiếp mạng | Service, ClusterIP, EndpointSlice | Địa chỉ ổn định và thông tin backend; dataplane thực hiện chuyển tiếp |
| Xử lý ứng dụng | Pod web | Tạo response |

Service là object, ClusterIP là địa chỉ ảo, EndpointSlice lưu thông tin endpoint;
không phải ba chương trình HTTP proxy. Tham chiếu §9–§13.

### Đáp án 2 — Outbound không đồng nghĩa chỉ truyền một chiều

`cloudflared` mở outbound tới Cloudflare, dùng QUIC/UDP hoặc HTTP/2/TCP 7844,
đăng ký connector rồi giữ kết nối. Router/NAT cho traffic thuộc kết nối đó quay về.
Edge truyền request qua chính kênh đã tồn tại, không mở connection mới tới IP lab.

Ở tầng HTTP, dữ liệu xuống connector vẫn là **request của người dùng**, chưa phải
response của app. Response chỉ được backend tạo ra sau khi xử lý request.
Thiết lập kênh tunnel không phải một GET/POST thông thường tới app.

§12.1 tạo tunnel/token; §12.2 chạy connector và kiểm tra Deployment, Pod,
dashboard Healthy; §12.3.3 gắn hostname và origin; §5.7 liên quan egress/firewall.
§13 kiểm tra đường public; §10.3 và §12.3.1 kiểm tra riêng đường nội bộ.

### Đáp án 3 — Hai hệ DNS độc lập

Browser nhận Edge IPv4/IPv6 vì record public Proxied. Connector hỏi CoreDNS
và nhận ClusterIP của Service `traefik` trong namespace `traefik`.
**Service URL ở §12.3.3** quyết định tên origin được phân giải.

Edge chỉ cần đưa request vào đúng tunnel/connector; connector mới trực tiếp gọi
origin và dùng DNS của cluster. Public DNS không phân giải tên
`*.svc.cluster.local` để giúp Edge gọi Service.

### Đáp án 4 — Thứ tự publish và gate

| Bước | Thao tác | Kết quả cần kiểm tra |
| --- | --- | --- |
| §11.1 | Mua domain tại Hostinger | Có quyền quản lý domain |
| §11.2 | Review record hiện có, xử lý DNSSEC cũ nếu có, thêm domain vào Cloudflare, đổi NS tại Hostinger theo đúng cặp được cấp | Resolver trả đúng NS và zone Active; hoàn tất gate DNSSEC theo runbook |
| §12.1 | Tạo homelab-k8s, lấy token | Đúng tunnel và token |
| §12.2 | Tạo namespace cloudflare, Secret token, Deployment cloudflared | Hai Pod Running, Deployment 2/2 và tunnel Healthy |
| §12.3.1 | Sửa Ingress hiện có sang app.hieupn.site, apply và test Host mới | Host đúng, curl nội bộ trả demo |
| §12.3.3 | Save route trong đúng tunnel, Service URL HTTP Traefik:80 | Route đúng, DNS record Proxied tồn tại đúng một lần, connector còn Healthy |
| §13.1–§13.3 | Kiểm tra DNS, HTTPS headers, body/browser | Edge IP, không lỗi TLS, status/nội dung đúng |

Không cần tạo Ingress thứ hai. Field `host` chứa `app.hieupn.site`,
không chứa `https://` hoặc dấu `/`. Không công khai token trong repo/log/chat.

### Đáp án 5 — Đích kết nối khác danh tính website

Hostname origin được phân giải thành IP để connector kết nối tới **Traefik**.
Host header giữ `app.hieupn.site` trong cấu hình lab và được Traefik dùng để
chọn website/router; path tiếp tục khớp rule đường dẫn.

`traefik.traefik.svc.cluster.local` là hostname; chuỗi có scheme và port là
Service URL đầy đủ. Host header không chỉ định trực tiếp Pod cuối cùng.

Override Host thành tên Service sẽ không khớp rule `app.hieupn.site`;
nếu không có router khác phù hợp, Traefik thường trả 404. Nó không đổi scheme/port:
origin vẫn HTTP:80 do Service URL quyết định. Tham chiếu §12.3.1 và §12.3.3.

### Đáp án 6 — Bắt đầu ở DNS, không restart backend

Test nội bộ chứng minh đường Service Traefik → Traefik khớp Ingress → backend
web trả được nội dung từ vị trí chạy test. Nó bỏ qua DNS public, Edge/TLS, route
Cloudflare và tunnel.

Bắt đầu bằng:

```bash
nslookup app.hieupn.site
nslookup app.hieupn.site 1.1.1.1
```

Đối chiếu **Cloudflare DNS → Records**: record app đúng tunnel và Proxied;
**domain Overview**: zone Active; **Hostinger**: đúng cặp NS Cloudflare;
**tunnel → Published application routes**: route đã lưu đúng hostname.
Nếu public resolver trả đúng nhưng máy/browser không resolve được, kiểm tra
resolver/cache, Secure DNS của browser và mạng client. Nếu SERVFAIL, kiểm tra
thêm delegation/DNSSEC; NXDOMAIN cần kiểm tra record và tên truy vấn.

Khi DNS đã đúng, tiếp tục:

```bash
curl -I https://app.hieupn.site
curl -sS https://app.hieupn.site
```

Hai lệnh curl đều dừng ở DNS nếu hostname chưa phân giải được. Không sửa Ingress
hay restart backend chỉ vì lỗi DNS public khi test nội bộ vẫn thành công.
Tham chiếu §11.2, §12.3.3, §13.1–§13.3.

### Đáp án 7 — Host không khớp

Traefik so Host với rule đã nạp. Không có router phù hợp thì thường trả 404;
request chưa được chuyển tới backend của rule đó. Kiểm tra:

```bash
kubectl get ingress web -o jsonpath='{.spec.rules[*].host}{"\n"}'
kubectl describe ingress web
ING_IP=$(kubectl -n traefik get svc traefik -o jsonpath='{.spec.clusterIP}')
curl -i -H 'Host: app.hieupn.site' "http://$ING_IP/"
```

Đối chiếu Host, path và IngressClass. Sửa source manifest, apply rồi test lại theo
§12.3.1. Một 404 nói chung cũng có thể do app trả; phải kiểm tra route/log để xác
định tầng, không kết luận mọi 404 đều do Ingress.

### Đáp án 8 — Healthy chưa chứng minh app đã public

**Có thể Healthy.** Trạng thái đó chứng minh connector kết nối tới Cloudflare,
không chứng minh hostname/route/origin đầy đủ. Nếu chưa có cấu hình khác cho hostname,
app chưa public được qua đường runbook này. §12.3.3 tạo ánh xạ hostname → tunnel
và origin, đồng thời tạo DNS record trong Full Setup.

### Đáp án 9 — CoreDNS hỏng

Request vẫn có thể tới connector qua tunnel đang mở. Nhưng với giả định không có
DNS cache hoặc connection origin sẵn có, connector không resolve được tên Service
nên không mở được kết nối tới Traefik bằng Service URL hiện tại. Tunnel Healthy
không chứng minh DNS nội bộ hay origin hoạt động. Kiểm tra §8.4 và log connector.

### Đáp án 10 — Chọn đúng tunnel và origin

§12.1 tạo tunnel có ID riêng/token; §12.2 đưa token vào connector để đăng ký đúng
tunnel; §12.3.3 tạo route trong tunnel đó. Subdomain + Domain xác định hostname,
tunnel đang chọn xác định tập connector, Service URL xác định origin.

Tên Service giống nhau giữa hai cluster không làm request tự nhảy sang cluster khác.
Tuy nhiên token **không khóa cluster**: connector ở cluster khác dùng cùng token
có thể tham gia cùng tunnel. Muốn tách đường vào thì dùng tunnel/token riêng.

### Đáp án 11 — DNS Service và kết nối origin

§6 triển khai CoreDNS; §8.4 kiểm tra DNS; §9.3 tạo Service traefik trong namespace
traefik kiểu ClusterIP; §12.2 dùng DNS policy mặc định ClusterFirst. CoreDNS
theo dõi Service Kubernetes và trả IP theo tên/namespace, không cần nhập record
riêng. §12.3.3 quyết định tên được hỏi qua Service URL.

Ví dụ DNS trả `10.96.123.45` thì connector kết nối TCP tới `10.96.123.45:80`.
Tên dùng để tìm IP; IP dùng để mở connection; port 80 lấy từ URL, không từ DNS.
Request bên trong connection vẫn mang Host public. Cache/connection có thể được
tái sử dụng. Origin cấu hình trên Cloudflare được cung cấp cho connector của
remotely-managed tunnel; không cần ghi địa chỉ Traefik trong manifest §12.2.

### Đáp án 12 — Nameserver và DNS record

Nameserver authoritative cung cấp bản ghi có thẩm quyền cho domain; recursive
resolver tra cứu/cache các câu trả lời đó. NS không phải web server.

Trước §11.2, nếu dùng mặc định Hostinger thì NS thường là
`ns1.dns-parking.com`/`ns2.dns-parking.com`; phải kiểm tra domain thực tế.
Sau khi đổi NS và propagation hoàn tất, đó là **đúng hai tên Cloudflare cấp cho
zone**, dạng `<tên>.ns.cloudflare.com`. Cloudflare quản DNS; Hostinger vẫn giữ
đăng ký/gia hạn. §11.2.3 lấy NS, §11.2.4 thay tại Hostinger, §11.2.5 verify.

Record app trong Full Setup: **CNAME app → <TUNNEL-UUID>.cfargotunnel.com,
Proxied**. DNS public thường trả Edge IP. Origin
`http://traefik.traefik.svc.cluster.local:80` nằm trong route tunnel, không nằm
trong record public. Root domain và www không tự phục vụ app; cần route/record
riêng nếu muốn dùng. Record email/website khác được giữ theo nhu cầu tại §11.2.

### Đáp án 13 — Request, tunnel và response

Mở outbound là thiết lập transport/đăng ký tunnel, không phải một GET/POST tới app.
Dữ liệu người dùng trong tunnel là biểu diễn HTTP request: metadata và body stream
được đóng gói/mã hóa theo giao thức; định dạng phụ thuộc transport/triển khai,
không phải luôn một JSON object.

Ví dụ HTTP/1.1 minh họa:

```http
GET /?page=2 HTTP/1.1
Host: app.hieupn.site
Accept: text/html

```

Method GET, path /, query page=2, body rỗng. Browser tạo Host từ URL
(HTTP/2/3 dùng :authority tương ứng). §12.3.1 khai Host **mong đợi** trong Ingress;
§12.3.3 có tùy chọn HTTP Host Header để override Host gửi origin.
Không override trong cấu hình lab thì giữ hostname public.

Connector nhận request, dùng origin URL để resolve tên Service rồi mở hoặc
tái sử dụng TCP connection tới ClusterIP:80, gửi method/path/query/headers/body.
Traefik khớp Host/path và proxy tới backend; headers hoặc framing có thể được proxy
điều chỉnh. Không có rule đổi method/rewrite path trong cấu hình đang xét.

Ví dụ POST, **chỉ khi thay demo bằng API hỗ trợ endpoint này**:

```http
POST /api/orders?dry_run=true HTTP/1.1
Host: app.hieupn.site
Content-Type: application/json

{"product_id":123,"quantity":2}
```

JSON chỉ là body; method là POST, query là dry_run=true. App quyết định response,
nginx demo không tự trở thành API tạo đơn hàng.

Response gồm status, headers, body, quay về Pod → Traefik → connector → tunnel →
Edge → browser, có thể truyền theo từng phần. DNS không chuyển response.
Browser–Edge dùng HTTPS, Edge–connector dùng tunnel mã hóa, connector–Traefik
và Traefik–web dùng HTTP nội bộ trong lab. curl -I gửi HEAD, không có response body;
curl -sS mặc định GET. Cache/edge rules có thể khiến request không tới origin;
headers Cloudflare hoặc status 200 tự chúng không chứng minh origin vừa được gọi.

### Đáp án 14 — Một điểm vào, nhiều ứng dụng

Traefik dùng Host `api.hieupn.site` và path để khớp Ingress của API, dù cả hai
hostname public đều đi tới cùng Service Traefik. ClusterIP/port chọn ingress
controller; Host/path chọn ứng dụng.

Lựa chọn connector và backend Pod độc lập. Connector replica 1 không bị ghép
với Pod web replica 1; một request thông thường không được broadcast tới mọi replica.

</details>

### Câu hỏi — vòng 2: vận hành và xử lý sự cố

#### Câu 1/6 — Tunnel mất kết nối

Trước đó `https://app.hieupn.site/` hoạt động bình thường. Hiện tại:

- `nslookup app.hieupn.site` vẫn trả Cloudflare Edge IP.
- Browser hiển thị lỗi Cloudflare **1033**.
- Dashboard tunnel `homelab-k8s` không còn connector Healthy.
- Gọi nội bộ qua ClusterIP Traefik với `Host: app.hieupn.site` vẫn trả đúng nội dung app.

Hãy trả lời:

1. Các kết quả trên giúp khoanh vùng lỗi ở phần nào? Những phần nào đã có bằng chứng hoạt động?
2. Bạn sẽ kiểm tra gì trong Kubernetes? Nêu lệnh và mục đích của từng lệnh.
3. Nếu Pod `cloudflared` vẫn `Running` nhưng log liên tục báo không kết nối được tới Edge,
   bạn kiểm tra những điều kiện mạng nào?
4. Nếu log báo lỗi xác thực tunnel, bạn kiểm tra cấu hình nào trong Deployment/Secret
   và đối chiếu với thông tin nào trên Cloudflare?

**Câu hỏi bổ sung:** Windows truy cập Internet bình thường, cả hai Pod connector đều
`Running`, nhưng log liên tục báo timeout khi kết nối Edge trên port `7844`.
Đã đủ cơ sở kết luận token sai chưa? Bạn ưu tiên kiểm tra gì tiếp theo?

<details>
<summary>Đáp án vòng 2 — câu 1/6: chỉ mở sau khi đã tự trả lời</summary>

**1. Khoanh vùng bằng bằng chứng**

Trong tình huống đã cho, tập trung vào **kết nối connector ↔ Cloudflare**:
1033 cùng trạng thái không có connector Healthy cho thấy Cloudflare không có
connector khả dụng để chuyển request xuống tunnel được chọn.

| Bằng chứng | Chứng minh được | Chưa chứng minh được |
| --- | --- | --- |
| DNS trả Edge IP | Resolver phân giải hostname public được | Connector đã kết nối hoặc DNS record đang trỏ đúng tunnel mong muốn |
| Browser nhận trang lỗi Cloudflare | Browser đã tới Cloudflare và nhận response | Origin trong cluster nhận request |
| Test ClusterIP Traefik với Host thật trả đúng app | Traefik/Ingress/backend hoạt động từ vị trí chạy test | Đường mạng từ Pod connector tới origin |
| Dashboard không có connector Healthy | Tunnel đang kiểm tra thiếu connector khả dụng | Nguyên nhân là token, DNS, firewall hay thành phần khác |

Đối chiếu thêm ID tunnel mà DNS record/route trỏ tới với tunnel đang xem trên
dashboard; không chỉ dựa vào tên hiển thị. Chưa có lý do sửa Ingress hoặc restart
Pod web khi test nội bộ vẫn PASS. Tham chiếu §12.1–§12.3.3 và §13.

**2. Kiểm tra Deployment, cả hai Pod, events và log**

```bash
kubectl -n cloudflare get deployment cloudflared
kubectl -n cloudflare get pods -l app=cloudflared -o wide
kubectl -n cloudflare describe deployment cloudflared
```

- Deployment: desired/ready/available replicas và tình trạng rollout.
- Pods: trạng thái, số restart, Pod IP và node đang chạy để khoanh đúng đường mạng.
- Describe Deployment: điều kiện và events liên quan việc tạo/duy trì replica.

Với **từng Pod** tìm được, thay `<pod-name>` bằng tên thật:

```bash
kubectl -n cloudflare describe pod <pod-name>
kubectl -n cloudflare logs <pod-name> --tail=100
```

- `describe pod`: xem events, trạng thái container, lỗi khởi động, probe và tham chiếu Secret.
- `logs`: phân biệt DNS lookup failure, connection timeout, TLS hoặc authentication error.

Nếu container đã restart, đọc log lần chạy trước nếu còn:

```bash
kubectl -n cloudflare logs <pod-name> --previous --tail=100
```

`get pods` **không đọc log**. Pod `Running` chỉ cho biết trạng thái chạy container,
không tự chứng minh connector đã đăng ký/kết nối thành công với Edge.
Không chỉ kiểm tra một replica rồi suy ra cả tunnel. Không chia sẻ log có token.

**3. Kiểm tra egress từ Pod/node connector**

Windows vào Internet được chưa chứng minh đường ra của Pod Kubernetes hoạt động.
Kiểm tra đường mạng từ connector qua node, gateway/NAT tới Cloudflare:

- DNS trong môi trường Pod có phân giải được các endpoint Edge không?
- Node/Pod có route ra ngoài và gateway/NAT hoạt động không?
- Firewall trên node, router hoặc NetworkPolicy có chặn egress không?
  NetworkPolicy chỉ có hiệu lực khi dataplane/CNI triển khai việc enforcement;
  việc có object policy không tự chứng minh nó đang chặn traffic.
- Outbound **UDP 7844** cho QUIC và **TCP 7844** cho HTTP/2 có được phép tới
  các endpoint tunnel của Cloudflare không? Kiểm tra theo giao thức thực tế trong log.
- Nếu chỉ UDP bị chặn, chế độ tự động có thể chuyển sang HTTP/2 qua TCP.
  Nếu ép QUIC, cần đối chiếu cấu hình protocol thay vì giả định luôn có fallback.
- Nếu log là lỗi certificate/TLS, kiểm tra thêm giờ hệ thống và việc can thiệp TLS
  trên đường mạng theo bằng chứng.

Truy cập web HTTPS trên port 443 thành công **không chứng minh port 7844 thông**.
Dùng log connector và test/debug trong môi trường mạng phù hợp; không mở inbound
router để chữa lỗi kết nối outbound. Tham chiếu §5.7 và §12.2.

**4. Kiểm tra token và cách container nhận token**

Đối chiếu chuỗi cấu hình của §12.2:

| Thành phần | Giá trị mong đợi |
| --- | --- |
| Deployment | `cloudflared`, namespace `cloudflare` |
| Biến môi trường | `TUNNEL_TOKEN` |
| Nguồn biến | `valueFrom.secretKeyRef` |
| Secret | `tunnel-token`, cùng namespace |
| Key | `token` |
| Token thực tế | Còn hợp lệ và thuộc đúng tunnel dự định dùng ở §12.1 |

Kiểm tra Secret tồn tại mà không in giá trị token:

```bash
kubectl -n cloudflare get secret tunnel-token
```

Xem tham chiếu env trong Deployment/Pod; không xuất toàn bộ Secret để đưa vào
nội dung chia sẻ. Xác nhận Secret thực sự chứa key `token`, token được nhập đúng,
không bị thu hồi/rotate và gắn với đúng tunnel/account trên Cloudflare.
Không kết luận token hợp lệ chỉ vì Secret tồn tại hoặc tên tunnel giống nhau.

Nếu token đã rotate, lấy token hợp lệ của đúng tunnel và cập nhật Secret bằng
cách riêng tư. Token truyền qua **biến môi trường không tự cập nhật trong container
đang chạy** khi Secret đổi; sau khi sửa cần tạo lại Pod để nạp giá trị mới:

```bash
kubectl -n cloudflare rollout restart deployment/cloudflared
kubectl -n cloudflare rollout status deployment/cloudflared --timeout=180s
kubectl -n cloudflare get pods -l app=cloudflared -o wide
```

Chỉ coi khôi phục khi các replica ổn định, log cho thấy connector đã kết nối,
dashboard đúng tunnel có connector Healthy và kiểm tra public theo §13 thành công.
Restart là bước nạp cấu hình đã sửa, không thay thế việc tìm nguyên nhân.

**Đáp án câu bổ sung**

**Chưa đủ cơ sở kết luận token sai.** Timeout ưu tiên gợi ý vấn đề reachability:
kiểm tra egress từ Pod/node, firewall/router/NetworkPolicy có hiệu lực, cùng outbound
UDP/TCP 7844 tới Edge. Token sai cần bằng chứng lỗi xác thực phù hợp; Internet
trên Windows và trạng thái Pod Running không thay thế kiểm tra đường mạng tunnel.

</details>

#### Câu 2/6 — Tunnel Healthy nhưng app trả 502

Bạn quan sát:

- `nslookup app.hieupn.site` trả Cloudflare Edge IP.
- Dashboard tunnel `homelab-k8s` hiển thị **Healthy**.
- Browser truy cập `https://app.hieupn.site/` nhận **502 Bad Gateway**.
- Origin trên Cloudflare được cấu hình là `http://traefik.traefik.svc.cluster.local:80`.

Trên **máy master**, chạy đúng hai lệnh:

```bash
ING_IP=$(kubectl -n traefik get svc traefik \
  -o jsonpath='{.spec.clusterIP}')

curl -v -H 'Host: app.hieupn.site' "http://$ING_IP/"
```

Lệnh curl trả đúng nội dung app. Request này xuất phát từ terminal trên máy master,
dùng trực tiếp ClusterIP; **không chạy trong Pod connector, không đi qua tunnel,
không phân giải hostname Service Traefik**. Lệnh kubectl phía trước chỉ lấy ClusterIP
từ Kubernetes API, không kiểm tra CoreDNS.

Hãy trả lời:

1. Tunnel Healthy chứng minh điều gì? Vì sao đường public vẫn có thể trả 502?
2. Lệnh curl trên master ở đề bài chứng minh những phần nào hoạt động? Nó đã chứng minh
   **Pod cloudflared phân giải được tên Service và kết nối được tới Traefik** chưa? Vì sao?
3. Để kiểm tra đường **connector → origin**, cần kiểm tra cấu hình gì trên Cloudflare
   và thực hiện các phép kiểm tra DNS/HTTP từ đâu? Nêu lệnh hoặc cách kiểm tra.
4. Nếu log connector báo `lookup traefik.traefik.svc.cluster.local ... no such host`,
   ưu tiên kiểm tra những gì? Đây có phải bằng chứng rule Ingress sai không?
   Có nên sửa token hoặc mở inbound router không?

**Câu hỏi bổ sung:** từ **môi trường mạng của chính Pod connector**, bằng debug container
có công cụ phù hợp, bạn thấy:

- Phân giải `traefik.traefik.svc.cluster.local` thất bại.
- Gọi trực tiếp ClusterIP Traefik trên port 80, với `Host: app.hieupn.site`,
  trả đúng nội dung app.

Khoanh vùng lỗi ở phần nào? Những nguyên nhân nào cần kiểm tra tiếp?
Có cần sửa Ingress hay restart Pod web không?

<details>
<summary>Đáp án vòng 2 — câu 2/6: chỉ mở sau khi đã tự trả lời</summary>

**1. Healthy chỉ chứng minh kết nối tunnel, không chứng minh origin**

Tunnel Healthy cho biết có kết nối connector hoạt động với Cloudflare.
Nó không tự chứng minh connector phân giải được hostname origin, gọi được Service
Traefik hoặc nhận được response backend đúng.

502 có thể liên quan đường connector → origin: sai Service URL/port, lỗi DNS,
không kết nối được hoặc origin gặp lỗi. Mã 502 riêng lẻ không xác định chính xác
thành phần tạo lỗi; cần đối chiếu log của request/connector và các phép test.
Nếu có nhiều replica, trạng thái Healthy chung cũng không chứng minh mọi replica
đều gọi được origin.

**2. Test trên master chưa chứng minh đường gọi từ connector**

| Phép kiểm tra | Đường được kiểm tra |
| --- | --- |
| Curl trên master tới ClusterIP, đặt Host public | Mạng master → Service Traefik → Traefik khớp Ingress → backend web |
| Connector gọi origin URL | DNS/mạng Pod connector → Service Traefik → Traefik → backend |

Hai nguồn request có thể có routing, firewall hoặc chính sách mạng khác nhau.
Master gọi được không có nghĩa Pod connector gọi được. Test trực tiếp ClusterIP
còn bỏ qua bước DNS nội bộ, nên không chứng minh hostname Service được phân giải.

**Hai môi trường phân giải tên khác nhau trong cấu hình lab:**

| Nơi chạy | Resolver thường được dùng | Khi hỏi tên Service `traefik.traefik.svc.cluster.local` |
| --- | --- | --- |
| Terminal trên master node | Resolver của Ubuntu theo Netplan §5.2; `systemd-resolved` có thể hiện `127.0.0.53` là stub cục bộ | DNS công cộng không biết tên Service nội bộ, trừ khi node được cấu hình riêng để hỏi DNS của cụm |
| Pod `cloudflared` với `dnsPolicy: ClusterFirst` | DNS của cụm (`kube-dns`/CoreDNS) do kubelet cấu hình cho Pod | CoreDNS trả ClusterIP của Service `traefik` trong namespace `traefik` |

Runbook [§5.2](../runbook-k8s-vmware.md#52-ip-tĩnh-netplan) cấu hình DNS cho
các node bằng `nameservers: addresses: [1.1.1.1, 8.8.8.8]`;
[§8.4](../runbook-k8s-vmware.md#84-tầng-4--dns) kiểm tra phân giải cả
tên Service nội bộ lẫn tên public từ Pod. Với tên ngoài cụm như `google.com`,
CoreDNS không tự giữ sẵn IP của Google: Corefile mặc định dùng
`forward . /etc/resolv.conf` để hỏi DNS upstream. Theo cấu hình node trong
runbook, upstream dự kiến là `1.1.1.1` hoặc `8.8.8.8`. Đây là **chọn DNS công
cộng làm upstream**, không có bước tự cài thêm một DNS server upstream.
Muốn biết cụm đang chạy thực sự dùng upstream nào thì xem ConfigMap/Corefile
CoreDNS và resolver thực tế trong Pod CoreDNS; đừng suy chỉ từ file hướng dẫn.

Riêng lệnh `curl -H 'Host: app.hieupn.site' "http://$ING_IP/"` không dùng DNS
để tìm Traefik: `$ING_IP` đã là ClusterIP lấy bằng `kubectl` từ Kubernetes API.
`Host` là HTTP header để Traefik chọn router, không phải tên mà `curl` phải
phân giải. Vì vậy test trên master chưa kiểm tra CoreDNS của Pod connector,
kể cả khi master và Pod tình cờ dùng chung DNS upstream cho tên public.

**3. Kiểm tra cấu hình và test từ đúng môi trường**

Trên Cloudflare, mở route trong **đúng tunnel → Published application routes**
theo §12.3.3. Đối chiếu scheme `http`, tên
`traefik.traefik.svc.cluster.local`, port `80` và các Host override nếu có.

Kiểm tra Service và endpoint để biết origin tồn tại:

```bash
kubectl -n traefik get svc traefik -o wide
kubectl -n traefik get endpointslice \
  -l kubernetes.io/service-name=traefik -o wide
```

Các lệnh get này đọc trạng thái qua API, không tự kiểm tra đường connector → origin.

Đọc log **connector hiện đang chạy**, đối chiếu cả hai Pod:

```bash
kubectl -n cloudflare get pods -l app=cloudflared -o wide
kubectl -n cloudflare logs <pod-cloudflared> --tail=100
```

Lệnh `cloudflared ... tunnel run` khởi chạy connector mới; không phải lệnh đọc
log Pod Kubernetes hiện tại. Không cần chạy thêm connector để chẩn đoán.

Thực hiện test bằng công cụ sẵn có trong container; nếu image thiếu shell/công cụ,
dùng debug container chia sẻ môi trường mạng của chính Pod connector.
Kiểm tra thêm cấu hình DNS của Pod, `/etc/resolv.conf` và DNS policy của môi trường
test để bảo đảm test phản ánh cách connector phân giải tên.

Trong môi trường đó:

```bash
nslookup traefik.traefik.svc.cluster.local

curl -v -H 'Host: app.hieupn.site' \
  http://traefik.traefik.svc.cluster.local:80/
```

So sánh thêm với request trực tiếp ClusterIP, thay IP minh họa bằng IP thật:

```bash
curl -v -H 'Host: app.hieupn.site' http://10.96.123.45:80/
```

Nếu dùng Pod test riêng, kết quả có ích nhưng chưa chắc đại diện connector vì
label, node, DNS configuration và chính sách mạng có thể khác.
Chỉ kiểm tra trên master không thay thế test từ môi trường connector.

**4. No such host khoanh vùng bước phân giải tên**

Lỗi này cho biết connector không phân giải được hostname origin. Nó chưa có IP để
mở connection tới Traefik cho request đó; vì vậy **chưa có bằng chứng Ingress sai**.

Ưu tiên kiểm tra:

| Hạng mục | Điều cần đối chiếu |
| --- | --- |
| Service URL §12.3.3 | Tên Service/namespace/domain nội bộ có gõ đúng không |
| Service §9.3 | Service traefik có tồn tại trong namespace traefik không |
| Pod connector §12.2 | dnsPolicy/dnsConfig và resolver thực tế có đúng không |
| DNS trong cluster §8.4 | CoreDNS Pods, Service DNS, endpoints, cấu hình/log và khả năng trả lời |
| Đường Pod → DNS | UDP/TCP 53, routing/firewall và NetworkPolicy có hiệu lực |

Không cần sửa token hoặc mở inbound router để xử lý lỗi DNS origin này.
Lý do là **lỗi quan sát được thuộc bước phân giải origin**, không phải vì Healthy
tự chứng minh mọi request đã xuống đúng connector hoặc mọi cấu hình token đều đúng.

**Đáp án câu bổ sung**

Khoanh vùng ở **phân giải DNS nội bộ của hostname origin**. Test trực tiếp IP
thành công chứng minh connector có đường tới Traefik/backend khi bỏ qua DNS.

Chưa đủ bằng chứng chốt CoreDNS đang chạy lỗi. Còn các khả năng: tên/namespace sai,
cấu hình resolver của Pod sai, đường Pod → DNS bị chặn, hoặc CoreDNS/configuration
DNS Kubernetes có vấn đề. Đối chiếu các hạng mục ở bảng trên trước khi sửa.

Không sửa Ingress hoặc restart Pod web để chữa lỗi này: request trực tiếp IP với
Host thật đã khớp route và nhận đúng nội dung app.

</details>

#### Câu 3/6 — App trả 404

Bạn quan sát:

- DNS public trả đúng Cloudflare Edge IP.
- Tunnel `homelab-k8s` Healthy.
- Browser mở `https://app.hieupn.site/` nhận **404 Not Found**.
- Published application route trỏ tới `http://traefik.traefik.svc.cluster.local:80`.

Trên **máy master**, chạy:

```bash
ING_IP=$(kubectl -n traefik get svc traefik \
  -o jsonpath='{.spec.clusterIP}')

curl -i -H 'Host: app.hieupn.site' "http://$ING_IP/"
```

Kết quả cũng là 404. Test này gọi trực tiếp Service Traefik từ master, không đi
qua Cloudflare hoặc connector.

1. Đã đủ bằng chứng kết luận rule Ingress sai chưa? Những thành phần nào có thể
   tạo response 404?
2. Cần kiểm tra những trường nào trong Ingress `web`? Nêu lệnh kiểm tra.
3. Nếu Ingress vẫn khai `host: app.example.com`, tại sao request không khớp?
   Sửa ở đâu và kiểm tra lại thế nào?
4. Nếu Host, path và IngressClass đúng, làm sao phân biệt **Traefik không tìm được
   router phù hợp** với **request đã tới ứng dụng nhưng ứng dụng trả 404**?

**Câu hỏi bổ sung:** Deployment web có hai replica. Gửi một request nhưng chỉ xem
log Pod web replica 1 và không thấy request đó: có thể kết luận Traefik không chuyển
request tới backend không? Cần kiểm tra thêm gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

**1. Chưa đủ kết luận Ingress sai**

Test nội bộ cũng trả 404 cho thấy có thể tái hiện lỗi mà không đi qua Cloudflare,
nhưng chưa xác định tầng nào tạo response. Hai khả năng cần phân biệt:

| Thành phần | Ví dụ tạo 404 |
| --- | --- |
| Traefik | Không có router khớp Host/path/entrypoint |
| Ứng dụng backend | Request đã tới app nhưng app không có resource/path được yêu cầu |

Đường public còn có thể có response do các rule/worker phía Edge nếu được cấu hình;
không dùng một status giống nhau để kết luận chắc chắn cả hai response cùng nguồn.

Ingress là object cấu hình, không tự phát HTTP response. Sai backend port hoặc
Service không có backend thường liên quan lỗi cấu hình/kết nối/không có backend,
có thể dẫn tới 502/503 hoặc router không được nạp tùy trường hợp; phải xem log,
không quy mọi lỗi Service/Deployment thành 404.

**2. Kiểm tra cả rule, class, namespace và router thực tế**

```bash
kubectl -n default get ingress web -o yaml
kubectl -n default describe ingress web
```

Trong lab, app nằm ở namespace default. Đối chiếu:

- `spec.ingressClassName: traefik`.
- `spec.rules[].host: app.hieupn.site`.
- Path `/` và `pathType: Prefix`.
- Backend Service name `web` và Service port `80`.
- Ingress và Service backend cùng namespace.
- Events và log Traefik về việc đọc/nạp cấu hình.

Nếu cần:

```bash
kubectl get ingressclass traefik -o yaml
kubectl -n default get svc web -o yaml
kubectl -n default get endpointslice \
  -l kubernetes.io/service-name=web -o wide
kubectl -n traefik logs deployment/traefik --tail=100
```

YAML đúng trên API chưa tự chứng minh Traefik đã nạp router đúng. Có thể xem
dashboard nội bộ theo §9.3.2 để đối chiếu router/service thực tế.

**3. Host khác nhau nên không khớp; sửa source manifest theo §12.3.1**

Request mang `Host: app.hieupn.site`; rule còn `app.example.com` nên không khớp.
Mở `~/demo-app.yaml`, trong object Ingress đổi:

```yaml
- host: app.example.com
```

thành:

```yaml
- host: app.hieupn.site
```

Sau đó:

```bash
kubectl apply --dry-run=client -f ~/demo-app.yaml
kubectl apply -f ~/demo-app.yaml

kubectl -n default get ingress web \
  -o jsonpath='{.spec.rules[*].host}{"\n"}'

ING_IP=$(kubectl -n traefik get svc traefik \
  -o jsonpath='{.spec.clusterIP}')

curl -i -H 'Host: app.hieupn.site' "http://$ING_IP/"
```

PASS khi object hiện Host đúng và request trả nội dung demo mong đợi. Field
host không có scheme `https://` hoặc dấu `/`. Khi test nội bộ PASS, kiểm tra
lại URL public theo §13.

**4. Dùng bằng chứng của đúng request, không suy từ việc thiếu log**

Đối chiếu cùng thời điểm, Host, path và dấu nhận diện request khi có:

- **Access log Traefik nếu đã bật:** router/backend nào được chọn, status trả client
  và origin status nếu có. Log thông thường của Deployment không tự thay thế access log.
- **Access log app nếu đã bật:** đúng request/path với status 404 là bằng chứng app
  đã nhận request và tạo response đó.
- **Dashboard/configuration Traefik:** router của Ingress có thực sự tồn tại và
  khớp entrypoint/Host/path không?
- **Gọi trực tiếp Service web** từ môi trường có đường mạng tới Service, giữ cùng
  Host/path để so sánh với chặng qua Traefik.

Ví dụ chạy trong Pod/debug container có curl và dùng cluster DNS:

```bash
curl -i -H 'Host: app.hieupn.site' \
  http://web.default.svc.cluster.local:80/
```

Test này bỏ qua Traefik và tunnel; kết hợp log để kết luận nguồn 404. Nếu app trả
404 trực tiếp, điều tra routing/resource phía app. Nếu app trả đúng nội dung nhưng
qua Traefik trả 404, điều tra router/configuration/header/path thực tế ở Traefik.

Không thấy request trong log chưa chứng minh app không nhận: có thể access log chưa
bật, log ở nơi khác, đang xem nhầm replica hoặc ngoài khoảng thời gian cần kiểm tra.

**Đáp án câu bổ sung**

**Không thể kết luận.** Request có thể tới replica 2; kiểm tra access log của cả
hai Pod và đối chiếu đúng request.

```bash
kubectl -n default get pods -l app=web -o wide
kubectl -n default logs <pod-web-1> --tail=100
kubectl -n default logs <pod-web-2> --tail=100
```

Ngay cả khi cả hai không có dòng log tương ứng, cần xác nhận access logging đã bật,
đang xem đúng nơi ghi log và đúng khoảng thời gian. Đối chiếu access log/router
Traefik trước khi kết luận request chưa tới backend.

</details>

#### Câu 4/6 — Service và EndpointSlice không có backend Ready

Bạn quan sát:

- Tunnel Healthy.
- Ingress web có đúng Host, path và IngressClass.
- Service web tồn tại, selector là `app: web`.
- Hai Pod web đều Running nhưng mang label `app: demo`.
- EndpointSlice của Service web không có endpoint.

Hãy trả lời:

1. Vì sao Service tồn tại và Pod Running nhưng request vẫn không tới ứng dụng?
2. Lỗi cấu hình nằm ở đâu? Sửa thế nào và dùng lệnh gì để kiểm tra?
3. Nếu label/selector đã khớp nhưng Pod có condition `Ready=False`, Pod có được dùng
   làm backend phục vụ bình thường không? Giả sử Service không bật
   `publishNotReadyAddresses`.
4. EndpointSlice có endpoint Ready đã đủ chứng minh app trả đúng HTTP response chưa?
   Cần test thêm gì?

**Câu hỏi bổ sung:** nếu gọi trực tiếp Pod IP trả đúng app nhưng gọi ClusterIP
Service thất bại, ưu tiên kiểm tra những cấu hình hoặc thành phần nào?
Khi so sánh phải gọi cùng giao thức, Host/path từ cùng vị trí mạng, đồng thời dùng
đúng Pod port và Service port tương ứng.

<details>
<summary>Đáp án vòng 2 — câu 4/6: chỉ mở sau khi đã tự trả lời</summary>

**1. Selector không khớp nên không có backend được chọn**

Service chọn Pod có label `app: web`; hai Pod hiện có mang `app: demo` nên không
được chọn. Service/ClusterIP tồn tại không đồng nghĩa có endpoint để chuyển tiếp.
Pod Running không thay thế việc label khớp và readiness đạt.

Selector Service **chọn backend**, không quản lý vòng đời Pod. Không có backend
qua Service này không có nghĩa không thể gọi Pod IP trực tiếp.

**2. Sửa sự không khớp, không đặt lại selector giống giá trị đang có**

Với dữ kiện đề bài, đổi Service selector từ app:web sang app:demo là một cách sửa,
nếu app:demo là label đúng của ứng dụng:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: demo
  ports:
    - port: 80
      targetPort: 80
```

Sửa trong source manifest demo-app.yaml, apply rồi kiểm tra:

```bash
kubectl apply -f demo-app.yaml
kubectl -n default get pods --show-labels
kubectl -n default get svc web -o yaml
kubectl -n default get endpointslice \
  -l kubernetes.io/service-name=web -o yaml
```

Không sửa thành `selector: { app: web }` rồi coi là xong: đó chính là giá trị
hiện tại nên lỗi vẫn còn. `get svc web -o wide` chưa chứng minh Service có
endpoint Ready; cần xem địa chỉ, port và `conditions.ready` trong EndpointSlice.

Cách khác là thống nhất label Pod template về app:web. Khi chọn cách này, selector
Deployment phải khớp template; selector Deployment đã tạo thường không sửa trực
tiếp được. Nếu nó đã là app:demo, cần lập kế hoạch tạo/thay Deployment phù hợp,
không chỉ apply một selector mới rồi mong Kubernetes chấp nhận. Không đổi label
Pod thủ công làm giải pháp lâu dài vì Deployment tạo lại Pod theo template.

**3. Ready=False không phục vụ traffic thông thường qua Service**

Với giả định không bật publishNotReadyAddresses, endpoint chưa Ready không được
chọn cho traffic thông thường qua Service; Traefik cũng thông thường chỉ dùng
backend Ready do Kubernetes provider cung cấp. Endpoint có thể vẫn xuất hiện trong
EndpointSlice với `ready: false`: có địa chỉ không đồng nghĩa đủ điều kiện phục vụ.

Pod/container vẫn có thể chạy. Readiness không tự kill/restart container;
các connection đang có và truy cập trực tiếp Pod IP không được bảo đảm dừng tức thì.

**4. Ready chưa chứng minh response đúng**

Cần gọi Service từ môi trường có cluster DNS/network, quan sát status và body:

```bash
curl -i -H 'Host: app.hieupn.site' \
  http://web.default.svc.cluster.local:80/
```

Sau đó test qua Traefik từ master theo §12.3.1:

```bash
ING_IP=$(kubectl -n traefik get svc traefik \
  -o jsonpath='{.spec.clusterIP}')
curl -i -H 'Host: app.hieupn.site' "http://$ING_IP/"
```

Khi đường nội bộ PASS, kiểm tra public theo §13. Endpoint Ready phản ánh trạng thái
readiness, không kiểm tra mọi method/path và tính đúng của dữ liệu ứng dụng.

**Đáp án câu bổ sung**

Ưu tiên đường **Service → backend và Service dataplane**, nhưng trước hết bảo đảm
hai phép test tương đương: cùng nguồn mạng, Host/path, giao thức và đúng các port.

- Service selector có khớp Pod label không?
- EndpointSlice có đúng Pod IP, port và endpoint Ready không?
- Service port/targetPort, protocol và named port nếu dùng có đúng không?
- App có thực sự lắng nghe Pod port được Service chỉ định không?
- kube-proxy hoặc dataplane thay thế có hoạt động và lập rule Service đúng không?
- Routing/CNI, firewall hoặc NetworkPolicy có hiệu lực có chặn đường tương ứng không?

Lệnh đối chiếu trong lab:

```bash
kubectl -n default get svc web -o yaml
kubectl -n default get pods --show-labels -o wide
kubectl -n default get endpointslice \
  -l kubernetes.io/service-name=web -o yaml
kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide
```

Nếu gọi trực tiếp IP, phép test không phụ thuộc DNS. Không vội sửa Ingress hay
restart app khi đường Service đang lỗi. Kiểm tra Service/kube-proxy theo §8.5,
mạng Pod theo §8.3 và selector/manifest theo §10.1.

</details>

#### Câu 5/6 — Một replica chết

Lab có **hai Pod cloudflared và hai Pod web**. Xét riêng:

- **A:** một Pod cloudflared chết; connector còn lại vẫn Healthy.
- **B:** một Pod web chết; Pod web còn lại Ready và hoạt động bình thường.

1. Trong mỗi tình huống, request mới có thể tiếp tục được phục vụ không?
   Điều kiện là gì?
2. Request đang xử lý qua Pod vừa chết có chắc chắn hoàn tất không? Vì sao?
3. Connector replica 1 có gắn cố định với Pod web replica 1 không?
   Ai chọn connector và ai chọn backend web?
4. Hai replica có tự bảo đảm chịu được việc một worker chết không?
   Cần kiểm tra cách đặt Pod trên node và cấu hình gì thêm?

**Thông tin về lab:** với cài mới đúng §9.3, chart Traefik 41.0.2 có mặc định
`deployment.kind: Deployment`, `deployment.replicas: 1`; lệnh runbook không
override số replica. Vì vậy mặc định là **2 connector, 1 Traefik, 2 web**.
Cluster đã được chỉnh riêng hoặc nâng cấp release cần kiểm tra số replica thực tế.

**Câu hỏi bổ sung:**

1. Hai Pod web nằm trên hai worker khác nhau, nhưng Traefik Pod duy nhất nằm trên
   worker vừa chết. Connector còn Healthy và một Pod web còn Ready.
   App public có chắc tiếp tục hoạt động không? Vì sao phải xét cả Traefik?
2. Nếu Traefik có hai Pod thay vì một, câu trả lời thay đổi thế nào?
   So sánh khi chúng nằm trên hai worker khác nhau và cùng nằm trên worker chết.

<details>
<summary>Đáp án vòng 2 — câu 5/6: chỉ mở sau khi đã tự trả lời</summary>

**1. Request mới có thể tiếp tục, nếu toàn bộ đường đi còn hoạt động**

| Tình huống | Điều kiện tiếp tục phục vụ |
| --- | --- |
| A — mất một connector | Cloudflare chọn được connector còn Healthy; connector đó phân giải/gọi được origin; Traefik và backend còn phục vụ |
| B — mất một Pod web | Traefik/Service dataplane chọn được backend web còn Ready và gọi được app; connector và Traefik vẫn hoạt động |

Pod còn lại phải đủ năng lực chịu tải và các dependency còn hoạt động. Không có
ngưỡng bắt buộc tải phải thấp/trung bình: cần tải không vượt khả năng thực tế.
Phát hiện lỗi/cập nhật endpoint và routing mất thời gian, nên request mới vẫn có
thể gặp lỗi trong giai đoạn chuyển tiếp; không bảo đảm failover tức thì.

**2. Request đang xử lý không được bảo đảm hoàn tất**

Pod có thể chết đột ngột dù CPU/memory trước đó bình thường. Connection/stream đang
xử lý có thể đứt; trạng thái trong bộ nhớ của process mất. Replica còn lại không
tự tiếp quản nguyên trạng request.

Nếu client/proxy retry, đó là gửi lại request theo chính sách cụ thể, không phải
tiếp tục process đã chết. POST có thể đã thực hiện thao tác nhưng client chưa nhận
response; retry có thể gây thao tác trùng. Khi cần retry an toàn, ứng dụng phải
thiết kế tính idempotent hoặc cơ chế chống trùng phù hợp.

**3. Hai lần lựa chọn độc lập**

| Quyết định | Bên thực hiện |
| --- | --- |
| Connector của tunnel | Cloudflare chọn trong các connector khả dụng |
| Pod web khi Traefik gọi endpoint trực tiếp | Traefik chọn backend từ thông tin Kubernetes |
| Backend khi kết nối qua Service ClusterIP | Service dataplane, ví dụ rule do kube-proxy lập, lựa chọn/chuyển tiếp |

Kernel trên master không chọn connector. Master không phải điểm trung chuyển bắt
buộc của request. Khi sử dụng kube-proxy, nó lập rule; dataplane xử lý traffic
trên node liên quan, không phải mọi request đều đi qua master.

Connector 1 không ghép cố định với web 1. Điều này không phụ thuộc việc workload
là Deployment hay StatefulSet; StatefulSet không tự tạo quan hệ ghép connector/backend.

**4. Replica phải được phân tán và đủ khả năng phục vụ**

Nếu cả hai replica trên worker chết, mất cả hai. Kiểm tra node của các Pod:

```bash
kubectl -n cloudflare get pods -l app=cloudflared -o wide
kubectl -n traefik get deployment traefik
kubectl -n traefik get pods -o wide
kubectl -n default get pods -l app=web -o wide
```

Dùng `topologySpreadConstraints` theo topology node hoặc Pod anti-affinity phù hợp
để phân tán replica. Ràng buộc mềm là ưu tiên, không bảo đảm tách node; ràng buộc
cứng có thể khiến Pod Pending nếu không đủ node/tài nguyên phù hợp.

Cần xét cả connector, Traefik, web, DNS/network và các dependency khác; còn đủ
tài nguyên, endpoint và đường mạng trên node sống. Replica của một tầng không
bảo đảm HA cho toàn bộ chuỗi. PDB hỗ trợ giới hạn gián đoạn tự nguyện như drain,
không ngăn worker chết đột ngột.

DaemonSet dùng khi muốn Pod trên mỗi node đủ điều kiện. Không cần đổi web/connector
sang DaemonSet chỉ để phân tán hai replica. Tăng replicas cho Deployment và cấu
hình phân tán thường phù hợp hơn với mục tiêu đó.

**Đáp án bổ sung 1 — Traefik duy nhất chết**

**Không chắc tiếp tục hoạt động; nếu chưa có Traefik thay thế Ready, đường public
bị gián đoạn.** Connector và web còn sống nhưng thiếu process ở giữa:

| Chặng | Trạng thái |
| --- | --- |
| Edge → connector | Có thể còn hoạt động |
| Connector → Service Traefik | ClusterIP tồn tại nhưng không còn Traefik backend hoạt động |
| Traefik → web | Không thực hiện được vì Traefik duy nhất đã chết |

Service là object/địa chỉ ảo, không chạy thay chương trình Traefik. Ingress là
cấu hình, không tự proxy request. Request không tự bỏ qua Traefik để tới web
vì Service URL vẫn trỏ Service Traefik.

Kubernetes có thể tạo lại Traefik trên node khác nếu scheduler, tài nguyên và
các điều kiện liên quan cho phép. Cần chờ Pod mới Ready, có mạng và nạp routing;
trong lúc đó vẫn có thể gián đoạn. Đây là lý do phải xét cả Traefik khi đánh giá
chịu lỗi worker.

Mặc định chart được đối chiếu tại:
[Traefik chart v41.0.2 — values.yaml](https://github.com/traefik/traefik-helm-chart/blob/v41.0.2/traefik/values.yaml).
Không suy ra số Pod hiện tại chỉ từ mặc định nếu release đã được thay đổi;
dùng các lệnh ở trên để xác nhận cluster thực tế.

**Đáp án bổ sung 2 — Có hai Traefik Pod**

- **Hai node khác nhau, còn một Traefik Ready trên node sống:** request mới có thể
  tiếp tục nếu connector gọi được Service, endpoint Traefik còn lại được chọn,
  Traefik gọi được web Ready và phần còn lại đủ năng lực phục vụ.
- **Cả hai trên worker chết:** vẫn mất toàn bộ Traefik và gián đoạn.
- Trong cả hai trường hợp, request đang chạy qua Traefik vừa chết có thể thất bại;
  replica còn lại không tự tiếp quản nguyên trạng request đó.

**Hai replica chỉ giúp chịu lỗi node khi cách đặt Pod và toàn bộ đường đi phù hợp.**

</details>

#### Câu 6/6 — HTTPS, SNI, certificate và ranh giới TLS

Trong bài, người dùng mở `https://app.hieupn.site/`. Cloudflare Tunnel có
Published application route với Service URL
`http://traefik.traefik.svc.cluster.local:80`; Traefik chuyển tới Pod web bằng
HTTP. Hãy trả lời trước khi mở đáp án:

1. Vì sao browser dùng HTTPS trong khi `cloudflared` gọi origin bằng HTTP?
   Public DNS quyết định giao thức không? Đoạn nội bộ HTTP được bảo vệ tới mức
   nào?
2. TLS của browser kết thúc ở đâu? Liệt kê giao thức trên các chặng tiếp theo;
   phân biệt các kết nối/phiên mã hóa độc lập.
3. Chỉ đổi Service URL sang `https://traefik.traefik.svc.cluster.local:443`
   đã đủ chưa? Cần chuẩn bị gì ở Traefik, certificate và `cloudflared`? Có nhất
   thiết phải đổi giá trị Host/path của Ingress không?
4. SNI xuất hiện vào lúc nào, giải quyết việc gì? So sánh SNI với DNS, HTTP
   `Host`/`:authority` và tên trong certificate.
5. Nếu certificate origin do CA riêng cấp hoặc chỉ có tên public
   `app.hieupn.site` trong khi Service URL mang tên nội bộ, client TLS sẽ kiểm
   tra những gì? `originServerName`, `caPool` và `noTLSVerify` khác nhau ra sao?

<details>
<summary>Đáp án vòng 2 — câu 6/6: chỉ mở sau khi đã tự trả lời</summary>

**Nhận xét câu trả lời ban đầu**

- **Câu 1:** đúng là Service URL `http://...:80` khiến `cloudflared` gọi Traefik
  bằng HTTP. Cần sửa hai chỗ: browser dùng HTTPS vì URL có scheme `https`,
  **không phải vì DNS hay dashboard trả IP cho HTTPS**; mạng nội bộ không phải
  bằng chứng đoạn HTTP đã được mã hóa/xác thực. Mức cô lập mạng và bảo vệ TLS
  là hai cơ chế khác nhau.
- **Câu 2:** đúng cho bài này: TLS của browser kết thúc ở Cloudflare Edge và
  Traefik → Pod web dùng HTTP. Còn một kết nối tunnel được mã hóa riêng giữa
  `cloudflared` và Cloudflare; `cloudflared` → Traefik đang dùng HTTP riêng.
  Không có một phiên TLS liên tục từ browser đến Pod.
- **Câu 3:** đúng khi nói chỉ đổi Service URL là chưa đủ. Nhưng **không nhất
  thiết đổi Host/path rule** của Ingress: rule định tuyến `app.hieupn.site` + `/`
  có thể giữ nguyên. Traefik cần listener/entrypoint HTTPS, certificate và
  private key phù hợp, cấu hình TLS/router phù hợp; `cloudflared` phải tin
  certificate và kiểm tra đúng tên. Tùy cấu hình controller mà cần thêm TLS vào
  Ingress hoặc loại cấu hình router tương ứng.

**Sơ đồ nền tảng: bất kỳ ứng dụng HTTPS nào**

Sơ đồ bắt đầu từ lúc người dùng **nhập URL HTTPS vào browser**. URL chưa phải
HTTP request: browser phải tìm địa chỉ, tạo kết nối và hoàn tất TLS trước.
“Public endpoint” có thể là chính server ứng dụng hoặc một proxy phía trước.

```mermaid
flowchart LR
    subgraph S1["1. Browser"]
        direction TB
        U["Nhận URL HTTPS"] --> D["DNS tìm IP"]
    end
    subgraph S2["2. Kết nối"]
        direction TB
        N["TCP/QUIC tới IP:443"] --> T["TLS: SNI → certificate → xác minh → khóa phiên"]
    end
    subgraph S3["3. HTTP"]
        direction TB
        H["Gửi method, path, Host/:authority, body nếu có"]
    end
    subgraph S4["4. Chuyển tiếp"]
        direction TB
        P{"Public endpoint là proxy?"}
        Q["Chọn route, kết nối tới chặng tiếp theo"]
        P -->|Có| Q
    end
    subgraph S5["5. Ứng dụng"]
        direction TB
        A["Nhận và xử lý HTTP request"] --> O["Response quay về qua các chặng"]
    end
    D --> N
    T --> H
    H --> P
    P -->|Không| A
    Q --> A
```

Nếu có nhiều proxy, lặp lại bước **chọn route → kết nối chặng kế tiếp → gửi
HTTP**. Proxy có thể tái sử dụng kết nối sẵn có, thay vì mở một TCP/TLS
connection mới cho từng request. Sơ đồ không giả định ứng dụng chạy trong
Kubernetes hay đặt sau Cloudflare.

**Từng bước: DNS, transport, TLS, rồi mới HTTP**

1. Browser tách URL thành `https`, hostname `host`, cổng mặc định `443`, path
   `/path` và query `query`. DNS (hoặc cache DNS) cung cấp IP của điểm vào;
   nó không quyết định scheme, certificate hay backend HTTP. Tên public có thể
   đi qua CNAME/proxy trước khi ra IP cuối cùng.
2. Browser mở kết nối mạng tới IP/cổng đó. Với HTTPS trên TCP, hai bên làm
   TCP handshake rồi bắt tay TLS. Với HTTP/3, QUIC chạy trên UDP và tích hợp
   thiết lập TLS; không mô tả HTTP/3 bằng TCP handshake.
3. Browser gửi TLS `ClientHello`: phiên bản/thuật toán hỗ trợ, dữ liệu trao đổi
   khóa, **SNI** cho hostname mà nó muốn gặp, và thường có **ALPN** để thương
   lượng giao thức HTTP. SNI giúp đầu nhận dùng chung IP/cổng chọn ngữ cảnh
   TLS và certificate **trước khi nó nhận HTTP request**. SNI là tên do client
   khai, không tự chứng minh server đáng tin hay cấp quyền truy cập.
4. Server gửi thông tin bắt tay và certificate. Browser kiểm tra certificate
   còn hợp lệ, chuỗi chứng nhận dẫn tới CA được tin cậy, và tên trong SAN của
   certificate khớp hostname trong URL. Bước bắt tay cũng xác minh server nắm
   private key tương ứng và tạo khóa đối xứng cho phiên. Certificate hỗ trợ
   xác thực danh tính; nó không phải khóa dùng trực tiếp để mã hóa mọi HTTP
   request. Nếu kiểm tra bắt buộc thất bại, browser không tiếp tục request
   một cách bình thường.
5. Khi TLS hoàn tất, browser gửi HTTP request được bảo vệ trong phiên TLS,
   ví dụ `GET /path?query`. Với HTTP/1.1 có `Host: host`; HTTP/2/3 dùng
   `:authority` mang tên đích tương ứng. POST có thể mang body. Server hoặc
   proxy giải mã HTTP, áp dụng route theo hostname/path và các rule liên quan.
6. Nếu điểm vào là proxy, proxy có thể kết thúc phiên TLS này và tạo một HTTP
   request cho chặng sau. HTTPS ở chặng sau đòi hỏi **một phiên TLS khác**:
   proxy lúc này là TLS client và phải xác minh server của chặng ấy. HTTP ở
   chặng sau không có TLS bảo vệ cho đoạn truyền đó. Cuối cùng ứng dụng nhận
   method, path/query, headers và body theo cách proxy chuyển tiếp; ứng dụng
   tạo response, mỗi proxy trả response trên kết nối tương ứng về browser.

Với TLS 1.3, các thông điệp như ServerHello, Certificate, CertificateVerify
và Finished có trình tự và mức mã hóa riêng trong handshake; sơ đồ trên rút
gọn chúng thành bước xác minh và tạo khóa. Session resumption/0-RTT cũng có
thể làm trình tự thực tế khác đi; phần giải thích dùng full handshake thông
thường.

| Thông tin | Có từ lúc nào? | Trả lời câu hỏi gì? |
| --- | --- | --- |
| DNS/IP | Trước khi kết nối | Mở kết nối mạng tới đâu? |
| SNI trong TLS ClientHello | Trước HTTP | Server cần chọn cấu hình TLS/certificate cho tên nào? |
| Certificate và SAN | Trong TLS handshake | Client có xác minh đúng server cho tên kỳ vọng không? |
| HTTP `Host` hoặc `:authority` | Sau khi lập TLS, nằm trong HTTP | HTTP request hướng tới hostname nào để chọn route? |
| HTTP path/query | Trong HTTP | Request nhắm tới tài nguyên/hành động nào? |

SNI **không thay thế** `Host`/`:authority`: TLS server cần chọn certificate
trước khi xử lý HTTP; HTTP router dựa trên request đã được nhận. Nếu dùng
Encrypted ClientHello (ECH), một phần ClientHello/SNI có thể được che giấu;
mô tả ở trên là trường hợp SNI thông thường.

**Áp dụng cho chuỗi Cloudflare Tunnel của bài**

| Chặng | Kết nối và giao thức | Vai trò, TLS |
| --- | --- | --- |
| Browser → Cloudflare Edge | Browser tìm Edge IP, kết nối cổng HTTPS, gửi SNI `app.hieupn.site`, xác minh certificate rồi gửi HTTP | TLS phiên 1 kết thúc tại Edge; Edge đọc hostname/path và tra route |
| `cloudflared` ↔ Cloudflare | Connector chủ động duy trì kết nối tunnel outbound được mã hóa; Edge chuyển request qua kết nối thích hợp | Phiên/kết nối bảo vệ riêng, không dùng khóa TLS của browser |
| `cloudflared` → Traefik | Connector phân giải Service URL qua DNS nội bộ, kết nối ClusterIP của Service Traefik port 80, gửi HTTP với Host public nếu không override | Hiện tại không có TLS, vì Service URL là `http://...:80` |
| Traefik → ứng dụng | Traefik khớp Ingress/router rồi gọi backend phù hợp | HTTP trong demo; ứng dụng nhận request và tạo response |

“Tunnel đã mở outbound” nghĩa là kết nối hai chiều được duy trì để nhận và
chuyển dữ liệu; request người dùng không phải **response HTTP** của một lệnh
outbound ban đầu. Request public tới Edge, được proxy vào tunnel; connector
lại tạo/chọn kết nối tới origin. Không cần mở inbound tới IP của lab.

**Đổi riêng chặng connector → origin sang HTTPS**

Với Service URL `https://traefik.traefik.svc.cluster.local:443`, connector sẽ
là **TLS client** và Traefik sẽ là **TLS server**. Trước khi chạy được, cần:

- Service/cổng/entrypoint Traefik thực sự phục vụ TLS; Traefik có certificate,
  private key và router/Ingress TLS phù hợp. Host/path rule có thể giữ nguyên.
- Xác định tên mà connector kỳ vọng ở certificate. Nếu certificate chỉ hợp lệ
  cho `app.hieupn.site` nhưng URL kết nối là tên `*.svc.cluster.local`, cần
  cấu hình phù hợp, chẳng hạn `originServerName: app.hieupn.site`, để kiểm tra
  đúng danh tính; đối chiếu thêm SNI nếu có cấu hình `matchSNItoHost`.
- Nếu dùng CA nội bộ, cung cấp CA cho connector bằng `caPool` và bảo đảm file
  CA thực sự có trong môi trường chạy của `cloudflared`. HTTP `Host`/`:authority`
  phục vụ routing sau TLS; `httpHostHeader` chỉ cần cấu hình khi muốn đặt lại
  hostname HTTP mà connector gửi tới origin.

`noTLSVerify: true` làm connector bỏ qua xác minh certificate origin: dữ liệu
vẫn được mã hóa bằng TLS, nhưng connector không còn xác thực chắc chắn danh
tính origin. Nó không sửa lỗi TCP sai port, listener không nói TLS hay HTTP
router sai. Nó cũng không thay đổi việc browser xác minh certificate của Edge.

Phân biệt khi chẩn đoán: **không nối được cổng** là lỗi network/listener;
**TLS handshake/certificate fail** là lỗi cấu hình TLS, tên hoặc CA;
**TLS thành công nhưng 404** thường cần kiểm tra HTTP `Host`, path và router.

Đọc thêm tài liệu gốc: [RFC 6066 — SNI](https://www.rfc-editor.org/rfc/rfc6066.html),
[RFC 8446 — TLS 1.3](https://www.rfc-editor.org/rfc/rfc8446.html),
[RFC 9525 — xác minh tên server](https://www.rfc-editor.org/rfc/rfc9525.html),
[RFC 9110 — HTTP semantics](https://www.rfc-editor.org/rfc/rfc9110.html),
[Cloudflare Tunnel origin parameters](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/configure-tunnels/origin-parameters/).

</details>

Câu nào chưa trả lời được thì quay lại §9–§13 và phần giải thích tương ứng trước
khi học Rancher ở §14 hoặc chuyển sang các tình huống xử lý sự cố nâng cao.

---
