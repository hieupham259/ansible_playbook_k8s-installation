# Các container tạm thời (Ephemeral Containers)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 3 → nhóm [3a](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời), bài 8/11 · Kiểm chứng
ở [Lab 3a](labs/LAB-3A-POD-VA-VONG-DOI.md).

Bài ngắn nhất nhóm và là **công cụ xử lý sự cố**, không phải cơ chế vận hành. Bạn sẽ dùng nó
nhiều ở các giai đoạn sau, nên lần đọc này chỉ cần nhớ nó giải quyết tình huống nào và bị cấm
những gì.

**Phải hiểu ở lần đọc này:**

- Không thể thêm container vào một Pod sau khi Pod đã được tạo. Ephemeral container là ngoại lệ
  dành riêng cho **hành động do người dùng khởi xướng để kiểm tra một Pod có sẵn** — dùng để
  inspect dịch vụ, **không** để xây dựng ứng dụng.
- Lý do nó không dùng được cho ứng dụng: ephemeral container **thiếu đảm bảo về tài nguyên và về
  việc thực thi**, và **không bao giờ được tự động khởi động lại**.
- Các trường bị cấm và lý do: không có port nên `ports`, `livenessProbe`, `readinessProbe` đều
  không được phép; việc phân bổ tài nguyên của Pod là bất biến nên `resources` cũng không.
- Cách thêm: qua handler `ephemeralcontainers` của API, **không** thêm được bằng `kubectl edit`.
  Đã thêm rồi thì **không sửa và không xóa được**.
- Khi nào cần đến nó thay vì `kubectl exec`: container đã crash, hoặc image không có shell và
  tiện ích gỡ lỗi — trường hợp điển hình là distroless image. Bật chia sẻ process namespace thì
  bạn nhìn được tiến trình của các container khác trong Pod.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ghi chú ephemeral container không được static pod hỗ trợ | chưa học static Pod | giai đoạn 3, nhóm 3b — bài [58](58-static-pods-vi.md) |
