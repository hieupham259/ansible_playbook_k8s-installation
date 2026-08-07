# Cơ chế admission bảo mật Pod (Pod Security Admission)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/pod-security-admission/>
>
> Tổng quan về Pod Security Admission Controller — admission controller có thể thực thi các Tiêu chuẩn bảo mật Pod (Pod Security Standards).

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 9](LO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), bài 7/18 · Kiểm chứng ở Lab 9b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài [115](115-pod-security-standards-vi.md) định nghĩa ba profile; bài này là **cách áp chúng
vào cluster thật**. Nó cũng chính là ví dụ cụ thể cho chặng thứ ba trong bài
[119](119-controlling-access-vi.md) — một admission controller tích hợp sẵn, quyết định dựa
trên **nội dung** của Pod. Bài ngắn, đọc kỹ toàn bộ trừ mục metric.

**Phải hiểu ở lần đọc này:**

- Pod Security là admission controller **tích hợp sẵn**; các hạn chế được áp **ở cấp namespace**
  và **tại thời điểm Pod được tạo**.
- Ba chế độ và hậu quả khác nhau của mỗi chế độ khi có vi phạm: **enforce** → Pod bị **từ
  chối**; **audit** → thêm một audit annotation vào sự kiện trong audit log, **Pod vẫn được
  phép**; **warn** → hiện cảnh báo cho người dùng, **Pod vẫn được phép**. Một namespace có thể
  bật nhiều chế độ cùng lúc, mỗi chế độ đặt ở một mức khác nhau.
- Cách khai báo bằng label trên namespace: `pod-security.kubernetes.io/<MODE>: <LEVEL>`, với
  `MODE` là `enforce`/`audit`/`warn` và `LEVEL` là `privileged`/`baseline`/`restricted`; cộng
  label tùy chọn `pod-security.kubernetes.io/<MODE>-version` để **ghim chính sách vào một phiên
  bản minor** thay vì `latest`.
- Điểm bất đối xứng dễ mắc bẫy: **audit và warn được áp cho cả workload resource** (Deployment,
  Job…) để phát hiện vi phạm sớm, nhưng **enforce KHÔNG áp cho workload resource** — nó chỉ áp
  cho các Pod object được tạo ra.
- Miễn trừ (exemption) được cấu hình **tĩnh** trong cấu hình Admission Controller, theo ba
  chiều: **username**, **RuntimeClassName**, **namespace**. Request thỏa tiêu chí miễn trừ sẽ bị
  bỏ qua hoàn toàn — cả `enforce`, `audit` lẫn `warn`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Danh sách trường được miễn kiểm tra khi cập nhật Pod | là chi tiết tra cứu lúc gỡ lỗi | giai đoạn 9, khi làm Lab 9b |
| Annotation seccomp/AppArmor đã lỗi thời trong danh sách đó | là dạng cũ, không dùng nữa | bài [127](127-linux-kernel-security-vi.md) |
| Cách cấu hình Admission Controller bằng file trên node control plane | là thao tác sửa cấu hình API server | giai đoạn 8, bài [03](03-control-plane-flags-vi.md) |
| Ba metric `pod_security_*` | chưa học endpoint `/metrics` và Prometheus | giai đoạn 11 |
| Link *Di trú từ PodSecurityPolicy* | chỉ cần khi tiếp quản cluster rất cũ | bài [117](117-pod-security-policy-vi.md) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.25 [stable]`

[Tiêu chuẩn bảo mật Pod (Pod Security Standards)](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
của Kubernetes định nghĩa các mức độ cô lập (isolation) khác nhau cho Pod. Các tiêu chuẩn này
cho phép bạn xác định cách bạn muốn hạn chế hành vi của các pod một cách rõ ràng và nhất quán.

Kubernetes cung cấp sẵn một admission controller tích hợp tên là _Pod Security_ để thực thi
các Tiêu chuẩn bảo mật Pod. Các hạn chế bảo mật pod được áp dụng ở cấp namespace
tại thời điểm pod được tạo.

### Thực thi Pod Security admission tích hợp sẵn (Built-in Pod Security admission enforcement)

Trang này là một phần của bộ tài liệu dành cho Kubernetes v1.36.
Nếu bạn đang chạy một phiên bản Kubernetes khác, hãy tham khảo tài liệu của phiên bản đó.

## Các mức bảo mật Pod (Pod Security levels)

Pod Security admission đặt ra các yêu cầu đối với [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
của Pod và các trường liên quan khác, theo ba mức được định nghĩa bởi
[Tiêu chuẩn bảo mật Pod](https://kubernetes.io/docs/concepts/security/pod-security-standards):
`privileged`, `baseline` và `restricted`. Hãy xem trang
[Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards)
để tìm hiểu chuyên sâu về các yêu cầu đó.

## Các label Pod Security Admission cho namespace (Pod Security Admission labels for namespaces)

Sau khi tính năng được bật hoặc webhook được cài đặt, bạn có thể cấu hình các namespace để
xác định chế độ kiểm soát admission mà bạn muốn dùng cho bảo mật pod trong từng namespace.
Kubernetes định nghĩa một tập các label mà bạn có thể đặt để chỉ định
mức Pod Security Standard định sẵn nào sẽ được dùng cho một namespace. Label mà bạn chọn
xác định hành động control plane sẽ thực hiện nếu phát hiện một vi phạm tiềm ẩn:

*Các chế độ của Pod Security Admission (Pod Security Admission modes)*

Chế độ (Mode) | Mô tả
:---------|:------------
**enforce** | Vi phạm chính sách sẽ khiến pod bị từ chối.
**audit** | Vi phạm chính sách sẽ kích hoạt việc thêm một audit annotation vào sự kiện được ghi trong [audit log](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/), nhưng pod vẫn được cho phép.
**warn** | Vi phạm chính sách sẽ kích hoạt một cảnh báo hiển thị cho người dùng, nhưng pod vẫn được cho phép.

Một namespace có thể cấu hình một, một vài hoặc tất cả các chế độ, thậm chí đặt
mức khác nhau cho từng chế độ khác nhau.

Với mỗi chế độ, có hai label xác định chính sách được sử dụng:

```yaml
# Label mức theo từng chế độ, cho biết mức chính sách nào được áp dụng cho chế độ đó.
#
# MODE phải là một trong `enforce`, `audit` hoặc `warn`.
# LEVEL phải là một trong `privileged`, `baseline` hoặc `restricted`.
pod-security.kubernetes.io/<MODE>: <LEVEL>

