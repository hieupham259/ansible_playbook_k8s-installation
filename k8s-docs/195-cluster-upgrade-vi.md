# Nâng cấp một Cluster (Upgrade A Cluster)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/cluster-upgrade/>
>
> Trang này cung cấp cái nhìn tổng quan về các bước bạn nên làm theo để nâng cấp một cluster
> Kubernetes.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 17 — Nâng cấp cluster](00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster),
bài 5/5 · thực hành trực tiếp trên cluster VM của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md).

Đây là trang tổng quan **không gắn với công cụ nào**: nó cho khung chung của mọi cuộc nâng cấp,
còn quy trình từng lệnh cho cluster kubeadm nằm ở bài *Upgrading kubeadm clusters* (giai đoạn 17 bài 1)
mà bạn làm trước bài này. Giá trị của bài nằm ở **thứ tự các bước** và **các việc sau nâng cấp** —
hai thứ dễ bị bỏ sót khi chỉ chăm chăm chạy lệnh.

**Phải hiểu ở lần đọc này:**

- Khung bốn bước ở mức tổng quan: nâng **control plane** → nâng **các node** → nâng **client**
  (kubectl) → **điều chỉnh manifest** theo các thay đổi API của phiên bản mới; kèm khuyến nghị
  luôn chạy patch release mới nhất của một minor release còn được hỗ trợ.
- Phạm vi phiên bản của trang: chỉ nói về nâng từ v1.35 lên v1.36; cluster đang chạy phiên bản
  khác thì phải đọc tài liệu của đúng phiên bản định nâng lên.
- Trình tự nâng control plane thủ công: etcd → kube-apiserver → kube-controller-manager →
  kube-scheduler → cloud controller manager (nếu dùng), rồi mới cài kubectl mới.
- Với mỗi node: **drain trước**, sau đó hoặc thay node mới chạy kubelet v1.36, hoặc nâng kubelet
  tại chỗ; và vì sao drain trước khi nâng kubelet lại quan trọng (mục *Triển khai thủ công*).
- Hai việc sau nâng cấp: ghi lại các đối tượng trong etcd bằng phiên bản API lưu trữ mới nhất
  (mục *Chuyển phiên bản API lưu trữ*), và dùng `kubectl convert` để cập nhật manifest.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Device Plugins* | chỉ liên quan cluster có thiết bị chuyên dụng (GPU, NIC đặc thù…) | đọc lại bài [184](184-device-plugins-vi.md) khi cluster của bạn chạy device plugin |
| Ghi chú về cgroup v1 và `FailCgroupV1` | nền tảng cgroup đã học riêng | bài [33](33-cgroups-vi.md), mục *Loại bỏ dần cgroup v1* |

---

Trang này cung cấp cái nhìn tổng quan về các bước bạn nên làm theo để nâng cấp một cluster
Kubernetes.

Dự án Kubernetes khuyến nghị nâng cấp kịp thời lên các bản phát hành vá lỗi (patch release)
mới nhất, đồng thời đảm bảo rằng bạn đang chạy một bản phát hành minor còn được hỗ trợ của
Kubernetes. Tuân theo khuyến nghị này giúp bạn duy trì sự an toàn.

Cách bạn nâng cấp một cluster phụ thuộc vào cách bạn triển khai nó lúc đầu và vào những thay
đổi sau đó (nếu có).

Ở mức tổng quan, các bước bạn thực hiện là:

- Nâng cấp control plane
- Nâng cấp các node trong cluster
- Nâng cấp các client như kubectl
- Điều chỉnh các manifest và tài nguyên khác dựa trên những thay đổi API đi kèm phiên bản
  Kubernetes mới

## Trước khi bạn bắt đầu (Before you begin)

Bạn phải có sẵn một cluster. Trang này nói về việc nâng cấp từ Kubernetes v1.35 lên Kubernetes
v1.36. Nếu cluster của bạn hiện không chạy Kubernetes v1.35 thì hãy xem tài liệu của phiên bản
Kubernetes mà bạn dự định nâng cấp lên.

