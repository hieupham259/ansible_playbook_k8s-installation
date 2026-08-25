# Thực thi Pod Security Standards bằng nhãn Namespace (Enforce Pod Security Standards with Namespace Labels)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 9 — Bảo mật và multi-tenancy](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy),
dòng **Thực hành**, bài 5/10 · Kiểm chứng ở
[Lab 9b — Pod Security và hardening](labs/LAB-9B-POD-SECURITY-VA-HARDENING.md), phần B0.3 (tạo
namespace bằng đúng khuôn manifest của bài), B1.2 (selector `!pod-security.kubernetes.io/enforce`)
và B3.5 (`kubectl label --dry-run=server`).

Bài này là mặt còn lại của bài [282](282-enforce-standards-admission-controller-vi.md): cùng một
chuẩn, hai chỗ đặt. Bài 282 đặt ở tầng cluster bằng file cấu hình của `kube-apiserver`; bài này
đặt ở **tầng namespace** bằng nhãn, đổi được bằng một lệnh `kubectl label`. Khi cả hai cùng có,
**nhãn namespace thắng** — giá trị `defaults` của file cấu hình chỉ áp dụng khi nhãn mode tương
ứng không được đặt. Hệ quả về quyền của việc "chính sách nằm trên một nhãn" — ai sửa được
namespace thì sửa được cả mức chính sách — nằm ở mục 1 của bài
[286](286-migrate-from-psp-vi.md), đọc ngay sau đây.

**Phải hiểu ở lần đọc này:**

- Cú pháp nhãn: `pod-security.kubernetes.io/<mode>` chọn **level**, `pod-security.kubernetes.io/<mode>-version`
  ghim **phiên bản chính sách**; ba mode `enforce`, `audit`, `warn` đặt độc lập nhau. Khuôn mẫu mà
  manifest `my-baseline-namespace` dạy: `enforce` ở mức bạn **bảo đảm được ngay**, còn `audit` và
  `warn` ở mức **chặt hơn mà bạn đang nhắm tới** (mục *Yêu cầu chuẩn `baseline` bằng nhãn
  namespace*).
- `kubectl label --dry-run=server` vẫn **chạy đủ các kiểm tra Pod Security** lên những Pod *hiện
  có* trong namespace và trả về cảnh báo cho Pod nào sẽ vi phạm, mà **không** thật sự cập nhật
  chính sách. Đây là cách thử trước khi siết (mục *Thêm nhãn vào các namespace hiện có*).
- Khi nhãn `enforce` (hoặc nhãn version của nó) được thêm hay đổi, admission plugin **kiểm lại
  từng Pod đang chạy** trong namespace theo chính sách mới và trả vi phạm về dưới dạng **cảnh
  báo** — chứ không xóa hay chặn Pod đang chạy. Vì vậy bước đầu an toàn mà bài khuyên là đặt
  `audit` và `warn` cho `--all` mà **không** đặt `enforce`, nhờ đó
  `kubectl get namespaces --selector='!pod-security.kubernetes.io/enforce'` liệt kê được đúng
  những namespace chưa được đánh giá tường minh (ghi chú đầu mục và mục *Áp dụng cho tất cả các
  namespace*).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Nội dung thật sự của ba mức `privileged`, `baseline`, `restricted` — cái gì bị chặn ở mức nào | ở bài này chúng mới chỉ là **giá trị** của nhãn | bài [115](115-pod-security-standards-vi.md), đọc ngay sau Lab 9a trong cùng giai đoạn 9 |
