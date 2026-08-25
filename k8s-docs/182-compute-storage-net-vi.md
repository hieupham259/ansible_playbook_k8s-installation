# Các phần mở rộng về Tính toán, Lưu trữ và Mạng (Compute, Storage, and Networking Extensions)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 14](00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng), bài 6/7 ·
Kiểm chứng ở [Lab 14](labs/LAB-14-CRD-VA-OPERATOR.md).

Giai đoạn này lộ trình ghi rõ là **dành cho platform administrator / người phát triển operator**.

**Bài này là trang mục lục, dài chưa tới một màn hình.** Nó gom ba nhóm phần mở rộng hạ tầng —
lưu trữ, thiết bị, mạng — mà bài [177](177-extend-kubernetes-vi.md) đã vẽ trên bản đồ. Hai trong
ba nhóm bạn đã học kỹ ở giai đoạn trước, nhóm còn lại là bài kế tiếp. Đọc để chốt ranh giới giữa
ba nhóm, không cần thêm gì.

**Phải hiểu ở lần đọc này:**

- Điểm chung của cả ba nhóm, nêu ngay câu đầu: đây là các phần mở rộng **không đi kèm sẵn trong
  bản thân Kubernetes**. Chúng hoặc tăng khả năng cho node, hoặc cung cấp hạ tầng mạng kết nối
  các Pod.
- **Storage plugin**: CSI là cách hiện hành để thêm loại volume mới; FlexVolume đã **deprecated
  từ Kubernetes v1.23**, và cơ chế của nó là **kubelet gọi một plugin dạng binary** để mount.
- **Device plugin**: cho node **khám phá các tiện ích mới, bổ sung cho** `cpu` và `memory` sẵn
  có, rồi cung cấp chúng cho các Pod có yêu cầu. Nó **thêm** resource chứ không thay thế.
- **Network plugin**: cluster **bắt buộc** phải có thì mạng Pod mới hoạt động và mô hình mạng
  Kubernetes mới được hỗ trợ. Kubernetes v1.36 tương thích với các network plugin CNI.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Chi tiết *Device plugin* | có bài riêng ngay sau, là bài cuối giai đoạn | bài [184](184-device-plugins-vi.md) |
| Chi tiết *Network plugin* | lộ trình đặt bài đó chính ở giai đoạn 5 | bài [183](183-network-plugins-vi.md) |
| Chi tiết CSI, các loại volume, lưu trữ tạm thời | đã học ở phần lưu trữ | giai đoạn 6 — bài [91](91-volumes-vi.md) và [92](92-persistent-volumes-vi.md) |
| Bản đề xuất thiết kế FlexVolume đã lưu trữ, Volume Plugin FAQ | tài liệu cho nhà cung cấp lưu trữ | Lab 14 |

---

Phần này trình bày về các phần mở rộng cho cluster của bạn mà không đi kèm sẵn trong bản thân
Kubernetes. Bạn có thể dùng những phần mở rộng này để tăng cường khả năng cho các node trong
cluster, hoặc để cung cấp hạ tầng mạng (network fabric) kết nối các Pod với nhau.

