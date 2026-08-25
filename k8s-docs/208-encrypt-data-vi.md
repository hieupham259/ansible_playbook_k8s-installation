# Mã hóa dữ liệu bí mật khi lưu trữ (Encrypting Confidential Data at Rest)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Checkpoint tiếp nối — nhánh `/docs/tasks/`](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 22 — Audit và mã hóa dữ liệu](00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu), bài 2/6 ·
Kiểm chứng bằng việc trả [nợ lab "Mã hóa Secret at rest"](labs/README.md#5-sổ-nợ-lab) phát sinh
từ Lab 3b: bật mã hóa trên `k8s-master`, verify bằng `etcdctl`, rồi mã hóa lại các Secret cũ.

Bài này là "phần còn thiếu" của bài [109 — Secret](109-secret-vi.md): ở đó bạn đã biết Secret chỉ
được encode base64 chứ không hề được mã hóa; bài này bịt lỗ hổng đó ở tầng lưu trữ etcd.

**Phải hiểu ở lần đọc này:**

- Cách xác định cluster đã bật mã hóa at rest hay chưa: nhìn flag `--encryption-provider-config`
  của `kube-apiserver`; không có flag, hoặc có flag nhưng provider đầu tiên trong danh sách là
  `identity`, thì dữ liệu vẫn nằm plain text — `identity` không mã hóa gì cả.
- Cách đọc `EncryptionConfiguration`: mỗi mục trong `resources` gồm danh sách resource cần mã hóa
  và danh sách `providers` **có thứ tự** — provider đầu tiên dùng để mã hóa khi ghi, còn khi đọc
  thì mọi provider (và mọi key trong đó) được thử lần lượt để giải mã; mục cụ thể phải đứng
  **trước** mục wildcard (`*.*`) thì mới có hiệu lực.
- Trình tự bật mã hóa trên cluster kubeadm: sinh key ngẫu nhiên 32 byte encode base64 → viết file
  cấu hình → mount file vào static Pod `kube-apiserver` (thêm flag + volumeMount + hostPath) →
  restart API server → verify bằng `etcdctl get ... | hexdump -C` thấy tiền tố
  `k8s:enc:aescbc:v1:key1`.
- Vì sao sau đó còn phải chạy `kubectl get secrets --all-namespaces -o json | kubectl replace -f -`:
  mã hóa chỉ áp dụng **khi ghi**, Secret tạo từ trước vẫn nằm plain text trong etcd cho tới khi
  được ghi lại; chỉ sau bước này mới được gỡ `identity` khỏi danh sách provider để chặn hẳn việc
  đọc dữ liệu plain text.
- Quy trình xoay key không downtime (thêm key mới làm entry **thứ hai** → restart **tất cả**
  `kube-apiserver` → backup key → đưa key mới lên đầu → restart lần nữa → replace toàn bộ Secret
  → gỡ key cũ) và giới hạn của key cục bộ: key nằm ngay trên host trong file cấu hình, nên chỉ
  chống được lộ etcd chứ không chống được chiếm quyền host — muốn hơn thì phải dùng KMS.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Hai hàng `kms` v1/v2 trong bảng provider và mục "Lưu trữ key được quản lý (KMS)" | cluster lab không có dịch vụ KMS bên ngoài; lần đọc này chỉ cần nắm ý tưởng envelope encryption (DEK/KEK) | bài Using a KMS provider, bài 4/6 của chính [giai đoạn 22](00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu) |
| Mã hóa custom resource (ví dụ `pandas.awesome.bears.example`) | chưa học CustomResourceDefinition | giai đoạn 14, bài [179](179-custom-resources-vi.md) |
| Mục "Cấu hình tự động nạp lại" (`--encryption-provider-config-automatic-reload`) | tối ưu vận hành khi xoay key thường xuyên, không thay đổi bản chất quy trình | khi thực hành xoay key trong bài tập của giai đoạn 22 và bài Decrypt kế tiếp (bài 3/6 của giai đoạn 22) |

---

Tất cả các API trong Kubernetes cho phép bạn ghi dữ liệu tài nguyên API bền vững (persistent) đều
hỗ trợ mã hóa khi lưu trữ (at-rest encryption). Ví dụ, bạn có thể bật mã hóa at rest cho Secrets.
Lớp mã hóa at rest này là lớp bổ sung, nằm ngoài bất kỳ cơ chế mã hóa mức hệ thống nào dành cho
cluster etcd hoặc cho filesystem trên các host đang chạy kube-apiserver.

Trang này hướng dẫn cách bật và cấu hình mã hóa dữ liệu API khi lưu trữ.

> **Ghi chú:** Tác vụ này đề cập đến việc mã hóa dữ liệu tài nguyên được lưu trữ thông qua
> Kubernetes API. Ví dụ, bạn có thể mã hóa các object Secret, bao gồm cả dữ liệu key-value mà
> chúng chứa.
>
> Nếu bạn muốn mã hóa dữ liệu trong các filesystem được mount vào container, thay vào đó bạn cần:
>
> - dùng một tích hợp lưu trữ (storage integration) cung cấp volume đã được mã hóa, hoặc
> - tự mã hóa dữ liệu ngay bên trong ứng dụng của bạn.

## Trước khi bạn bắt đầu (Before you begin)

* Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
  với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
  vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
  [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
  chơi (playground) Kubernetes sau:

  * [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
  * [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
  * [KodeKloud](https://kodekloud.com/public-playgrounds)

* Tác vụ này giả định rằng bạn đang chạy Kubernetes API server dưới dạng static pod trên mỗi node
  control plane.

* Control plane của cluster **phải** dùng etcd v3.x (major version 3, minor version bất kỳ).

* Để mã hóa một custom resource, cluster của bạn phải chạy Kubernetes v1.26 trở lên.

* Để dùng ký tự đại diện (wildcard) khớp resource, cluster của bạn phải chạy Kubernetes v1.27
  trở lên.

Để kiểm tra phiên bản, nhập `kubectl version`.

## Xác định mã hóa at rest đã được bật hay chưa (Determine whether encryption at rest is already enabled) {#determining-whether-encryption-at-rest-is-already-enabled}

Theo mặc định, API server lưu các biểu diễn plain text của tài nguyên vào etcd, không có mã hóa
at rest.

Tiến trình `kube-apiserver` chấp nhận đối số `--encryption-provider-config` chỉ định đường dẫn
tới một file cấu hình. Nội dung của file đó, nếu bạn chỉ định, sẽ điều khiển cách dữ liệu
Kubernetes API được mã hóa trong etcd. Nếu bạn đang chạy kube-apiserver mà không có đối số dòng
lệnh `--encryption-provider-config`, bạn chưa bật mã hóa at rest. Nếu bạn đang chạy
kube-apiserver với đối số dòng lệnh `--encryption-provider-config`, và file mà nó tham chiếu
chỉ định provider `identity` là provider mã hóa đầu tiên trong danh sách, thì bạn cũng chưa bật
mã hóa at rest (**provider `identity` mặc định không cung cấp bất kỳ sự bảo vệ tính bí mật
nào.**)

Nếu bạn đang chạy kube-apiserver với đối số dòng lệnh `--encryption-provider-config`, và file mà
nó tham chiếu chỉ định một provider khác `identity` làm provider mã hóa đầu tiên trong danh sách,
thì bạn đã bật mã hóa at rest. Tuy nhiên, phép kiểm tra đó không cho bạn biết liệu lần di trú
(migration) sang lưu trữ mã hóa trước đây đã thành công hay chưa. Nếu bạn không chắc, hãy xem
[bảo đảm mọi dữ liệu liên quan đều được mã hóa](#ensure-all-secrets-are-encrypted).

## Hiểu cấu hình mã hóa at rest (Understanding the encryption at rest configuration) {#understanding-the-encryption-at-rest-configuration}

```yaml
---
#
# THẬN TRỌNG: đây là một cấu hình ví dụ.
#             Đừng dùng nó cho cluster của chính bạn!
#
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      - configmaps
      - pandas.awesome.bears.example # một API custom resource
    providers:
      # Cấu hình này KHÔNG bảo đảm tính bí mật của dữ liệu. Provider
      # đầu tiên được cấu hình là cơ chế "identity", tức là lưu
      # tài nguyên dưới dạng plain text.
      #
      - identity: {} # plain text, nói cách khác là KHÔNG mã hóa
      - aesgcm:
          keys:
            - name: key1
              secret: c2VjcmV0IGlzIHNlY3VyZQ==
            - name: key2
              secret: dGhpcyBpcyBwYXNzd29yZA==
      - aescbc:
          keys:
            - name: key1
              secret: c2VjcmV0IGlzIHNlY3VyZQ==
            - name: key2
              secret: dGhpcyBpcyBwYXNzd29yZA==
      - secretbox:
          keys:
            - name: key1
              secret: YWJjZGVmZ2hpamtsbW5vcHFyc3R1dnd4eXoxMjM0NTY=
  - resources:
      - events
    providers:
      - identity: {} # không mã hóa Events dù bên dưới có khai báo *.*
  - resources:
      - '*.apps' # khớp wildcard yêu cầu Kubernetes 1.27 trở lên
    providers:
      - aescbc:
          keys:
          - name: key2
            secret: c2VjcmV0IGlzIHNlY3VyZSwgb3IgaXMgaXQ/Cg==
  - resources:
      - '*.*' # khớp wildcard yêu cầu Kubernetes 1.27 trở lên
    providers:
      - aescbc:
          keys:
          - name: key3
            secret: c2VjcmV0IGlzIHNlY3VyZSwgSSB0aGluaw==
```

Mỗi phần tử trong mảng `resources` là một cấu hình riêng biệt và chứa một cấu hình hoàn chỉnh.
Trường `resources.resources` là mảng tên các tài nguyên Kubernetes (`resource` hoặc
`resource.group`) cần được mã hóa, chẳng hạn Secrets, ConfigMaps hoặc các tài nguyên khác.

Nếu custom resource được thêm vào `EncryptionConfiguration` và phiên bản cluster là 1.26 trở lên,
mọi custom resource mới tạo được nhắc đến trong `EncryptionConfiguration` sẽ được mã hóa. Mọi
custom resource đã tồn tại trong etcd trước phiên bản và cấu hình đó sẽ vẫn không được mã hóa cho
tới lần ghi kế tiếp vào bộ lưu trữ. Đây cũng chính là hành vi của các tài nguyên tích hợp sẵn
(built-in). Xem mục [Bảo đảm mọi Secret đều được mã hóa](#ensure-all-secrets-are-encrypted).

Mảng `providers` là danh sách có thứ tự các provider mã hóa khả dụng cho những API mà bạn đã
liệt kê. Mỗi provider hỗ trợ nhiều key — các key được thử theo thứ tự khi giải mã, và nếu
provider đó là provider đầu tiên thì key đầu tiên được dùng để mã hóa.

Mỗi phần tử chỉ được chỉ định một loại provider (`identity` hoặc `aescbc` đều được, nhưng không
được cả hai trong cùng một phần tử). Provider đầu tiên trong danh sách được dùng để mã hóa các
tài nguyên ghi vào bộ lưu trữ. Khi đọc tài nguyên từ bộ lưu trữ, mỗi provider khớp với dữ liệu đã
lưu sẽ lần lượt thử giải mã dữ liệu. Nếu không provider nào đọc được dữ liệu đã lưu do sai khác
về định dạng hoặc secret key, một lỗi sẽ được trả về và ngăn client truy cập tài nguyên đó.

`EncryptionConfiguration` hỗ trợ dùng wildcard để chỉ định các tài nguyên cần mã hóa. Dùng
'`*.<group>`' để mã hóa mọi tài nguyên trong một group (ví dụ '`*.apps`' ở ví dụ trên) hoặc
'`*.*`' để mã hóa mọi tài nguyên. '`*.`' có thể được dùng để mã hóa mọi tài nguyên trong core
group. '`*.*`' sẽ mã hóa tất cả tài nguyên, kể cả custom resource được thêm sau khi API server
khởi động.

> **Ghi chú:** Không được phép dùng các wildcard chồng lấn nhau trong cùng một danh sách resource
> hoặc giữa nhiều phần tử, vì khi đó một phần của cấu hình sẽ không có hiệu lực. Thứ tự xử lý và
> độ ưu tiên của danh sách `resources` được xác định theo thứ tự nó được liệt kê trong cấu hình.

Nếu bạn có một wildcard bao trùm các tài nguyên nhưng lại muốn loại một loại tài nguyên cụ thể ra
khỏi mã hóa at rest, bạn thực hiện điều đó bằng cách thêm một phần tử `resources` riêng chứa tên
tài nguyên muốn miễn trừ, theo sau là phần tử mảng `providers` trong đó bạn chỉ định provider
`identity`. Bạn thêm phần tử này vào danh sách sao cho nó xuất hiện **trước** cấu hình mà bạn có
chỉ định mã hóa (provider khác `identity`).

Ví dụ, nếu '`*.*`' được bật và bạn muốn loại trừ mã hóa cho Events và ConfigMaps, hãy thêm một
phần tử mới đứng **trước** vào `resources`, theo sau là phần tử mảng providers với `identity` làm
provider. Phần tử cụ thể hơn phải đứng trước phần tử wildcard.

Phần tử mới sẽ trông tương tự như sau:

```yaml
  ...
  - resources:
      - configmaps. # chỉ lấy từ core API group,
                    # vì có dấu "." ở cuối
      - events
    providers:
      - identity: {}
  # và sau đó là các phần tử khác trong resources
```

Hãy bảo đảm phần miễn trừ được liệt kê _trước_ phần tử wildcard '`*.*`' trong mảng resources để
nó có độ ưu tiên cao hơn.

Để biết thông tin chi tiết hơn về struct `EncryptionConfiguration`, hãy tham khảo
[API cấu hình mã hóa](https://kubernetes.io/docs/reference/config-api/apiserver-config.v1/).

> **Thận trọng:** Nếu bất kỳ tài nguyên nào không thể đọc được qua cấu hình mã hóa (do key đã bị
> thay đổi), và bạn không thể khôi phục một cấu hình hoạt động được, cách duy nhất còn lại là xóa
> trực tiếp entry đó khỏi etcd bên dưới.
>
> Mọi lời gọi tới Kubernetes API cố đọc tài nguyên đó sẽ thất bại cho tới khi nó bị xóa hoặc một
> key giải mã hợp lệ được cung cấp.

### Các provider khả dụng (Available providers) {#providers}

Trước khi cấu hình mã hóa at rest cho dữ liệu trong Kubernetes API của cluster, bạn cần chọn
(các) provider sẽ dùng.

Bảng sau mô tả từng provider khả dụng.

<table class="complex-layout">
<caption style="display: none;">Các provider cho mã hóa at rest của Kubernetes</caption>
<thead>
  <tr>
  <th>Tên</th>
  <th>Mã hóa</th>
  <th>Độ mạnh</th>
  <th>Tốc độ</th>
  <th>Độ dài key</th>
  </tr>
</thead>
<tbody id="encryption-providers-identity">
  <tr>
  <th rowspan="2" scope="row"><tt>identity</tt></th>
  <td><strong>Không có</strong></td>
  <td>N/A</td>
  <td>N/A</td>
  <td>N/A</td>
  </tr>
  <tr>
  <td colspan="4">Tài nguyên được ghi nguyên trạng, không mã hóa. Khi được đặt làm provider đầu
   tiên, tài nguyên sẽ được giải mã khi giá trị mới được ghi vào. Các tài nguyên đã mã hóa từ
   trước <strong>không</strong> tự động bị ghi đè bằng dữ liệu plain text.
   Provider <tt>identity</tt> là mặc định nếu bạn không chỉ định gì khác.</td>
  </tr>
</tbody>
<tbody id="encryption-providers-that-encrypt">
  <tr>
  <th rowspan="2" scope="row"><tt>aescbc</tt></th>
  <td>AES-CBC với padding <a href="https://datatracker.ietf.org/doc/html/rfc2315">PKCS#7</a></td>
  <td>Yếu</td>
  <td>Nhanh</td>
  <td>16, 24 hoặc 32 byte</td>
  </tr>
  <tr>
  <td colspan="4">Không được khuyến nghị do CBC dễ bị tấn công padding oracle. Key material truy
  cập được từ host control plane.</td>
  </tr>
  <tr>
  <th rowspan="2" scope="row"><tt>aesgcm</tt></th>
  <td>AES-GCM với nonce ngẫu nhiên</td>
  <td>Phải xoay key sau mỗi 200.000 lần ghi</td>
  <td>Nhanh nhất</td>
  <td>16, 24 hoặc 32 byte</td>
  </tr>
  <tr>
  <td colspan="4">Không được khuyến nghị, trừ khi đã triển khai một cơ chế xoay key tự động. Key
  material truy cập được từ host control plane.</td>
  </tr>
  <tr>
  <th rowspan="2" scope="row"><tt>kms</tt> v1 <em>(deprecated kể từ Kubernetes v1.28)</em></th>
  <td>Dùng cơ chế envelope encryption, với một DEK cho mỗi tài nguyên.</td>
  <td>Mạnh nhất</td>
  <td>Chậm (<em>so với <tt>kms</tt> version 2</em>)</td>
  <td>32 byte</td>
  </tr>
  <tr>
  <td colspan="4">
    Dữ liệu được mã hóa bằng các data encryption key (DEK) dùng AES-GCM;
    DEK được mã hóa bằng các key encryption key (KEK) theo cấu hình
    trong Key Management Service (KMS).
    Việc xoay key đơn giản: một DEK mới được sinh ra cho mỗi lần mã hóa,
    còn việc xoay KEK do người dùng kiểm soát.
    <br />
    Đọc cách <a href="https://kubernetes.io/docs/tasks/administer-cluster/kms-provider#configuring-the-kms-provider-kms-v1">cấu hình KMS V1 provider</a>.
    </td>
  </tr>
  <tr>
  <th rowspan="2" scope="row"><tt>kms</tt> v2 </th>
  <td>Dùng cơ chế envelope encryption, với một DEK cho mỗi API server.</td>
  <td>Mạnh nhất</td>
  <td>Nhanh</td>
  <td>32 byte</td>
  </tr>
  <tr>
  <td colspan="4">
    Dữ liệu được mã hóa bằng các data encryption key (DEK) dùng AES-GCM; DEK
    được mã hóa bằng các key encryption key (KEK) theo cấu hình
    trong Key Management Service (KMS).
    Kubernetes sinh một DEK mới cho mỗi lần mã hóa từ một seed bí mật.
    Seed được xoay mỗi khi KEK được xoay.<br/>
    Một lựa chọn tốt nếu bạn dùng công cụ bên thứ ba để quản lý key.
    Khả dụng ở mức stable từ Kubernetes v1.29.
    <br />
    Đọc cách <a href="https://kubernetes.io/docs/tasks/administer-cluster/kms-provider#configuring-the-kms-provider-kms-v2">cấu hình KMS V2 provider</a>.
    </td>
  </tr>
  <tr>
  <th rowspan="2" scope="row"><tt>secretbox</tt></th>
  <td>XSalsa20 và Poly1305</td>
  <td>Mạnh</td>
  <td>Nhanh hơn</td>
  <td>32 byte</td>
  </tr>
  <tr>
  <td colspan="4">Dùng các công nghệ mã hóa tương đối mới, có thể không được chấp nhận trong
  những môi trường đòi hỏi mức độ thẩm định cao. Key material truy cập được từ host control
  plane.</td>
  </tr>
</tbody>
</table>

Provider `identity` là mặc định nếu bạn không chỉ định gì khác. **Provider `identity` không mã
hóa dữ liệu được lưu và _không_ cung cấp thêm bất kỳ sự bảo vệ tính bí mật nào.**

### Lưu trữ key (Key storage)

#### Lưu trữ key cục bộ (Local key storage)

Mã hóa dữ liệu secret bằng một key được quản lý cục bộ giúp chống lại việc etcd bị xâm phạm,
nhưng không bảo vệ được trước việc host bị chiếm quyền. Vì các key mã hóa được lưu trên host
trong file YAML EncryptionConfiguration, một kẻ tấn công có kỹ năng có thể truy cập file đó và
trích xuất các key mã hóa.

#### Lưu trữ key được quản lý (KMS) (Managed (KMS) key storage) {#kms-key-storage}

Provider KMS dùng _envelope encryption_ (mã hóa phong bì): Kubernetes mã hóa tài nguyên bằng một
data key, rồi mã hóa chính data key đó bằng dịch vụ mã hóa được quản lý. Kubernetes sinh một
data key duy nhất cho mỗi tài nguyên. API server lưu phiên bản đã mã hóa của data key trong etcd
cùng với bản mã (ciphertext); khi đọc tài nguyên, API server gọi dịch vụ mã hóa được quản lý và
cung cấp cả bản mã lẫn data key (đã mã hóa). Bên trong dịch vụ mã hóa được quản lý, provider dùng
một _key encryption key_ để giải mã data key, giải mã data key đó, và cuối cùng khôi phục bản rõ.
Giao tiếp giữa control plane và KMS yêu cầu bảo vệ khi truyền (in-transit), chẳng hạn TLS.

Việc dùng envelope encryption tạo ra sự phụ thuộc vào key encryption key, vốn không được lưu
trong Kubernetes. Trong trường hợp KMS, kẻ tấn công muốn truy cập trái phép các giá trị bản rõ sẽ
phải xâm phạm cả etcd **và** KMS provider của bên thứ ba.

### Bảo vệ các key mã hóa (Protection for encryption keys)

Bạn nên áp dụng các biện pháp phù hợp để bảo vệ thông tin bí mật cho phép giải mã, dù đó là một
key mã hóa cục bộ, hay một token xác thực cho phép API server gọi KMS.

Ngay cả khi bạn dựa vào một provider để quản lý việc sử dụng và vòng đời của (các) key mã hóa
chính, bạn vẫn chịu trách nhiệm bảo đảm rằng các kiểm soát truy cập và các biện pháp bảo mật khác
cho dịch vụ mã hóa được quản lý là phù hợp với nhu cầu bảo mật của bạn.

## Mã hóa dữ liệu của bạn (Encrypt your data) {#encrypting-your-data}

### Sinh key mã hóa (Generate the encryption key) {#generate-key-no-kms}

Các bước sau giả định rằng bạn không dùng KMS, và do đó cũng giả định rằng bạn cần sinh một key
mã hóa. Nếu bạn đã có sẵn key mã hóa, hãy bỏ qua và chuyển tới
[Viết file cấu hình mã hóa](#write-an-encryption-configuration-file).

> **Thận trọng:** Lưu key mã hóa dạng thô trong EncryptionConfig chỉ cải thiện mức độ bảo mật
> của bạn ở mức vừa phải, so với việc không mã hóa.
>
> Để bí mật hơn nữa, hãy cân nhắc dùng provider `kms` vì cơ chế này dựa trên các key được giữ bên
> ngoài cluster Kubernetes của bạn. Các hiện thực của `kms` có thể làm việc với hardware security
> module hoặc với các dịch vụ mã hóa do nhà cung cấp cloud của bạn quản lý.
>
> Để tìm hiểu cách thiết lập mã hóa at rest bằng KMS, xem
> [Dùng KMS provider để mã hóa dữ liệu](213-kms-provider-vi.md).
> Plugin KMS provider mà bạn dùng cũng có thể kèm theo tài liệu riêng bổ sung.

Bắt đầu bằng việc sinh một key mã hóa mới, rồi encode nó bằng base64:

#### Linux

Sinh một key ngẫu nhiên 32 byte và encode base64. Bạn có thể dùng lệnh này:

```shell
head -c 32 /dev/urandom | base64
```

Bạn có thể dùng `/dev/hwrng` thay cho `/dev/urandom` nếu muốn dùng nguồn entropy phần cứng tích
hợp của máy. Không phải mọi thiết bị Linux đều cung cấp bộ sinh số ngẫu nhiên phần cứng.

#### macOS

Sinh một key ngẫu nhiên 32 byte và encode base64. Bạn có thể dùng lệnh này:

```shell
head -c 32 /dev/urandom | base64
```

#### Windows

Sinh một key ngẫu nhiên 32 byte và encode base64. Bạn có thể dùng lệnh này:

```powershell
# Đừng chạy lệnh này trong session mà bạn đã đặt seed
# cho bộ sinh số ngẫu nhiên.
[Convert]::ToBase64String((1..32|%{[byte](Get-Random -Max 256)}))
```

> **Ghi chú:** Hãy giữ bí mật key mã hóa, kể cả trong lúc bạn sinh nó và lý tưởng nhất là ngay cả
> sau khi bạn không còn tích cực sử dụng nó nữa.

### Nhân bản key mã hóa (Replicate the encryption key)

Dùng một cơ chế truyền file an toàn, tạo một bản sao của key mã hóa đó cho mọi host control plane
khác.

Ở mức tối thiểu, hãy dùng mã hóa khi truyền — ví dụ, secure shell (SSH). Để an toàn hơn, hãy dùng
mã hóa bất đối xứng giữa các host, hoặc thay đổi cách tiếp cận để chuyển sang dựa vào mã hóa KMS.

### Viết file cấu hình mã hóa (Write an encryption configuration file) {#write-an-encryption-configuration-file}

> **Thận trọng:** File cấu hình mã hóa có thể chứa các key giải mã được nội dung trong etcd. Nếu
> file cấu hình chứa bất kỳ key material nào, bạn phải giới hạn quyền truy cập một cách chặt chẽ
> trên tất cả host control plane, sao cho chỉ user chạy kube-apiserver mới đọc được cấu hình này.

Tạo một file cấu hình mã hóa mới. Nội dung nên tương tự như:

```yaml
---
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      - configmaps
      - pandas.awesome.bears.example
    providers:
      - aescbc:
          keys:
            - name: key1
              # Xem đoạn văn phía dưới để biết chi tiết
              # về giá trị secret
              secret: <BASE 64 ENCODED SECRET>
      - identity: {} # fallback này cho phép đọc các secret chưa mã hóa;
                     # ví dụ, trong giai đoạn di trú (migration) ban đầu
```

Để tạo một key mã hóa mới (không dùng KMS), xem [Sinh key mã hóa](#generate-key-no-kms).

### Sử dụng file cấu hình mã hóa mới (Use the new encryption configuration file)

Bạn sẽ cần mount file cấu hình mã hóa mới vào static pod `kube-apiserver`. Dưới đây là một ví dụ
về cách làm điều đó:

1. Lưu file cấu hình mã hóa mới vào `/etc/kubernetes/enc/enc.yaml` trên node control-plane.
1. Sửa manifest của static pod `kube-apiserver`: `/etc/kubernetes/manifests/kube-apiserver.yaml`
   sao cho nó tương tự như:

   ```yaml
   ---
   #
   # Đây là một đoạn trích manifest của một static Pod.
   # Hãy kiểm tra xem nó có đúng với cluster và API server của bạn không.
   #
   apiVersion: v1
   kind: Pod
   metadata:
     annotations:
       kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 10.20.30.40:443
     creationTimestamp: null
     labels:
       app.kubernetes.io/component: kube-apiserver
       tier: control-plane
     name: kube-apiserver
     namespace: kube-system
   spec:
     containers:
     - command:
       - kube-apiserver
       ...
       - --encryption-provider-config=/etc/kubernetes/enc/enc.yaml  # thêm dòng này
       volumeMounts:
       ...
       - name: enc                           # thêm dòng này
         mountPath: /etc/kubernetes/enc      # thêm dòng này
         readOnly: true                      # thêm dòng này
       ...
     volumes:
     ...
     - name: enc                             # thêm dòng này
       hostPath:                             # thêm dòng này
         path: /etc/kubernetes/enc           # thêm dòng này
         type: DirectoryOrCreate             # thêm dòng này
     ...
   ```

1. Khởi động lại API server của bạn.

> **Thận trọng:** File cấu hình của bạn chứa các key có thể giải mã nội dung trong etcd, vì vậy
> bạn phải giới hạn quyền truy cập chặt chẽ trên các node control-plane, sao cho chỉ user chạy
> `kube-apiserver` mới đọc được nó.

Bây giờ bạn đã có mã hóa trên **một** host control plane. Một cluster Kubernetes điển hình có
nhiều host control plane, nên còn nhiều việc phải làm.

### Cấu hình lại các host control plane khác (Reconfigure other control plane hosts) {#api-server-config-update-more}

Nếu cluster của bạn có nhiều API server, bạn nên triển khai các thay đổi lần lượt cho từng API
server.

> **Thận trọng:** Với các cấu hình cluster có từ hai node control plane trở lên, cấu hình mã hóa
> phải giống hệt nhau trên mỗi node control plane.
>
> Nếu có sự khác biệt về cấu hình encryption provider giữa các node control plane, sự khác biệt
> này có thể khiến kube-apiserver không giải mã được dữ liệu.

Khi bạn lên kế hoạch cập nhật cấu hình mã hóa của cluster, hãy lập kế hoạch sao cho các API
server trong control plane luôn có thể giải mã dữ liệu đã lưu (ngay cả khi đang triển khai thay
đổi dở dang).

Hãy bảo đảm bạn dùng **cùng một** cấu hình mã hóa trên mỗi host control plane.

### Xác minh dữ liệu mới ghi đã được mã hóa (Verify that newly written data is encrypted) {#verifying-that-data-is-encrypted}

Dữ liệu được mã hóa khi ghi vào etcd. Sau khi khởi động lại `kube-apiserver`, mọi Secret mới tạo
hoặc mới cập nhật (hoặc loại tài nguyên khác được cấu hình trong `EncryptionConfiguration`) sẽ
được mã hóa khi lưu trữ.

Để kiểm tra điều này, bạn có thể dùng chương trình dòng lệnh `etcdctl` để đọc nội dung dữ liệu
secret của bạn.

Ví dụ sau cho thấy cách kiểm tra điều này với việc mã hóa Secret API.

1. Tạo một Secret mới tên `secret1` trong namespace `default`:

   ```shell
   kubectl create secret generic secret1 -n default --from-literal=mykey=mydata
   ```

1. Dùng công cụ dòng lệnh `etcdctl`, đọc Secret đó ra khỏi etcd:

   ```
   ETCDCTL_API=3 etcdctl get /registry/secrets/default/secret1 [...] | hexdump -C
   ```

   trong đó `[...]` phải là các đối số bổ sung để kết nối tới etcd server.

   Ví dụ:

   ```shell
   ETCDCTL_API=3 etcdctl \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt   \
      --cert=/etc/kubernetes/pki/etcd/server.crt \
      --key=/etc/kubernetes/pki/etcd/server.key  \
      get /registry/secrets/default/secret1 | hexdump -C
   ```

   Kết quả tương tự như sau (đã rút gọn):

   ```hexdump
   00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
   00000010  73 2f 64 65 66 61 75 6c  74 2f 73 65 63 72 65 74  |s/default/secret|
   00000020  31 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |1.k8s:enc:aescbc|
   00000030  3a 76 31 3a 6b 65 79 31  3a c7 6c e7 d3 09 bc 06  |:v1:key1:.l.....|
   00000040  25 51 91 e4 e0 6c e5 b1  4d 7a 8b 3d b9 c2 7c 6e  |%Q...l..Mz.=..|n|
   00000050  b4 79 df 05 28 ae 0d 8e  5f 35 13 2c c0 18 99 3e  |.y..(..._5.,...>|
   [...]
   00000110  23 3a 0d fc 28 ca 48 2d  6b 2d 46 cc 72 0b 70 4c  |#:..(.H-k-F.r.pL|
   00000120  a5 fc 35 43 12 4e 60 ef  bf 6f fe cf df 0b ad 1f  |..5C.N`..o......|
   00000130  82 c4 88 53 02 da 3e 66  ff 0a                    |...S..>f..|
   0000013a
   ```

1. Xác minh Secret được lưu có tiền tố `k8s:enc:aescbc:v1:`, cho biết provider `aescbc` đã mã
   hóa dữ liệu kết quả. Xác nhận rằng tên key hiển thị trong `etcd` khớp với tên key được chỉ
   định trong `EncryptionConfiguration` nói trên. Trong ví dụ này, bạn có thể thấy key mã hóa
   tên `key1` được dùng trong `etcd` và trong `EncryptionConfiguration`.

1. Xác minh Secret được giải mã đúng khi truy xuất qua API:

   ```shell
   kubectl get secret secret1 -n default -o yaml
   ```

   Kết quả phải chứa `mykey: bXlkYXRh`, với nội dung `mydata` được encode bằng base64; đọc
   [giải mã một Secret](327-secret-kubectl-vi.md#decoding-secret)
   để tìm hiểu cách decode Secret một cách đầy đủ.

### Bảo đảm mọi dữ liệu liên quan đều được mã hóa (Ensure all relevant data are encrypted) {#ensure-all-secrets-are-encrypted}

Thường thì việc bảo đảm các object mới được mã hóa là chưa đủ: bạn còn muốn việc mã hóa đó áp
dụng cho cả các object đã được lưu từ trước.

Trong ví dụ này, bạn đã cấu hình cluster để Secret được mã hóa khi ghi. Thực hiện thao tác
replace cho từng Secret sẽ mã hóa nội dung đó khi lưu trữ (at rest), trong khi bản thân object
không thay đổi.

Bạn có thể thực hiện thay đổi này trên tất cả Secret trong cluster:

```shell
# Chạy lệnh này với tư cách quản trị viên có quyền đọc và ghi mọi Secret
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

Lệnh trên đọc tất cả Secret rồi cập nhật chúng với chính dữ liệu đó, nhằm áp dụng mã hóa phía
server.

> **Ghi chú:** Nếu xảy ra lỗi do một thao tác ghi xung đột, hãy chạy lại lệnh. Chạy lệnh đó
> nhiều lần là an toàn.
>
> Với các cluster lớn hơn, bạn có thể muốn chia nhỏ các Secret theo namespace, hoặc viết script
> cập nhật.

## Ngăn việc truy xuất plain text (Prevent plain text retrieval) {#cleanup-all-secrets-encrypted}

Nếu bạn muốn chắc chắn rằng cách truy cập duy nhất tới một loại API cụ thể là thông qua mã hóa,
bạn có thể loại bỏ khả năng API server đọc dữ liệu nền của API đó ở dạng plaintext.

> **Cảnh báo:** Thay đổi này khiến API server không thể truy xuất các tài nguyên được đánh dấu
> là đã mã hóa at rest nhưng thực tế lại đang được lưu ở dạng không mã hóa.
>
> Khi bạn đã cấu hình mã hóa at rest cho một API (ví dụ: loại API `Secret`, đại diện cho tài
> nguyên `secrets` trong core API group), bạn **phải** bảo đảm rằng tất cả các tài nguyên đó
> trong cluster này thực sự đã được mã hóa at rest. Hãy kiểm tra điều này trước khi tiếp tục các
> bước kế tiếp.

Khi tất cả Secret trong cluster đã được mã hóa, bạn có thể gỡ phần `identity` khỏi cấu hình mã
hóa. Ví dụ:

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
            - name: key1
              secret: <BASE 64 ENCODED SECRET>
      - identity: {} # XÓA DÒNG NÀY
```

…rồi khởi động lại lần lượt từng API server. Thay đổi này ngăn API server truy cập một Secret
dạng plain text, kể cả khi vô tình.

## Xoay key giải mã (Rotate a decryption key) {#rotating-a-decryption-key}

Thay đổi một key mã hóa cho Kubernetes mà không gây downtime đòi hỏi một thao tác nhiều bước,
đặc biệt khi có triển khai tính sẵn sàng cao (highly-available) với nhiều tiến trình
`kube-apiserver` đang chạy.

1. Sinh một key mới và thêm nó làm entry key thứ hai cho provider hiện tại trên tất cả các node
   control plane.
1. Khởi động lại **tất cả** các tiến trình `kube-apiserver`, để bảo đảm mỗi server có thể giải
   mã bất kỳ dữ liệu nào được mã hóa bằng key mới.
1. Tạo một bản sao lưu an toàn của key mã hóa mới. Nếu bạn mất tất cả bản sao của key này, bạn
   sẽ phải xóa mọi tài nguyên đã được mã hóa bằng key bị mất, và các workload có thể không hoạt
   động như mong đợi trong thời gian mã hóa at rest bị hỏng.
1. Đưa key mới lên làm entry đầu tiên trong mảng `keys` để nó được dùng cho mã hóa at rest với
   các lần ghi mới.
1. Khởi động lại tất cả các tiến trình `kube-apiserver` để bảo đảm mỗi host control plane giờ
   đây mã hóa bằng key mới.
1. Với tư cách một user có đặc quyền, chạy
   `kubectl get secrets --all-namespaces -o json | kubectl replace -f -` để mã hóa tất cả
   Secret hiện có bằng key mới.
1. Sau khi bạn đã cập nhật tất cả Secret hiện có để dùng key mới và đã sao lưu an toàn key mới,
   hãy gỡ key giải mã cũ khỏi cấu hình.

## Giải mã toàn bộ dữ liệu (Decrypt all data) {#decrypting-all-data}

Ví dụ này cho thấy cách dừng mã hóa Secret API at rest. Nếu bạn đang mã hóa các loại API khác,
hãy điều chỉnh các bước cho phù hợp.

Để tắt mã hóa at rest, đặt provider `identity` làm entry đầu tiên trong file cấu hình mã hóa của
bạn:

```yaml
---
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      # liệt kê ở đây mọi resource khác mà trước đó
      # bạn đã mã hóa at rest
    providers:
      - identity: {} # thêm dòng này
      - aescbc:
          keys:
            - name: key1
              secret: <BASE 64 ENCODED SECRET> # giữ nguyên dòng này
                                               # bảo đảm nó đứng sau "identity"
```

Sau đó chạy lệnh sau để buộc giải mã tất cả Secret:

```shell
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

Khi bạn đã replace tất cả tài nguyên mã hóa hiện có bằng dữ liệu nền không dùng mã hóa, bạn có
thể gỡ các thiết lập mã hóa khỏi `kube-apiserver`.

## Cấu hình tự động nạp lại (Configure automatic reloading) {#configure-automatic-reloading}

Bạn có thể cấu hình tự động nạp lại (reload) cấu hình encryption provider. Thiết lập đó quyết
định việc API server chỉ nạp file mà bạn chỉ định cho `--encryption-provider-config` một lần lúc
khởi động, hay tự động nạp lại mỗi khi bạn thay đổi file đó. Bật tùy chọn này cho phép bạn thay
đổi các key cho mã hóa at rest mà không cần khởi động lại API server.

Để cho phép tự động nạp lại, cấu hình API server chạy với:
`--encryption-provider-config-automatic-reload=true`. Khi được bật, các thay đổi file được kiểm
tra (poll) mỗi phút để phát hiện sự chỉnh sửa. Metric
`apiserver_encryption_config_controller_automatic_reload_last_timestamp_seconds` cho biết thời
điểm cấu hình mới có hiệu lực. Điều này cho phép xoay các key mã hóa mà không cần khởi động lại
API server.

## Tiếp theo (What's next)

* Đọc về [giải mã dữ liệu đã được mã hóa at rest từ trước](202-decrypt-data-vi.md)
* Tìm hiểu thêm về [API cấu hình EncryptionConfiguration (v1)](https://kubernetes.io/docs/reference/config-api/apiserver-config.v1/).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 22:

1. Trên `k8s-master` của cluster lab, bạn thêm flag `--encryption-provider-config` trỏ tới một
   file trong đó danh sách provider là `identity` đứng đầu, `aescbc` đứng sau. Secret mới tạo có
   được mã hóa trong etcd không? Cluster lúc này được coi là đã bật mã hóa at rest chưa?
2. **Câu bẫy.** Bạn đã đặt `aescbc` làm provider đầu tiên, restart `kube-apiserver` thành công,
   rồi dùng `etcdctl` đọc một Secret tạo từ tuần trước — và vẫn thấy dữ liệu plain text. Đây có
   phải lỗi cấu hình không? Cần làm gì tiếp, và điều kiện nào phải thỏa trước khi bạn được phép
   gỡ `identity` khỏi danh sách provider?
3. Trong một mục cấu hình có nhiều provider và mỗi provider có nhiều key: provider nào và key
   nào được dùng khi **ghi**? Khi **đọc**, chuyện gì diễn ra, và điều gì xảy ra nếu không
   provider nào giải mã được dữ liệu đã lưu?
4. Mã hóa bằng key cục bộ trong file `EncryptionConfiguration` chống được kịch bản tấn công nào
   và không chống được kịch bản nào? Cơ chế envelope encryption của KMS thay đổi điều đó ra sao?
5. Trong quy trình xoay key không downtime, vì sao key mới phải được thêm làm entry **thứ hai**
   và tất cả `kube-apiserver` phải restart xong, rồi mới được đưa key mới lên vị trí đầu tiên?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không — Secret mới vẫn được lưu plain text, và cluster chưa được coi là bật mã hóa at
   rest.** Provider **đầu tiên** trong danh sách là provider dùng để mã hóa khi ghi; `identity`
   đứng đầu nghĩa là mọi thao tác ghi đều ra plain text (`aescbc` đứng sau chỉ được dùng để
   *giải mã* dữ liệu cũ nếu khớp). Bài nói rõ: có flag nhưng provider đầu là `identity` thì
   tương đương chưa bật mã hóa, vì `identity` không cung cấp bất kỳ sự bảo vệ tính bí mật nào.
2. **Không phải lỗi cấu hình.** Mã hóa chỉ áp dụng **khi ghi vào etcd**: object tồn tại từ trước
   giữ nguyên dạng cũ cho tới lần ghi kế tiếp. Cần chạy
   `kubectl get secrets --all-namespaces -o json | kubectl replace -f -` để ghi lại mọi Secret
   với chính dữ liệu của nó, qua đó áp mã hóa phía server (chạy lại nhiều lần là an toàn nếu gặp
   xung đột ghi). Chỉ khi **tất cả** Secret thực sự đã được mã hóa at rest mới được gỡ
   `identity`; gỡ sớm sẽ khiến API server không truy xuất được các Secret còn nằm plain text.
3. Khi ghi: **provider đầu tiên trong danh sách, với key đầu tiên của nó**. Khi đọc: mỗi
   provider khớp với dữ liệu đã lưu lần lượt thử giải mã, và trong mỗi provider các key được
   thử theo thứ tự. Nếu không provider nào đọc được (sai định dạng hoặc sai secret key), API
   server trả về **lỗi và client không truy cập được tài nguyên đó**; nếu không khôi phục được
   cấu hình hợp lệ thì lối thoát duy nhất là xóa entry đó trực tiếp trong etcd.
4. Key cục bộ chống được **etcd bị xâm phạm** (kẻ có bản sao dữ liệu etcd không đọc được nội
   dung), nhưng không chống được **host control plane bị chiếm quyền** — key nằm ngay trên host
   trong file YAML nên kẻ tấn công đọc file là trích được key. Với KMS, dữ liệu được mã hóa bằng
   DEK, DEK lại được mã hóa bằng KEK giữ **bên ngoài cluster**; kẻ tấn công phải xâm phạm cả
   etcd **và** KMS provider bên thứ ba mới thu được bản rõ.
5. Vì trong cluster HA các `kube-apiserver` được cập nhật lần lượt. Nếu đưa key mới lên vị trí
   mã hóa ngay, một server đã cập nhật sẽ **ghi bằng key mới** trong khi server chưa restart
   **không có key đó để giải mã** — đọc dữ liệu sẽ lỗi. Thêm key ở vị trí thứ hai trước bảo đảm
   mọi server đều *giải mã được* key mới (dù chưa dùng để ghi); sau khi tất cả đã restart mới
   đưa key mới lên đầu để bắt đầu mã hóa bằng nó, rồi restart lần nữa, replace toàn bộ Secret và
   cuối cùng gỡ key cũ.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng, rồi trả
[nợ lab "Mã hóa Secret at rest"](labs/README.md#5-sổ-nợ-lab) trên cluster lab trước khi sang bài
Decrypt kế tiếp của [giai đoạn 22](00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu).