# Tùy chọn: label phiên bản theo từng chế độ, có thể dùng để ghim chính sách vào
# phiên bản đi kèm một phiên bản minor nhất định của Kubernetes (ví dụ v1.36).
#
# MODE phải là một trong `enforce`, `audit` hoặc `warn`.
# VERSION phải là một phiên bản minor hợp lệ của Kubernetes, hoặc `latest`.
pod-security.kubernetes.io/<MODE>-version: <VERSION>
```

Xem [Thực thi Pod Security Standards bằng label của namespace](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels)
để biết ví dụ sử dụng.

## Tài nguyên workload và Pod template (Workload resources and Pod templates) {#workload-resources-and-pod-templates}

Pod thường được tạo một cách gián tiếp, thông qua việc tạo một
[workload object](https://kubernetes.io/docs/concepts/workloads/controllers/) như Deployment
hoặc Job. Workload object định nghĩa một _Pod template_ và một controller
của workload resource sẽ tạo các Pod dựa trên template đó. Để giúp phát hiện vi phạm sớm,
cả chế độ audit và warn đều được áp dụng cho các workload resource. Tuy nhiên, chế độ enforce
**không** được áp dụng cho workload resource, mà chỉ áp dụng cho các pod object được tạo ra.

## Miễn trừ (Exemptions)

Bạn có thể định nghĩa các _miễn trừ_ (exemption) khỏi việc thực thi bảo mật pod để cho phép
tạo những pod lẽ ra đã bị cấm bởi chính sách gắn với một namespace nhất định.
Miễn trừ có thể được cấu hình tĩnh trong
[cấu hình Admission Controller](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-admission-controller/#configure-the-admission-controller).

Miễn trừ phải được liệt kê một cách tường minh. Các request thỏa mãn tiêu chí miễn trừ sẽ bị
Admission Controller _bỏ qua_ (mọi hành vi `enforce`, `audit` và `warn` đều được bỏ qua).
Các chiều miễn trừ bao gồm:

- **Usernames:** request từ người dùng có username đã xác thực (hoặc được mạo danh — impersonated)
  thuộc diện miễn trừ sẽ bị bỏ qua.
- **RuntimeClassNames:** pod và [workload resource](#workload-resources-and-pod-templates) chỉ định
  một runtime class name thuộc diện miễn trừ sẽ bị bỏ qua.
- **Namespaces:** pod và [workload resource](#workload-resources-and-pod-templates) nằm trong một namespace thuộc diện miễn trừ sẽ bị bỏ qua.

> **Thận trọng:**
>
> Hầu hết pod được tạo bởi một controller để đáp ứng một
> [workload resource](#workload-resources-and-pod-templates), nghĩa là việc miễn trừ một người dùng cuối
> chỉ miễn trừ họ khỏi việc thực thi khi tạo pod trực tiếp, chứ không phải khi tạo một workload resource.
> Các service account của controller (chẳng hạn `system:serviceaccount:kube-system:replicaset-controller`)
> nói chung không nên được miễn trừ, vì làm vậy sẽ ngầm miễn trừ cho bất kỳ người dùng nào có thể tạo
> workload resource tương ứng.

Các cập nhật đối với những trường pod sau đây được miễn kiểm tra chính sách, nghĩa là nếu một
request cập nhật pod chỉ thay đổi các trường này, request đó sẽ không bị từ chối ngay cả khi pod
đang vi phạm mức chính sách hiện tại:

- Mọi cập nhật metadata **ngoại trừ** các thay đổi đối với annotation seccomp hoặc AppArmor:
  - `seccomp.security.alpha.kubernetes.io/pod` (đã lỗi thời — deprecated)
  - `container.seccomp.security.alpha.kubernetes.io/*` (đã lỗi thời)
  - `container.apparmor.security.beta.kubernetes.io/*` (đã lỗi thời)
- Các cập nhật hợp lệ đối với `.spec.activeDeadlineSeconds`
- Các cập nhật hợp lệ đối với `.spec.tolerations`

## Số liệu đo (Metrics)

Dưới đây là các metric Prometheus được kube-apiserver cung cấp:

- `pod_security_errors_total`: Metric này cho biết số lượng lỗi ngăn cản quá trình đánh giá bình thường.
  Các lỗi không nghiêm trọng (non-fatal) có thể dẫn đến việc profile restricted mới nhất được dùng để thực thi.
- `pod_security_evaluations_total`: Metric này cho biết số lần đánh giá chính sách đã diễn ra,
  không tính các request bị bỏ qua hoặc được miễn trừ trong quá trình xuất số liệu.
- `pod_security_exemptions_total`: Metric này cho biết số lượng request được miễn trừ, không tính
  các request bị bỏ qua hoặc nằm ngoài phạm vi.

## Tiếp theo (What's next)

- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards)
- [Thực thi Pod Security Standards](https://kubernetes.io/docs/setup/best-practices/enforcing-pod-security-standards)
- [Thực thi Pod Security Standards bằng cách cấu hình Admission Controller tích hợp sẵn](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-admission-controller)
- [Thực thi Pod Security Standards bằng label của namespace](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels)

Nếu bạn đang chạy một phiên bản Kubernetes cũ hơn và muốn nâng cấp
lên một phiên bản Kubernetes không còn PodSecurityPolicy,
hãy đọc [Di trú từ PodSecurityPolicy sang PodSecurity Admission Controller tích hợp sẵn](https://kubernetes.io/docs/tasks/configure-pod-container/migrate-from-psp).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 9:

1. Bạn gắn label `pod-security.kubernetes.io/enforce: restricted` cho một namespace, rồi
   `kubectl apply` một Deployment có container `privileged: true`. Lệnh apply thành công hay
   thất bại? Pod có chạy không? Vì sao?
2. Ba chế độ của Pod Security Admission là gì, và mỗi chế độ làm gì khi phát hiện vi phạm?
3. Trên cluster lab, namespace `default` đang có sẵn workload chạy. Bạn muốn biết chúng có đạt
   mức `restricted` hay không **mà chưa chặn gì cả**. Dùng tổ hợp label nào?
4. Label `pod-security.kubernetes.io/<MODE>-version` để làm gì, và vì sao nên đặt nó thay vì
   để `latest`?
5. Vì sao bài cảnh báo không nên miễn trừ các service account của controller, chẳng hạn
   `system:serviceaccount:kube-system:replicaset-controller`?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Lệnh apply thành công, nhưng Pod không bao giờ chạy.** Đây là điểm bất đối xứng dễ nhầm
   nhất của bài: chế độ **enforce không được áp dụng cho workload resource**, mà **chỉ áp dụng
   cho các pod object được tạo ra**. Deployment vì thế được API server chấp nhận bình thường;
   đến khi controller tạo Pod thì Pod mới bị từ chối, và bạn thấy Deployment có 0 replica sẵn
   sàng. Nếu namespace cũng bật `warn`, bạn sẽ nhận được cảnh báo ngay lúc apply — vì
   **audit và warn thì có** áp cho workload resource.
2. **enforce**: vi phạm khiến **pod bị từ chối**. **audit**: vi phạm khiến một **audit
   annotation** được thêm vào sự kiện ghi trong audit log, **nhưng pod vẫn được cho phép**.
   **warn**: vi phạm kích hoạt một **cảnh báo hiển thị cho người dùng**, **pod vẫn được cho
   phép**.
3. Đặt **`pod-security.kubernetes.io/warn: restricted`** và
   **`pod-security.kubernetes.io/audit: restricted`**, và **không** đặt `enforce` ở mức đó.
   Bài nói rõ một namespace có thể cấu hình một, một vài hoặc tất cả các chế độ, **thậm chí đặt
   mức khác nhau cho từng chế độ** — nên đây là cách chạy thử trước khi siết.
4. Nó **ghim chính sách vào phiên bản đi kèm một phiên bản minor nhất định của Kubernetes**
   thay vì `latest`. Ghim để nội dung của một mức chính sách không tự thay đổi dưới chân bạn khi
   cluster được nâng cấp lên phiên bản minor mới có định nghĩa profile chặt hơn.
5. Vì **hầu hết Pod được tạo bởi một controller để đáp ứng một workload resource**. Miễn trừ
   service account của controller nghĩa là **ngầm miễn trừ cho bất kỳ người dùng nào có thể tạo
   workload resource tương ứng** — tức là bất kỳ ai tạo được Deployment cũng chạy được Pod vi
   phạm. Ngược lại, miễn trừ một người dùng cuối chỉ miễn trừ họ khi họ **tạo pod trực tiếp**.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
