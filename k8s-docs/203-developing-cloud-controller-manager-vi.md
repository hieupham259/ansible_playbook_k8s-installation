# Phát triển Cloud Controller Manager (Developing Cloud Controller Manager)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/developing-cloud-controller-manager/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 1 — Mô hình Kubernetes](00-ALO-TRINH-ADMIN.md#giai-đoạn-1--mô-hình-kubernetes)
→ nhóm [1c. Vòng đời và cơ chế nền của object](00-ALO-TRINH-ADMIN.md#1c-vòng-đời-và-cơ-chế-nền-của-object),
bài 1/1 của dòng **Thực hành** · Kiểm chứng ở
[Lab 1c — Vòng đời và cơ chế nền của object](labs/LAB-1C-VONG-DOI-VA-CO-CHE-NEN-CUA-OBJECT.md)
phần B6, nơi lab chứng minh cluster self-managed không có Pod `cloud-controller-manager` và cả
ba Node đều để trống `spec.providerID`.

Bài mở rộng bài [34 — Cloud Controller Manager](34-cloud-controller-vi.md) của cùng nhóm 1c, và
có liên hệ với bài [198 — Leader Migration](198-controller-manager-leader-migration-vi.md) ở
[giai đoạn 12](00-ALO-TRINH-ADMIN.md#giai-đoạn-12--quản-trị-cluster-nâng-cao).

Bài rất ngắn và viết cho **người phát triển** muốn xây dựng cloud-controller-manager cho một
cloud provider — không phải cho người vận hành. Với cluster lab on-premise của bạn, chỉ cần
nắm mô hình plug-in để hiểu vì sao code lõi Kubernetes không chứa code đặc thù cloud.

**Phải hiểu ở lần đọc này:**

- Lý do tách code đặc thù cloud ra binary `cloud-controller-manager` riêng: cloud provider
  phát triển và phát hành theo nhịp độ khác với dự án Kubernetes, nên cần được tiến hóa độc
  lập với code lõi.
- Mô hình plug-in: Kubernetes cung cấp khung code (skeleton) với các Go interface; mỗi cloud
  provider hiện thực `cloudprovider.Interface` và tự đăng ký bằng cách gọi
  `cloudprovider.RegisterCloudProvider` (trong khối `init` của package) để ghi tên mình vào
  biến toàn cục chứa danh sách provider khả dụng.
- Phân biệt hai con đường: **out-of-tree** (viết package riêng ngoài repo Kubernetes, build
  binary từ `main.go` mẫu của core) và **in-tree** (chạy cloud controller manager có sẵn
  trong core dưới dạng một DaemonSet trong cluster).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ba bước build out-of-tree và các link mã nguồn Go (`cloud.go`, `plugins.go`, `main.go`) | chỉ cần khi bạn thực sự viết một cloud-controller-manager mới | khi làm việc trực tiếp với repo `kubernetes/cloud-provider` |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.11 [beta]`

cloud-controller-manager là một thành phần thuộc control plane của Kubernetes, nhúng logic
điều khiển đặc thù cho từng cloud. Cloud controller manager cho phép bạn kết nối cluster của
mình với API của nhà cung cấp cloud (cloud provider), đồng thời tách các thành phần tương
tác với nền tảng cloud đó khỏi các thành phần chỉ tương tác với cluster của bạn.

Bằng cách tách rời logic tương tác giữa Kubernetes và hạ tầng cloud bên dưới, thành phần
cloud-controller-manager giúp các nhà cung cấp cloud có thể phát hành tính năng theo nhịp độ
khác với dự án Kubernetes chính.

## Bối cảnh (Background)

Vì các cloud provider phát triển và phát hành theo nhịp độ khác so với dự án Kubernetes,
việc trừu tượng hóa phần code đặc thù của từng provider vào binary `cloud-controller-manager`
cho phép các nhà cung cấp cloud tiến hóa độc lập với code lõi của Kubernetes.

Dự án Kubernetes cung cấp khung code (skeleton) cloud-controller-manager kèm các Go
interface để cho phép bạn (hoặc nhà cung cấp cloud của bạn) cắm phần hiện thực của riêng
mình vào. Điều này nghĩa là một cloud provider có thể hiện thực một cloud-controller-manager
bằng cách import các package từ Kubernetes core; mỗi cloud provider sẽ đăng ký code của họ
bằng cách gọi `cloudprovider.RegisterCloudProvider` để cập nhật một biến toàn cục chứa danh
sách các cloud provider khả dụng.

## Phát triển (Developing)

### Out of tree

Để build một cloud-controller-manager out-of-tree cho cloud của bạn:

1. Tạo một package Go với phần hiện thực thỏa mãn
   [cloudprovider.Interface](https://github.com/kubernetes/cloud-provider/blob/master/cloud.go).
2. Dùng [`main.go` trong cloud-controller-manager](https://github.com/kubernetes/kubernetes/blob/master/cmd/cloud-controller-manager/main.go)
   của Kubernetes core làm khuôn mẫu cho `main.go` của bạn. Như đã nói ở trên, khác biệt duy
   nhất chỉ nên là package cloud sẽ được import.
3. Import package cloud của bạn trong `main.go`, bảo đảm package của bạn có một khối `init`
   để chạy [`cloudprovider.RegisterCloudProvider`](https://github.com/kubernetes/cloud-provider/blob/master/plugins.go).

Nhiều cloud provider công bố code controller manager của họ dưới dạng mã nguồn mở. Nếu bạn
đang tạo một cloud-controller-manager mới từ đầu, bạn có thể lấy một cloud controller
manager out-of-tree có sẵn làm điểm xuất phát.

### In tree

Với các cloud provider in-tree, bạn có thể chạy cloud controller manager in-tree như một
DaemonSet trong cluster của mình. Xem
[Quản trị Cloud Controller Manager](254-running-cloud-controller-vi.md)
để biết thêm chi tiết.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc này:

1. Vì sao Kubernetes tách logic đặc thù cloud thành binary `cloud-controller-manager` riêng
   thay vì giữ nó trong code lõi?
2. Một cloud provider muốn "cắm" code của họ vào khung cloud-controller-manager thì phải làm
   những việc gì?
3. **Câu bẫy.** Cluster lab on-premise của bạn không đặt `--cloud-provider` và không chạy
   cloud-controller-manager nào. Điều đó có nghĩa là cluster đang thiếu một thành phần, và
   bạn phải tự phát triển một cloud-controller-manager theo bài này không?
4. Out-of-tree và in-tree khác nhau ở đâu, và cách chạy in-tree được bài mô tả như thế nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Vì nhịp phát triển và phát hành khác nhau.** Cloud provider ra tính năng theo lịch
   riêng của họ; trừu tượng hóa code đặc thù provider vào binary `cloud-controller-manager`
   cho phép họ tiến hóa độc lập với code lõi Kubernetes, đồng thời tách các thành phần tương
   tác với nền tảng cloud khỏi các thành phần chỉ tương tác với cluster.
2. **Hai việc chính:** (a) viết một package Go hiện thực thỏa mãn `cloudprovider.Interface`;
   (b) tạo `main.go` từ khuôn mẫu `main.go` của Kubernetes core (khác biệt duy nhất là
   package cloud được import), trong đó package của họ phải có khối `init` gọi
   `cloudprovider.RegisterCloudProvider` để tự ghi tên vào biến toàn cục chứa danh sách các
   provider khả dụng.
3. **Không.** cloud-controller-manager tồn tại để kết nối cluster với API của một cloud
   provider; cluster on-premise không có nền tảng cloud nào để tích hợp nên không cần thành
   phần này — đúng như bài [34](34-cloud-controller-vi.md) đã nêu. Bài này chỉ dành cho
   người xây dựng tích hợp cho một cloud cụ thể. Trực giác sai ở chỗ coi
   cloud-controller-manager là thành phần bắt buộc của mọi control plane.
4. **Out-of-tree:** code provider nằm ngoài repo Kubernetes; bạn build một binary riêng từ
   package của mình cộng với `main.go` mẫu của core — đây là con đường để provider tiến hóa
   độc lập. **In-tree:** code provider nằm sẵn trong Kubernetes core; bài cho biết bạn có
   thể chạy cloud controller manager in-tree như một **DaemonSet** trong cluster, chi tiết ở
   trang Cloud Controller Manager Administration.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
