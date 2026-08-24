# Bật hoặc tắt một Kubernetes API (Enable Or Disable A Kubernetes API)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/enable-disable-api/>
>
> Trang này hướng dẫn cách bật hoặc tắt một phiên bản API trong control plane của cluster.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Checkpoint tiếp nối — CP5 Cấu hình lại cluster đang chạy](00-ALO-TRINH-ADMIN.md#cp5--cấu-hình-lại-cluster-đang-chạy),
bài 5/6 · thực hành trực tiếp trên cluster VM của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md).

Bài này rất ngắn: nó chỉ dạy một flag. Đừng nhầm `--runtime-config` (bật/tắt **phiên bản
API** — nhóm resource mà API server phục vụ) với `--feature-gates` (bật/tắt **tính năng**)
vừa học ở bài [196](196-configure-feature-gates-vi.md). Trên cluster kubeadm của lab, cách
đưa flag vào kube-apiserver dùng lại đúng cơ chế sửa static pod manifest của bài 196.

**Phải hiểu ở lần đọc này:**

- Phiên bản API được bật hoặc tắt bằng flag dòng lệnh `--runtime-config=api/<version>` của
  **API server** — đây là cấu hình control plane, không phải một đối tượng trong cluster.
- Giá trị của flag là danh sách các phiên bản API phân tách bằng dấu phẩy; **giá trị đứng sau
  ghi đè giá trị đứng trước**.
- Hai khóa đặc biệt: `api/all` đại diện cho mọi API đã biết, `api/legacy` chỉ đại diện cho
  các API legacy — tức các API đã bị deprecated một cách tường minh.
- Đọc hiểu ví dụ chuẩn: `--runtime-config=api/all=false,api/v1=true` tắt mọi phiên bản API
  trừ v1.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Chi tiết chính sách deprecation đứng sau khóa `api/legacy` | bài chỉ trỏ link, không trình bày | trang [Deprecation Policy](https://kubernetes.io/docs/reference/using-api/deprecation-policy/) khi bạn cần tra một API sắp bị gỡ |

---

Trang này hướng dẫn cách bật hoặc tắt một phiên bản API trong control plane của cluster.

Các phiên bản API cụ thể có thể được bật hoặc tắt bằng cách truyền
`--runtime-config=api/<version>` làm đối số dòng lệnh cho API server. Giá trị của đối số này
là một danh sách các phiên bản API phân tách bằng dấu phẩy. Giá trị đứng sau sẽ ghi đè giá
trị đứng trước.

Đối số dòng lệnh `runtime-config` cũng hỗ trợ 2 khóa đặc biệt:

- `api/all`, đại diện cho tất cả các API đã biết
- `api/legacy`, chỉ đại diện cho các API legacy. API legacy là bất kỳ API nào đã bị
  [deprecated](https://kubernetes.io/docs/reference/using-api/deprecation-policy/) một cách
  tường minh.

Ví dụ, để tắt tất cả các phiên bản API ngoại trừ v1, hãy truyền
`--runtime-config=api/all=false,api/v1=true` cho `kube-apiserver`.

## Tiếp theo (What's next)

Đọc [tài liệu đầy đủ](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)
về thành phần `kube-apiserver`.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu dưới đây mà không nhìn lại bài là đủ cho lần đọc ở checkpoint này.

1. Muốn tắt một phiên bản API trong cluster, bạn tác động vào thành phần nào, bằng flag nào,
   và giá trị của flag có dạng gì?
2. Vì sao `--runtime-config=api/all=false,api/v1=true` lại cho kết quả "tắt tất cả trừ v1" mà
   không phải "tắt sạch mọi thứ"?
3. `api/legacy` có phải là "mọi API phiên bản cũ" (ví dụ mọi API v1beta1) không? Nếu không,
   nó gồm những gì?
4. Trên cluster lab của bạn, flag này thuộc về tiến trình nào và tiến trình đó đang chạy ở
   đâu — bạn sẽ sửa nó bằng cơ chế nào đã học trong CP5?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Tác động vào **API server** (`kube-apiserver`), bằng flag **`--runtime-config=api/<version>`**.
   Giá trị là **một danh sách các phiên bản API phân tách bằng dấu phẩy**, trong đó giá trị
   đứng sau ghi đè giá trị đứng trước.
2. Vì **giá trị đứng sau ghi đè giá trị đứng trước**: `api/all=false` tắt mọi API đã biết,
   nhưng `api/v1=true` đứng sau nên ghi đè lại riêng cho v1 — kết quả là chỉ v1 còn bật. Nếu
   chỉ nhìn `api/all=false` mà kết luận "tắt hết" là bỏ qua quy tắc ghi đè theo thứ tự.
3. **Không.** Đây là chỗ dễ nhầm: `api/legacy` không có nghĩa là "phiên bản cũ" nói chung, mà
   chỉ gồm các API **đã bị deprecated một cách tường minh** theo chính sách deprecation. Một
   API v1beta1 chưa bị deprecated không thuộc `api/legacy`.
4. Flag thuộc tiến trình **`kube-apiserver`**, chạy dưới dạng **static pod trên `k8s-master`**.
   Cách sửa dùng lại cơ chế của bài [196](196-configure-feature-gates-vi.md): thêm flag vào
   danh sách `command` trong manifest `/etc/kubernetes/manifests/kube-apiserver.yaml`; lưu
   file xong static pod tự khởi động lại, không cần restart thủ công.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
