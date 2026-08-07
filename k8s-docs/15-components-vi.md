# Các thành phần của Kubernetes (Kubernetes Components)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/overview/components/>
>
> Tổng quan về các thành phần chính cấu thành một cluster Kubernetes.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 1 → nhóm [1a](LO-TRINH-ADMIN.md#1a-kiến-trúc-và-mô-hình-điều-khiển),
bài 2/8 · Kiểm chứng ở [Lab 1a](labs/LAB-1A-KIEN-TRUC-VA-MO-HINH-DIEU-KHIEN.md) phần B2.

Bài này rất ngắn và chỉ là danh mục. Giá trị của nó nằm ở **ranh giới phân loại**, không ở
chi tiết từng thành phần — chi tiết nằm ở bài [Kiến trúc cluster](22-architecture-vi.md) ngay
sau đây.

**Phải hiểu ở lần đọc này:**

- Ba nhóm tách bạch: thành phần **control plane**, thành phần **node**, và **addon**. Nhầm
  nhóm là lỗi phổ biến nhất ở giai đoạn 1.
- Năm thành phần control plane: `kube-apiserver`, `etcd`, `kube-scheduler`,
  `kube-controller-manager`, và `cloud-controller-manager` (tùy chọn).
- Ba thành phần node: `kubelet`, `kube-proxy` (tùy chọn), container runtime.
- Vì sao hai thành phần được đánh dấu *tùy chọn*: cluster lab on-premise không có
  cloud-controller-manager; `kube-proxy` có thể được network plugin thay thế.
- CoreDNS là **addon**, không phải thành phần bắt buộc của control plane.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mô tả một dòng của từng thành phần | quá ngắn để hiểu thật | bài [22](22-architecture-vi.md) ngay sau |
| Addon Web UI, monitoring, cluster logging | là chủ đề observability | giai đoạn 11 |

---

Trang này cung cấp cái nhìn tổng quan ở mức cao về các thành phần thiết yếu cấu thành một cluster Kubernetes.

![Các thành phần của Kubernetes](https://kubernetes.io/images/docs/components-of-kubernetes.svg)

*Các thành phần của một cluster Kubernetes*

## Các thành phần cốt lõi (Core Components)

Một cluster Kubernetes bao gồm một control plane và một hoặc nhiều worker node.
Dưới đây là tổng quan ngắn gọn về các thành phần chính:

### Các thành phần của control plane (Control Plane Components)

Quản lý trạng thái tổng thể của cluster:

* **[kube-apiserver](https://kubernetes.io/docs/concepts/architecture/#kube-apiserver)**:
  Server thành phần cốt lõi expose HTTP API của Kubernetes.
* **[etcd](https://kubernetes.io/docs/concepts/architecture/#etcd)**:
  Kho lưu trữ key-value nhất quán và có tính sẵn sàng cao (highly-available) cho toàn bộ dữ liệu của API server.
* **[kube-scheduler](https://kubernetes.io/docs/concepts/architecture/#kube-scheduler)**:
  Tìm các Pod chưa được gắn (bound) với node nào, và gán mỗi Pod vào một node phù hợp.
* **[kube-controller-manager](https://kubernetes.io/docs/concepts/architecture/#kube-controller-manager)**:
  Chạy các controller để hiện thực hóa hành vi của Kubernetes API.
* **[cloud-controller-manager](https://kubernetes.io/docs/concepts/architecture/#cloud-controller-manager)** (tùy chọn):
  Tích hợp với (các) nhà cung cấp cloud bên dưới.

### Các thành phần của node (Node Components)

Chạy trên mọi node, duy trì các pod đang chạy và cung cấp môi trường runtime của Kubernetes:

* **[kubelet](https://kubernetes.io/docs/concepts/architecture/#kubelet)**:
  Bảo đảm các Pod, bao gồm các container của chúng, đang chạy.
* **[kube-proxy](https://kubernetes.io/docs/concepts/architecture/#kube-proxy)** (tùy chọn):
  Duy trì các quy tắc mạng (network rules) trên các node để hiện thực hóa các Service.
* **[Container runtime](https://kubernetes.io/docs/concepts/architecture/#container-runtime)**:
  Phần mềm chịu trách nhiệm chạy các container. Đọc
  [Container Runtimes](./00-container-runtimes-vi.md) để tìm hiểu thêm.

> **Ghi chú:** Mục này đề cập đến sản phẩm hoặc dự án bên thứ ba cung cấp chức năng mà Kubernetes cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về sản phẩm hoặc dự án bên thứ ba đó. Để biết thêm chi tiết, hãy đọc [hướng dẫn về nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content).

Cluster của bạn có thể cần thêm phần mềm bổ sung trên mỗi node; ví dụ, bạn cũng có thể
chạy [systemd](https://systemd.io/) trên một node Linux để giám sát các thành phần cục bộ.

## Addons

Addon mở rộng chức năng của Kubernetes. Một vài ví dụ quan trọng bao gồm:

* **[DNS](https://kubernetes.io/docs/concepts/architecture/#dns)**:
  Để phân giải DNS trong toàn cluster.
* **[Web UI](https://kubernetes.io/docs/concepts/architecture/#web-ui-dashboard)** (Dashboard):
  Để quản lý cluster thông qua giao diện web.
* **[Giám sát tài nguyên container (Container Resource Monitoring)](https://kubernetes.io/docs/concepts/architecture/#container-resource-monitoring)**:
  Để thu thập và lưu trữ các chỉ số (metrics) của container.
* **[Logging cấp cluster (Cluster-level Logging)](https://kubernetes.io/docs/concepts/architecture/#cluster-level-logging)**:
  Để lưu log của container vào một kho log tập trung.

## Sự linh hoạt trong kiến trúc (Flexibility in Architecture)

Kubernetes cho phép sự linh hoạt trong cách các thành phần này được triển khai và quản lý.
Kiến trúc có thể được điều chỉnh cho nhiều nhu cầu khác nhau, từ các môi trường phát triển nhỏ
cho đến các hệ thống production quy mô lớn.

Để biết thông tin chi tiết hơn về từng thành phần và các cách khác nhau để cấu hình
kiến trúc cluster của bạn, hãy xem trang [Kiến trúc cluster (Cluster Architecture)](https://kubernetes.io/docs/concepts/architecture/).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 1:

1. Trên control-plane node có chạy `kubelet` và container runtime. Hai thứ đó có phải thành
   phần control plane không? Vì sao?
2. Cluster lab on-premise của bạn thiếu thành phần control plane nào trong danh sách, và vì
   sao thiếu nó vẫn chạy được?
3. CoreDNS thuộc nhóm nào trong ba nhóm: control plane, thành phần node, hay addon?
4. Nếu network plugin tự hiện thực chuyển tiếp cho Service thì node còn cần `kube-proxy`
   không?

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
