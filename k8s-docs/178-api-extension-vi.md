# Mở rộng Kubernetes API (Extending the Kubernetes API)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 14](00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng), bài 2/7 ·
Kiểm chứng ở Lab 14 (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Giai đoạn này lộ trình ghi rõ là **dành cho platform administrator / người phát triển operator**.

**Bài này chỉ dài hai gạch đầu dòng.** Nó là trang mục lục của nhánh *API extension*, đọc hết
trong hai phút. Giá trị của nó là đóng khung đúng một câu hỏi: có **đúng hai cách** thêm custom
resource, và hai bài kế tiếp sẽ mổ từng cách. Đừng cố hiểu sâu ở đây.

**Phải hiểu ở lần đọc này:**

- Kubernetes cung cấp **đúng hai cách** thêm custom resource: **CRD** và **tầng tổng hợp
  (aggregation layer)**. Không có cách thứ ba.
- Với CRD, bạn khai báo API group, kind và schema; **control plane của Kubernetes phục vụ và
  đảm nhận việc lưu trữ** custom resource đó — bạn không phải viết và vận hành API server nào.
- Với tầng tổng hợp, **bạn viết và triển khai API server của riêng mình**; nó nằm phía sau API
  server chính, và API server chính đóng vai trò **proxy**, ủy quyền request tới server của bạn.
- Điểm chung của hai cách: với client, Kubernetes API trông như đã được mở rộng — cùng một
  endpoint, cùng các công cụ.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Chi tiết CRD: schema, validation, versioning | bài này chỉ nêu tên | bài [179](179-custom-resources-vi.md) |
| Tiêu chí chọn giữa CRD và aggregated API | có ba bảng so sánh riêng | bài [179](179-custom-resources-vi.md) |
| Cơ chế APIService và cách tầng tổng hợp proxy request | bài này chỉ nói "nằm phía sau API server chính" | bài [180](180-apiserver-aggregation-vi.md) |
| Vì sao cần custom controller đi kèm custom resource | là chủ đề của mẫu operator | bài [181](181-operator-vi.md) |

---

Custom resource là các phần mở rộng của Kubernetes API. Kubernetes cung cấp hai cách để thêm custom resource vào cluster của bạn:

- Cơ chế [CustomResourceDefinition](179-custom-resources-vi.md)
  (CRD) cho phép bạn định nghĩa một custom API mới theo cách khai báo (declarative) với API group, kind và
  schema do bạn chỉ định.
  Control plane của Kubernetes phục vụ và đảm nhận việc lưu trữ custom resource của bạn. CRD cho phép bạn
  tạo các loại resource mới cho cluster mà không cần viết và vận hành một API server tùy chỉnh.
- [Tầng tổng hợp (aggregation layer)](180-apiserver-aggregation-vi.md)
  nằm phía sau API server chính, và API server chính đóng vai trò như một proxy.
  Cách bố trí này được gọi là API Aggregation (AA), cho phép bạn cung cấp
  các hiện thực chuyên biệt cho các custom resource của mình bằng cách viết và
  triển khai API server của riêng bạn.
  API server chính ủy quyền (delegate) các request tới API server của bạn đối với những custom API mà bạn chỉ định,
  giúp chúng khả dụng cho tất cả các client của nó.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 14:

1. Bài nói có mấy cách thêm custom resource vào cluster? Với mỗi cách, **ai** lưu trữ dữ liệu
   của custom resource đó?
2. Với tầng tổng hợp, client `kubectl` gọi thẳng vào API server của bạn hay gọi vào API server
   chính? Bài dùng đúng hai từ nào để mô tả vai trò của API server chính?
3. Trên cluster lab v1.35.6 của bạn chưa cài phần mở rộng nào, nên `kubectl api-resources` chỉ
   in ra resource dựng sẵn. Nếu bạn thêm một CRD thì resource mới xuất hiện ở đó nhờ thành phần
   nào phục vụ? Còn nếu bạn đăng ký một aggregated API thì nhờ thành phần nào?
4. Câu nào trong bài cho biết dùng CRD thì bạn **không** phải làm việc gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Đúng hai cách: CRD và tầng tổng hợp.** Với **CRD**, "control plane của Kubernetes phục vụ
   và đảm nhận việc lưu trữ custom resource của bạn" — tức kho lưu trữ của cluster. Với **tầng
   tổng hợp**, dữ liệu do **API server của chính bạn** đảm nhận, vì bạn tự "cung cấp các hiện
   thực chuyên biệt cho các custom resource của mình bằng cách viết và triển khai API server của
   riêng bạn".
2. Client vẫn gọi vào **API server chính**. Bài mô tả API server chính đóng vai trò **proxy** và
   **ủy quyền (delegate)** request tới API server của bạn đối với những custom API bạn chỉ định,
   "giúp chúng khả dụng cho tất cả các client của nó". Đây là điểm dễ nhầm: aggregated API
   **không** bắt client biết tới một endpoint thứ hai.
3. Với CRD: **chính control plane / kube-apiserver** phục vụ và lưu trữ, không có tiến trình nào
   được thêm. Với aggregated API: **API server do bạn triển khai** phục vụ, còn kube-apiserver
   chỉ chuyển tiếp request tới đó.
4. Câu "CRD cho phép bạn tạo các loại resource mới cho cluster **mà không cần viết và vận hành
   một API server tùy chỉnh**". Đó chính là ranh giới phân chia hai cách: CRD bỏ được cả việc
   viết lẫn việc *vận hành* một tiến trình thêm.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