| Cơ chế phía sau nhãn: thứ tự đánh giá, miễn trừ, hành vi khác nhau giữa Pod và workload resource | bài này chỉ là công thức đặt nhãn, không giải thích plugin | bài [116](116-pod-security-admission-vi.md); [Lab 9b](labs/LAB-9B-POD-SECURITY-VA-HARDENING.md) B1.2 và B3 làm rõ từng chế độ |
| Audit annotation mà chế độ `audit` ghi ra khi Pod vi phạm | đọc được nội dung đó cần một audit backend đang chạy, tức phải sửa cờ của `kube-apiserver` | [giai đoạn 22 — Audit và mã hóa dữ liệu](00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu), bài [306](306-audit-vi.md) |
| Con số phiên bản `v1.36` trong các ví dụ nhãn `-version` | là phiên bản của trang gốc, không phải của cluster bạn | phiên bản khóa nằm ở [bảng A1.3 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); [Lab 9b](labs/LAB-9B-POD-SECURITY-VA-HARDENING.md) B0.2 suy nhãn version từ chính cluster |

---

Các namespace có thể được gán nhãn (label) để thực thi các
[Pod Security Standard](./115-pod-security-standards-vi.md). Ba chính sách
[privileged](./115-pod-security-standards-vi.md#privileged),
[baseline](./115-pod-security-standards-vi.md#baseline)
và [restricted](./115-pod-security-standards-vi.md#restricted) bao phủ rộng khắp phổ bảo mật
và được hiện thực bởi admission controller [Pod Security](./116-pod-security-admission-vi.md).

## Trước khi bạn bắt đầu (Before you begin)

Pod Security Admission đã khả dụng theo mặc định trong Kubernetes v1.23, ở trạng thái beta.
Từ phiên bản 1.25 trở đi, Pod Security Admission đạt mức phổ biến rộng rãi (generally
available).

Kubernetes server của bạn phải ở phiên bản v1.36 hoặc mới hơn. Để kiểm tra phiên bản, hãy
nhập `kubectl version`.

## Yêu cầu chuẩn Pod Security Standard `baseline` bằng nhãn namespace (Requiring the `baseline` Pod Security Standard with namespace labels)

Manifest này định nghĩa một Namespace `my-baseline-namespace` mà:

- _Chặn_ mọi pod không thỏa mãn các yêu cầu của chính sách `baseline`.
- Sinh ra một cảnh báo hiển thị cho người dùng và thêm một audit annotation vào bất kỳ pod
  nào được tạo mà không đáp ứng các yêu cầu của chính sách `restricted`.
- Ghim (pin) phiên bản của các chính sách `baseline` và `restricted` vào v1.36.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-baseline-namespace
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: v1.36

    # Chúng ta đặt các nhãn này theo mức `enforce` mà mình _mong muốn_.
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.36
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.36
```

## Thêm nhãn vào các namespace hiện có bằng `kubectl label` (Add labels to existing namespaces with `kubectl label`)

> **Ghi chú:** Khi một nhãn chính sách `enforce` (hoặc nhãn phiên bản) được thêm vào hoặc
> thay đổi, admission plugin sẽ kiểm tra từng pod trong namespace đó theo chính sách mới.
> Các vi phạm được trả về cho người dùng dưới dạng cảnh báo.

Sẽ rất hữu ích khi áp dụng flag `--dry-run` lúc mới bắt đầu đánh giá các thay đổi về profile
bảo mật cho các namespace. Các kiểm tra của Pod Security Standard vẫn được chạy ở chế độ
_dry run_, cho bạn thông tin về việc chính sách mới sẽ đối xử với các pod hiện có như thế
nào, mà không thực sự cập nhật chính sách.

```shell
kubectl label --dry-run=server --overwrite ns --all \
    pod-security.kubernetes.io/enforce=baseline
```

### Áp dụng cho tất cả các namespace (Applying to all namespaces)

Nếu bạn mới bắt đầu với các Pod Security Standard, một bước đầu tiên phù hợp là cấu hình tất
cả các namespace với audit annotation cho một mức nghiêm ngặt hơn, chẳng hạn `baseline`:

```shell
kubectl label --overwrite ns --all \
  pod-security.kubernetes.io/audit=baseline \
  pod-security.kubernetes.io/warn=baseline
```

Lưu ý rằng thao tác này không đặt mức enforce, nhờ đó có thể phân biệt được các namespace
chưa được đánh giá một cách tường minh. Bạn có thể liệt kê các namespace chưa được đặt mức
enforce tường minh bằng lệnh sau:

```shell
kubectl get namespaces --selector='!pod-security.kubernetes.io/enforce'
```

### Áp dụng cho một namespace duy nhất (Applying to a single namespace)

Bạn cũng có thể cập nhật một namespace cụ thể. Lệnh này thêm chính sách `enforce=restricted`
vào `my-existing-namespace`, ghim phiên bản của chính sách restricted vào v1.36.

```shell
kubectl label --overwrite ns my-existing-namespace \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=v1.36
```

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 9:

1. Trên `lab-k8s-master`, một namespace của bạn đang chạy vài Pod và bạn định siết nó lên
   `enforce=restricted`. Lệnh nào cho bạn biết trước Pod nào sẽ vi phạm mà **chưa** đổi chính
   sách gì?
2. **Câu bẫy.** Bạn chạy `kubectl label --overwrite ns ... pod-security.kubernetes.io/enforce=restricted`
   trong khi namespace đang có một Pod vi phạm mức đó **đang chạy**. Pod đó bị xóa, bị chặn, hay
   không sao cả?
3. Manifest `my-baseline-namespace` đặt `enforce: baseline` nhưng `audit` và `warn` lại là
   `restricted`. Vì sao lệch nhau như vậy? Và khi mới bắt đầu áp Pod Security Standards cho cả
   cluster, bài khuyên đặt nhãn nào cho `--all`?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. `kubectl label --dry-run=server --overwrite ns <namespace> pod-security.kubernetes.io/enforce=restricted`.
   Điểm mấu chốt là **`--dry-run=server`**: bài nói rõ **các kiểm tra của Pod Security vẫn được
   chạy ở chế độ dry run**, nên bạn nhận đủ cảnh báo cho từng Pod hiện có sẽ vi phạm, **mà chính
   sách không thật sự được cập nhật**. Đây là lý do nó là bước đầu tiên khi đánh giá một thay đổi
   profile bảo mật.
2. **Không sao cả — Pod vẫn chạy, và bạn nhận một cảnh báo.** Ghi chú của bài nói chính xác điều
   xảy ra: khi nhãn `enforce` (hoặc nhãn version) được thêm vào hoặc thay đổi, admission plugin
   **kiểm tra từng Pod trong namespace theo chính sách mới**, và **các vi phạm được trả về cho
   người dùng dưới dạng cảnh báo**. Chỗ dễ nhầm là tưởng "enforce" nghĩa là dọn sạch những gì
   đang vi phạm — không phải: nó chặn ở **cửa tạo Pod**, còn Pod đã ở trong thì chỉ bị điểm danh.
   Pod vi phạm sẽ chỉ biến mất khi có ai đó tạo lại nó.
3. Vì hai nhãn phục vụ hai mục đích khác nhau: `enforce` đặt ở mức bạn **thực sự chặn được ngay**,
   còn `audit`/`warn` đặt theo mức **mong muốn** — chính chú thích trong manifest ghi "chúng ta
   đặt các nhãn này theo mức `enforce` mà mình _mong muốn_". Nhờ vậy bạn thấy trước những gì sẽ
   vỡ nếu nâng `enforce` lên `restricted`, mà chưa vỡ gì hôm nay. Phần sau: bài khuyên đặt
   **`pod-security.kubernetes.io/audit=baseline` và `pod-security.kubernetes.io/warn=baseline` cho
   `--all`, và cố ý *không* đặt `enforce`** — nhờ đó vẫn phân biệt được namespace nào chưa được
   đánh giá tường minh, bằng
   `kubectl get namespaces --selector='!pod-security.kubernetes.io/enforce'`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
