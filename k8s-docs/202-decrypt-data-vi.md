# Giải mã dữ liệu bí mật đã được mã hóa khi lưu trữ (Decrypt Confidential Data that is Already Encrypted at Rest)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/decrypt-data/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 22 — Audit và mã hóa dữ liệu](00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu),
bài 3/6 · Phần II không có lab riêng: kiểm chứng bằng **Checkpoint của chính giai đoạn 22** trên
cluster lab dựng ở [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) — bật mã hóa Secret at rest, tạo
Secret mới, chứng minh bằng `etcdctl get` rằng giá trị trong etcd không còn đọc được dưới dạng
thường, rồi đảo ngược đúng quy trình của bài này.

Bài là thao tác đảo ngược của bài
[Encrypting Confidential Data at Rest](208-encrypt-data-vi.md) — bài 2/6 của cùng giai đoạn — và
nối dài phần lưu trữ Secret của bài [109 — Secret](109-secret-vi.md).

Bài này mô tả thao tác **tắt** mã hóa. Hãy đọc nó để hiểu cơ chế thứ tự provider trong
`EncryptionConfiguration`; chỉ làm thật khi cluster của bạn đã bật mã hóa từ trước (theo bài
Encrypting Confidential Data at Rest của giai đoạn 22).

**Phải hiểu ở lần đọc này:**

- Cách nhận biết cluster có bật mã hóa khi lưu trữ hay không: flag
  `--encryption-provider-config` của `kube-apiserver`. Không có flag này thì dữ liệu đang ở
  dạng rõ, vì provider mặc định là `identity` — không bảo vệ gì cả.
- Quy tắc thứ tự provider: provider **đứng đầu** danh sách quyết định cách **ghi** dữ liệu
  mới; các provider phía sau vẫn được dùng để **đọc** (giải mã) dữ liệu cũ.
- Quy trình giải mã an toàn: thêm `identity: {}` lên đầu danh sách providers → restart
  kube-apiserver → lặp lại lần lượt trên mọi control plane node với cấu hình giống hệt nhau
  → ép ghi lại toàn bộ bằng
  `kubectl get secrets --all-namespaces -o json | kubectl replace -f -`.
- Chỉ sau khi **mọi** resource đã được ghi lại ở dạng không mã hóa mới được gỡ
  `--encryption-provider-config` (và `--encryption-provider-config-automatic-reload`) rồi
  restart thêm lần nữa.
- Phạm vi của bài: mã hóa dữ liệu resource ghi qua Kubernetes API vào etcd — không phải mã
  hóa filesystem mount vào container, cũng không phải mã hóa ổ đĩa của host.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Chi tiết provider `aescbc` và định dạng key trong ví dụ YAML | bài này chỉ cần vị trí provider trong danh sách; cấu hình mã hóa và quản lý key thuộc bài bật mã hóa | bài [Encrypting Confidential Data at Rest](208-encrypt-data-vi.md) trong [giai đoạn 22](00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu) |
| API `EncryptionConfiguration` (v1) đầy đủ | tài liệu tham chiếu field | khi cần tra cứu lúc vận hành thật |

---

Tất cả các API trong Kubernetes cho phép bạn ghi dữ liệu resource API bền vững (persistent)
đều hỗ trợ mã hóa khi lưu trữ (at-rest encryption). Ví dụ, bạn có thể bật mã hóa khi lưu trữ
cho Secret. Lớp mã hóa này là phần bổ sung bên cạnh bất kỳ mã hóa cấp hệ thống nào dành cho
etcd cluster hoặc cho các filesystem trên những host đang chạy kube-apiserver.

Trang này chỉ ra cách chuyển từ trạng thái mã hóa dữ liệu API khi lưu trữ về trạng thái dữ
liệu API được lưu không mã hóa. Bạn có thể muốn làm việc này để cải thiện hiệu năng; tuy
nhiên, thông thường nếu việc mã hóa một số dữ liệu từng là ý tưởng đúng đắn thì việc giữ
chúng ở trạng thái mã hóa cũng là ý tưởng đúng đắn.

> **Ghi chú:** Tác vụ này áp dụng cho mã hóa dữ liệu resource được lưu trữ thông qua
> Kubernetes API. Ví dụ, bạn có thể mã hóa các object Secret, bao gồm cả dữ liệu key-value
> mà chúng chứa.
>
> Nếu bạn muốn quản lý mã hóa cho dữ liệu trong các filesystem được mount vào container,
> thay vào đó bạn cần:
>
> - dùng một tích hợp lưu trữ (storage integration) cung cấp volume đã mã hóa, hoặc
> - tự mã hóa dữ liệu bên trong ứng dụng của bạn

