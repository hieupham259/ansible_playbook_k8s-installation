# Quản lý Secret (Managing Secrets)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configmap-secret/>
>
> Quản lý dữ liệu thiết lập bí mật (confidential settings data) bằng Secret.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 3 — Pod và cấu hình](00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình)
→ nhóm [3b. Cấu hình ứng dụng](00-ALO-TRINH-ADMIN.md#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod),
bài 3/12 · Kiểm chứng ở [Lab 3b](labs/LAB-3B-CAU-HINH-UNG-DUNG.md) phần B6 — ba đường tạo Secret
chạy liền nhau trong ba mục con B6.1, B6.2 và B6.3.

Đây là **trang mục lục**, đọc trong vài phút. Nó chỉ tồn tại để dẫn bạn sang ba bài kế tiếp.

**Phải hiểu ở lần đọc này:**

- Trang này không dạy cơ chế nào cả: nội dung duy nhất là mục *Danh sách các trang trong mục này*,
  liệt kê đúng **ba trang con** — bằng [`kubectl`](327-secret-kubectl-vi.md), bằng
  [file cấu hình](326-secret-config-file-vi.md), bằng [Kustomize](328-secret-kustomize-vi.md).
- Ba trang đó tạo ra **cùng một loại object `Secret`**; chúng khác nhau ở **đường tạo**, không phải
  ở loại Secret hay mức bảo vệ. Ba bài kế tiếp là ba lần làm cùng một việc, nên đọc để chọn đường
  chứ không phải để học ba cơ chế.
- Dùng trang này như điểm vào khi cần tra "tạo Secret bằng cách nào": mở đúng trang con, rồi đối
  chiếu cả ba ở phần B6 của [Lab 3b](labs/LAB-3B-CAU-HINH-UNG-DUNG.md).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Cụm *file kustomization.yaml* trong mô tả trang con thứ ba | Kustomize là công cụ quản lý manifest, chưa học ở giai đoạn 3; bài [328](328-secret-kustomize-vi.md) chỉ dùng đúng một trường của nó | bài [322](322-kustomization-vi.md), [giai đoạn 5](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng) |

---

Trang gốc là trang mục lục của phần *Tasks → Managing Secrets*: nội dung trang là danh sách
các trang con, được liệt kê dưới đây theo đúng thứ tự hiển thị trên trang web, kèm phần mô tả
của từng trang.

## Danh sách các trang trong mục này (Pages in this section)

- [Quản lý Secret bằng kubectl (Managing Secrets using kubectl)](327-secret-kubectl-vi.md) —
  Tạo các đối tượng Secret bằng công cụ dòng lệnh kubectl.
- [Quản lý Secret bằng file cấu hình (Managing Secrets using Configuration File)](326-secret-config-file-vi.md) —
  Tạo các đối tượng Secret bằng file cấu hình tài nguyên (resource configuration file).
- [Quản lý Secret bằng Kustomize (Managing Secrets using Kustomize)](328-secret-kustomize-vi.md) —
  Tạo các đối tượng Secret bằng file kustomization.yaml.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Mục này gồm mấy trang con, và điểm khác nhau giữa chúng nằm ở đâu?
2. **Câu bẫy.** Chọn đường tạo nào trong ba đường thì Secret được bảo vệ tốt hơn?
3. Trên `lab-k8s-master`, bạn cần một Secret dùng một lần rồi thôi trong lúc đang xử lý sự cố; lần
   khác bạn cần một Secret lưu cùng các manifest của lab trong `~/lab-work/3b/` để apply lại nhiều
   lần. Mỗi tình huống mở trang con nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Ba trang con**: quản lý Secret bằng `kubectl`, bằng file cấu hình tài nguyên, và bằng file
   `kustomization.yaml`. Khác nhau **chỉ ở cách tạo ra object**, tức ở công cụ và ở dạng đầu vào —
   sản phẩm cuối cùng vẫn là một object `Secret` trong cluster.
2. **Không đường nào cả.** Cả ba đều tạo ra cùng một object `Secret`, nên mức bảo vệ giống hệt
   nhau; trang mục lục này chỉ phân loại theo **công cụ**, không phân loại theo độ an toàn. Trực
   giác "dùng file cấu hình chắc kín hơn gõ dòng lệnh" nhầm ở chỗ nó so hai thứ khác nhau: rủi ro
   nằm ở nơi giá trị đi qua (lịch sử shell, file manifest), không nằm ở object kết quả.
3. Dùng một lần trong lúc vận hành → **[Quản lý Secret bằng kubectl](327-secret-kubectl-vi.md)**,
   vì đó là đường tạo bằng công cụ dòng lệnh, không để lại file nào. Cần apply lại nhiều lần từ
   thư mục manifest → **[Quản lý Secret bằng file cấu hình](326-secret-config-file-vi.md)**, vì đó
   là đường tạo object từ một file cấu hình tài nguyên mà bạn giữ lại được.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
