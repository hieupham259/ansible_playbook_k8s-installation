# Sử dụng Custom Resource (Use Custom Resources)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/>
>
> Tìm hiểu cách mở rộng Kubernetes API bằng các kiểu tài nguyên do bạn tự định nghĩa
> (custom resource).

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 28 — Mở rộng Kubernetes](00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes),
bài 3/11 · Phần II không có lab riêng: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) và tự chấm bằng **Checkpoint** ghi ở cuối mục giai đoạn
28. Trang này là mục lục con, không có gì để chạy; phần thực hành nằm ở hai trang con của nó.

**Đây là nhánh bạn đã làm nhiều nhất, nhưng chưa đọc đủ.** Ở
[Lab 14 — CRD và Operator](labs/LAB-14-CRD-VA-OPERATOR.md) của giai đoạn 14 bạn đã chạy trọn một
vòng đời CRD: tạo `CustomResourceDefinition`, apply custom resource, siết schema, đổi bề mặt CLI, mở
subresource `status`, tự tay đóng vai controller, và đi qua cả `status.storedVersions`. Nhưng chính
Lab 14 tự ghi rằng nó **không thay thế** bài [378](378-custom-resource-definitions-vi.md) — ở lab
bạn chỉ dùng đúng những trường mà bài [179](179-custom-resources-vi.md) đã nêu tên, còn mọi tinh
chỉnh sâu hơn thì để dành cho giai đoạn 28. Trang mục lục này cho biết phần để dành đó gồm hai bài.

**Phải hiểu ở lần đọc này:**

- Đây là **trang mục lục** của phần *Tasks → Extend Kubernetes → Use Custom Resources*, chỉ gồm
  **hai** trang con: [378](378-custom-resource-definitions-vi.md) — thêm một kiểu tài nguyên mới vào
  Kubernetes API bằng `CustomResourceDefinition`, và
  [377](377-custom-resource-definition-versioning-vi.md) — quản lý **nhiều version của cùng một
  custom resource** khi lược đồ của nó thay đổi theo thời gian.
- Ranh giới mà trang tự đặt: **phần khái niệm nền tảng về custom resource nằm ở bài
  [179](179-custom-resources-vi.md)**, không lặp lại ở đây. Nhóm này là phần *tasks* — cú pháp và
  thao tác — nên đọc nó với câu hỏi "viết thế nào", không phải "là cái gì" hay "khi nào nên dùng".
- Dùng trang này khi cần định vị: hỏi về **cú pháp và trường của CRD** thì vào 378; hỏi về **đổi
  schema mà không làm vỡ dữ liệu cũ** thì vào 377. Thứ tự lộ trình cũng là thứ tự phụ thuộc.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Link tới bài *Mở rộng Kubernetes API bằng CustomResourceDefinition* | đây mới là trang bản đồ; nội dung nằm ở chính trang đó, và lộ trình gọi nó là **bài xương sống của nhóm** | bài [378](378-custom-resource-definitions-vi.md), ngay sau trang này trong giai đoạn 28 |
| Link tới bài *Version trong CustomResourceDefinition* | chỉ có nghĩa sau khi đã nắm cú pháp một CRD một version; Lab 14 mới chạm phần `storedVersions` bằng `conversion.strategy: None` | bài [377](377-custom-resource-definition-versioning-vi.md), đọc sau bài 378 |
| Câu dẫn về phần khái niệm nền tảng | là bài đã đọc rồi, dẫn ở đây chỉ để phân định ranh giới | bài [179](179-custom-resources-vi.md), đã đọc ở [giai đoạn 14](00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng) |

---

Trang gốc là trang mục lục của phần *Tasks → Extend Kubernetes → Use Custom Resources*: nội dung
trang chỉ gồm danh sách các trang con, được liệt kê dưới đây theo đúng thứ tự hiển thị trên trang
web. Các trang này hướng dẫn cách thêm một kiểu tài nguyên (resource) mới vào Kubernetes API bằng
CustomResourceDefinition, và cách quản lý nhiều version của cùng một custom resource khi lược đồ
(schema) của nó thay đổi theo thời gian. Phần khái niệm nền tảng về custom resource nằm ở bài
[Tài nguyên tùy chỉnh (Custom Resources)](179-custom-resources-vi.md).

## Danh sách các trang trong mục này (Pages in this section)

- [Mở rộng Kubernetes API bằng CustomResourceDefinition (Extend the Kubernetes API with CustomResourceDefinitions)](378-custom-resource-definitions-vi.md)
- [Version trong CustomResourceDefinition (Versions in CustomResourceDefinitions)](377-custom-resource-definition-versioning-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 28:

1. Trang này chia nhánh custom resource thành đúng hai trang con. Mỗi trang trả lời câu hỏi gì, và
   vì sao thứ tự bắt buộc là 378 trước rồi mới tới 377?
2. **Câu bẫy.** Ở Lab 14 bạn đã tạo CRD, apply custom resource, bật subresource `status` và chạy một
   vòng lặp điều khiển. Vậy hai trang con này còn gì để đọc nữa?
3. Trên `lab-k8s-master`, `kubectl get crd` in ra một danh sách khác rỗng ngay cả khi bạn chưa tự
   tạo CRD nào — CNI và dynamic provisioner cài từ Lab 5b và Lab 6a mang CRD theo. Theo ranh giới
   mà trang này đặt: muốn hiểu **cú pháp** của những CRD đó thì đọc trang nào, còn muốn hiểu **khi
   nào nên chọn CRD thay vì một API server thứ hai** thì đọc trang nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Trang [378](378-custom-resource-definitions-vi.md) trả lời **"thêm một kiểu tài nguyên mới vào
   Kubernetes API bằng `CustomResourceDefinition` như thế nào"**; trang
   [377](377-custom-resource-definition-versioning-vi.md) trả lời **"quản lý nhiều version của cùng
   một custom resource ra sao khi lược đồ của nó thay đổi theo thời gian"**. Thứ tự là **quan hệ phụ
   thuộc, không phải sở thích biên tập**: phải có một CRD viết đúng cú pháp đã, rồi mới đặt được câu
   hỏi version thứ hai của nó trông thế nào và chuyển đổi giữa hai version ra sao.
2. **Còn gần như toàn bộ phần cú pháp.** Lab 14 tự ghi rằng nó **không thay thế bài 378** — trang
   xương sống về cú pháp CRD — và rằng ở lab bạn **chỉ dùng đúng những trường mà bài
   [179](179-custom-resources-vi.md) đã nêu tên**, mọi tinh chỉnh sâu hơn để dành cho giai đoạn 28.
   Với bài 377 cũng vậy: lab chỉ chạy nhánh `conversion.strategy: None`, tức trường hợp mọi version
   dùng chung một schema, và **không** làm conversion webhook. Chỗ dễ nhầm ở đây là lẫn giữa *đã
   chạy được một CRD* và *đã đọc hết những gì CRD làm được*.
3. Cú pháp thì đọc **[378](378-custom-resource-definitions-vi.md)** — trang này nói rõ nhóm *tasks*
   là phần cú pháp và thao tác. Còn câu hỏi **nên chọn CRD hay aggregated API** là câu hỏi khái
   niệm, và trang này đẩy nó về **[179](179-custom-resources-vi.md)** — phần khái niệm nền tảng, bạn
   đã đọc ở [giai đoạn 14](00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng). Đó cũng đúng là
   ranh giới mà trang mục lục này tồn tại để giữ.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
