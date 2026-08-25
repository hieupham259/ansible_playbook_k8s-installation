# Quản lý tài nguyên Memory, CPU và API (Manage Memory, CPU, and API Resources)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 25 — Quản trị tài nguyên theo namespace](00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace),
bài 2/7 · Phần II không có lab riêng: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md), tự chấm bằng **Checkpoint giai đoạn 25**.

Đây là **trang mục lục**, không phải bài học: nó chỉ giới thiệu sáu trang con. Đọc nó để nắm
bản đồ, rồi làm lần lượt sáu trang con trước khi sang bài 3/7.

**Phải hiểu ở lần đọc này:**

- Sáu trang con thuộc **hai cơ chế khác nhau**: bốn trang đầu tác động lên **từng Pod/container**
  trong namespace, hai trang cuối đặt **giới hạn tổng thể cho cả namespace** (quota memory/CPU và
  quota số Pod). Đây đúng là cặp khái niệm mà Checkpoint giai đoạn 25 bắt bạn phân biệt.
- Trong bốn trang đầu, phân biệt hai việc mà chính lời mô tả của trang gốc đã tách ra: **đặt giá
  trị mặc định** ([232](232-memory-default-namespace-vi.md), [230](230-cpu-default-namespace-vi.md)
  — Pod mới được cấu hình sẵn limit) và **định nghĩa khoảng hợp lệ**
  ([231](231-memory-constraint-namespace-vi.md), [229](./229-cpu-constraint-namespace-vi.md) — Pod
  mới phải nằm trong khoảng bạn cấu hình).
- Quota có hai cách đo, và cả hai đều có trang riêng: đo theo **lượng tài nguyên**
  ([233](233-quota-memory-cpu-namespace-vi.md)) và đo theo **số lượng object**
  ([234](234-quota-pod-namespace-vi.md) — hạn chế số Pod tạo được).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Nội dung chi tiết của sáu trang con (manifest LimitRange/ResourceQuota, thông báo từ chối cụ thể) | trang này chỉ là mục lục, mỗi trang con là một bài thực hành riêng | chính sáu trang con, đọc ngay sau trang này trong [giai đoạn 25](00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace) |
| Quota cho các loại object khác ngoài Pod | mục lục này chỉ liệt kê quota memory/CPU và quota số Pod | bài [252 — Cấu hình Quota cho các đối tượng API](252-quota-api-object-vi.md), bài 3/7 của giai đoạn 25 |

---

Đây là trang mục lục của mục tác vụ (task) *Manage Memory, CPU, and API Resources* trong tài
liệu Kubernetes: các tác vụ dành cho người quản trị cluster để đặt giá trị mặc định, ràng buộc
và quota cho tài nguyên memory, CPU trong từng namespace.

---

Các trang trong mục này:

* [Cấu hình memory request và limit mặc định cho một Namespace (Configure Default Memory Requests and Limits for a Namespace)](232-memory-default-namespace-vi.md)

  Định nghĩa memory resource limit mặc định cho một namespace, để mọi Pod mới trong namespace
  đó được cấu hình sẵn memory resource limit.

* [Cấu hình CPU request và limit mặc định cho một Namespace (Configure Default CPU Requests and Limits for a Namespace)](230-cpu-default-namespace-vi.md)

  Định nghĩa CPU resource limit mặc định cho một namespace, để mọi Pod mới trong namespace đó
  được cấu hình sẵn CPU resource limit.

* [Cấu hình ràng buộc memory tối thiểu và tối đa cho một Namespace (Configure Minimum and Maximum Memory Constraints for a Namespace)](231-memory-constraint-namespace-vi.md)

  Định nghĩa một khoảng giá trị memory resource limit hợp lệ cho một namespace, để mọi Pod
  mới trong namespace đó nằm trong khoảng mà bạn cấu hình.

* [Cấu hình ràng buộc CPU tối thiểu và tối đa cho một Namespace (Configure Minimum and Maximum CPU Constraints for a Namespace)](./229-cpu-constraint-namespace-vi.md)

  Định nghĩa một khoảng giá trị CPU resource limit hợp lệ cho một namespace, để mọi Pod mới
  trong namespace đó nằm trong khoảng mà bạn cấu hình.

* [Cấu hình quota memory và CPU cho một Namespace (Configure Memory and CPU Quotas for a Namespace)](233-quota-memory-cpu-namespace-vi.md)

  Định nghĩa giới hạn tổng thể về tài nguyên memory và CPU cho một namespace.

* [Cấu hình quota Pod cho một Namespace (Configure a Pod Quota for a Namespace)](234-quota-pod-namespace-vi.md)

  Hạn chế số lượng Pod bạn có thể tạo trong một namespace.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại trang mục lục là đủ để bắt đầu sáu trang con:

1. Sáu trang con chia thành hai cơ chế. Kể tên hai cơ chế đó và xếp đúng từng trang vào nhóm của
   nó — đâu là nhóm tác động lên từng Pod, đâu là nhóm đặt trần cho cả namespace?
2. **Câu bẫy.** Trang [232](232-memory-default-namespace-vi.md) nói "định nghĩa memory resource
   limit **mặc định**", còn trang [231](231-memory-constraint-namespace-vi.md) nói "định nghĩa một
   **khoảng giá trị hợp lệ**". Chỉ đặt mặc định thôi thì có chặn được một Pod tự khai `limits`
   memory cực lớn không?
3. Hai worker của bạn — `lab-k8s-worker1` và `lab-k8s-worker2` — mỗi máy 2 vCPU và 6 GB RAM. Bạn
   muốn namespace `dev` không bao giờ dùng quá 2 CPU tính trên toàn cluster, đồng thời không
   container nào trong đó xin quá `500m`. Theo mục lục này, bạn phải mở **hai** trang con nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Nhóm tác động lên từng Pod/container**: [232](232-memory-default-namespace-vi.md) và
   [230](230-cpu-default-namespace-vi.md) (mặc định memory/CPU),
   [231](231-memory-constraint-namespace-vi.md) và [229](./229-cpu-constraint-namespace-vi.md)
   (ràng buộc min/max memory/CPU) — cả bốn đều nói "mọi Pod **mới** trong namespace đó".
   **Nhóm đặt trần cho cả namespace**: [233](233-quota-memory-cpu-namespace-vi.md) (giới hạn
   **tổng thể** về memory và CPU) và [234](234-quota-pod-namespace-vi.md) (hạn chế **số lượng**
   Pod tạo được).
2. **Không.** Đặt mặc định chỉ nói: Pod nào **không khai** thì được điền sẵn giá trị đó — nó
   không phát biểu gì về Pod có khai. Muốn chặn giá trị quá lớn phải dùng trang
   [231](231-memory-constraint-namespace-vi.md): định nghĩa **khoảng hợp lệ**, để mọi Pod mới đều
   phải nằm trong khoảng bạn cấu hình. Trực giác "đặt mặc định là đã giới hạn rồi" sai vì hai
   trang này giải quyết hai bài toán khác nhau; thường phải dùng cả hai.
3. Trần 2 CPU cho **cả namespace** là quota tổng thể → trang
   [233 — Cấu hình quota memory và CPU cho một Namespace](233-quota-memory-cpu-namespace-vi.md).
   Trần `500m` cho **từng container** là ràng buộc tối đa → trang
   [229 — Cấu hình ràng buộc CPU tối thiểu và tối đa cho một Namespace](./229-cpu-constraint-namespace-vi.md).
   Hai yêu cầu, hai cơ chế, không thay thế được cho nhau.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
