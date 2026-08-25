# Thực thi Pod Security Standards bằng cách cấu hình Admission Controller tích hợp sẵn (Enforce Pod Security Standards by Configuring the Built-in Admission Controller)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-admission-controller/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 9 — Bảo mật và multi-tenancy](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy),
dòng **Thực hành**, bài 4/10 · Kiểm chứng ở
[Lab 9b — Pod Security và hardening](labs/LAB-9B-POD-SECURITY-VA-HARDENING.md), phần B1.3 (namespace
không có nhãn thì mặc định là gì) và B7.1 (apiserver của cluster lab **không** có
`--admission-control-config-file`, nên bộ mặc định trong bài đang có hiệu lực nguyên vẹn).

Bài này và bài [283](283-enforce-standards-namespace-labels-vi.md) là **hai cách thực thi cùng một
chuẩn**. Bài này là tầng cluster: một file cấu hình truyền cho `kube-apiserver`, đặt mặc định cho
toàn bộ namespace và khai báo miễn trừ — chỉ người vận hành control plane đổi được. Bài 283 là
tầng namespace: một nhãn, đổi bằng `kubectl label`. Khi cả hai cùng có mặt, **nhãn namespace
thắng**: chính chú thích trong YAML dưới đây nói giá trị `defaults` chỉ áp dụng **khi một nhãn
mode không được đặt**.

**Phải hiểu ở lần đọc này:**

- Đây là cấu hình **tĩnh, ở tầng cluster**: một object `AdmissionConfiguration` chứa
  `PodSecurityConfiguration` cho plugin tên `PodSecurity`, và nó chỉ có hiệu lực khi được truyền
  cho `kube-apiserver` qua cờ `--admission-control-config-file` (ghi chú cuối bài).
- Khối `defaults` gồm ba cặp `enforce`/`audit`/`warn` và nhãn version tương ứng. Mặc định của cả
  ba mode là **`privileged`**, mặc định của cả ba version là **`latest`** — và các giá trị này
  **chỉ áp dụng cho namespace không đặt nhãn mode tương ứng**. Đó là lý do một namespace trần
  không chặn gì cả.
- Khối `exemptions` miễn trừ theo đúng ba trục: `usernames`, `runtimeClasses`, `namespaces`. Miễn
  trừ **chỉ khai báo được ở file này** — không có nhãn namespace nào làm được việc đó.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Thao tác **áp** file này vào cluster: truyền cờ `--admission-control-config-file` cho `kube-apiserver` | phải sửa cờ và manifest static Pod của control plane; Lab 9b cấm việc đó và biến điều đó thành gate | [giai đoạn 8](00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm) bài [03](03-control-plane-flags-vi.md) và [giai đoạn 20](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) |