* Các storage plugin [CSI](91-volumes-vi.md#csi) và [FlexVolume](91-volumes-vi.md#flexvolume)

  Các plugin Container Storage Interface (CSI) cung cấp một cách để mở rộng Kubernetes với khả
  năng hỗ trợ các loại volume mới. Các volume này có thể được hậu thuẫn bởi hệ thống lưu trữ
  ngoài bền vững, hoặc cung cấp lưu trữ tạm thời (ephemeral), hoặc chúng có thể cung cấp một
  giao diện chỉ đọc tới thông tin theo mô hình hệ thống tập tin (filesystem).

  Kubernetes cũng bao gồm hỗ trợ cho các plugin [FlexVolume](91-volumes-vi.md#flexvolume),
  vốn đã bị deprecated (không khuyến khích dùng) từ Kubernetes v1.23 (thay bằng CSI).

  Các plugin FlexVolume cho phép người dùng mount những loại volume mà Kubernetes không hỗ trợ
  sẵn. Khi bạn chạy một Pod dựa vào lưu trữ FlexVolume, kubelet gọi một plugin dạng binary để
  mount volume đó. Bản đề xuất thiết kế [FlexVolume](https://git.k8s.io/design-proposals-archive/storage/flexvolume-deployment.md)
  đã được lưu trữ có thêm chi tiết về cách tiếp cận này.

  Tài liệu [Kubernetes Volume Plugin FAQ for Storage Vendors](https://github.com/kubernetes/community/blob/main/sig-storage/volume-plugin-faq.md#kubernetes-volume-plugin-faq-for-storage-vendors)
  bao gồm thông tin tổng quát về các storage plugin.

* [Device plugin](184-device-plugins-vi.md)

  Device plugin cho phép một node khám phá (discover) các tiện ích mới của Node (bổ sung cho các
  resource có sẵn của node như `cpu` và `memory`), và cung cấp những tiện ích cục bộ trên node
  tùy chỉnh này cho các Pod có yêu cầu chúng.

* [Network plugin](183-network-plugins-vi.md)

  Network plugin cho phép Kubernetes làm việc với các topology và công nghệ mạng khác nhau.
  Cluster Kubernetes của bạn cần một _network plugin_ để có được một mạng Pod hoạt động được
  và để hỗ trợ các khía cạnh khác của mô hình mạng Kubernetes.

  Kubernetes v1.36 tương thích với các network plugin CNI.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 14:

1. Trong ba nhóm phần mở rộng bài liệt kê, nhóm nào cluster **bắt buộc** phải có mới chạy được?
   Bài dùng chữ gì để nói điều đó?
2. Câu bẫy: FlexVolume và CSI cùng để "thêm loại volume mới". Chúng khác nhau ở cơ chế và ở
   trạng thái vòng đời như thế nào?
3. Device plugin công bố resource kiểu gì — nó **thay thế** hay **bổ sung cho** `cpu` và
   `memory`? Ai là bên "khám phá" ra chúng?
4. Cluster lab ba VM Ubuntu của bạn đang chạy Flannel. Theo cách phân loại của bài, Flannel
   thuộc nhóm nào, và nếu gỡ nó ra thì phần nào của cluster hỏng?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Network plugin.** Bài viết: "Cluster Kubernetes của bạn **cần** một *network plugin* để có
   được một mạng Pod hoạt động được và để hỗ trợ các khía cạnh khác của mô hình mạng Kubernetes."
   Storage plugin và device plugin thì chỉ cần khi bạn có nhu cầu tương ứng — chúng "tăng cường
   khả năng cho các node", còn network plugin là điều kiện để mô hình mạng tồn tại.
2. **Cơ chế:** với FlexVolume, "kubelet gọi một plugin dạng binary để mount volume đó" — tức
   Kubernetes **thực thi một chương trình nhị phân trên node**. CSI là giao diện chuẩn hóa để
   mở rộng Kubernetes với các loại volume mới, và là thứ thay thế FlexVolume. **Vòng đời:**
   FlexVolume **đã bị deprecated từ Kubernetes v1.23, thay bằng CSI**; nó chỉ còn ý nghĩa khi
   bạn tiếp quản cluster cũ. Trực giác "hai cái tương đương, chọn cái nào cũng được" sai ở chỗ
   một cái đã hết vòng đời.
3. **Bổ sung.** Bài nói device plugin cho một node khám phá "các tiện ích mới của Node (**bổ sung
   cho** các resource có sẵn của node như `cpu` và `memory`)", rồi cung cấp những tiện ích cục bộ
   đó cho các Pod **có yêu cầu chúng**. Bên khám phá là **node** — cụ thể hơn sẽ thấy ở bài
   [184](184-device-plugins-vi.md) rằng plugin đăng ký với kubelet còn kubelet công bố lên API
   server.
4. Flannel là một **network plugin (CNI)**. Gỡ nó ra thì **mạng Pod ngừng hoạt động**: Pod không
   có được mạng hoạt động được và mô hình mạng Kubernetes không còn được hỗ trợ — đây là nhóm
   duy nhất trong ba nhóm mà thiếu là cluster không chạy workload được, chứ không chỉ mất một
   khả năng bổ sung.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