## Trước khi bạn bắt đầu (Before you begin)

* Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
  tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
  không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
  bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một
  trong các sân chơi (playground) Kubernetes sau:

  * [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
  * [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
  * [KodeKloud](https://kodekloud.com/public-playgrounds)

* Tác vụ này giả định rằng bạn đang chạy Kubernetes API server dưới dạng một
  [static pod](58-static-pods-vi.md) trên mỗi control plane node.

* Control plane của cluster **phải** dùng etcd v3.x (major version 3, minor version bất kỳ).

* Để mã hóa một custom resource, cluster của bạn phải chạy Kubernetes v1.26 trở lên.

* Bạn cần có sẵn một số dữ liệu API đã được mã hóa.

Kubernetes server của bạn phải ở phiên bản v1.36 hoặc mới hơn. Để kiểm tra phiên bản, nhập
`kubectl version`.

## Xác định mã hóa khi lưu trữ đã được bật hay chưa (Determine whether encryption at rest is already enabled)

Mặc định, API server dùng một provider tên là `identity`, lưu các resource ở dạng biểu diễn
văn bản rõ (plain text). **Provider `identity` mặc định không cung cấp bất kỳ sự bảo vệ bí
mật nào.**

Tiến trình `kube-apiserver` nhận một tham số `--encryption-provider-config` chỉ định đường
dẫn tới một file cấu hình. Nội dung của file đó, nếu bạn chỉ định, điều khiển cách dữ liệu
Kubernetes API được mã hóa trong etcd. Nếu tham số này không được chỉ định, bạn chưa bật mã
hóa khi lưu trữ.

Định dạng của file cấu hình đó là YAML, biểu diễn một kind API cấu hình có tên
[`EncryptionConfiguration`](https://kubernetes.io/docs/reference/config-api/apiserver-config.v1/).
Bạn có thể xem một ví dụ cấu hình trong
[Cấu hình mã hóa khi lưu trữ](208-encrypt-data-vi.md#understanding-the-encryption-at-rest-configuration).

Nếu `--encryption-provider-config` được đặt, hãy kiểm tra những resource nào (chẳng hạn
`secrets`) được cấu hình mã hóa, và provider nào đang được dùng. Hãy chắc chắn rằng provider
được ưu tiên cho loại resource đó **không phải** là `identity`; bạn chỉ đặt `identity`
(_không mã hóa_) làm mặc định khi muốn tắt mã hóa khi lưu trữ. Hãy xác nhận rằng provider
đứng đầu danh sách của một resource là thứ gì đó **khác** `identity` — điều đó nghĩa là mọi
thông tin mới ghi vào các resource loại đó sẽ được mã hóa theo cấu hình. Nếu bạn thấy
`identity` đứng đầu danh sách provider của bất kỳ resource nào, điều đó nghĩa là các
resource đó đang được ghi ra etcd mà không mã hóa.

## Giải mã toàn bộ dữ liệu (Decrypt all data) {#decrypting-all-data}

Ví dụ này chỉ ra cách dừng mã hóa Secret API khi lưu trữ. Nếu bạn đang mã hóa các kind API
khác, hãy điều chỉnh các bước cho khớp.

### Xác định vị trí file cấu hình mã hóa (Locate the encryption configuration file)

Trước tiên, tìm các file cấu hình của API server. Trên mỗi control plane node, manifest
static Pod của kube-apiserver chỉ định một tham số dòng lệnh
`--encryption-provider-config`. Nhiều khả năng bạn sẽ thấy file này được mount vào static
Pod bằng một volume mount kiểu [`hostPath`](91-volumes-vi.md#hostpath). Khi đã xác định được
volume, bạn có thể tìm file đó trên filesystem của node và kiểm tra nội dung.

### Cấu hình API server để giải mã object (Configure the API server to decrypt objects)

Để tắt mã hóa khi lưu trữ, hãy đặt provider `identity` làm mục đầu tiên trong file cấu hình
mã hóa của bạn.

Ví dụ, nếu file EncryptionConfiguration hiện tại của bạn có nội dung:

```yaml
---
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            # Không dùng key ví dụ (không hợp lệ) này để mã hóa
            - name: example
              secret: 2KfZgdiq2K0g2YrYpyDYs9mF2LPZhQ==
```

thì đổi thành:

```yaml
---
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - identity: {} # thêm dòng này
      - aescbc:
          keys:
            - name: example
              secret: 2KfZgdiq2K0g2YrYpyDYs9mF2LPZhQ==
```

và restart Pod kube-apiserver trên node đó.

### Cấu hình lại các control plane host khác (Reconfigure other control plane hosts) {#api-server-config-update-more-1}

Nếu cluster của bạn có nhiều API server, bạn nên triển khai thay đổi lần lượt cho từng API
server.

Hãy chắc chắn rằng bạn dùng cùng một cấu hình mã hóa trên mọi control plane host.

### Ép giải mã (Force decryption)

Sau đó, chạy lệnh sau để ép giải mã toàn bộ Secret:

```shell
# Nếu bạn đang giải mã một loại object khác, hãy đổi "secrets" cho khớp.
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

Khi bạn đã thay thế **tất cả** các resource mã hóa hiện có bằng dữ liệu nền không dùng mã
hóa, bạn có thể gỡ bỏ các thiết lập mã hóa khỏi `kube-apiserver`.

Các tùy chọn dòng lệnh cần gỡ bỏ là:

- `--encryption-provider-config`
- `--encryption-provider-config-automatic-reload`

Restart Pod kube-apiserver một lần nữa để áp dụng cấu hình mới.

### Cấu hình lại các control plane host khác (Reconfigure other control plane hosts) {#api-server-config-update-more-2}

Nếu cluster của bạn có nhiều API server, bạn lại cần triển khai thay đổi lần lượt cho từng
API server.

Hãy chắc chắn rằng bạn dùng cùng một cấu hình mã hóa trên mọi control plane host.

## Tiếp theo (What's next)

* Tìm hiểu thêm về
  [EncryptionConfiguration configuration API (v1)](https://kubernetes.io/docs/reference/config-api/apiserver-config.v1/).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc này:

1. **Câu bẫy.** Bạn thêm `identity: {}` lên đầu danh sách providers và restart
   kube-apiserver. Các Secret cũ trong etcd bây giờ đã ở dạng rõ chưa? Nếu chưa, lệnh nào
   biến chúng thành dạng rõ và vì sao lệnh đó có tác dụng?
2. Vì sao khi cấu hình API server để giải mã, ta đưa `identity` lên đầu nhưng **giữ nguyên**
   mục `aescbc` phía dưới thay vì xóa nó luôn?
3. Trên `lab-k8s-master` của cluster lab (dựng bằng kubeadm), bạn kiểm tra thế nào để biết
   cluster có đang bật mã hóa khi lưu trữ hay không, và nếu có thì tìm file cấu hình bằng
   cách nào?
4. Điều kiện nào phải thỏa trước khi được phép gỡ `--encryption-provider-config` khỏi
   kube-apiserver? Chuyện gì xảy ra nếu gỡ sớm?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Chưa.** Thứ tự provider chỉ quyết định cách **ghi** dữ liệu mới; dữ liệu cũ vẫn nằm mã
   hóa trong etcd cho tới khi được ghi lại. Lệnh
   `kubectl get secrets --all-namespaces -o json | kubectl replace -f -` đọc mọi Secret (API
   server giải mã được nhờ provider `aescbc` vẫn còn trong danh sách) rồi ghi đè lại chính
   chúng — lần ghi này đi qua provider đứng đầu là `identity`, nên bản lưu mới trong etcd
   không còn mã hóa. Trực giác "đổi cấu hình xong là dữ liệu tự đổi theo" sai vì cấu hình
   chỉ tác động vào các thao tác ghi từ thời điểm đó trở đi.
2. Vì etcd vẫn còn dữ liệu đã mã hóa bằng key của `aescbc`. **Provider đứng đầu quyết định
   cách ghi, các provider phía sau vẫn được dùng để đọc/giải mã dữ liệu cũ.** Nếu xóa
   `aescbc` ngay, API server không còn cách nào giải mã các Secret cũ khi đọc chúng.
3. Xem manifest static Pod của kube-apiserver trên control plane node (với cluster kubeadm
   của bạn là file trong `/etc/kubernetes/manifests/`) và tìm flag
   `--encryption-provider-config`. **Không có flag → chưa bật mã hóa** (API server dùng
   provider mặc định `identity`). Nếu có, lần theo volume mount kiểu `hostPath` của static
   Pod để xác định vị trí file trên filesystem của node rồi kiểm tra nội dung.
4. Điều kiện: **mọi** resource mã hóa hiện có đã được thay thế bằng dữ liệu nền không mã hóa
   (đã chạy bước ép giải mã sau khi tất cả API server dùng cùng cấu hình có `identity` đứng
   đầu). Nếu gỡ sớm, trong etcd vẫn còn resource mã hóa trong khi kube-apiserver không còn
   cấu hình (và key) để giải mã chúng — các resource đó trở nên không đọc được.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
