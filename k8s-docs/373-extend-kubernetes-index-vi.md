# Mở rộng Kubernetes (Extend Kubernetes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/extend-kubernetes/>
>
> Tìm hiểu những cách nâng cao để điều chỉnh cluster Kubernetes của bạn cho phù hợp với nhu cầu
> của môi trường làm việc.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 28 — Mở rộng Kubernetes](00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes),
bài 1/11 · Phần II không có lab riêng: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) và tự chấm bằng **Checkpoint** ghi ở cuối mục giai đoạn
28. Trang này là bản đồ, không có gì để chạy.

**Bạn đã đi được một nửa nhóm này rồi.** Giai đoạn 28 là nhóm thực hành của
[giai đoạn 14](00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng), mà ở giai đoạn 14 bạn đã làm
[Lab 14 — CRD và Operator](labs/LAB-14-CRD-VA-OPERATOR.md): tạo `CustomResourceDefinition`, apply
custom resource, siết schema, mở subresource `status`, rồi tự tay đóng vai controller. Bạn cũng đã
**đọc** một extension API server đang chạy thật trên cluster lab. Cái Lab 14 **không** làm được là
tự **dựng** một API server thứ hai — và đó chính là hai bài kế tiếp của giai đoạn này. Đọc trang mục
lục này để biết bảy trang con chia việc ra sao, rồi đi theo thứ tự lộ trình.

**Phải hiểu ở lần đọc này:**

- Đây là **trang mục lục** của phần *Tasks → Extend Kubernetes*: nội dung chỉ gồm danh sách bảy
  trang con, xếp theo đúng thứ tự kubernetes.io hiển thị. Không có thao tác nào ở đây.
- Bốn nhóm việc mà đoạn dẫn gom lại: **thêm API mới vào cluster** (aggregation layer và extension
  API server), **định nghĩa kiểu tài nguyên riêng** bằng custom resource, **chạy thêm scheduler**
  bên cạnh scheduler mặc định, và **truy cập Kubernetes API** qua proxy hoặc qua dịch vụ
  Konnectivity.
- Bảy trang ở đây **không trùng** với danh sách 11 bài của giai đoạn 28 trong lộ trình: hai bài
  [378](378-custom-resource-definitions-vi.md) và
  [377](377-custom-resource-definition-versioning-vi.md) là trang con của
  [376](376-custom-resources-index-vi.md) nên không hiện ở cấp này, còn
  [372](372-kubectl-plugins-vi.md) thuộc nhóm `tasks/extend-kubectl/` — một nhóm khác hẳn.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Hai link về extension API server trong danh sách | đây mới là trang bản đồ; nội dung nằm ở chính hai trang đó, và chúng là phần Lab 14 chưa làm được | bài [374](374-configure-aggregation-layer-vi.md) ngay sau trang này, rồi bài [380](380-setup-extension-api-server-vi.md), cùng thuộc giai đoạn 28 |
| Link *Cấu hình nhiều Scheduler* | là nhánh riêng về lập lịch, không liên quan tới việc thêm API | bài [375](375-configure-multiple-schedulers-vi.md) của chính giai đoạn 28 |
| Ba link về đường truy cập API: HTTP proxy, SOCKS5 proxy, dịch vụ Konnectivity | không phải "mở rộng" theo nghĩa thêm kiểu tài nguyên hay thêm API server, mà là **đường mạng tới API**; lộ trình xếp chúng xuống cuối giai đoạn | bài [379](379-http-proxy-access-api-vi.md), [382](382-socks5-proxy-access-api-vi.md) và [381](381-setup-konnectivity-vi.md) của chính giai đoạn 28 |

---

Trang gốc là trang mục lục của phần *Tasks → Extend Kubernetes*: nội dung trang chỉ gồm danh sách
các trang con, được liệt kê dưới đây theo đúng thứ tự hiển thị trên trang web. Các trang này hướng
dẫn cách mở rộng Kubernetes bằng các cơ chế bổ sung: thêm API mới vào cluster thông qua
aggregation layer và extension API server, định nghĩa kiểu tài nguyên riêng bằng custom resource,
chạy thêm scheduler bên cạnh scheduler mặc định, cũng như truy cập Kubernetes API thông qua proxy
hoặc thông qua dịch vụ Konnectivity.

