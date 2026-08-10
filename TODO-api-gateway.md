# TODO — Gateway API và API Gateway trên cụm homelab

> Ghi lại từ phần thảo luận quanh bullet cảnh báo `networkExposure.type` ở
> [§14.3 của runbook](runbook-k8s-vmware.md#143-cài-rancher-helm-pin-2143). Chưa có việc nào được
> thực hiện; file này là quyết định kiến trúc + điều kiện kiến thức, không phải runbook thao tác.
>
> Ngày ghi: 10/08/2026. Baseline đối chiếu: Kubernetes `v1.35.6`, Traefik chart `41.0.2`,
> Rancher `2.14.3`.

---

## 1. Hai khái niệm trùng tên, phải tách bạch trước

| | **Gateway API** | **API Gateway** |
| --- | --- | --- |
| Là gì | Bộ CRD `gateway.networking.k8s.io` của SIG-Network, kế nhiệm Ingress | Hạng mục sản phẩm: Kong, Tyk, Gloo, Apigee, KrakenD |
| Trả lời câu hỏi | Request HTTP này đi tới Service nào | Ai được gọi API này, bao nhiêu lần/giây, key/JWT có hợp lệ không, biến đổi request ra sao |
| Ai cài | Người vận hành apply CRD, rồi một controller phục vụ | Cài như một workload/proxy trong cluster |
| Có sẵn trong Kubernetes | **Không** — là add-on | Không |

Nhiều sản phẩm nằm ở **cả hai ô**: vừa là API Gateway, vừa implement Gateway API. Danh sách
implementation của SIG-Network hiện có **16+ sản phẩm** thuộc nhóm gateway/ingress — Envoy
Gateway, Gloo Gateway, kgateway, Kong Operator, NGINX Gateway Fabric, WSO2 Gateway, HAProxy,
Cilium, Airlock Microgateway, agentgateway, AWS Load Balancer Controller, GKE, **và Traefik**.
Đây là nguồn gốc của gần như mọi nhầm lẫn về chủ đề này.

> Kong xuất hiện trong file này chỉ với vai trò **ví dụ cụ thể** để nói về cơ chế. Mọi lập luận
> bên dưới áp dụng cho bất kỳ API gateway nào đi theo đường Gateway API.

---

## 2. Vì sao runbook chốt Ingress + Traefik cho Rancher

Chart Rancher `2.14.3` có mode selector `networkExposure.type` với ba giá trị `ingress` |
`gateway` | `none`:

```
{{- define "rancher.ingressEnabled" -}}
{{- if and (eq .Values.networkExposure.type "ingress") .Values.ingress.enabled -}}

{{- define "rancher.gatewayEnabled" -}}
{{- if eq .Values.networkExposure.type "gateway" -}}
```

Ở mode `gateway`, chart render `gateway.yaml`, `httproute.yaml`, `httproute-redirect.yaml`,
`gateway-cert.yaml` thay cho `ingress.yaml`, và tham chiếu `gateway.gatewayClass.name` mặc định
là `traefik`.

Cái bẫy: giá trị mặc định đó **chỉ là một chuỗi ký tự**. Nó không cài Gateway API CRD, không tạo
GatewayClass, không kiểm tra có controller nào phục vụ hay không. Cụm hiện tại không có thứ nào
trong ba thứ đó — [§9](runbook-k8s-vmware.md#93-cài-đặt-traefik) chỉ bật
`providers.kubernetesIngress` và `providers.kubernetesCRD` (CRD **riêng của Traefik**:
`IngressRoute`, `Middleware`), không bật `providers.kubernetesGateway` và không cài Gateway API
CRD.

---

## 3. Điều gì đổi khi cài một API gateway theo đường Gateway API

Gateway API là add-on: **CRD phải được apply trước khi controller khởi động**, bằng một lệnh
riêng (`standard-install.yaml` từ repo `kubernetes-sigs/gateway-api`). Điều này đúng với mọi
implementation, không riêng sản phẩm nào. Nghĩa là từ thời điểm cài bất kỳ API gateway nào theo
hướng Gateway API, cụm **có** CRD Gateway API — và kiểu hỏng của việc đổi Rancher sang mode
`gateway` đổi bản chất:

| | Trước khi có API gateway | Sau khi CRD Gateway API vào cụm |
| --- | --- | --- |
| `helm install rancher` mode `gateway` | Abort ngay: `no matches for kind "Gateway"` | **Thành công**, `helm status` báo `deployed` |
| Hậu quả | Cụm chưa bị đụng, biết ngay là sai | `Gateway` kẹt `Accepted=False`, Rancher không có route, Pod vẫn `Running` |

Lý do hỏng im lặng: mỗi implementation đăng ký GatewayClass **mang tên của nó**, không phải tên
`traefik` mà `gateway.gatewayClass.name` của chart Rancher đặt mặc định. Ví dụ với Kong,
GatewayClass tên `kong`, `spec.controllerName` là `konghq.com/kic-gateway-controller`, còn cần
annotation `konghq.com/gatewayclass-unmanaged=true`. Trừ khi có một GatewayClass đúng tên
`traefik` và đang `Accepted`, mode `gateway` của Rancher không có gì phục vụ nó.

> **Không gỡ gate render ở §14.3** khi cụm "đã có Gateway API rồi". Dòng
> `grep -Eq '^kind: (Gateway|HTTPRoute)$'` là thứ duy nhất còn bắt được tình huống trên, vì Helm
> lúc đó không còn báo lỗi nữa.

Ghi chú phụ: ở mode `ingress`, Certificate do `ingress-shim` của cert-manager sinh ra từ
annotation trên Ingress; ở mode `gateway`, chart render thẳng `gateway-cert.yaml`. Tên và cơ chế
khác nhau, nên gate §14.4 cũng sai theo nếu đổi mode.

---

## 4. Runbook đã chuẩn bị sẵn gì cho một ingress controller thứ hai

Chính sách ở [§9.1.7](runbook-k8s-vmware.md#917-provider-của-traefik-và-default-ingressclass) —
*khai tường minh `ingressClassName: traefik` trong mọi Ingress dù đã có default* — được thực thi
đủ ở Ingress mẫu §9.1.5, Ingress app §10 và values Rancher §14.3. Một ingress controller thứ hai
vào cụm sẽ **không** cướp Ingress nào của Traefik, vì nó chỉ nhận object khai đúng class của nó.

Điểm duy nhất phải canh: Traefik đang được cài với `ingressClass.isDefaultClass=true`. **Không để
controller mới cũng đặt class của nó làm default** — từ hai default trở lên, Kubernetes từ chối
tạo Ingress không khai class. Nhiều chart ingress controller mặc định bật cờ này, phải kiểm
values trước khi cài.

---

## 5. Ba kiến trúc, và lựa chọn

**(0) Không thêm sản phẩm nào — dùng chính Traefik.** Traefik đã nằm trong danh sách
implementation Gateway API, nên nếu chỉ cần *routing kiểu Gateway API* thì chỉ việc apply CRD và
bật `providers.kubernetesGateway`. Quan trọng hơn: một phần đáng kể nhu cầu "API gateway" —
rate limit, forward-auth, thao tác header, retry, circuit breaker — Traefik đã có qua CRD
`Middleware`, mà `providers.kubernetesCRD` ở §9 **đã bật sẵn**. **Đánh giá phương án này trước
tiên**; chỉ loại nó khi đã chỉ ra được tính năng cụ thể mà Traefik không làm.

**(a) Một API gateway thay Traefik hoàn toàn.** Một cửa vào duy nhất. Phải sửa §9, đổi Service
URL của Published application ở §12.3.3, đổi `ingressClassName` của Rancher ở §14.3, làm lại
snapshot và toàn bộ gate. Kéo cả mặt phẳng quản trị vào một thành phần đang thí nghiệm.

**(b) API gateway đứng cạnh Traefik, chỉ phục vụ API sản phẩm.** ✅ **Chọn phương án này nếu (0)
không đủ.** Hostname riêng (vd `api.hieupn.site`), thêm **một Published application route thứ hai
trong cùng tunnel** trỏ tới Service của gateway đó. Traefik giữ nguyên Rancher và app hạ tầng.
Đúng pattern mà [§15 "Thêm app/domain mới sau này"](runbook-k8s-vmware.md#thêm-appdomain-mới-sau-này)
đã mô tả, và không đụng gì tới §9/§14.

Nguyên tắc chi phối: **không để Rancher — mặt phẳng quản trị — phụ thuộc vào thứ đang thử
nghiệm.** Gateway mới hỏng thì vẫn còn UI Rancher và Traefik để chẩn đoán.

### 5.1. Tiêu chí chọn sản phẩm khi thật sự cần (a) hoặc (b)

Vì phần API-gateway-đặc-thù **chưa được chuẩn hóa** (xem 6.3), tiêu chí quan trọng nhất là: sản
phẩm đó đẩy được bao nhiêu cấu hình vào Gateway API chuẩn, và bao nhiêu phải viết bằng CRD riêng
của nó. Càng nhiều thứ nằm trong CRD riêng thì càng khóa vendor và càng khó thay về sau.

Nhu cầu tối thiểu phải viết ra trước khi so sánh sản phẩm: rate limit theo ai (IP, API key,
consumer), xác thực kiểu gì (API key, JWT, OIDC, mTLS), có cần quota/monetization không, có cần
developer portal không.

---

## 6. Đọc tới đâu trong lộ trình thì hiểu được và tự làm được

Đối chiếu với [`k8s-docs/LO-TRINH-ADMIN.md`](k8s-docs/LO-TRINH-ADMIN.md).

### 6.1. Để **hiểu** toàn bộ file này

Đủ khi đọc hết nhóm Ingress của
[Giai đoạn 5 — Mạng nền tảng](k8s-docs/LO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng), theo đúng
thứ tự ba bài:

| Thứ tự | Bài | Cho biết điều gì trong file này |
| --- | --- | --- |
| bài 10/16 | [Ingress](k8s-docs/11-ingress-vi.md) | rule, pathType, **IngressClass**, TLS |
| bài 11/16 | [Ingress Controllers](k8s-docs/12-ingress-controllers-vi.md) | không có controller thì Ingress vô nghĩa; mục *"Các ingress controller của bên thứ ba"* liệt kê **29 sản phẩm** dưới dạng link trần — minh hoạ đúng chính sách biên tập nói ở 6.3 |
| **bài 12/16** | [**Gateway API**](k8s-docs/13-gateway-vi.md) | **mốc chốt** — sau bài này là hiểu được cả mục 2 và 3 ở trên |

Bài `13-gateway-vi.md` nói thẳng hai điều là nền của toàn bộ cảnh báo §14.3:

- Gateway API là **add-on** gồm CustomResourceDefinition, *"không cài CRD và một bản hiện thực
  thì không có gì chạy"*;
- **GatewayClass** là nơi *"khai controller nào quản class đó"* — chính là lý do chuỗi
  `gatewayClass.name: traefik` vô nghĩa khi không có class nào tên như vậy.

→ **Kết luận: đọc xong [`13-gateway-vi.md`](k8s-docs/13-gateway-vi.md) ở Giai đoạn 5 là đủ để
hiểu file này.**

### 6.2. Để **tự làm** thì còn thiếu gì

| Cần cho việc gì | Nằm ở đâu trong lộ trình | Trạng thái |
| --- | --- | --- |
| Dựng/gỡ một ingress controller thật, có snapshot để rollback | 🧪 **Lab 5b — NetworkPolicy, Ingress và CNI**, [bản đồ lab](k8s-docs/labs/README.md#4-bản-đồ-lab) | ⬜ **chưa viết** — và lab này cài ingress controller, **không** cài Gateway |
| Hiểu vì sao "CRD tồn tại nhưng không controller" = hỏng im lặng | [Giai đoạn 14](k8s-docs/LO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng) → [Tài nguyên tùy chỉnh](k8s-docs/179-custom-resources-vi.md) | bài đã có; checkpoint giai đoạn 14 hỏi đúng ý này |
| RBAC cho controller mới — mọi Gateway API controller cần quyền watch Gateway/HTTPRoute và cập nhật status | [Giai đoạn 9](k8s-docs/LO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy) → [Kiểm soát truy cập](k8s-docs/119-controlling-access-vi.md), [RBAC good practices](k8s-docs/120-rbac-good-practices-vi.md) | bài đã có; 🧪 Lab 9a ⬜ chưa viết |
| Hiểu admission controller tự điền `ingressClassName` từ default class | Giai đoạn 5 (§9.1.7 của runbook giải thích) + Giai đoạn 9 (chặng admission) | bài đã có |

### 6.3. Vì sao lộ trình không có mục "API Gateway" — và ranh giới thật nằm ở đâu

Đây **không** phải thiếu sót của bản dịch. kubernetes.io chỉ tài liệu hóa API và điểm mở rộng của
chính dự án Kubernetes; với mọi hạng mục do bên thứ ba hiện thực, nó làm đúng hai việc: định
nghĩa **contract**, rồi **liệt kê** các bản hiện thực kèm link ra ngoài. Pattern này lặp lại
nguyên xi trong repo:

| Hạng mục | Bài định nghĩa contract | Cách xử lý sản phẩm |
| --- | --- | --- |
| Ingress | [11-ingress-vi.md](k8s-docs/11-ingress-vi.md) | [12-ingress-controllers-vi.md](k8s-docs/12-ingress-controllers-vi.md) có mục *"Các ingress controller của bên thứ ba"*: **29 dòng link trần**, không dòng nào hướng dẫn |
| CNI | [183-network-plugins-vi.md](k8s-docs/183-network-plugins-vi.md) | tài liệu hóa **yêu cầu** plugin phải đạt (loopback CNI, hostPort, traffic shaping); không nhắc tên plugin cụ thể |
| Gateway API | [13-gateway-vi.md](k8s-docs/13-gateway-vi.md) | danh sách implementation nằm ở `gateway-api.sigs.k8s.io` |

Bằng chứng sắc nhất nằm ngay trong bài Ingress Controllers: dự án Kubernetes tự nhận **chỉ bảo
trì đúng hai** ingress controller (AWS và GCE), rồi liệt kê **29 sản phẩm bên thứ ba** dưới dạng
link trần. Nếu kubernetes.io không viết hướng dẫn cho 29 ingress controller thì cũng sẽ không
viết hướng dẫn cho các API gateway.

**Vì vậy `13-gateway-vi.md` chính là bài về API Gateway ở tầng Kubernetes** — nó dạy đúng cái
contract mà các sản phẩm API gateway đang hội tụ về. Phần lớn trong 16+ implementation của SIG
tự mô tả bằng cụm "next-generation API gateway".

Ranh giới thật nằm ở chỗ contract đã chuẩn hóa tới đâu:

| Tầng | Trạng thái | Học ở đâu |
| --- | --- | --- |
| Routing L7, tách vai trò, TLS listener | **Stable v1**: GatewayClass, Gateway, HTTPRoute, GRPCRoute | [13-gateway-vi.md](k8s-docs/13-gateway-vi.md) |
| Gắn policy vào route/backend | **EXPERIMENTAL** — Policy Attachment (GEP-713); project mới định nghĩa `BackendTLSPolicy`, `BackendTrafficPolicy` | chưa vào lộ trình |
| Rate limit, API key, JWT/OIDC, quota, transformation, developer portal, analytics, monetization | **Chưa chuẩn hóa** — mỗi vendor một bộ CRD riêng (`KongPlugin`, Envoy Gateway `SecurityPolicy`, Traefik `Middleware`…) | tài liệu sản phẩm |

Nghĩa là: **phần đặc thù của API gateway chưa có chuẩn để tài liệu hóa một cách trung lập.**
Chừng nào Policy Attachment còn experimental và mỗi vendor còn CRD riêng, kubernetes.io không thể
viết một trang "cách rate-limit API" mà không thiên vị ai.

Hệ quả cho việc học: phần **khả chuyển** (Gateway API + CRD) học ở Giai đoạn 5 và 14, đủ để cài
và vận hành **bất kỳ** API gateway nào ở tầng Kubernetes. Phần **khóa vendor** đọc tài liệu sản
phẩm — và chính tỉ lệ giữa hai phần đó là tiêu chí chọn sản phẩm ở mục 5.1.

---

## 7. Checklist khi tới lúc làm

Điều kiện tiên quyết về kiến thức:

- [ ] Đọc xong [Ingress](k8s-docs/11-ingress-vi.md) → [Ingress Controllers](k8s-docs/12-ingress-controllers-vi.md) → [Gateway API](k8s-docs/13-gateway-vi.md)
- [ ] Đọc xong [Tài nguyên tùy chỉnh (CRD)](k8s-docs/179-custom-resources-vi.md) ở Giai đoạn 14
- [ ] Lab 5b đã được viết và đã làm xong (hoặc chấp nhận làm không có lab, ghi rõ rủi ro)

Quyết định trước khi chọn sản phẩm:

- [ ] Viết ra nhu cầu cụ thể: rate limit theo ai, xác thực kiểu gì, có cần quota/portal không
- [ ] Đối chiếu nhu cầu đó với `Middleware` của Traefik — **chốt phương án (0) hay không** trước khi đánh giá sản phẩm mới
- [ ] Nếu cần sản phẩm mới: so tỉ lệ cấu hình nằm trong Gateway API chuẩn vs CRD riêng của vendor (mục 5.1)

Trước khi đụng cụm:

- [ ] Snapshot etcd + `/etc/kubernetes` theo [§15](runbook-k8s-vmware.md#backup-etcd-và-cấu-hình-trước-thay-đổi-lớn), copy ra ngoài VM
- [ ] Ghi lại `kubectl get ingressclass -o yaml` hiện tại để biết class nào đang là default
- [ ] Xác nhận không có Gateway API CRD nào đang tồn tại: `kubectl get crd | grep gateway.networking.k8s.io`

Khi cài theo phương án (b):

- [ ] Kiểm phiên bản Gateway API CRD tương thích với controller sẽ dùng **trước** khi apply
- [ ] Đặt gateway ở namespace riêng, class riêng, **không** đặt làm default IngressClass
- [ ] Hostname riêng `api.hieupn.site`, thêm Published application route thứ hai trong cùng tunnel `homelab-k8s`
- [ ] Gate: Rancher vẫn vào được qua `rancher.hieupn.site`, app cũ vẫn vào được qua `app.hieupn.site`, cả hai vẫn đi qua Traefik
- [ ] Gate: `kubectl get ingressclass` chỉ có **một** class mang annotation default
- [ ] Gate: `kubectl get gatewayclass` — mọi class phải `Accepted=True`; không để class mồ côi

Sau khi cài xong:

- [ ] Cập nhật bảng phiên bản §2 của runbook nếu gateway mới trở thành thành phần thường trực
- [ ] Ghi vào [§15 troubleshooting](runbook-k8s-vmware.md#15-vận-hành--troubleshooting) triệu chứng phân biệt lỗi Traefik và lỗi gateway mới

---

## 8. Nguồn

- Chart Rancher `v2.14.3`: [`chart/values.yaml`](https://raw.githubusercontent.com/rancher/rancher/v2.14.3/chart/values.yaml), [`chart/templates/ingate/_helpers.tpl`](https://raw.githubusercontent.com/rancher/rancher/v2.14.3/chart/templates/ingate/_helpers.tpl)
- Gateway API — [trang chủ](https://gateway-api.sigs.k8s.io/)
- Gateway API — [danh sách implementation](https://gateway-api.sigs.k8s.io/implementations/) (nguồn của con số 16+ và của việc Traefik nằm trong danh sách)
- Gateway API — [Policy Attachment / GEP-713](https://gateway-api.sigs.k8s.io/reference/policy-attachment/) (nguồn của trạng thái EXPERIMENTAL)
- Kong (ví dụ cụ thể) — [Gateway API](https://developer.konghq.com/kubernetes-ingress-controller/gateway-api/), [Understanding IngressClass / GatewayClass](https://developer.konghq.com/kubernetes-ingress-controller/class-annotations/)