| Ghi chú tương thích: `v1beta1` cho v1.23–v1.24, `v1alpha1` cho v1.22 | cluster lab dùng phiên bản khóa ở [bảng A1.3 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa), tức nhánh `v1` | chỉ cần khi tiếp quản cluster rất cũ — cùng nhóm với bài [117](117-pod-security-policy-vi.md), lộ trình xếp là **tài liệu lịch sử** ở cuối giai đoạn 9 |
| Nội dung thật sự của ba mức `privileged`, `baseline`, `restricted` | ở đây chúng mới chỉ là **giá trị hợp lệ** của nhãn level | bài [115](115-pod-security-standards-vi.md), đọc ngay sau Lab 9a trong cùng giai đoạn 9 |
| Cơ chế miễn trừ hoạt động ra sao trong chuỗi admission | bài này chỉ đưa cú pháp khai báo | bài [116](116-pod-security-admission-vi.md), mục [Miễn trừ](116-pod-security-admission-vi.md#miễn-trừ-exemptions) |

---

Kubernetes cung cấp một
[admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#podsecurity)
tích hợp sẵn để thực thi các [Pod Security Standard](./115-pod-security-standards-vi.md).
Bạn có thể cấu hình admission controller này để đặt các giá trị mặc định trên toàn cluster và
các [miễn trừ (exemptions)](./116-pod-security-admission-vi.md#miễn-trừ-exemptions).

## Trước khi bạn bắt đầu (Before you begin)

Sau bản phát hành alpha trong Kubernetes v1.22, Pod Security Admission trở nên khả dụng theo
mặc định trong Kubernetes v1.23, ở trạng thái beta. Từ phiên bản 1.25 trở đi, Pod Security
Admission đạt mức phổ biến rộng rãi (generally available).

Để kiểm tra phiên bản, hãy nhập `kubectl version`.

Nếu bạn không chạy Kubernetes 1.36, bạn có thể chuyển sang xem trang này trong tài liệu của
phiên bản Kubernetes mà bạn đang chạy.

## Cấu hình Admission Controller (Configure the Admission Controller) {#configure-the-admission-controller}

> **Ghi chú:** Cấu hình `pod-security.admission.config.k8s.io/v1` yêu cầu v1.25+.
> Với v1.23 và v1.24, dùng
> [v1beta1](https://v1-24.docs.kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-admission-controller/).
> Với v1.22, dùng
> [v1alpha1](https://v1-22.docs.kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-admission-controller/).

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1 # xem ghi chú về tương thích
    kind: PodSecurityConfiguration
    # Giá trị mặc định được áp dụng khi một nhãn (label) mode không được đặt.
    #
    # Giá trị nhãn level phải là một trong:
    # - "privileged" (mặc định)
    # - "baseline"
    # - "restricted"
    #
    # Giá trị nhãn version phải là một trong:
    # - "latest" (mặc định)
    # - phiên bản cụ thể, ví dụ "v1.36"
    defaults:
      enforce: "privileged"
      enforce-version: "latest"
      audit: "privileged"
      audit-version: "latest"
      warn: "privileged"
      warn-version: "latest"
    exemptions:
      # Mảng các username đã xác thực được miễn trừ.
      usernames: []
      # Mảng các tên runtime class được miễn trừ.
      runtimeClasses: []
      # Mảng các namespace được miễn trừ.
      namespaces: []
```

> **Ghi chú:** Manifest ở trên cần được chỉ định cho kube-apiserver thông qua flag
> `--admission-control-config-file`.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 9:

1. Cluster kubeadm của bạn không truyền `--admission-control-config-file` cho `kube-apiserver`.
   Bạn tạo một namespace mới trên `lab-k8s-master` và không gắn nhãn nào. Namespace đó đang chịu
   mức `enforce` nào, và vì sao?
2. **Câu bẫy.** Bạn muốn miễn trừ hẳn một namespace khỏi Pod Security. Gắn thêm một nhãn lên
   namespace đó có làm được việc này không?
3. Một namespace **có** nhãn `pod-security.kubernetes.io/enforce`, đồng thời cluster **có** file
   cấu hình với `defaults.enforce` khác giá trị nhãn. Namespace đó chịu mức nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **`privileged`.** Hai bước lập luận. Một: giá trị mặc định của `enforce` trong khối `defaults`
   là `privileged`, và chú thích trong YAML nói mặc định này được áp dụng **khi một nhãn mode
   không được đặt** — namespace trần đúng là trường hợp đó. Hai: cluster không truyền
   `--admission-control-config-file` nên không có file nào ghi đè bộ mặc định ấy; nó đang có hiệu
   lực **nguyên vẹn**. Kết quả: namespace mới không chặn gì cả.
2. **Không.** Miễn trừ là thứ **chỉ khai báo được trong khối `exemptions` của file cấu hình
   admission controller**, theo ba trục `usernames`, `runtimeClasses`, `namespaces` — và file đó
   phải truyền cho `kube-apiserver`. Nhãn trên namespace chỉ làm được một việc: chọn **mode** và
   **level** cho namespace đó. Thứ gần nhất với "miễn trừ" mà nhãn làm được là đặt level
   `privileged`, nhưng đó là **áp một chính sách rộng**, không phải bỏ qua kiểm tra.
3. **Mức của nhãn namespace** — nhãn thắng. Chú thích ngay trong YAML là câu trả lời trực tiếp:
   `defaults` là "giá trị mặc định được áp dụng **khi một nhãn mode không được đặt**". Namespace
   đã đặt nhãn thì không còn rơi vào mặc định nữa. Hệ quả cần nhớ: file cấu hình là **sàn cho
   namespace chưa khai báo gì**, không phải trần chặn đè lên nhãn.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