| Thao tác bật *chia sẻ process namespace* | là cấu hình Pod, chỉ làm khi thật sự cần debug | Lab 3a |
| Mục *Tiếp theo* — hướng dẫn gỡ lỗi pod bằng ephemeral container | là tài liệu tác vụ, làm khi có Pod hỏng thật | Lab 3a |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.25 [stable]`

Trang này cung cấp cái nhìn tổng quan về ephemeral container (container tạm thời): một
loại container đặc biệt chạy tạm thời trong một Pod có sẵn nhằm thực hiện các hành động
do người dùng khởi xướng, chẳng hạn như xử lý sự cố (troubleshooting). Bạn dùng
ephemeral container để kiểm tra (inspect) các dịch vụ chứ không phải để xây dựng ứng
dụng.

## Hiểu về ephemeral container (Understanding ephemeral containers)

Pod là khối xây dựng nền tảng của các ứng dụng Kubernetes. Vì Pod được thiết kế để có thể
vứt bỏ và thay thế được (disposable and replaceable), bạn không thể thêm một container
vào Pod sau khi nó đã được tạo. Thay vào đó, bạn thường xóa và thay thế các Pod theo cách
có kiểm soát bằng các deployment.

Tuy nhiên, đôi khi vẫn cần kiểm tra trạng thái của một Pod có sẵn, ví dụ để xử lý một lỗi
khó tái hiện. Trong những trường hợp này, bạn có thể chạy một ephemeral container trong
một Pod có sẵn để kiểm tra trạng thái của nó và chạy các lệnh tùy ý.

### Ephemeral container là gì? (What is an ephemeral container?)

Ephemeral container khác các container khác ở chỗ chúng thiếu các đảm bảo về tài nguyên
hay việc thực thi, và chúng sẽ không bao giờ được tự động khởi động lại, do đó chúng
không phù hợp để xây dựng ứng dụng. Ephemeral container được mô tả bằng cùng
`ContainerSpec` như các container thông thường, nhưng nhiều trường không tương thích và
không được phép sử dụng cho ephemeral container.

- Ephemeral container không được có port, vì vậy các trường như `ports`,
  `livenessProbe`, `readinessProbe` không được phép sử dụng.
- Việc phân bổ tài nguyên của Pod là bất biến (immutable), vì vậy không được phép đặt
  `resources`.
- Để xem danh sách đầy đủ các trường được phép, hãy xem
  [tài liệu tham khảo về EphemeralContainer](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#ephemeralcontainer-v1-core).

Ephemeral container được tạo thông qua một handler đặc biệt tên là `ephemeralcontainers`
trong API thay vì được thêm trực tiếp vào `pod.spec`, do đó không thể thêm một ephemeral
container bằng `kubectl edit`.

Giống như các container thông thường, bạn không thể thay đổi hoặc xóa một ephemeral
container sau khi đã thêm nó vào một Pod.

> **Ghi chú:**
> Ephemeral container không được hỗ trợ bởi
> [static pod](293-static-pod-tasks-vi.md).

## Công dụng của ephemeral container (Uses for ephemeral containers)

Ephemeral container hữu ích cho việc xử lý sự cố theo cách tương tác khi `kubectl exec`
là không đủ, do container đã bị crash hoặc container image không bao gồm các tiện ích gỡ
lỗi (debugging utilities).

Đặc biệt, [distroless image](https://github.com/GoogleContainerTools/distroless) cho phép
bạn triển khai các container image tối giản nhằm giảm bề mặt tấn công (attack surface)
và mức độ phơi nhiễm trước lỗi cũng như lỗ hổng bảo mật. Vì distroless image không bao
gồm shell hay bất kỳ tiện ích gỡ lỗi nào, rất khó để xử lý sự cố với distroless image
nếu chỉ dùng `kubectl exec`.

Khi sử dụng ephemeral container, sẽ hữu ích nếu bật
[chia sẻ process namespace](292-share-process-namespace-vi.md)
để bạn có thể xem các tiến trình (process) trong những container khác.

## Tiếp theo (What's next)

* Tìm hiểu cách [gỡ lỗi pod bằng ephemeral container](300-debug-running-pod-vi.md#ephemeral-container).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Vì sao ephemeral container không được đặt `resources`, `ports` hay `livenessProbe`?
2. Một Pod trên `lab-k8s-worker2` dùng image distroless và container chính đã crash. Vì sao
   `kubectl exec` không giúp được gì, ephemeral container giải quyết ra sao, và bạn nên bật thêm
   gì để nhìn được tiến trình của container kia?
3. Bạn thêm nhầm một ephemeral container. Sửa lại bằng `kubectl edit` được không?
4. Vì sao bài nói ephemeral container "không phù hợp để xây dựng ứng dụng"?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Vì hai ràng buộc khác nhau. **`ports` — và kéo theo `livenessProbe`, `readinessProbe` — bị cấm
   vì ephemeral container không được có port.** **`resources` bị cấm vì việc phân bổ tài nguyên
   của Pod là bất biến**: Pod đã tồn tại rồi, không thể xin thêm phần tài nguyên mới cho nó. Đây
   cũng là lý do ephemeral container được mô tả bằng cùng `ContainerSpec` như container thường
   nhưng nhiều trường trong đó không được phép dùng.
2. `kubectl exec` cần một container còn sống và một shell trong image. Ở đây **container đã crash**
   và **distroless image không có shell hay bất kỳ tiện ích gỡ lỗi nào** — đó chính xác là hai
   tình huống bài nêu. Ephemeral container cho bạn **chạy một container tạm mang sẵn công cụ ngay
   trong Pod đang có** để kiểm tra trạng thái và chạy lệnh tùy ý. Nên **bật chia sẻ process
   namespace** để thấy được các tiến trình trong những container khác.
3. **Không.** Ephemeral container được tạo qua handler đặc biệt **`ephemeralcontainers`** trong
   API chứ không thêm trực tiếp vào `pod.spec`, nên `kubectl edit` không thêm được. Và giống các
   container thông thường, **đã thêm vào Pod rồi thì bạn không thay đổi hay xóa nó được nữa**.
4. Vì nó **thiếu các đảm bảo về tài nguyên và về việc thực thi**, và **không bao giờ được tự động
   khởi động lại**. Nói cách khác không có gì bảo đảm nó chạy, chạy đủ, hay chạy lại sau khi
   chết — ba thứ tối thiểu mà một thành phần ứng dụng cần.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
