# Quản lý các đối tượng Kubernetes (Manage Kubernetes Objects)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/manage-kubernetes-objects/>
>
> Các mô hình khai báo (declarative) và mệnh lệnh (imperative) để tương tác với Kubernetes API.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 1 — Mô hình Kubernetes](00-ALO-TRINH-ADMIN.md#giai-đoạn-1--mô-hình-kubernetes)
→ nhóm [1b. Làm việc với object và kubectl](00-ALO-TRINH-ADMIN.md#1b-làm-việc-với-object-và-kubectl),
bài 5/8 của dòng **Thực hành** · Kiểm chứng ở
[Lab 1b — Object, label, kubectl và kubeconfig](labs/LAB-1B-OBJECT-LABEL-KUBECTL-VA-KUBECONFIG.md)
phần B5, nơi lab chạy lần lượt ba kỹ thuật quản lý object trên `lab-k8s-master`.

Đây là **trang mục lục**, không phải bài học: nó chỉ liệt kê các trang con. Đọc trong hai phút,
mục đích là biết trang nào nằm ở đâu để về sau mở đúng trang cần.

**Phải hiểu ở lần đọc này:**

- Trang gom các trang con của nhánh *Manage Kubernetes Objects* dưới hai mô hình mà dòng dẫn nêu
  ngay đầu trang: **khai báo (declarative)** và **mệnh lệnh (imperative)**.
- Ba trang trong danh sách ứng đúng ba kỹ thuật mà bài
  [27 — Quản lý object trong Kubernetes](27-object-management-vi.md) so sánh:
  [320](320-imperative-command-vi.md) là lệnh imperative, [321](321-imperative-config-vi.md) là
  imperative bằng file cấu hình, [319](319-declarative-config-vi.md) là declarative bằng file
  cấu hình. Cần thao tác cụ thể của kỹ thuật nào thì vào trang đó.
- Ba trang còn lại **không phải kỹ thuật thứ tư**: [322](322-kustomization-vi.md) (Kustomize) là
  một biến thể của mô hình declarative, [324](324-kubectl-patch-vi.md) (`kubectl patch`) là cách
  sửa object tại chỗ bằng strategic merge patch hoặc JSON merge patch, còn
  [323](323-storage-version-migration-vi.md) (Storage Version Migration) là việc di trú dữ liệu
  đã lưu, không phải cách viết manifest.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục [322 — Kustomize](322-kustomization-vi.md) | thêm một lớp sinh manifest lên trên `apply`, chưa cần khi mới học ba kỹ thuật gốc | dòng **Thực hành** của [Giai đoạn 5 — Mạng nền tảng](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng) |
| Mục [324 — `kubectl patch`](324-kubectl-patch-vi.md) | cần hiểu Deployment và các field do controller quản lý trước | dòng **Thực hành** của nhóm [4a. ReplicaSet, Deployment và rollout](00-ALO-TRINH-ADMIN.md#4a-replicaset-deployment-và-rollout) |
| Mục [323 — Storage Version Migration](323-storage-version-migration-vi.md) | thuộc chuyện phiên bản lưu trữ trong etcd, không phải chuyện quản lý manifest | dòng **Thực hành** của [Giai đoạn 14 — Khả năng mở rộng](00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng) |

---

Mục này bao gồm các trang sau:

* [Quản lý đối tượng Kubernetes theo kiểu khai báo bằng file cấu hình (Declarative Management of Kubernetes Objects Using Configuration Files)](319-declarative-config-vi.md)
* [Quản lý đối tượng Kubernetes theo kiểu khai báo bằng Kustomize (Declarative Management of Kubernetes Objects Using Kustomize)](322-kustomization-vi.md)
* [Quản lý đối tượng Kubernetes bằng các lệnh mệnh lệnh (Managing Kubernetes Objects Using Imperative Commands)](320-imperative-command-vi.md)
* [Quản lý đối tượng Kubernetes theo kiểu mệnh lệnh bằng file cấu hình (Imperative Management of Kubernetes Objects Using Configuration Files)](321-imperative-config-vi.md)
* [Cập nhật đối tượng API tại chỗ bằng kubectl patch (Update API Objects in Place Using kubectl patch)](324-kubectl-patch-vi.md)
  — dùng kubectl patch để cập nhật các đối tượng Kubernetes API tại chỗ; thực hiện một strategic
  merge patch hoặc một JSON merge patch.
* [Di trú đối tượng Kubernetes bằng Storage Version Migration (Migrate Kubernetes Objects Using Storage Version Migration)](323-storage-version-migration-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở nhóm 1b:

1. Trang này không dạy thao tác nào. Vậy nó dùng vào lúc nào, và ba trang con nào của nó tương
   ứng với ba kỹ thuật quản lý object mà bài [27](27-object-management-vi.md) so sánh?
2. **Câu bẫy.** Danh sách có sáu mục xếp ngang hàng nhau. Có phải Kubernetes có sáu cách quản lý
   object không?
3. Phần B5 của Lab 1b chạy `kubectl diff` rồi `kubectl apply -f` trên `lab-k8s-master`. Từ trang
   mục lục này, bạn mở trang con nào để đọc kỹ kỹ thuật đó, và vì sao không phải trang
   [321](321-imperative-config-vi.md)?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Dùng nó **như một điểm rẽ**: khi đã biết mình cần kỹ thuật nào thì vào đây lấy đúng trang con,
   thay vì nhớ đường dẫn từng trang. Ba trang tương ứng ba kỹ thuật là
   **[320](320-imperative-command-vi.md)** (lệnh mệnh lệnh), **[321](321-imperative-config-vi.md)**
   (mệnh lệnh bằng file cấu hình) và **[319](319-declarative-config-vi.md)** (khai báo bằng file
   cấu hình).
2. **Không.** Trang gom mọi thứ liên quan tới việc tương tác với Kubernetes API dưới **hai mô
   hình** là khai báo và mệnh lệnh; chỉ **ba** mục đầu là ba kỹ thuật quản lý object. Ba mục còn
   lại nằm cùng danh sách vì cùng chủ đề, chứ không phải vì chúng ngang hàng: Kustomize là biến
   thể của mô hình khai báo, `kubectl patch` là cách sửa tại chỗ, còn Storage Version Migration
   là di trú dữ liệu đã lưu. Danh sách phẳng của một trang mục lục không có nghĩa các mục cùng
   loại.
3. Mở **[319 — Quản lý đối tượng Kubernetes theo kiểu khai báo bằng file cấu hình](319-declarative-config-vi.md)**,
   vì `apply` là kỹ thuật **khai báo**: bạn đưa file mô tả trạng thái mong muốn và để kubectl tự
   quyết định tạo hay cập nhật. Trang [321](321-imperative-config-vi.md) cũng dùng file, nhưng
   người vận hành vẫn phải tự chỉ định thao tác (`create`, `replace`, `delete`) — đó là mô hình
   mệnh lệnh, chỉ khác ở chỗ đầu vào là file.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