## Danh sách các trang trong mục này (Pages in this section)

- [Cấu hình tầng tổng hợp (Configure the Aggregation Layer)](374-configure-aggregation-layer-vi.md)
- [Sử dụng Custom Resource (Use Custom Resources)](376-custom-resources-index-vi.md)
- [Thiết lập một Extension API Server (Set up an Extension API Server)](380-setup-extension-api-server-vi.md)
- [Cấu hình nhiều Scheduler (Configure Multiple Schedulers)](375-configure-multiple-schedulers-vi.md)
- [Dùng HTTP Proxy để truy cập Kubernetes API (Use an HTTP Proxy to Access the Kubernetes API)](379-http-proxy-access-api-vi.md)
- [Dùng SOCKS5 Proxy để truy cập Kubernetes API (Use a SOCKS5 Proxy to Access the Kubernetes API)](382-socks5-proxy-access-api-vi.md)
- [Thiết lập dịch vụ Konnectivity (Set up Konnectivity service)](381-setup-konnectivity-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 28:

1. Đoạn dẫn của trang gom bảy trang con thành mấy nhóm việc? Kể tên từng nhóm và nói mỗi nhóm mở
   rộng đúng cái gì của Kubernetes.
2. **Câu bẫy.** Ở Lab 14 bạn đã tạo CRD, apply custom resource và tự chạy một vòng lặp điều khiển.
   Vậy giai đoạn 28 chỉ là làm lại những thứ đó cho kỹ hơn?
3. Cluster lab ba node của bạn (`lab-k8s-master`, `lab-k8s-worker1`, `lab-k8s-worker2`) đã có sẵn
   **một** extension API server đang chạy — bạn đã đọc nó ở Lab 14. Đó là thành phần nào, và trong
   bảy trang con liệt kê ở đây, hai trang nào dạy bạn tự dựng một cái như vậy?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Bốn nhóm.** Một: **thêm API mới vào cluster**, qua aggregation layer và extension API server —
   mở rộng chính bề mặt API. Hai: **định nghĩa kiểu tài nguyên riêng** bằng custom resource — thêm
   kind mới mà không cần server thứ hai. Ba: **chạy thêm scheduler** bên cạnh scheduler mặc định —
   mở rộng phần quyết định Pod chạy ở đâu. Bốn: **truy cập Kubernetes API** qua proxy hoặc qua dịch
   vụ Konnectivity — không thêm gì vào API, chỉ đổi đường đi tới nó.
2. **Không.** Phần lớn giai đoạn 28 là thứ Lab 14 **cố ý không làm**. Rõ nhất là nhánh extension API
   server: Lab 14 ghi thẳng lý do trong bảng "không kiểm chứng được" — dựng nó **phải viết và build
   binary cùng image, rồi cấu hình cụm cờ `--requestheader-*` cho kube-apiserver**, nên lab đẩy sang
   đúng hai bài [374](374-configure-aggregation-layer-vi.md) và
   [380](380-setup-extension-api-server-vi.md) của giai đoạn 28. Ngoài ra
   [378](378-custom-resource-definitions-vi.md) mới là trang xương sống về cú pháp CRD, còn
   scheduler thứ hai và các đường truy cập API thì Lab 14 không đụng tới.
3. Đó là **metrics-server** — APIService duy nhất trên cluster lab có `.spec.service`, tức API group
   được **proxy sang một server khác** thay vì do kube-apiserver tự phục vụ. Hai trang dạy tự dựng
   một cái như vậy là **[Cấu hình tầng tổng hợp](374-configure-aggregation-layer-vi.md)** — làm cho
   kube-apiserver chấp nhận proxy sang server ngoài — và **[Thiết lập một Extension API
   Server](380-setup-extension-api-server-vi.md)** — dựng chính server đó.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
