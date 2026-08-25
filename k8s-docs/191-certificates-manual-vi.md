# Tạo certificate thủ công (Generate Certificates Manually)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/certificates/>
>
> Khi dùng xác thực bằng client certificate, bạn có thể tạo certificate thủ công bằng
> `easyrsa`, `openssl` hoặc `cfssl`.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 18 — Vòng đời chứng chỉ](00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ), bài 4/7 ·
Giai đoạn 18 **không có lab riêng**: bạn tự chấm bằng **Checkpoint** ghi ở cuối mục giai đoạn đó
trong lộ trình, chạy trên chính cluster ba VM dựng ở
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md).

**Đây là chỗ trả nợ #7 — quản lý vòng đời certificate.** Nợ phát sinh ở giai đoạn 12 tại bài
[156](156-certificates-vi.md), vốn chỉ là trang trỏ hướng sáu dòng và **chính là trang trỏ tới bài
này**. Theo quy tắc trả nợ của [sổ nợ lộ trình](00-ALO-TRINH-ADMIN.md#sổ-nợ-lộ-trình), **đọc lại
bài [156](156-certificates-vi.md) trước khi làm giai đoạn 18**.

Bài này **không phải cách bạn sẽ vận hành certificate của cluster lab**. Cluster lab dựng bằng
kubeadm nên PKI đã có sẵn, và việc kiểm hạn, gia hạn, xoay CA nằm ở bài
[219](219-kubeadm-certs-vi.md) — bài xương sống của giai đoạn 18. Đọc bài này để thấy **thứ mà
kubeadm đang làm thay bạn** trông như thế nào khi làm tay, và để nhận ra ba bộ công cụ khi tiếp
quản một cluster không dùng kubeadm.

**Phải hiểu ở lần đọc này:**

- Ba công cụ `easyrsa`, `openssl` và `cfssl` làm **cùng một dãy việc**, chỉ khác cú pháp: tạo CA
  tạo key cho server, sinh CSR rồi ký nó bằng CA, cuối cùng giao ba file kết quả cho API server.
  Nắm mô hình đó rồi thì đọc mục nào cũng ra cùng một thứ.
- Ba tham số khởi động API server mà cả mục `easyrsa` lẫn mục `openssl` đều kết thúc bằng, và ý
  nghĩa chia hai phía: `--client-ca-file` để **xác thực client**, còn `--tls-cert-file` và
  `--tls-private-key-file` để **phục vụ TLS** cho chính API server.
- Phần dễ sai nhất là danh sách tên thay thế — `--subject-alt-name` của easyrsa, `[ alt_names ]`
  của openssl, `hosts` của cfssl: certificate của API server phải liệt kê **mọi IP và tên DNS mà
  client sẽ dùng để gọi tới nó**, gồm `MASTER_IP`, `MASTER_CLUSTER_IP` và năm tên từ `kubernetes`
  tới `kubernetes.default.svc.cluster.local`.
- `MASTER_CLUSTER_IP` là **địa chỉ IP đầu tiên trong service CIDR**, tức dải truyền qua
  `--service-cluster-ip-range` cho **cả** API server lẫn controller manager — không phải một giá
  trị bạn tự chọn.
- Thời hạn là do bạn đặt và chỉ do bạn: `--days` của easyrsa, `-days` của openssl, `expiry` trong
  `ca-config.json` của cfssl — ví dụ trong bài đặt `10000` ngày. Certificate làm tay **không kèm
  theo quy trình canh hạn hay gia hạn nào**; đó chính là khoảng trống mà hai mục cuối bài và bài
  [219](219-kubeadm-certs-vi.md) lấp vào.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Toàn bộ mục `cfssl` — bốn file `config.json`, `csr.json`, `ca-config.json`, `server-csr.json` | ba công cụ cùng một mô hình; đọc kỹ `openssl` là đủ vì đó là công cụ có sẵn trên node lab | không cần cho lộ trình; mô hình chung đã nằm ở hai mục `easyrsa` và `openssl` của chính bài này |
| Việc thật sự thay ba tham số `--client-ca-file` / `--tls-cert-file` / `--tls-private-key-file` của một API server đang chạy | là sửa cấu hình control plane của cluster đang chạy | [giai đoạn 20](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) — bài [220](220-kubeadm-reconfigure-vi.md) |
| Mục *Certificates API* ở cuối bài | ở đây chỉ là một dòng trỏ hướng | hai bài kế của chính giai đoạn 18: [399](399-managing-tls-in-a-cluster-vi.md) và [397](397-certificate-issue-client-csr-vi.md) |
| Dùng bộ CA vừa tạo để thay CA của một cluster đang chạy | là thao tác nguy hiểm nhất nhóm, 12 bước, làm sai là mất quyền truy cập cluster | bài [400](400-manual-rotation-of-ca-certificates-vi.md), bài cuối của giai đoạn 18 |

---

Khi dùng xác thực bằng client certificate (client certificate authentication), bạn có
thể tạo certificate thủ công bằng [`easyrsa`](https://github.com/OpenVPN/easy-rsa),
[`openssl`](https://github.com/openssl/openssl) hoặc
[`cfssl`](https://github.com/cloudflare/cfssl).

### easyrsa

**easyrsa** có thể tạo certificate thủ công cho cluster của bạn.

1. Tải về, giải nén và khởi tạo phiên bản đã được vá (patched) của `easyrsa3`.

   ```shell
   curl -LO https://dl.k8s.io/easy-rsa/easy-rsa.tar.gz
   tar xzf easy-rsa.tar.gz
   cd easy-rsa-master/easyrsa3
   ./easyrsa init-pki
   ```
1. Tạo một certificate authority (CA) mới. `--batch` đặt chế độ tự động;
   `--req-cn` chỉ định Common Name (CN) cho root certificate mới của CA.

   ```shell
   ./easyrsa --batch "--req-cn=${MASTER_IP}@`date +%s`" build-ca nopass
   ```

1. Tạo certificate và key cho server.

   Đối số `--subject-alt-name` đặt các địa chỉ IP và tên DNS khả dĩ mà API server sẽ
   được truy cập qua đó. `MASTER_CLUSTER_IP` thường là địa chỉ IP đầu tiên trong
   service CIDR, tức dải được chỉ định qua đối số `--service-cluster-ip-range` cho cả
   API server lẫn thành phần controller manager. Đối số `--days` dùng để đặt số ngày
   mà sau đó certificate sẽ hết hạn.
   Ví dụ dưới đây cũng giả định rằng bạn đang dùng `cluster.local` làm tên miền DNS
   mặc định.

   ```shell
   ./easyrsa --subject-alt-name="IP:${MASTER_IP},"\
   "IP:${MASTER_CLUSTER_IP},"\
   "DNS:kubernetes,"\
   "DNS:kubernetes.default,"\
   "DNS:kubernetes.default.svc,"\
   "DNS:kubernetes.default.svc.cluster,"\
   "DNS:kubernetes.default.svc.cluster.local" \
   --days=10000 \
   build-server-full server nopass
   ```

1. Sao chép `pki/ca.crt`, `pki/issued/server.crt` và `pki/private/server.key` vào
   thư mục của bạn.

1. Điền và thêm các tham số sau vào các tham số khởi động của API server:

   ```shell
   --client-ca-file=/yourdirectory/ca.crt
   --tls-cert-file=/yourdirectory/server.crt
   --tls-private-key-file=/yourdirectory/server.key
   ```

### openssl

**openssl** có thể tạo certificate thủ công cho cluster của bạn.

1. Tạo ca.key với độ dài 2048 bit:

   ```shell
   openssl genrsa -out ca.key 2048
   ```

1. Dựa trên ca.key, tạo ca.crt (dùng `-days` để đặt thời gian hiệu lực của
   certificate):

   ```shell
   openssl req -x509 -new -noenc -key ca.key -subj "/CN=${MASTER_IP}" -days 10000 -out ca.crt
   ```

1. Tạo server.key với độ dài 2048 bit:

   ```shell
   openssl genrsa -out server.key 2048
   ```

1. Tạo một file cấu hình để sinh Certificate Signing Request (CSR).

   Nhớ thay các giá trị được đánh dấu bằng dấu ngoặc nhọn (ví dụ `<MASTER_IP>`)
   bằng giá trị thật trước khi lưu vào file (ví dụ `csr.conf`).
   Lưu ý rằng giá trị của `MASTER_CLUSTER_IP` là service cluster IP của
   API server như đã mô tả ở tiểu mục trước.
   Ví dụ dưới đây cũng giả định rằng bạn đang dùng `cluster.local` làm tên miền DNS
   mặc định.

   ```ini
   [ req ]
   default_bits = 2048
   prompt = no
   default_md = sha256
   req_extensions = req_ext
   distinguished_name = dn

   [ dn ]
   C = <country>
   ST = <state>
   L = <city>
   O = <organization>
   OU = <organization unit>
   CN = <MASTER_IP>

   [ req_ext ]
   subjectAltName = @alt_names

   [ alt_names ]
   DNS.1 = kubernetes
   DNS.2 = kubernetes.default
   DNS.3 = kubernetes.default.svc
   DNS.4 = kubernetes.default.svc.cluster
   DNS.5 = kubernetes.default.svc.cluster.local
   IP.1 = <MASTER_IP>
   IP.2 = <MASTER_CLUSTER_IP>

   [ v3_ext ]
   authorityKeyIdentifier=keyid,issuer:always
   basicConstraints=CA:FALSE
   keyUsage=keyEncipherment,dataEncipherment
   extendedKeyUsage=serverAuth,clientAuth
   subjectAltName=@alt_names
   ```

1. Sinh certificate signing request dựa trên file cấu hình:

   ```shell
   openssl req -new -key server.key -out server.csr -config csr.conf
   ```

1. Sinh certificate cho server bằng ca.key, ca.crt và server.csr:

   ```shell
   openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key \
       -CAcreateserial -out server.crt -days 10000 \
       -extensions v3_ext -extfile csr.conf -sha256
   ```

1. Xem certificate signing request:

   ```shell
   openssl req  -noout -text -in ./server.csr
   ```

1. Xem certificate:

   ```shell
   openssl x509  -noout -text -in ./server.crt
   ```

1. Điền và thêm các tham số sau vào các tham số khởi động của API server:

   ```shell
   --client-ca-file=/yourdirectory/ca.crt
   --tls-cert-file=/yourdirectory/server.crt
   --tls-private-key-file=/yourdirectory/server.key
   ```

### cfssl

**cfssl** là một công cụ khác để tạo certificate.

1. Tải về, giải nén và chuẩn bị các công cụ dòng lệnh như dưới đây.

   Lưu ý rằng bạn có thể cần điều chỉnh các lệnh mẫu tùy theo kiến trúc phần cứng và
   phiên bản cfssl mà bạn đang dùng.

   ```shell
   curl -L https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssl_1.5.0_linux_amd64 -o cfssl
   chmod +x cfssl
   curl -L https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssljson_1.5.0_linux_amd64 -o cfssljson
   chmod +x cfssljson
   curl -L https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssl-certinfo_1.5.0_linux_amd64 -o cfssl-certinfo
   chmod +x cfssl-certinfo
   ```

1. Tạo một thư mục để chứa các artifact và khởi tạo cfssl:

   ```shell
   mkdir cert
   cd cert
   ../cfssl print-defaults config > config.json
   ../cfssl print-defaults csr > csr.json
   ```

1. Tạo một file cấu hình JSON để sinh file CA, ví dụ `ca-config.json`:

   ```json
   {
     "signing": {
       "default": {
         "expiry": "8760h"
       },
       "profiles": {
         "kubernetes": {
           "usages": [
             "signing",
             "key encipherment",
             "server auth",
             "client auth"
           ],
           "expiry": "8760h"
         }
       }
     }
   }
   ```

1. Tạo một file cấu hình JSON cho CA certificate signing request (CSR), ví dụ
   `ca-csr.json`. Nhớ thay các giá trị được đánh dấu bằng dấu ngoặc nhọn bằng
   giá trị thật mà bạn muốn dùng.

   ```json
   {
     "CN": "kubernetes",
     "key": {
       "algo": "rsa",
       "size": 2048
     },
     "names":[{
       "C": "<country>",
       "ST": "<state>",
       "L": "<city>",
       "O": "<organization>",
       "OU": "<organization unit>"
     }]
   }
   ```

1. Sinh CA key (`ca-key.pem`) và certificate (`ca.pem`):

   ```shell
   ../cfssl gencert -initca ca-csr.json | ../cfssljson -bare ca
   ```

1. Tạo một file cấu hình JSON để sinh key và certificate cho API server, ví dụ
   `server-csr.json`. Nhớ thay các giá trị trong ngoặc nhọn bằng giá trị thật mà bạn
   muốn dùng. `<MASTER_CLUSTER_IP>` là service cluster IP của API server như đã mô tả
   ở tiểu mục trước.
   Ví dụ dưới đây cũng giả định rằng bạn đang dùng `cluster.local` làm tên miền DNS
   mặc định.

   ```json
   {
     "CN": "kubernetes",
     "hosts": [
       "127.0.0.1",
       "<MASTER_IP>",
       "<MASTER_CLUSTER_IP>",
       "kubernetes",
       "kubernetes.default",
       "kubernetes.default.svc",
       "kubernetes.default.svc.cluster",
       "kubernetes.default.svc.cluster.local"
     ],
     "key": {
       "algo": "rsa",
       "size": 2048
     },
     "names": [{
       "C": "<country>",
       "ST": "<state>",
       "L": "<city>",
       "O": "<organization>",
       "OU": "<organization unit>"
     }]
   }
   ```

1. Sinh key và certificate cho API server; theo mặc định, chúng được lưu lần lượt
   vào hai file `server-key.pem` và `server.pem`:

   ```shell
   ../cfssl gencert -ca=ca.pem -ca-key=ca-key.pem \
        --config=ca-config.json -profile=kubernetes \
        server-csr.json | ../cfssljson -bare server
   ```

## Phân phối certificate CA tự ký (Distributing Self-Signed CA Certificate)

Một node client có thể từ chối công nhận một certificate CA tự ký (self-signed) là
hợp lệ. Với một triển khai không dùng cho production, hoặc một triển khai chạy phía
sau firewall của công ty, bạn có thể phân phối certificate CA tự ký đến tất cả các
client và làm mới danh sách cục bộ các certificate hợp lệ.

Trên mỗi client, thực hiện các thao tác sau:

```shell
sudo cp ca.crt /usr/local/share/ca-certificates/kubernetes.crt
sudo update-ca-certificates
```

```none
Updating certificates in /etc/ssl/certs...
1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d....
done.
```

## Certificates API

Bạn có thể dùng API `certificates.k8s.io` để cấp phát (provision) các certificate
x509 dùng cho việc xác thực, như được mô tả trong trang tác vụ
[Quản lý TLS trong một cluster](399-managing-tls-in-a-cluster-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 18:

1. Bỏ qua khác biệt cú pháp, ba công cụ trong bài đi qua đúng một dãy bốn bước. Dãy đó là gì, và
   kết thúc bằng việc giao những file nào cho API server?
2. **Câu bẫy.** Bạn tạo certificate cho API server bằng `openssl` với `CN = <MASTER_IP>` đúng địa
   chỉ của `lab-k8s-master`, nhưng để trống mục `[ alt_names ]`. Certificate có `CN` đúng rồi thì
   client gọi qua `https://kubernetes.default.svc` có dùng được không, và vì sao?
3. Bài nói `MASTER_CLUSTER_IP` thường là địa chỉ IP đầu tiên trong service CIDR. Trên cluster lab
   của bạn, giá trị đó đến từ tham số nào và của những thành phần nào? Vì sao nó bắt buộc phải nằm
   trong danh sách tên thay thế của certificate API server?
4. Ba tham số khởi động ở cuối mục `easyrsa` và mục `openssl` chia làm hai nhóm chức năng. Nhóm nào
   dùng để xác thực client, nhóm nào để phục vụ TLS, và file nào phải có mặt ở cả hai phía của kết
   nối?
5. Ví dụ trong bài đặt thời hạn `10000` ngày. Hệ quả vận hành của việc tự đặt hạn như vậy là gì, và
   bài chỉ ra con đường nào để không phải tự làm và tự canh hạn?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Tạo CA, tạo private key cho server, sinh CSR rồi ký nó bằng CA, cuối cùng cấu hình API server
   dùng kết quả.** Ba file được giao cho API server: **`ca.crt`** (certificate của CA),
   **`server.crt`** (certificate của API server đã được CA ký) và **`server.key`** (private key của
   server) — tên đổi theo công cụ, ví dụ `ca.pem`, `server.pem` và `server-key.pem` với cfssl.
2. **Không dùng được.** Chỗ khai các tên gọi là `--subject-alt-name` / `[ alt_names ]` / `hosts`,
   và bài nói rõ nó đặt **các địa chỉ IP và tên DNS khả dĩ mà API server sẽ được truy cập qua đó**
   — danh sách đầy đủ gồm `kubernetes`, `kubernetes.default`, `kubernetes.default.svc`,
   `kubernetes.default.svc.cluster`, `kubernetes.default.svc.cluster.local` cùng hai địa chỉ IP.
   Đây là chỗ dễ sai vì `CN` **trông như** đã định danh xong server, nhưng nó chỉ phục vụ đúng một
   cách gọi; mọi tên khác mà client dùng đều phải có mặt trong danh sách tên thay thế.
3. Đến từ **`--service-cluster-ip-range`**, và bài nhấn mạnh dải này được chỉ định cho **cả API
   server lẫn thành phần controller manager** — nên trên cluster lab bạn đọc nó từ tham số khởi
   động của hai thành phần đó, không tự chọn. Nó bắt buộc phải nằm trong danh sách tên thay thế vì
   đó chính là **địa chỉ mà API server được truy cập từ bên trong cluster**; thiếu nó thì client
   gọi qua đường nội bộ sẽ không chấp nhận certificate.
4. **`--client-ca-file` dùng để xác thực client**: API server lấy CA này kiểm certificate mà client
   xuất trình. **`--tls-cert-file` và `--tls-private-key-file` dùng để phục vụ TLS**: cặp
   certificate và key mà API server trình ra. File phải có mặt ở **cả hai phía** là **`ca.crt`** —
   server dùng nó để kiểm client, còn client phải tin nó thì mới chấp nhận `server.crt`. Đó chính
   là lý do bài có mục *Phân phối certificate CA tự ký*.
5. Hệ quả: **hạn dùng là con số bạn chọn một lần**, và bài **không kèm bất kỳ quy trình canh hạn
   hay gia hạn nào** — làm tay thì tự chịu trách nhiệm nhớ. Con đường thay thế nằm ở mục
   *Certificates API* cuối bài: dùng API **`certificates.k8s.io`** để cấp phát certificate x509
   dùng cho việc xác thực, như mô tả ở bài [399](399-managing-tls-in-a-cluster-vi.md). Với cluster
   kubeadm thì phần vòng đời do kubeadm lo, xem bài [219](219-kubeadm-certs-vi.md).

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau. Cả giai đoạn 18
chấm bằng **Checkpoint** ở cuối mục
[Giai đoạn 18 — Vòng đời chứng chỉ](00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ), làm
trên cluster lab chứ không có lab riêng.
