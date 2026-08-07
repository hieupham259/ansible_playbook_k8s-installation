# Giám sát tình trạng volume (Volume Health Monitoring)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/storage/volume-health-monitoring/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 6](LO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), bài 16/16 · Kiểm chứng ở
Lab 6b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài cuối của giai đoạn 6, dài chưa tới một trang, và toàn bộ là tính năng alpha phụ thuộc CSI
driver. Đọc để biết **khi lưu trữ bên dưới hỏng thì Kubernetes báo cho bạn ở đâu** — đó là câu
hỏi vận hành duy nhất bài này trả lời.

**Phải hiểu ở lần đọc này:**

- Giám sát tình trạng volume là một phần trong cách Kubernetes hiện thực CSI, và được hiện thực
  ở **hai chỗ**: External Health Monitor controller và kubelet. Driver không hỗ trợ thì không
  có gì để báo — mục *Giám sát tình trạng volume*.
- Phía controller: phát hiện điều kiện bất thường của một CSI volume thì báo một Event **trên
  PersistentVolumeClaim** liên quan. Đặt thêm cờ `enable-node-watcher` là true thì controller
  cũng theo dõi lỗi node và báo Event trên PVC để cho biết Pod đang dùng PVC này nằm trên một
  node bị lỗi — cùng mục.
- Phía node: báo Event **trên mọi Pod đang dùng PVC**, và phơi bày metric
  `kubelet_volume_stats_health_status_abnormal` với hai label `namespace` và
  `persistentvolumeclaim`; giá trị **1 là volume không khỏe mạnh, 0 là khỏe mạnh** — cùng mục.
- Điều kiện bật phía node: feature gate **`CSIVolumeHealth`** — ghi chú cuối mục.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Link KEP về thay đổi metric của kubelet | tài liệu thiết kế | không cần |
| Danh sách CSI driver đã hiện thực tính năng trong mục *Tiếp theo* | tra khi chọn driver cho cluster | Lab 6a, khi chọn provisioner |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.21 [alpha]`

Tính năng giám sát tình trạng volume của CSI cho phép các CSI Driver
phát hiện các điều kiện bất thường của volume từ hệ thống lưu trữ bên
dưới và báo cáo chúng dưới dạng event trên các PVC hoặc Pod.

## Giám sát tình trạng volume (Volume health monitoring)

*Giám sát tình trạng volume* (volume health monitoring) trong Kubernetes
là một phần trong cách Kubernetes hiện thực Container Storage Interface
(CSI). Tính năng giám sát tình trạng volume được hiện thực trong hai
thành phần: một controller giám sát tình trạng bên ngoài (External
Health Monitor controller), và kubelet.

Nếu một CSI Driver hỗ trợ tính năng Giám sát tình trạng volume từ phía
controller, một event sẽ được báo cáo trên PersistentVolumeClaim (PVC)
liên quan khi một điều kiện bất thường của volume được phát hiện trên
một CSI volume.

Controller External Health Monitor cũng theo dõi các event lỗi node
(node failure). Bạn có thể bật giám sát lỗi node bằng cách đặt cờ
`enable-node-watcher` là true. Khi external health monitor phát hiện
một event lỗi node, controller sẽ báo cáo một Event trên PVC để cho
biết các Pod đang dùng PVC này nằm trên một node bị lỗi.

Nếu một CSI Driver hỗ trợ tính năng Giám sát tình trạng volume từ phía
node, một Event sẽ được báo cáo trên mọi Pod đang dùng PVC khi một điều
kiện bất thường của volume được phát hiện trên một CSI volume. Ngoài
ra, thông tin tình trạng volume được cung cấp dưới dạng các metric
VolumeStats của Kubelet. Một metric mới
kubelet_volume_stats_health_status_abnormal được thêm vào. Metric này
gồm hai nhãn (label): `namespace` và `persistentvolumeclaim`. Giá trị
đếm là 1 hoặc 0. 1 cho biết volume không khỏe mạnh, 0 cho biết volume
khỏe mạnh. Để biết thêm thông tin, hãy xem
[KEP](https://github.com/kubernetes/enhancements/tree/master/keps/sig-storage/1432-volume-health-monitor#kubelet-metrics-changes).

> **Ghi chú:**
> Bạn cần bật [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
> `CSIVolumeHealth` để dùng tính năng này từ phía node.

## Tiếp theo (What's next)

Xem [tài liệu về CSI driver](https://kubernetes-csi.github.io/docs/drivers.html)
để biết những CSI driver nào đã hiện thực tính năng này.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 6:

1. Cùng một volume gặp sự cố. Event được báo trên PVC hay trên Pod? Điều đó phụ thuộc vào cái
   gì?
2. Metric `kubelet_volume_stats_health_status_abnormal` bằng `1` nghĩa là gì, và bạn lọc nó
   theo những chiều nào?
3. Cluster lab của bạn sau Lab 6a có một CSI driver. Bạn cần đủ những gì để thấy được bất kỳ
   thông tin tình trạng volume nào, và phần nào cần feature gate?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Phụ thuộc vào việc CSI driver hỗ trợ phía nào.** Nếu driver hỗ trợ giám sát **từ phía
   controller**, Event được báo **trên PersistentVolumeClaim**. Nếu driver hỗ trợ **từ phía
   node**, Event được báo **trên mọi Pod đang dùng PVC đó**. Đây là chỗ dễ nhầm: không phải hai
   cách báo cùng một sự kiện, mà là **hai phần hiện thực khác nhau** — controller nhìn từ tầng
   lưu trữ, kubelet nhìn từ node. Riêng trường hợp **lỗi node**, controller báo Event trên PVC
   khi bạn đã bật cờ `enable-node-watcher`.
2. **Volume không khỏe mạnh** (giá trị `0` mới là khỏe mạnh). Metric có hai label:
   **`namespace`** và **`persistentvolumeclaim`**, nên bạn lọc được theo namespace và theo từng
   PVC cụ thể. Metric này thuộc nhóm VolumeStats của kubelet, tức đến từ **phía node**.
3. Cần ba thứ. Thứ nhất, **CSI driver phải có hiện thực tính năng giám sát tình trạng volume** —
   không phải driver nào cũng có, phải tra danh sách driver. Thứ hai, tùy phía bạn muốn: **phía
   controller cần External Health Monitor controller** đang chạy (và cờ `enable-node-watcher`
   nếu muốn theo dõi lỗi node). Thứ ba, **phía node cần bật feature gate `CSIVolumeHealth`** —
   chỉ phía node mới cần gate này. Toàn bộ tính năng vẫn ở mức alpha, nên đừng xây quy trình
   vận hành dựa vào nó.

</details>

Đây là bài cuối của giai đoạn 6. Trả lời trôi cả ba câu thì chuyển sang **Lab 6b — Snapshot và
volume nâng cao** (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)); phần nào driver
không hỗ trợ thì giữ nguyên trong [sổ nợ lab](labs/README.md#5-sổ-nợ-lab) chứ đừng bỏ qua.