> **Ghi chú:** Trên các node Linux, kubelet mặc định chỉ hỗ trợ cgroups v2. Với Kubernetes
> v1.36, tùy chọn cấu hình kubelet `FailCgroupV1` được đặt là `true` theo mặc định.
>
> Để tìm hiểu thêm, hãy xem [tài liệu về việc loại bỏ dần cgroup v1 của Kubernetes](33-cgroups-vi.md#loại-bỏ-dần-cgroup-v1-deprecation-of-cgroup-v1).

## Các cách tiếp cận nâng cấp (Upgrade approaches)

### kubeadm {#upgrade-kubeadm}

Nếu cluster của bạn được triển khai bằng công cụ `kubeadm`, hãy xem
[Nâng cấp cluster kubeadm](221-kubeadm-upgrade-vi.md)
để biết thông tin chi tiết về cách nâng cấp cluster.

Sau khi đã nâng cấp cluster, hãy nhớ
[cài đặt phiên bản mới nhất của `kubectl`](185-tools-vi.md).

### Triển khai thủ công (Manual deployments)

> **Thận trọng:** Các bước này không tính đến các phần mở rộng (extension) của bên thứ ba như
> plugin mạng và plugin lưu trữ.

Bạn nên cập nhật control plane theo cách thủ công theo trình tự sau:

- etcd (tất cả các instance)
- kube-apiserver (tất cả các host control plane)
- kube-controller-manager
- kube-scheduler
- cloud controller manager, nếu bạn có dùng

Đến thời điểm này, bạn nên
[cài đặt phiên bản mới nhất của `kubectl`](185-tools-vi.md).

Với mỗi node trong cluster, hãy
[drain](255-safely-drain-node-vi.md) node đó, sau đó
hoặc thay thế nó bằng một node mới dùng kubelet v1.36, hoặc nâng cấp kubelet trên node đó và
đưa node trở lại hoạt động.

> **Thận trọng:** Việc drain node trước khi nâng cấp kubelet đảm bảo rằng các pod được kết nạp
> lại (re-admit) và các container được tạo lại — điều này có thể cần thiết để khắc phục một số
> vấn đề bảo mật hoặc các lỗi quan trọng khác.

### Các cách triển khai khác (Other deployments) {#upgrade-other}

Hãy tham khảo tài liệu của công cụ triển khai cluster mà bạn dùng để biết các bước bảo trì
được khuyến nghị.

## Các việc sau nâng cấp (Post-upgrade tasks)

### Chuyển phiên bản API lưu trữ của cluster (Switch your cluster's storage API version)

Các đối tượng được tuần tự hóa (serialize) vào etcd — dạng biểu diễn nội bộ của cluster cho các
resource Kubernetes đang hoạt động trong cluster — được ghi bằng một phiên bản API cụ thể.

Khi API được hỗ trợ thay đổi, các đối tượng này có thể cần được ghi lại bằng API mới hơn.
Không làm việc này thì cuối cùng sẽ dẫn đến những resource mà API server của Kubernetes không
còn giải mã (decode) hoặc sử dụng được nữa.

Với mỗi đối tượng bị ảnh hưởng, hãy lấy (fetch) nó về bằng API mới nhất được hỗ trợ, rồi ghi
nó trở lại cũng bằng API mới nhất được hỗ trợ.

### Cập nhật các manifest (Update manifests)

Nâng cấp lên một phiên bản Kubernetes mới có thể mang đến các API mới.

Bạn có thể dùng lệnh `kubectl convert` để chuyển đổi manifest giữa các phiên bản API khác nhau.
Ví dụ:

```shell
kubectl convert -f pod.yaml --output-version v1
```

Công cụ `kubectl` thay thế nội dung của `pod.yaml` bằng một manifest có `kind` là Pod
(không đổi), nhưng với `apiVersion` đã được sửa lại.

### Device Plugins

Nếu cluster của bạn đang chạy các device plugin và node cần được nâng cấp lên một bản phát
hành Kubernetes có phiên bản device plugin API mới hơn, thì các device plugin phải được nâng
cấp để hỗ trợ cả hai phiên bản **trước khi** node được nâng cấp, nhằm đảm bảo việc cấp phát
thiết bị tiếp tục hoàn thành thành công trong suốt quá trình nâng cấp.

Hãy xem [Tương thích API](184-device-plugins-vi.md#tương-thích-api-api-compatibility) và
[Các phiên bản API của Kubelet Device Manager](https://kubernetes.io/docs/reference/node/device-plugin-api-versions/)
để biết thêm chi tiết.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu dưới đây mà không nhìn lại bài là đủ cho lần đọc ở checkpoint này.

1. Bạn phải nâng cấp thủ công một control plane không dựng bằng kubeadm. Hãy kể đúng trình tự
   các thành phần, và cho biết bước cài `kubectl` phiên bản mới nằm ở chỗ nào trong quy trình.
2. Cluster của bạn đang chạy v1.34 và bạn muốn lên v1.36. Có làm thẳng theo trang này được
   không? Vì sao?
3. Đến lượt nâng cấp `lab-k8s-worker2` trong cluster lab của bạn: bài yêu cầu làm gì với node
   trước tiên, sau đó bạn có những lựa chọn nào, và vì sao bước đầu tiên đó lại quan trọng khi
   nâng kubelet?
4. Sau khi nâng cấp xong mà bạn bỏ qua việc "chuyển phiên bản API lưu trữ", chuyện gì sẽ xảy
   ra về lâu dài, và thao tác khắc phục cho từng đối tượng bị ảnh hưởng là gì?
5. Chạy `kubectl convert -f pod.yaml --output-version v1` có tạo ra một file mới không? Lệnh
   này thay đổi những gì trong manifest?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Trình tự: **etcd (tất cả các instance) → kube-apiserver (tất cả các host control plane) →
   kube-controller-manager → kube-scheduler → cloud controller manager (nếu dùng)**. Ngay sau
   khi xong control plane — "đến thời điểm này" theo lời bài — bạn cài phiên bản mới nhất của
   `kubectl`, rồi mới bắt đầu xử lý từng node.
2. **Không.** Trang này chỉ nói về việc nâng từ Kubernetes v1.35 lên v1.36; bài viết rõ: nếu
   cluster của bạn hiện không chạy v1.35 thì hãy xem tài liệu của phiên bản Kubernetes mà bạn
   dự định nâng cấp lên. Trực giác "cứ chạy lệnh nâng cấp là xong" sai ở chỗ tài liệu nâng cấp
   luôn gắn với một cặp phiên bản cụ thể, không phải một quy trình dùng chung cho mọi khoảng
   cách phiên bản.
3. Trước tiên phải **drain** node đó. Sau đó có hai lựa chọn: **thay thế bằng một node mới
   dùng kubelet v1.36**, hoặc **nâng cấp kubelet ngay trên node rồi đưa node trở lại hoạt
   động**. Drain trước khi nâng kubelet quan trọng vì nó đảm bảo **các pod được kết nạp lại
   (re-admit) và các container được tạo lại** — điều có thể cần thiết để khắc phục một số vấn
   đề bảo mật hoặc các lỗi quan trọng khác.
4. Các đối tượng trong etcd được ghi bằng một phiên bản API cụ thể; khi API được hỗ trợ thay
   đổi mà không ghi lại, **cuối cùng sẽ có những resource mà API server không còn giải mã hoặc
   sử dụng được nữa**. Khắc phục: với mỗi đối tượng bị ảnh hưởng, **fetch nó bằng API mới nhất
   được hỗ trợ rồi ghi nó trở lại cũng bằng API mới nhất được hỗ trợ**.
5. **Không tạo file mới** — `kubectl` **thay thế nội dung của chính `pod.yaml`** bằng một
   manifest trong đó `kind` giữ nguyên là Pod, còn **`apiVersion` được sửa lại** theo phiên
   bản đích.

</details>

Đây là bài cuối của **giai đoạn 17 — Nâng cấp cluster**. Trả lời được cả năm câu thì chốt checkpoint
bằng một cuộc nâng cấp thật trên cluster lab theo bài
[Upgrading kubeadm clusters](221-kubeadm-upgrade-vi.md),
rồi mới sang [Giai đoạn 18 — Vòng đời chứng chỉ](00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ).
