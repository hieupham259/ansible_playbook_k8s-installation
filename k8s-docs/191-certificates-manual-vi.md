# Tạo certificate thủ công (Generate Certificates Manually)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/certificates/>
>
> Khi dùng xác thực bằng client certificate, bạn có thể tạo certificate thủ công bằng
> `easyrsa`, `openssl` hoặc `cfssl`.

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
[Quản lý TLS trong một cluster](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster).
