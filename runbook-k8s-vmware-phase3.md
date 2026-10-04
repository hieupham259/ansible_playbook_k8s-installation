# Runbook Phase 3: Bật TLS giữa backend FastAPI và MongoDB

> **Phụ thuộc:** hoàn thành Phase 1 trong [`runbook-k8s-vmware.md`](runbook-k8s-vmware.md) — đặc biệt cert-manager ở [§14.1](runbook-k8s-vmware.md#141-cài-cert-manager-rancher-cần-để-cấp-tls-nội-bộ) — và toàn bộ [checklist §17 của Phase 2](runbook-k8s-vmware-phase2.md#17-checklist-hoàn-tất) trong [`runbook-k8s-vmware-phase2.md`](runbook-k8s-vmware-phase2.md).
>
> **Phạm vi Phase 3:** xử lý đúng một mục trong [§15.2 của Phase 2](runbook-k8s-vmware-phase2.md#152-những-gì-baseline-chưa-giải-quyết) — *"MongoDB traffic trong cluster chưa bật TLS"*. Sau Phase 3, MongoDB chỉ nhận kết nối TLS; backend, probe, `mongosh`, `mongodump` và `mongorestore` đều nói chuyện với MongoDB qua TLS và kiểm tra certificate của server. Không đổi dữ liệu, PVC, credential, frontend, Traefik hay Cloudflare Tunnel.
>
> **Cách chạy bắt buộc:** giống Phase 2 — thực hiện từng checkpoint theo thứ tự. Sau mỗi khối có nhãn **DỪNG — GỬI OUTPUT**, gửi nguyên output để kiểm tra; chỉ sang checkpoint tiếp theo khi kết quả được xác nhận **PASS**. Không gửi password, private key hay giá trị Secret.
>
> **Môi trường:** mọi lệnh `kubectl`, `openssl` chạy trên `k8s-master` bằng user `ubuntu`, từ thư mục `~/three-tier-crud`, trừ khi tiêu đề ghi rõ **máy build** hoặc **worker**. Máy build là máy Windows, chạy Git Bash trong repository ứng dụng `E:/courses/Ansible/three-tier-crud` như [Phase 2 §4.1](runbook-k8s-vmware-phase2.md#41-các-giá-trị-duy-nhất-được-dùng-trong-toàn-runbook).
>
> **Ngày đối chiếu tài liệu:** 04/10/2026.

---

## Mục lục

1. [Mục tiêu, cơ chế và giới hạn](#1-mục-tiêu-cơ-chế-và-giới-hạn)
2. [Baseline và mức gián đoạn](#2-baseline-và-mức-gián-đoạn)
3. [Gate đầu vào](#3-gate-đầu-vào)
4. [Chốt tên, đường dẫn và quy trình sửa manifest](#4-chốt-tên-đường-dẫn-và-quy-trình-sửa-manifest)
5. [Cấp CA và certificate bằng cert-manager](#5-cấp-ca-và-certificate-bằng-cert-manager)
6. [Bước 1 — MongoDB nhận cả TLS lẫn plaintext](#6-bước-1--mongodb-nhận-cả-tls-lẫn-plaintext)
7. [Bước 2 — Chuyển backend sang TLS](#7-bước-2--chuyển-backend-sang-tls)
8. [Bước 3 — MongoDB chỉ nhận TLS](#8-bước-3--mongodb-chỉ-nhận-tls)
9. [Kiểm thử end-to-end sau khi bật TLS](#9-kiểm-thử-end-to-end-sau-khi-bật-tls)
10. [Backup và restore qua TLS](#10-backup-và-restore-qua-tls)
11. [Rollback](#11-rollback)
12. [Vận hành: gia hạn certificate](#12-vận-hành-gia-hạn-certificate)
13. [Troubleshooting](#13-troubleshooting)
14. [Checklist hoàn tất](#14-checklist-hoàn-tất)
15. [Nguồn official](#15-nguồn-official)

---

## 1. Mục tiêu, cơ chế và giới hạn

### 1.1. Kết quả cần đạt

- MongoDB chạy `--tlsMode requireTLS`: kết nối plaintext bị từ chối.
- Backend kết nối bằng URI có `tls=true`, kiểm tra certificate server bằng CA nội bộ và kiểm tra hostname `mongodb` có trong certificate.
- Probe, `mongosh`, `mongodump`, `mongorestore` trong Pod MongoDB đều dùng TLS.
- Certificate do cert-manager cấp từ một CA riêng của namespace `three-tier`; có quy trình gia hạn và kiểm tra.
- CRUD qua Traefik và qua domain public vẫn `200`; số document trước và sau Phase 3 bằng nhau; backup/restore rehearsal qua TLS PASS.

### 1.2. Chặng nào được mã hóa

Trước Phase 3:

```text
Browser ──HTTPS──► Cloudflare Edge ──tunnel, có mã hóa──► cloudflared
────────────────────── từ đây trở vào: không mã hóa ──────────────────────
cloudflared ─HTTP:80─► Traefik ─HTTP─► Nginx (frontend) ─HTTP:8000─► FastAPI (backend)
FastAPI ──TCP thường :27017──► mongodb-0
```

Sau Phase 3, chỉ chặng cuối đổi:

```text
FastAPI ──TLS :27017, verify cert + hostname "mongodb"──► mongodb-0
```

Chặng cuối được chọn trước vì nó mang toàn bộ dữ liệu database. Khi backend và `mongodb-0` nằm trên hai worker khác nhau, gói tin đi qua Flannel VXLAN (UDP 8472) trên LAN `192.168.100.0/24` — mạng Bridged, tức chính mạng nhà. VXLAN chỉ đóng gói, không mã hóa. MongoDB đăng nhập bằng SCRAM nên password không đi nguyên văn, nhưng mọi query và document sau khi đăng nhập thì có. Các chặng HTTP khác trong cụm vẫn plaintext; Phase 3 không xử lý chúng.

### 1.3. Chuỗi tin cậy — ai giữ gì

```text
Issuer mongodb-selfsigned ──ký──► Certificate mongodb-ca (isCA)  ──► Secret mongodb-ca
                                                                        (CA cert + CA key)
Issuer mongodb-ca-issuer ──dùng Secret mongodb-ca──ký──► Certificate mongodb-server
                                                          ──► Secret mongodb-tls
                                                              (server cert + server key + ca.crt)
```

| Object | Chứa | Ai được đọc |
| --- | --- | --- |
| Secret `mongodb-ca` | CA certificate **và CA private key** | chỉ cert-manager. Không mount vào Pod nào |
| Secret `mongodb-tls` | server certificate, **server private key**, `ca.crt` | chỉ Pod MongoDB: initContainer đọc cert + key, container chính chỉ nhận `ca.crt` |
| ConfigMap `mongodb-ca-bundle` | chỉ CA certificate (public) | backend |

Backend chỉ cần CA certificate để kiểm tra server; nó không bao giờ được nhận Secret chứa private key. Vì vậy CA được chép sang một ConfigMap riêng ở [§5.3](#53-tạo-configmap-ca-cho-backend).

### 1.4. Vì sao đi ba bước

`tlsMode` là tham số khởi động của `mongod`, nằm trong manifest; đổi nó nghĩa là restart `mongodb-0`. MongoDB chỉ có một replica nên mỗi lần restart là một khoảng `/api/*` lỗi tạm thời. Bật thẳng `requireTLS` khi backend còn dùng URI plaintext thì backend mất kết nối cho tới khi đổi xong backend. Ba bước dưới đây giữ mỗi lần thay đổi ở trạng thái mà cả hai phía vẫn nói chuyện được:

| Bước | Server MongoDB | Backend | Probe/tool trong Pod MongoDB |
| --- | --- | --- | --- |
| Trước Phase 3 | không TLS | plaintext | plaintext |
| §6 — Bước 1 | `preferTLS`: nhận **cả** TLS lẫn plaintext | plaintext — vẫn chạy | TLS |
| §7 — Bước 2 | `preferTLS` | **TLS** | TLS |
| §8 — Bước 3 | `requireTLS`: chỉ nhận TLS | TLS | TLS |

Theo tài liệu MongoDB, `allowTLS` và `preferTLS` đều nhận cả TLS lẫn plaintext ở kết nối đến; khác nhau ở kết nối giữa các server với nhau. MongoDB ở đây là standalone nên hai giá trị tương đương; runbook dùng `preferTLS`.

Backend PyMongo với `tls=true` **không bao giờ lùi về plaintext**. Vì vậy ở Bước 2, Pod backend mới đạt Ready nghĩa là TLS handshake, kiểm tra CA và kiểm tra hostname đều đã thành công.

### 1.5. Ba quyết định cấu hình phải hiểu trước khi làm

**CA file bắt buộc trên server.** Tài liệu MongoDB 8.0: khi bật TLS cho `mongod` phải khai `--tlsCAFile` (hoặc `tlsUseSystemCA`). Nhưng khi đã có CA file, mặc định server **đòi client trình certificate**. Backend chỉ dùng TLS để mã hóa và xác thực server, còn đăng nhập vẫn bằng username/password, nên runbook thêm `--tlsAllowConnectionsWithoutCertificates`.

**Một file PEM cho `mongod`.** `--tlsCertificateKeyFile` phải là một file chứa cả certificate lẫn private key. cert-manager ghi hai key riêng `tls.crt`, `tls.key`. Một initContainer ghép hai file này vào `emptyDir` chạy trên RAM (`medium: Memory`), đổi owner sang user `mongodb` và mode `0400`. Hệ quả: initContainer chỉ chạy khi Pod khởi động, nên certificate gia hạn chỉ có hiệu lực sau khi restart `mongodb-0` — quy trình ở [§12](#12-vận-hành-gia-hạn-certificate).

**Danh sách tên trong certificate (SAN).** Client kiểm tra tên nó dùng để kết nối phải nằm trong certificate:

| Tên | Ai dùng |
| --- | --- |
| `mongodb` | backend — URI `mongodb://...@mongodb:27017/...` của Phase 2 §9.1 |
| `mongodb.three-tier`, `mongodb.three-tier.svc`, `mongodb.three-tier.svc.cluster.local` | các dạng FQDN của Service, dự phòng khi đổi URI |
| `mongodb-0.mongodb.three-tier.svc.cluster.local` | DNS ổn định của Pod qua Service headless |
| `localhost`, IP `127.0.0.1` | probe và `mongosh`/`mongodump` chạy ngay trong Pod với `--host 127.0.0.1` |

IP của Pod **cố ý không** nằm trong certificate — [§8.3](#83-gate-âm--chứng-minh-server-chỉ-nhận-tls-và-client-thật-sự-kiểm-tra) dùng chính điều đó để chứng minh client có kiểm tra hostname.

Script init trong ConfigMap `mongodb-init` của Phase 2 không cần sửa: nó chỉ chạy khi data directory rỗng, và lúc đó entrypoint của image tự chạy server tạm với `--tlsMode allowTLS` nếu thấy `--tlsCertificateKeyFile`, nên script `mongosh --host 127.0.0.1` plaintext vẫn chạy được.

### 1.6. Giới hạn — Phase 3 không làm

- Không dùng client certificate (x.509) để đăng nhập; backend vẫn dùng password.
- Không mã hóa các chặng HTTP khác trong cụm.
- Không thay NetworkPolicy: Pod khác trong cụm vẫn kết nối được tới `mongodb:27017`, chỉ là phải nói TLS và phải có password.
- Secret `mongodb-ca`, `mongodb-tls` nằm trong etcd dạng base64 như mọi Secret khác — encryption at rest vẫn là mục mở của Phase 2 §15.2.
- CA private key nằm ngay trong cụm. Ai đọc được Secret `mongodb-ca` thì ký được certificate giả cho MongoDB.

---

## 2. Baseline và mức gián đoạn

| Thành phần | Giá trị | Ghi chú |
| --- | --- | --- |
| Kubernetes | v1.35.6 | Phase 1, không đổi |
| cert-manager | chart `v1.21.1` | Phase 1 §14.1 — dùng lại, không cài thêm |
| MongoDB | `docker.io/library/mongo:8.0.29-noble` | Phase 2, không đổi. `mongosh`, `mongodump`, `mongorestore` có sẵn trong image; initContainer dùng cùng image nên không pull thêm |
| Backend image | giữ nguyên digest Phase 2 | PyMongo `4.18.0` hỗ trợ `tls`/`tlsCAFile` trong URI — không build lại |
| CA | RSA 2048, `duration: 87600h` (10 năm) | cert-manager tự gia hạn ở khoảng 2/3 thời hạn |
| Server certificate | RSA 2048, `duration: 8760h` (1 năm), `renewBefore: 2160h` (90 ngày) | gia hạn xong phải restart `mongodb-0`, có 90 ngày để làm |

Phase 3 không tải phần mềm mới và không pull image mới.

Gián đoạn dự kiến:

| Mục | Restart | Ảnh hưởng |
| --- | --- | --- |
| §6 | `mongodb-0` | `/api/*` trả lỗi cho tới khi `mongodb-0` Ready lại; frontend tĩnh vẫn phục vụ |
| §7 | rolling update backend | không gián đoạn: `maxUnavailable: 0`, Pod mới phải Ready mới thay Pod cũ |
| §8 | `mongodb-0` | như §6 |

Thời gian mỗi lần restart phụ thuộc tốc độ `mongod` khởi động và chu kỳ probe; không coi con số nào là cam kết.

---

## 3. Gate đầu vào

### 3.1. Phase 2 đang khỏe và chưa có object của Phase 3

Trên `k8s-master`, trong `~/three-tier-crud`:

```bash
cd ~/three-tier-crud
test -f ~/phase2-app.env && source ~/phase2-app.env && printf 'APP_HOST=%s\nING_IP=%s\n' "$APP_HOST" "$ING_IP"
kubectl get nodes
kubectl -n three-tier get statefulset,deploy,pod -o wide
curl -sS -o /dev/null -w 'api-ready-via-traefik=%{http_code}\n' \
  -H "Host: $APP_HOST" "http://$ING_IP/api/health/ready"
helm list -n cert-manager
kubectl -n cert-manager get deploy
kubectl get crd certificates.cert-manager.io issuers.cert-manager.io
kubectl -n three-tier get secret/mongodb-ca secret/mongodb-tls \
  secret/backend-mongodb-uri-tls configmap/mongodb-ca-bundle 2>&1
kubectl -n three-tier get certificates.cert-manager.io,issuers.cert-manager.io 2>&1
printf 'mongodb-args=[%s]\n' \
  "$(kubectl -n three-tier get statefulset mongodb -o jsonpath='{.spec.template.spec.containers[0].args}')"
printf 'uri-has-tls=%s\n' \
  "$(kubectl -n three-tier get secret backend-mongodb-uri -o jsonpath='{.data.MONGODB_URI}' | base64 -d | grep -c 'tls=')"
```

PASS khi:

- `APP_HOST` là domain thật của Phase 2 (`crud.hieupn.site`), `ING_IP` không rỗng;
- 3 node `Ready`; `mongodb-0` `1/1 Running`, backend `2/2`, frontend `2/2`;
- `api-ready-via-traefik=200`;
- release `cert-manager` ở trạng thái `deployed`, chart `cert-manager-v1.21.1`; ba Deployment `cert-manager`, `cert-manager-cainjector`, `cert-manager-webhook` đều `1/1`; hai CRD tồn tại;
- bốn dòng `NotFound` cho `mongodb-ca`, `mongodb-tls`, `backend-mongodb-uri-tls`, `mongodb-ca-bundle`; namespace chưa có Certificate/Issuer nào (`No resources found`);
- `mongodb-args=[]` và `uri-has-tls=0`.

Nếu một object của Phase 3 đã tồn tại, đây là lần chạy lại: dừng và gửi output để xác định checkpoint resume, không apply đè.

> **DỪNG — GỬI OUTPUT CHECKPOINT 3.1.**

### 3.2. Backup trước khi thay đổi

**Backup logic MongoDB.** Chạy nguyên văn [§14.2 của Phase 2](runbook-k8s-vmware-phase2.md#142-backup-logic-bằng-mongodump). Lúc này MongoDB chưa bật TLS nên khối của Phase 2 vẫn đúng.

**Backup etcd.** Chạy tay script cron đã cài theo [§8.1 của runbook restore](runbook-k8s-vmware-etcd-restore.md#81-backup-etcd-định-kỳ-tự-động), đúng môi trường cron dùng:

```bash
sudo env -i PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
  /usr/local/sbin/etcd-backup.sh
ls -l ~/k8s-backups/ | tail -3
```

**Đếm document hiện có** để so sánh ở §9.3:

```bash
install -d -m 700 ~/phase3
kubectl -n three-tier exec mongodb-0 -- mongosh --quiet --host 127.0.0.1 --eval '
  const appDb = db.getSiblingDB(process.env.MONGO_APP_DATABASE)
  if (!appDb.auth(process.env.MONGO_APP_USERNAME, process.env.MONGO_APP_PASSWORD)) {
    throw new Error("MongoDB app-user authentication failed")
  }
  print(appDb.items.countDocuments({}))
' | tee ~/phase3/items-count-before.txt
```

PASS khi: §14.2 Phase 2 PASS (archive mới > 0 byte, mode `600`); script etcd in `PASS: <STAMP> <sha256>`; file `~/phase3/items-count-before.txt` chứa đúng một số nguyên.

> Muốn có bản etcd nằm ngoài VM trước thay đổi, chạy thêm bước 2 của [Phase 1 §14.0.1](runbook-k8s-vmware.md#1401-backup-etcd-và-cấu-hình-trước-thay-đổi-lớn) với `STAMP` và hash vừa in.

> **DỪNG — GỬI OUTPUT CHECKPOINT 3.2; KHÔNG GỬI FILE BACKUP.**

### 3.3. Lưu manifest Phase 2 để rollback

Trên `k8s-master`, trước khi đồng bộ bất kỳ manifest mới nào:

```bash
cd ~/three-tier-crud
install -d -m 700 ~/phase3-rollback
cp k8s/10-mongodb.yaml ~/phase3-rollback/10-mongodb.phase2.yaml
cp k8s/20-backend.yaml ~/phase3-rollback/20-backend.phase2.yaml
grep -c 'tlsMode' ~/phase3-rollback/10-mongodb.phase2.yaml
grep -n 'name: backend-mongodb-uri' ~/phase3-rollback/20-backend.phase2.yaml
ls -l ~/phase3-rollback/
```

PASS khi `grep -c` in `0`, dòng `secretKeyRef` trỏ `name: backend-mongodb-uri` (không có hậu tố `-tls`), và hai file tồn tại.

> **DỪNG — GỬI OUTPUT CHECKPOINT 3.3.**

### 3.4. (Tùy chọn) Bắt gói tin plaintext — ảnh chụp "trước"

Mục này cho **thấy tận mắt** dữ liệu đang đi nguyên văn; không phải gate bắt buộc. Cần `tcpdump` có sẵn trên worker. Nếu worker không có `tcpdump`, bỏ qua §3.4 và §9.2 — runbook không cài thêm gói.

Tìm worker đang chạy `mongodb-0` (trên `k8s-master`):

```bash
kubectl -n three-tier get pod mongodb-0 -o jsonpath='{.spec.nodeName}{"\n"}'
```

**Terminal A — trên worker vừa in ra.** Mọi traffic tới `mongodb-0`, kể cả từ backend ở worker khác, đều đi qua bridge `cni0` của Flannel trên node này:

```bash
command -v tcpdump && ip -br link show cni0
sudo -v
sudo timeout 40 tcpdump -l -i cni0 -nn -A -s 0 'tcp port 27017' \
  > /tmp/phase3-wire-before.txt 2>/dev/null &
```

**Terminal B — trên `k8s-master`, trong vòng 40 giây:**

```bash
source ~/phase2-app.env
curl -sS -o /dev/null -w 'create=%{http_code}\n' -X POST -H "Host: $APP_HOST" \
  -H 'Content-Type: application/json' "http://$ING_IP/api/items" \
  -d '{"id":"phase3-wire-before","name":"phase3-wire-before","description":"plaintext capture marker"}'
```

**Terminal A — sau khi `timeout` kết thúc:**

```bash
wait
printf 'packets-27017=%s\nmarker-hits=%s\n' \
  "$(grep -c '\.27017' /tmp/phase3-wire-before.txt)" \
  "$(grep -c 'phase3-wire-before' /tmp/phase3-wire-before.txt)"
rm -f /tmp/phase3-wire-before.txt
```

PASS khi `create=201`, `packets-27017` > 0 và `marker-hits` > 0: chuỗi vừa gửi qua API xuất hiện nguyên văn trong gói tin tới MongoDB. File capture chứa dữ liệu ứng dụng nên bị xóa ngay. Item `phase3-wire-before` được xóa ở §9.2.

> **DỪNG — GỬI OUTPUT CHECKPOINT 3.4 (hoặc ghi "bỏ qua §3.4").**

---

## 4. Chốt tên, đường dẫn và quy trình sửa manifest

### 4.1. Giá trị dùng trong toàn runbook

| Mục | Giá trị |
| --- | --- |
| Namespace | `three-tier` |
| Issuer tự ký | `mongodb-selfsigned` |
| Certificate CA → Secret | `mongodb-ca` → `mongodb-ca` |
| Issuer CA | `mongodb-ca-issuer` |
| Certificate server → Secret | `mongodb-server` → `mongodb-tls` |
| ConfigMap CA cho backend | `mongodb-ca-bundle`, key `ca.crt` |
| Secret URI mới của backend | `backend-mongodb-uri-tls`, key `MONGODB_URI` |
| File PEM ghép (container `mongodb`) | `/etc/mongodb/tls/mongodb.pem` |
| CA trong container `mongodb` | `/etc/mongodb/ca/ca.crt` |
| CA trong container `backend` | `/etc/mongodb-ca/ca.crt` |
| Hậu tố thêm vào URI backend | `&tls=true&tlsCAFile=/etc/mongodb-ca/ca.crt` |
| Manifest mới | `k8s/05-mongodb-tls.yaml` (số `05` để đứng trước `10-mongodb.yaml`) |
| Thư mục làm việc trên master | `~/phase3` (bằng chứng public), `~/phase3-rollback` (manifest cũ) |

Secret URI cũ `backend-mongodb-uri` được **giữ nguyên** tới hết Phase 3 — nó là đường rollback của backend.

### 4.2. Quy trình sửa manifest

Manifest là source of truth trong repository ứng dụng, như Phase 2 §8.1. Mỗi lần runbook bảo sửa một file `k8s/*.yaml`:

1. Sửa trên **máy build**, trong `E:/courses/Ansible/three-tier-crud`, rồi commit.
2. Đồng bộ thư mục `k8s/` lên master theo đúng cách đã dùng ở [Phase 2 §8.1](runbook-k8s-vmware-phase2.md#81-tạo-manifest-database):

   ```bash
   # Cách A — master là bản clone: push từ máy build, rồi trên k8s-master:
   cd ~/three-tier-crud && git pull --ff-only

   # Cách B — chép tay: chạy trên máy build, từ thư mục repo
   cd E:/courses/Ansible/three-tier-crud
   scp -r k8s ubuntu@192.168.100.111:~/three-tier-crud/
   ```

3. Trên master, chạy gate `grep` của bước đó **trước** khi `kubectl apply`. Gate kiểm nội dung thay vì so hash, vì file trên máy build Windows có thể mang CRLF còn bản Git trên master là LF.

---

## 5. Cấp CA và certificate bằng cert-manager

Mục này chỉ tạo object cert-manager và Secret; **không** đụng tới Pod nào, không gián đoạn.

### 5.1. Manifest `k8s/05-mongodb-tls.yaml`

Trên **máy build**, tạo `k8s/05-mongodb-tls.yaml` với nội dung:

```yaml
# Phase 3 — CA riêng của namespace three-tier và certificate cho MongoDB.
# Chuỗi: Issuer tự ký -> Certificate CA (isCA) -> Issuer CA -> Certificate server.
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: mongodb-selfsigned
  namespace: three-tier
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: mongodb-ca
  namespace: three-tier
spec:
  isCA: true
  # commonName cho CA một Subject DN không rỗng; certificate do CA này ký mới có Issuer DN hợp lệ.
  commonName: three-tier-mongodb-ca
  secretName: mongodb-ca
  duration: 87600h
  privateKey:
    algorithm: RSA
    size: 2048
  issuerRef:
    name: mongodb-selfsigned
    kind: Issuer
    group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: mongodb-ca-issuer
  namespace: three-tier
spec:
  ca:
    secretName: mongodb-ca
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: mongodb-server
  namespace: three-tier
spec:
  secretName: mongodb-tls
  commonName: mongodb.three-tier.svc.cluster.local
  # Mọi tên mà client dùng để kết nối phải có ở đây; client kiểm tra hostname theo danh sách này.
  dnsNames:
    - mongodb
    - mongodb.three-tier
    - mongodb.three-tier.svc
    - mongodb.three-tier.svc.cluster.local
    - mongodb-0.mongodb.three-tier.svc.cluster.local
    - localhost
  ipAddresses:
    - 127.0.0.1
  duration: 8760h
  renewBefore: 2160h
  usages:
    - server auth
    - digital signature
    - key encipherment
  privateKey:
    algorithm: RSA
    size: 2048
    rotationPolicy: Always
  issuerRef:
    name: mongodb-ca-issuer
    kind: Issuer
    group: cert-manager.io
```

Commit trên máy build:

```bash
cd E:/courses/Ansible/three-tier-crud
git add k8s/05-mongodb-tls.yaml
git commit -m "add: add 05-mongodb-tls.yaml"
git log -1 --format='%h %s'
```

Đồng bộ lên master theo [§4.2](#42-quy-trình-sửa-manifest), rồi trên `k8s-master`:

```bash
cd ~/three-tier-crud
grep -c '^kind: ' k8s/05-mongodb-tls.yaml
grep -A7 'dnsNames:' k8s/05-mongodb-tls.yaml
kubectl apply --dry-run=server -f k8s/05-mongodb-tls.yaml
```

PASS khi `grep -c` in `4`, danh sách tên đúng sáu DNS như trên, và dry-run trả về bốn object `created (server dry run)` không lỗi.

> **DỪNG — GỬI OUTPUT CHECKPOINT 5.1.**

### 5.2. Apply và kiểm tra certificate

```bash
cd ~/three-tier-crud
kubectl apply -f k8s/05-mongodb-tls.yaml
kubectl -n three-tier wait --for=condition=Ready certificate.cert-manager.io/mongodb-ca --timeout=120s
kubectl -n three-tier wait --for=condition=Ready issuer.cert-manager.io/mongodb-ca-issuer --timeout=120s
kubectl -n three-tier wait --for=condition=Ready certificate.cert-manager.io/mongodb-server --timeout=120s
kubectl -n three-tier get issuers.cert-manager.io,certificates.cert-manager.io
kubectl -n three-tier get secret mongodb-ca mongodb-tls \
  -o go-template='{{range .items}}{{.metadata.name}} type={{.type}} keys={{range $k,$v := .data}}{{$k}} {{end}}{{"\n"}}{{end}}'
```

Chỉ lấy phần **public** của certificate ra master để kiểm tra; không lấy `tls.key`:

```bash
kubectl -n three-tier get secret mongodb-tls -o jsonpath='{.data.tls\.crt}' | base64 -d > ~/phase3/server.crt
kubectl -n three-tier get secret mongodb-tls -o jsonpath='{.data.ca\.crt}' | base64 -d > ~/phase3/ca.crt
openssl x509 -in ~/phase3/server.crt -noout -subject -issuer -dates -ext subjectAltName,extendedKeyUsage
openssl x509 -in ~/phase3/ca.crt -noout -subject -dates -ext basicConstraints
openssl verify -CAfile ~/phase3/ca.crt ~/phase3/server.crt
```

PASS khi:

- hai Issuer và hai Certificate đều `READY=True`;
- hai Secret type `kubernetes.io/tls`, mỗi Secret có key `ca.crt tls.crt tls.key`;
- server certificate: issuer `CN = three-tier-mongodb-ca`; Subject Alternative Name có đủ sáu `DNS:` ở §5.1 và `IP Address:127.0.0.1`; Extended Key Usage có `TLS Web Server Authentication`; `notAfter` cách hiện tại khoảng một năm;
- CA certificate: `CA:TRUE`; `notAfter` cách hiện tại khoảng mười năm;
- `openssl verify` in `OK`.

> **DỪNG — GỬI OUTPUT CHECKPOINT 5.2.**

### 5.3. Tạo ConfigMap CA cho backend

ConfigMap chỉ chứa CA certificate, lấy từ `ca.crt` vừa kiểm tra. Đối chiếu fingerprint với chính certificate trong Secret CA để chắc chắn backend sẽ tin đúng CA đang ký certificate MongoDB:

```bash
kubectl -n three-tier create configmap mongodb-ca-bundle \
  --from-file=ca.crt="$HOME/phase3/ca.crt" \
  --dry-run=client -o yaml | kubectl apply -f -
CM_FP=$(kubectl -n three-tier get configmap mongodb-ca-bundle -o jsonpath='{.data.ca\.crt}' \
  | openssl x509 -noout -fingerprint -sha256)
CA_FP=$(kubectl -n three-tier get secret mongodb-ca -o jsonpath='{.data.tls\.crt}' | base64 -d \
  | openssl x509 -noout -fingerprint -sha256)
printf 'configmap: %s\nca-secret: %s\n' "$CM_FP" "$CA_FP"
if [ -n "$CM_FP" ] && [ "$CM_FP" = "$CA_FP" ]; then
  echo 'PASS: ConfigMap chứa đúng CA đang ký certificate MongoDB'
else
  echo 'STOP: ConfigMap và Secret CA lệch nhau'
fi
unset CM_FP CA_FP
```

PASS khi hai fingerprint giống nhau và in dòng `PASS`. ConfigMap này **không** nằm trong Git vì nội dung sinh ra từ cert-manager; khi CA đổi, tạo lại theo [§12.3](#123-khi-ca-tự-gia-hạn--hiếm-có-thứ-tự-bắt-buộc).

> **DỪNG — GỬI OUTPUT CHECKPOINT 5.3.**

---

## 6. Bước 1 — MongoDB nhận cả TLS lẫn plaintext

Bước này restart `mongodb-0`. Backend vẫn dùng URI plaintext và vẫn chạy được sau restart vì server ở `preferTLS`.

### 6.1. Sửa `k8s/10-mongodb.yaml`

Trên **máy build**, thay **toàn bộ** nội dung `k8s/10-mongodb.yaml` bằng manifest dưới đây. So với Phase 2, chỉ StatefulSet đổi: thêm initContainer `tls-bundle`, `args` của `mongod`, ba volume TLS, và probe dùng TLS. Service và ConfigMap `mongodb-init` giữ nguyên từng dòng.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb
  namespace: three-tier
  labels:
    app.kubernetes.io/name: mongodb
spec:
  clusterIP: None
  selector:
    app.kubernetes.io/name: mongodb
  ports:
    - name: mongodb
      port: 27017
      targetPort: mongodb
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-init
  namespace: three-tier
data:
  01-create-app-user.sh: |
    #!/bin/bash
    set -eu
    mongosh --quiet --host 127.0.0.1 <<'EOF'
    const adminDb = db.getSiblingDB("admin")
    if (!adminDb.auth(
      process.env.MONGO_INITDB_ROOT_USERNAME,
      process.env.MONGO_INITDB_ROOT_PASSWORD
    )) {
      throw new Error("MongoDB root authentication failed")
    }
    const appDb = db.getSiblingDB(process.env.MONGO_APP_DATABASE)
    appDb.createUser({
      user: process.env.MONGO_APP_USERNAME,
      pwd: process.env.MONGO_APP_PASSWORD,
      roles: [{ role: "readWrite", db: process.env.MONGO_APP_DATABASE }]
    })
    appDb.items.createIndex({ id: 1 }, { unique: true })
    EOF
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
  namespace: three-tier
  labels:
    app.kubernetes.io/name: mongodb
spec:
  serviceName: mongodb
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: mongodb
  template:
    metadata:
      labels:
        app.kubernetes.io/name: mongodb
    spec:
      terminationGracePeriodSeconds: 60
      automountServiceAccountToken: false
      initContainers:
        # mongod cần MỘT file PEM chứa cả certificate lẫn private key; cert-manager ghi hai key
        # riêng. Ghép vào emptyDir trên RAM, owner mongodb, mode 0400. Chỉ chạy lúc Pod khởi động.
        - name: tls-bundle
          image: docker.io/library/mongo:8.0.29-noble
          imagePullPolicy: IfNotPresent
          command:
            - bash
            - -ec
            - |
              umask 077
              cat /tls-src/tls.crt /tls-src/tls.key > /tls-out/mongodb.pem
              chown mongodb:mongodb /tls-out/mongodb.pem
              chmod 0400 /tls-out/mongodb.pem
          volumeMounts:
            - name: tls-secret
              mountPath: /tls-src
              readOnly: true
            - name: tls-runtime
              mountPath: /tls-out
          resources:
            requests:
              cpu: 10m
              memory: 16Mi
            limits:
              cpu: 100m
              memory: 64Mi
      containers:
        - name: mongodb
          image: docker.io/library/mongo:8.0.29-noble
          imagePullPolicy: IfNotPresent
          # Đối số bắt đầu bằng "--" nên entrypoint của image tự thêm "mongod" phía trước,
          # vẫn tự thêm --bind_ip_all và --auth như Phase 2.
          args:
            - --tlsMode
            - preferTLS
            - --tlsCertificateKeyFile
            - /etc/mongodb/tls/mongodb.pem
            - --tlsCAFile
            - /etc/mongodb/ca/ca.crt
            - --tlsAllowConnectionsWithoutCertificates
          ports:
            - name: mongodb
              containerPort: 27017
          env:
            - name: MONGO_INITDB_DATABASE
              value: cruddb
            - name: MONGO_APP_DATABASE
              value: cruddb
          envFrom:
            - secretRef:
                name: mongodb-credentials
          volumeMounts:
            - name: data
              mountPath: /data/db
            - name: init
              mountPath: /docker-entrypoint-initdb.d/01-create-app-user.sh
              subPath: 01-create-app-user.sh
              readOnly: true
            - name: tls-runtime
              mountPath: /etc/mongodb/tls
              readOnly: true
            - name: tls-ca
              mountPath: /etc/mongodb/ca
              readOnly: true
          startupProbe:
            exec:
              command: ["mongosh", "--quiet", "--tls", "--tlsCAFile", "/etc/mongodb/ca/ca.crt", "--host", "127.0.0.1", "--eval", "db.adminCommand('ping').ok"]
            periodSeconds: 5
            timeoutSeconds: 5
            failureThreshold: 30
          readinessProbe:
            exec:
              command: ["mongosh", "--quiet", "--tls", "--tlsCAFile", "/etc/mongodb/ca/ca.crt", "--host", "127.0.0.1", "--eval", "db.adminCommand('ping').ok"]
            periodSeconds: 15
            timeoutSeconds: 5
            failureThreshold: 4
          livenessProbe:
            tcpSocket:
              port: mongodb
            periodSeconds: 30
            timeoutSeconds: 3
            failureThreshold: 6
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: "1"
              memory: 1536Mi
      volumes:
        - name: init
          configMap:
            name: mongodb-init
            defaultMode: 0444
        # Toàn bộ Secret mongodb-tls (có private key): chỉ initContainer mount, chỉ root đọc.
        - name: tls-secret
          secret:
            secretName: mongodb-tls
            defaultMode: 0400
        # Chỉ ca.crt (public) cho container mongodb: mongod dùng làm --tlsCAFile, probe và
        # mongosh/mongodump dùng để kiểm tra certificate server. User mongodb phải đọc được.
        - name: tls-ca
          secret:
            secretName: mongodb-tls
            items:
              - key: ca.crt
                path: ca.crt
                mode: 0444
        # File PEM ghép nằm trên RAM của node, mất khi Pod bị xóa.
        - name: tls-runtime
          emptyDir:
            medium: Memory
            sizeLimit: 1Mi
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: local-path
        resources:
          requests:
            storage: 10Gi
```

Kiểm tra và commit trên máy build:

```bash
cd E:/courses/Ansible/three-tier-crud
git diff --stat k8s/10-mongodb.yaml
grep -c 'preferTLS' k8s/10-mongodb.yaml
grep -c 'requireTLS' k8s/10-mongodb.yaml
git add k8s/10-mongodb.yaml
git commit -m "update: update 10-mongodb.yaml - enable preferTLS"
```

PASS khi `git diff --stat` chỉ liệt kê `k8s/10-mongodb.yaml`, `preferTLS` đếm `1`, `requireTLS` đếm `0`.

> **DỪNG — GỬI OUTPUT CHECKPOINT 6.1.**

### 6.2. Apply và kiểm tra `mongod` đã nạp cấu hình TLS

Đồng bộ lên master theo [§4.2](#42-quy-trình-sửa-manifest). Trên `k8s-master`:

```bash
cd ~/three-tier-crud
grep -c 'preferTLS' k8s/10-mongodb.yaml
grep -c 'name: tls-bundle' k8s/10-mongodb.yaml
kubectl apply --dry-run=server -f k8s/10-mongodb.yaml
```

PASS khi hai `grep -c` đều in `1` và dry-run không lỗi. Chỉ apply sau dòng PASS này — từ lệnh apply, `/api/*` gián đoạn cho tới khi `mongodb-0` Ready:

```bash
kubectl apply -f k8s/10-mongodb.yaml
kubectl -n three-tier rollout status statefulset/mongodb --timeout=300s
kubectl -n three-tier get pod mongodb-0 -o wide
kubectl -n three-tier get pod mongodb-0 -o jsonpath='{range .status.initContainerStatuses[*]}init {.name}: exitCode={.state.terminated.exitCode}{"\n"}{end}{range .status.containerStatuses[*]}{.name}: ready={.ready} restarts={.restartCount}{"\n"}{end}'
kubectl -n three-tier exec mongodb-0 -c mongodb -- ls -l /etc/mongodb/tls/ /etc/mongodb/ca/
kubectl -n three-tier logs mongodb-0 -c mongodb --tail=60
```

Đọc tham số `mongod` thực sự đang dùng. Lệnh chạy qua TLS nên đồng thời chứng minh TLS trong Pod hoạt động:

```bash
kubectl -n three-tier exec mongodb-0 -c mongodb -- mongosh --quiet \
  --tls --tlsCAFile /etc/mongodb/ca/ca.crt --host 127.0.0.1 --eval '
  const adminDb = db.getSiblingDB("admin")
  if (!adminDb.auth(process.env.MONGO_INITDB_ROOT_USERNAME, process.env.MONGO_INITDB_ROOT_PASSWORD)) {
    throw new Error("MongoDB root authentication failed")
  }
  printjson(adminDb.runCommand({getCmdLineOpts: 1}).parsed.net)
'
```

PASS khi:

- `mongodb-0` `1/1 Running`, init `tls-bundle` `exitCode=0`, `restarts` không tăng;
- `/etc/mongodb/tls/mongodb.pem` thuộc `mongodb mongodb`, mode `-r--------`; `/etc/mongodb/ca/` có `ca.crt` (dạng symlink của Secret volume là bình thường);
- log không có `ERROR`/`FATAL` và không có lỗi nạp certificate;
- `parsed.net` có `bindIpAll: true` và khối `tls` với mode `preferTLS`, `certificateKeyFile: '/etc/mongodb/tls/mongodb.pem'`, `CAFile: '/etc/mongodb/ca/ca.crt'`, `allowConnectionsWithoutCertificates: true`.

> **DỪNG — GỬI OUTPUT CHECKPOINT 6.2.**

### 6.3. Gate: TLS nhìn từ ngoài Pod, plaintext vẫn chạy

Từ `k8s-master`, bắt tay TLS thẳng vào Pod IP bằng `openssl`, kiểm tra certificate bằng CA đã lấy ở §5.2 và kiểm tra tên `mongodb` — đúng tên backend sẽ dùng:

```bash
MONGO_POD_IP=$(kubectl -n three-tier get pod mongodb-0 -o jsonpath='{.status.podIP}')
openssl s_client -connect "${MONGO_POD_IP}:27017" -servername mongodb \
  -verify_hostname mongodb -CAfile ~/phase3/ca.crt -verify_return_error </dev/null 2>/dev/null \
  | grep -E '^(subject|issuer)=|Verify return code'
unset MONGO_POD_IP
```

Server ở `preferTLS` nên backend plaintext phải vẫn khỏe:

```bash
source ~/phase2-app.env
kubectl -n three-tier wait --for=condition=Ready pod -l app.kubernetes.io/name=backend --timeout=180s
curl -sS -o /dev/null -w 'api-ready-via-traefik=%{http_code}\n' \
  -H "Host: $APP_HOST" "http://$ING_IP/api/health/ready"
```

PASS khi `openssl` in `issuer=CN = three-tier-mongodb-ca` (hoặc `issuer=CN=three-tier-mongodb-ca`, tùy định dạng) và `Verify return code: 0 (ok)`; hai Pod backend Ready; `api-ready-via-traefik=200`.

> **DỪNG — GỬI OUTPUT CHECKPOINT 6.3.**

---

## 7. Bước 2 — Chuyển backend sang TLS

Bước này không gián đoạn: rolling update giữ Pod cũ (plaintext) phục vụ cho tới khi Pod mới (TLS) Ready.

### 7.1. Tạo Secret URI có TLS

Không nhập lại password. Lệnh đọc URI hiện tại từ Secret cũ, kiểm tra đúng dạng Phase 2 §9.1, rồi ghép hậu tố TLS vào Secret **mới**. Secret cũ giữ nguyên để rollback:

```bash
OLD_URI=$(kubectl -n three-tier get secret backend-mongodb-uri -o jsonpath='{.data.MONGODB_URI}' | base64 -d)
case "$OLD_URI" in
  'mongodb://crudapp:'*'@mongodb:27017/cruddb?authSource=cruddb')
    printf 'MONGODB_URI=%s&tls=true&tlsCAFile=/etc/mongodb-ca/ca.crt\n' "$OLD_URI" | \
      kubectl -n three-tier create secret generic backend-mongodb-uri-tls \
      --from-env-file=/dev/stdin \
      --dry-run=client -o yaml | kubectl apply -f -
    echo 'PASS: đã tạo backend-mongodb-uri-tls từ URI Phase 2' ;;
  *)
    echo 'STOP: URI hiện tại không đúng dạng Phase 2 §9.1; không tạo Secret mới' ;;
esac
unset OLD_URI
kubectl -n three-tier get secret backend-mongodb-uri-tls -o jsonpath='{.data.MONGODB_URI}' \
  | base64 -d | sed -E 's#^(mongodb://[^:]+:)[^@]*@#\1***@#'; echo
```

Password trong URI Phase 2 đã được percent-encode nên không chứa `@` thô; `sed` thay đúng phần password bằng `***` trước khi in. PASS khi có dòng `PASS: đã tạo...` và dòng cuối là đúng:

```text
mongodb://crudapp:***@mongodb:27017/cruddb?authSource=cruddb&tls=true&tlsCAFile=/etc/mongodb-ca/ca.crt
```

> **DỪNG — GỬI OUTPUT CHECKPOINT 7.1; CHỈ GỬI DÒNG ĐÃ CHE PASSWORD.**

### 7.2. Sửa `k8s/20-backend.yaml`

File `20-backend.yaml` trong repository có comment riêng và image digest thật, nên **không thay cả file**; sửa đúng ba chỗ trên **máy build**:

**(a)** Trong `env`, đổi Secret của `MONGODB_URI`:

```yaml
          env:
            - name: MONGODB_URI
              valueFrom:
                secretKeyRef:
                  name: backend-mongodb-uri-tls
                  key: MONGODB_URI
```

**(b)** Trong `volumeMounts` của container `backend`, thêm mount thứ hai ngay sau `tmp`:

```yaml
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: mongodb-ca
              mountPath: /etc/mongodb-ca
              readOnly: true
```

**(c)** Trong `volumes` của Pod, thêm volume thứ hai ngay sau `tmp`:

```yaml
      volumes:
        - name: tmp
          emptyDir: {}
        - name: mongodb-ca
          configMap:
            name: mongodb-ca-bundle
            items:
              - key: ca.crt
                path: ca.crt
            defaultMode: 0444
```

Cập nhật luôn phần comment đầu file: dòng `Secret backend-mongodb-uri (key MONGODB_URI)` thành `Secret backend-mongodb-uri-tls (key MONGODB_URI)`, và thêm một dòng `ConfigMap mongodb-ca-bundle (key ca.crt)` vào danh sách object phải có sẵn trước khi apply.

Backend chạy bằng UID `10001`, root filesystem read-only; ConfigMap mount mode `0444` để UID đó đọc được, và `/etc/mongodb-ca` là mount riêng nên không cần ghi vào root filesystem.

Kiểm tra và commit trên máy build:

```bash
cd E:/courses/Ansible/three-tier-crud
grep -n 'name: backend-mongodb-uri' k8s/20-backend.yaml
grep -n -A2 'name: mongodb-ca$' k8s/20-backend.yaml
grep -c 'image: .*@sha256:' k8s/20-backend.yaml
git diff --stat
git add k8s/20-backend.yaml
git commit -m "update: update 20-backend.yaml - connect MongoDB over TLS"
```

PASS khi dòng `secretKeyRef` in ra là `name: backend-mongodb-uri-tls`; `name: mongodb-ca` xuất hiện **hai** lần — một lần kèm `mountPath: /etc/mongodb-ca`, một lần kèm `configMap:`; image vẫn có `@sha256:` (đếm `1`); `git diff --stat` chỉ có `k8s/20-backend.yaml`.

> **DỪNG — GỬI OUTPUT CHECKPOINT 7.2.**

### 7.3. Apply và chứng minh backend đã dùng TLS

Đồng bộ lên master theo [§4.2](#42-quy-trình-sửa-manifest). Trên `k8s-master`:

```bash
cd ~/three-tier-crud
grep -n 'name: backend-mongodb-uri' k8s/20-backend.yaml
kubectl apply --dry-run=server -f k8s/20-backend.yaml
kubectl apply -f k8s/20-backend.yaml
kubectl -n three-tier rollout status deploy/backend --timeout=300s
kubectl -n three-tier get deploy backend -o jsonpath='uri-secret={.spec.template.spec.containers[0].env[?(@.name=="MONGODB_URI")].valueFrom.secretKeyRef.name}{"\n"}'
kubectl -n three-tier get pod -l app.kubernetes.io/name=backend -o wide
kubectl -n three-tier exec deploy/backend -- ls -lL /etc/mongodb-ca/
kubectl -n three-tier logs -l app.kubernetes.io/name=backend --tail=80 --prefix \
  | grep -iE 'certificate|ssl|tls|ServerSelection|readiness failed' || echo 'no TLS/connection errors in recent logs'
source ~/phase2-app.env
curl -sS -o /dev/null -w 'api-ready-via-traefik=%{http_code}\n' \
  -H "Host: $APP_HOST" "http://$ING_IP/api/health/ready"
curl -sS -o /dev/null -w 'api-list-via-traefik=%{http_code}\n' \
  -H "Host: $APP_HOST" "http://$ING_IP/api/items"
```

PASS khi:

- rollout `successfully rolled out`; `uri-secret=backend-mongodb-uri-tls`; hai Pod backend mới `1/1 Running`;
- `/etc/mongodb-ca/ca.crt` tồn tại, mode `-r--r--r--`;
- không có dòng log lỗi TLS/kết nối (in `no TLS/connection errors in recent logs`);
- `api-ready-via-traefik=200`, `api-list-via-traefik=200`.

Vì sao đây là bằng chứng: Pod backend mới chỉ được đưa vào thay Pod cũ khi readiness `/api/health/ready` — lệnh `ping` tới MongoDB — trả `200`. URI của Pod mới có `tls=true`, và PyMongo không lùi về plaintext. Pod mới Ready nghĩa là TLS handshake, kiểm tra CA và kiểm tra hostname `mongodb` đều đã qua. §8 sẽ chứng minh thêm từ phía server.

> **DỪNG — GỬI OUTPUT CHECKPOINT 7.3.**

---

## 8. Bước 3 — MongoDB chỉ nhận TLS

Bước này restart `mongodb-0` lần hai.

### 8.1. Lưu manifest Bước 1 để rollback

Trên `k8s-master`, **trước** khi đồng bộ manifest mới. Lúc này `k8s/10-mongodb.yaml` trên master vẫn là bản `preferTLS` của §6:

```bash
cd ~/three-tier-crud
grep -c 'preferTLS' k8s/10-mongodb.yaml
cp k8s/10-mongodb.yaml ~/phase3-rollback/10-mongodb.step1-prefertls.yaml
ls -l ~/phase3-rollback/
```

PASS khi `grep -c` in `1` và `~/phase3-rollback/` có ba file.

> **DỪNG — GỬI OUTPUT CHECKPOINT 8.1.**

### 8.2. Đổi sang `requireTLS` và apply

Trên **máy build** — đổi đúng một giá trị:

```bash
cd E:/courses/Ansible/three-tier-crud
sed -i 's/preferTLS/requireTLS/' k8s/10-mongodb.yaml
git diff -U0 k8s/10-mongodb.yaml
grep -c 'preferTLS' k8s/10-mongodb.yaml
grep -c 'requireTLS' k8s/10-mongodb.yaml
git add k8s/10-mongodb.yaml
git commit -m "update: update 10-mongodb.yaml - require TLS"
```

PASS khi `git diff -U0` chỉ có một cặp dòng `-            - preferTLS` / `+            - requireTLS`, `preferTLS` đếm `0`, `requireTLS` đếm `1`.

Đồng bộ lên master theo [§4.2](#42-quy-trình-sửa-manifest). Trên `k8s-master`:

```bash
cd ~/three-tier-crud
grep -c 'requireTLS' k8s/10-mongodb.yaml
kubectl apply --dry-run=server -f k8s/10-mongodb.yaml
kubectl apply -f k8s/10-mongodb.yaml
kubectl -n three-tier rollout status statefulset/mongodb --timeout=300s
kubectl -n three-tier get pod mongodb-0 -o wide
kubectl -n three-tier exec mongodb-0 -c mongodb -- mongosh --quiet \
  --tls --tlsCAFile /etc/mongodb/ca/ca.crt --host 127.0.0.1 --eval '
  const adminDb = db.getSiblingDB("admin")
  if (!adminDb.auth(process.env.MONGO_INITDB_ROOT_USERNAME, process.env.MONGO_INITDB_ROOT_PASSWORD)) {
    throw new Error("MongoDB root authentication failed")
  }
  print("tls.mode=" + adminDb.runCommand({getCmdLineOpts: 1}).parsed.net.tls.mode)
'
kubectl -n three-tier wait --for=condition=Ready pod -l app.kubernetes.io/name=backend --timeout=180s
```

PASS khi `grep -c` in `1`, `mongodb-0` `1/1 Running`, in `tls.mode=requireTLS`, và hai Pod backend Ready trở lại. Backend không cần rollout: PyMongo tự kết nối lại sau khi `mongod` restart, và lần này chỉ có TLS.

> **DỪNG — GỬI OUTPUT CHECKPOINT 8.2.**

### 8.3. Gate âm — chứng minh server chỉ nhận TLS và client thật sự kiểm tra

Mỗi phép thử dùng connection string có `serverSelectionTimeoutMS=5000` để thất bại nhanh. Phép thử dương chạy trước để chắc `kubectl exec` hoạt động — tránh trường hợp `exec` hỏng bị đọc nhầm thành "bị từ chối":

```bash
mongo_ping() {
  kubectl -n three-tier exec mongodb-0 -c mongodb -- \
    mongosh --quiet "$1" --eval 'db.adminCommand("ping").ok' >/dev/null 2>&1
}
CA=/etc/mongodb/ca/ca.crt
MONGO_POD_IP=$(kubectl -n three-tier get pod mongodb-0 -o jsonpath='{.status.podIP}')

mongo_ping "mongodb://127.0.0.1:27017/?tls=true&tlsCAFile=${CA}&serverSelectionTimeoutMS=5000" \
  && echo 'PASS dương: TLS + CA nội bộ + tên 127.0.0.1 kết nối được' \
  || echo 'FAIL dương: TLS hợp lệ cũng không kết nối được — dừng, không đọc các dòng âm'

mongo_ping "mongodb://127.0.0.1:27017/?serverSelectionTimeoutMS=5000" \
  && echo 'FAIL âm 1: plaintext vẫn kết nối được' \
  || echo 'PASS âm 1: plaintext bị từ chối'

if kubectl -n three-tier exec mongodb-0 -c mongodb -- test -s /etc/ssl/certs/ca-certificates.crt; then
  mongo_ping "mongodb://127.0.0.1:27017/?tls=true&tlsCAFile=/etc/ssl/certs/ca-certificates.crt&serverSelectionTimeoutMS=5000" \
    && echo 'FAIL âm 2: client tin CA công cộng mà vẫn chấp nhận certificate nội bộ' \
    || echo 'PASS âm 2: client không có CA nội bộ thì không tin server'
else
  echo 'SKIP âm 2: image không có CA bundle hệ thống'
fi

mongo_ping "mongodb://${MONGO_POD_IP}:27017/?tls=true&tlsCAFile=${CA}&serverSelectionTimeoutMS=5000" \
  && echo 'FAIL âm 3: kết nối bằng tên không có trong certificate vẫn được chấp nhận' \
  || echo 'PASS âm 3: tên không có trong certificate bị từ chối (kiểm tra hostname đang bật)'

unset -f mongo_ping; unset CA MONGO_POD_IP
```

Kiểm tra lại từ phía người dùng — backend vẫn phục vụ khi server từ chối mọi kết nối plaintext:

```bash
source ~/phase2-app.env
curl -sS -o /dev/null -w 'api-ready-via-traefik=%{http_code}\n' \
  -H "Host: $APP_HOST" "http://$ING_IP/api/health/ready"
```

PASS khi có `PASS dương`, `PASS âm 1`, `PASS âm 3`, `PASS âm 2` hoặc `SKIP âm 2`, và `api-ready-via-traefik=200`.

Ý nghĩa ba phép âm:

| Phép thử | Chứng minh |
| --- | --- |
| Âm 1 — plaintext | server thực thi `requireTLS`; từ giờ không client nào đọc/ghi được dữ liệu mà không mã hóa |
| Âm 2 — CA công cộng | client thật sự kiểm tra chữ ký certificate; một server giả không có certificate do CA nội bộ ký sẽ bị từ chối |
| Âm 3 — Pod IP | client thật sự kiểm tra hostname; đây là lý do SAN ở §1.5 phải có đúng tên client dùng |

Kết hợp với `api-ready=200`: server không nhận plaintext mà backend vẫn chạy, nên backend chắc chắn đang dùng TLS.

> **DỪNG — GỬI OUTPUT CHECKPOINT 8.3.**

---

## 9. Kiểm thử end-to-end sau khi bật TLS

### 9.1. CRUD qua Traefik

Các request đi qua Traefik và Nginx của frontend như Phase 2 §12, giờ có thêm chặng TLS backend → MongoDB:

```bash
source ~/phase2-app.env
BASE="http://$ING_IP/api/items"
curl -sS -o /dev/null -w 'create=%{http_code}\n' -X POST -H "Host: $APP_HOST" \
  -H 'Content-Type: application/json' "$BASE" \
  -d '{"id":"phase3-smoke-001","name":"TLS smoke","description":"created after requireTLS"}'
curl -sS -o /dev/null -w 'read=%{http_code}\n' -H "Host: $APP_HOST" "$BASE/phase3-smoke-001"
curl -sS -o /dev/null -w 'update=%{http_code}\n' -X PUT -H "Host: $APP_HOST" \
  -H 'Content-Type: application/json' "$BASE/phase3-smoke-001" \
  -d '{"name":"TLS smoke v2","description":"updated over TLS"}'
curl -sS -H "Host: $APP_HOST" "$BASE/phase3-smoke-001"; echo
curl -sS -o /dev/null -w 'delete=%{http_code}\n' -X DELETE -H "Host: $APP_HOST" "$BASE/phase3-smoke-001"
curl -sS -o /dev/null -w 'read-after-delete=%{http_code}\n' -H "Host: $APP_HOST" "$BASE/phase3-smoke-001"
unset BASE
```

PASS khi lần lượt `create=201`, `read=200`, `update=200`, GET in ra item có `"name":"TLS smoke v2"`, `delete=204`, `read-after-delete=404`.

> **DỪNG — GỬI OUTPUT CHECKPOINT 9.1.**

### 9.2. (Tùy chọn) Bắt gói tin sau khi bật TLS — ảnh chụp "sau"

Chỉ làm nếu đã làm §3.4. Lặp lại đúng quy trình §3.4 với marker mới. `mongodb-0` có thể đã đổi node — chạy lại lệnh tìm node trước.

```bash
# k8s-master
kubectl -n three-tier get pod mongodb-0 -o jsonpath='{.spec.nodeName}{"\n"}'
```

```bash
# Terminal A — worker vừa in ra
sudo -v
sudo timeout 40 tcpdump -l -i cni0 -nn -A -s 0 'tcp port 27017' \
  > /tmp/phase3-wire-after.txt 2>/dev/null &
```

```bash
# Terminal B — k8s-master, trong vòng 40 giây
source ~/phase2-app.env
curl -sS -o /dev/null -w 'create=%{http_code}\n' -X POST -H "Host: $APP_HOST" \
  -H 'Content-Type: application/json' "http://$ING_IP/api/items" \
  -d '{"id":"phase3-wire-after","name":"phase3-wire-after","description":"tls capture marker"}'
```

```bash
# Terminal A — sau khi timeout kết thúc
wait
printf 'packets-27017=%s\nmarker-hits=%s\n' \
  "$(grep -c '\.27017' /tmp/phase3-wire-after.txt)" \
  "$(grep -c 'phase3-wire-after' /tmp/phase3-wire-after.txt)"
rm -f /tmp/phase3-wire-after.txt
```

Dọn hai item marker (trên `k8s-master`; `404` nghĩa là item không tồn tại, cũng chấp nhận):

```bash
source ~/phase2-app.env
for id in phase3-wire-before phase3-wire-after; do
  curl -sS -o /dev/null -w "delete-$id=%{http_code}\n" -X DELETE \
    -H "Host: $APP_HOST" "http://$ING_IP/api/items/$id"
done
```

PASS khi `create=201`, `packets-27017` > 0 (capture thật sự thấy traffic) và `marker-hits=0` (nội dung không còn đọc được); hai lệnh delete trả `204` hoặc `404`. So với §3.4: cùng API, cùng node, cùng bộ lọc — chỉ khác chặng backend → MongoDB đã được mã hóa.

> **DỪNG — GỬI OUTPUT CHECKPOINT 9.2 (hoặc ghi "bỏ qua §9.2").**

### 9.3. Dữ liệu còn nguyên và endpoint public

```bash
kubectl -n three-tier exec mongodb-0 -c mongodb -- mongosh --quiet \
  --tls --tlsCAFile /etc/mongodb/ca/ca.crt --host 127.0.0.1 --eval '
  const appDb = db.getSiblingDB(process.env.MONGO_APP_DATABASE)
  if (!appDb.auth(process.env.MONGO_APP_USERNAME, process.env.MONGO_APP_PASSWORD)) {
    throw new Error("MongoDB app-user authentication failed")
  }
  print(appDb.items.countDocuments({}))
' | tee ~/phase3/items-count-after.txt
printf 'before=%s after=%s\n' "$(cat ~/phase3/items-count-before.txt)" "$(cat ~/phase3/items-count-after.txt)"
source ~/phase2-app.env
curl -sS -o /dev/null -w 'public-frontend=%{http_code}\n' "https://$APP_HOST/"
curl -sS -o /dev/null -w 'public-api-ready=%{http_code}\n' "https://$APP_HOST/api/health/ready"
```

PASS khi `before` và `after` bằng nhau (item test của §9.1 đã xóa; item marker của §3.4/§9.2 đã dọn ở §9.2), `public-frontend=200` và `public-api-ready=200`.

Nếu đã làm §3.4 nhưng bỏ qua §9.2, chạy riêng khối "Dọn hai item marker" ở §9.2 trước khi đếm.

> **DỪNG — GỬI OUTPUT CHECKPOINT 9.3.**

---

## 10. Backup và restore qua TLS

Từ Phase 3, các khối ở [Phase 2 §14.2](runbook-k8s-vmware-phase2.md#142-backup-logic-bằng-mongodump) và [§14.3](runbook-k8s-vmware-phase2.md#143-rehearse-restore-vào-database-tạm) **không còn chạy được** vì chúng kết nối plaintext. Hai khối dưới đây thay thế chúng; chỉ khác ở tham số TLS và `-c mongodb`. Database Tools dùng tên tham số `--ssl`/`--sslCAFile` (không phải `--tls...` như `mongosh`). `--host 127.0.0.1` vẫn đúng vì `127.0.0.1` có trong certificate.

### 10.1. Backup bằng `mongodump` qua TLS

```bash
umask 077
BACKUP_DIR="$HOME/backups/three-tier"
install -d -m 0700 "$BACKUP_DIR"
chmod 700 "$BACKUP_DIR"
BACKUP_TS=$(date -u +%Y%m%dT%H%M%SZ)
BACKUP_FILE="${BACKUP_DIR}/cruddb-${BACKUP_TS}.archive"
kubectl -n three-tier exec mongodb-0 -c mongodb -- env BACKUP_TS="$BACKUP_TS" bash -ec '
  umask 077
  CONFIG_FILE=$(mktemp /tmp/mongodb-tools.XXXXXX.yaml)
  trap "rm -f \"$CONFIG_FILE\"" EXIT
  printf "%s\n" "$MONGO_TOOLS_CONFIG" > "$CONFIG_FILE"
  mongodump --config="$CONFIG_FILE" \
    --ssl --sslCAFile=/etc/mongodb/ca/ca.crt \
    --host 127.0.0.1 \
    --username "$MONGO_INITDB_ROOT_USERNAME" \
    --authenticationDatabase admin \
    --db cruddb \
    --archive="/tmp/cruddb-${BACKUP_TS}.archive" \
    --gzip
'
kubectl -n three-tier cp -c mongodb \
  "mongodb-0:/tmp/cruddb-${BACKUP_TS}.archive" "$BACKUP_FILE"
chmod 600 "$BACKUP_FILE"
test -s "$BACKUP_FILE" && ls -lh "$BACKUP_FILE"
stat -c '%a %n' "$BACKUP_DIR" "$BACKUP_FILE"
kubectl -n three-tier exec mongodb-0 -c mongodb -- rm -f "/tmp/cruddb-${BACKUP_TS}.archive"
printf 'backup file: %s\n' "$BACKUP_FILE"
unset BACKUP_TS BACKUP_FILE BACKUP_DIR
```

PASS khi `mongodump` không lỗi, archive > 0 byte, directory mode `700`, file mode `600`.

> **DỪNG — GỬI OUTPUT CHECKPOINT 10.1; KHÔNG GỬI FILE BACKUP.**

### 10.2. Rehearse restore qua TLS vào database tạm

```bash
BACKUP_DIR="$HOME/backups/three-tier"
BACKUP_FILE=$(ls -1t "$BACKUP_DIR"/cruddb-*.archive 2>/dev/null | head -1)
test -n "$BACKUP_FILE" && test -s "$BACKUP_FILE" && echo "restore source: $BACKUP_FILE"
kubectl -n three-tier cp -c mongodb "$BACKUP_FILE" mongodb-0:/tmp/restore-test.archive
kubectl -n three-tier exec mongodb-0 -c mongodb -- bash -ec '
  umask 077
  CONFIG_FILE=$(mktemp /tmp/mongodb-tools.XXXXXX.yaml)
  trap "rm -f \"$CONFIG_FILE\"" EXIT
  printf "%s\n" "$MONGO_TOOLS_CONFIG" > "$CONFIG_FILE"
  mongorestore --config="$CONFIG_FILE" \
    --ssl --sslCAFile=/etc/mongodb/ca/ca.crt \
    --host 127.0.0.1 \
    --username "$MONGO_INITDB_ROOT_USERNAME" \
    --authenticationDatabase admin \
    --archive=/tmp/restore-test.archive \
    --gzip \
    --nsFrom="cruddb.*" \
    --nsTo="cruddb_restore_test.*" \
    --drop
'
kubectl -n three-tier exec mongodb-0 -c mongodb -- mongosh --quiet \
  --tls --tlsCAFile /etc/mongodb/ca/ca.crt --host 127.0.0.1 --eval '
  const adminDb = db.getSiblingDB("admin")
  if (!adminDb.auth(process.env.MONGO_INITDB_ROOT_USERNAME, process.env.MONGO_INITDB_ROOT_PASSWORD)) {
    throw new Error("MongoDB root authentication failed")
  }
  print("restored=" + db.getSiblingDB("cruddb_restore_test").items.countDocuments({}))
  print("live=" + db.getSiblingDB("cruddb").items.countDocuments({}))
  printjson(db.getSiblingDB("cruddb_restore_test").dropDatabase())
'
kubectl -n three-tier exec mongodb-0 -c mongodb -- rm -f /tmp/restore-test.archive
unset BACKUP_FILE BACKUP_DIR
```

PASS khi `mongorestore` không lỗi, `restored` bằng `live` (không có ghi mới giữa §10.1 và §10.2), và `dropDatabase` trả `ok: 1`.

> **DỪNG — GỬI OUTPUT CHECKPOINT 10.2; KHÔNG GỬI FILE BACKUP HOẶC SECRET.**

---

## 11. Rollback

Rollback dùng các file đã lưu ở §3.3 và §8.1. **Thứ tự quan trọng**: server phải nhận plaintext **trước khi** backend quay về plaintext, nếu không backend mất kết nối.

| Đang ở | Triệu chứng cần rollback | Làm |
| --- | --- | --- |
| §6 thất bại | `mongodb-0` không Ready với manifest TLS | bước 3 bên dưới |
| §7 thất bại | Pod backend mới không Ready (Pod cũ vẫn phục vụ do `maxUnavailable: 0`) | bước 2 |
| §8 thất bại | `mongodb-0` không Ready hoặc backend mất kết nối sau `requireTLS` | bước 1 |
| Muốn bỏ hẳn Phase 3 | — | bước 1 → 2 → 3, rồi bước 4 |

Trên `k8s-master`:

```bash
# Bước 1 — MongoDB về preferTLS (chỉ khi đang ở requireTLS)
kubectl apply -f ~/phase3-rollback/10-mongodb.step1-prefertls.yaml
kubectl -n three-tier rollout status statefulset/mongodb --timeout=300s

# Bước 2 — backend về URI plaintext và bỏ mount CA
kubectl apply -f ~/phase3-rollback/20-backend.phase2.yaml
kubectl -n three-tier rollout status deploy/backend --timeout=300s

# Bước 3 — MongoDB về manifest Phase 2 (không TLS)
kubectl apply -f ~/phase3-rollback/10-mongodb.phase2.yaml
kubectl -n three-tier rollout status statefulset/mongodb --timeout=300s

# Kiểm tra sau mỗi bước
source ~/phase2-app.env
curl -sS -o /dev/null -w 'api-ready-via-traefik=%{http_code}\n' \
  -H "Host: $APP_HOST" "http://$ING_IP/api/health/ready"
```

Bước 4 — chỉ khi bỏ hẳn Phase 3, sau khi bước 3 PASS:

```bash
kubectl -n three-tier delete certificate.cert-manager.io/mongodb-server certificate.cert-manager.io/mongodb-ca \
  issuer.cert-manager.io/mongodb-ca-issuer issuer.cert-manager.io/mongodb-selfsigned
kubectl -n three-tier delete secret/mongodb-tls secret/mongodb-ca secret/backend-mongodb-uri-tls \
  configmap/mongodb-ca-bundle
```

Sau rollback, đưa Git về khớp với cụm: trên máy build `git revert` các commit Phase 3 trong repository ứng dụng, rồi đồng bộ lại theo §4.2. PASS rollback khi `api-ready-via-traefik=200` và §3.1 cho lại đúng kết quả ban đầu.

---

## 12. Vận hành: gia hạn certificate

### 12.1. Lịch gia hạn

| Certificate | Thời hạn | cert-manager gia hạn khi | Việc của người vận hành |
| --- | --- | --- | --- |
| `mongodb-server` | 1 năm | còn 90 ngày (`renewBefore: 2160h`) | restart `mongodb-0` trong 90 ngày đó — §12.2 |
| `mongodb-ca` | 10 năm | khoảng 2/3 thời hạn (mặc định) | chuyển CA cho backend theo đúng thứ tự — §12.3 |

Xem thời điểm cụ thể:

```bash
kubectl -n three-tier get certificate.cert-manager.io/mongodb-server certificate.cert-manager.io/mongodb-ca \
  -o custom-columns='NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,NOT_AFTER:.status.notAfter,RENEWAL:.status.renewalTime'
```

Ghi `RENEWAL` của cả hai vào lịch. `rotationPolicy: Always` (mặc định từ cert-manager v1.18) nghĩa là mỗi lần gia hạn server certificate có private key mới — client không bị ảnh hưởng vì client tin CA, không tin một certificate cụ thể.

### 12.2. Sau khi server certificate được gia hạn

`mongod` không tự đọc lại file: certificate mới trong Secret chỉ có hiệu lực khi initContainer chạy lại, tức khi `mongodb-0` restart. Kiểm tra định kỳ, và luôn kiểm tra sau mốc `RENEWAL`:

```bash
SECRET_FP=$(kubectl -n three-tier get secret mongodb-tls -o jsonpath='{.data.tls\.crt}' | base64 -d \
  | openssl x509 -noout -fingerprint -sha256)
SECRET_CA_FP=$(kubectl -n three-tier get secret mongodb-tls -o jsonpath='{.data.ca\.crt}' | base64 -d \
  | openssl x509 -noout -fingerprint -sha256)
BUNDLE_CA_FP=$(kubectl -n three-tier get configmap mongodb-ca-bundle -o jsonpath='{.data.ca\.crt}' \
  | openssl x509 -noout -fingerprint -sha256)
MONGO_POD_IP=$(kubectl -n three-tier get pod mongodb-0 -o jsonpath='{.status.podIP}')
SERVED_FP=$(openssl s_client -connect "${MONGO_POD_IP}:27017" -servername mongodb </dev/null 2>/dev/null \
  | openssl x509 -noout -fingerprint -sha256)
printf 'secret: %s\nserved: %s\nca-in-secret: %s\nca-in-bundle: %s\n' \
  "$SECRET_FP" "$SERVED_FP" "$SECRET_CA_FP" "$BUNDLE_CA_FP"
if [ "$SECRET_CA_FP" != "$BUNDLE_CA_FP" ]; then
  echo 'STOP: CA đã đổi — KHÔNG restart mongodb-0; làm §12.3'
elif [ -n "$SERVED_FP" ] && [ "$SECRET_FP" = "$SERVED_FP" ]; then
  echo 'PASS: mongod đang dùng certificate mới nhất'
else
  echo 'ACTION: certificate đã gia hạn nhưng mongod còn dùng bản cũ — restart mongodb-0'
fi
unset SECRET_FP SECRET_CA_FP BUNDLE_CA_FP MONGO_POD_IP SERVED_FP
```

Nếu nhận `ACTION`, restart rồi chạy lại khối kiểm tra:

```bash
kubectl -n three-tier rollout restart statefulset/mongodb
kubectl -n three-tier rollout status statefulset/mongodb --timeout=300s
kubectl -n three-tier wait --for=condition=Ready pod -l app.kubernetes.io/name=backend --timeout=180s
```

PASS khi khối kiểm tra in `PASS: mongod đang dùng certificate mới nhất`. Restart gây gián đoạn `/api/*` như §6; backend không cần đổi gì.

### 12.3. Khi CA tự gia hạn — hiếm, có thứ tự bắt buộc

Theo tài liệu cert-manager, đổi Secret của CA **không** tự cấp lại certificate lá. Lần gia hạn server certificate kế tiếp sẽ được ký bởi CA mới, trong khi backend vẫn chỉ tin CA cũ trong ConfigMap. Nếu restart `mongodb-0` lúc đó, backend mất kết nối. Khối §12.2 bắt trường hợp này bằng dòng `STOP: CA đã đổi`.

Thứ tự đúng — luôn để backend tin **cả hai** CA trước khi server đổi certificate:

1. Lấy CA mới từ `ca.crt` của Secret `mongodb-tls` (hoặc `tls.crt` của Secret `mongodb-ca`) và CA cũ từ ConfigMap, ghép thành một file chứa **hai** certificate, tạo lại ConfigMap `mongodb-ca-bundle` từ file đó.
2. `kubectl -n three-tier rollout restart deploy/backend` — Pod mới đọc bundle hai CA.
3. Nếu Secret `mongodb-tls` vẫn là certificate do CA cũ ký, xóa Secret này để cert-manager cấp lại bằng CA mới; chờ `certificate.cert-manager.io/mongodb-server` Ready.
4. Restart `mongodb-0` theo §12.2; chạy lại §8.3.
5. Tạo lại ConfigMap chỉ với CA mới, restart backend lần nữa, chạy lại §8.3.

Sau bước 4, mọi client mới phải dùng bundle có CA mới; `/etc/mongodb/ca/ca.crt` trong Pod MongoDB tự cập nhật vì nó là Secret volume không dùng `subPath`.

---

## 13. Troubleshooting

| Triệu chứng | Kiểm tra | Nguyên nhân thường gặp |
| --- | --- | --- |
| Certificate `mongodb-server` không `Ready` | `kubectl -n three-tier describe certificate.cert-manager.io mongodb-server`; `kubectl -n three-tier get certificaterequests.cert-manager.io`; log `deploy/cert-manager` ở namespace `cert-manager` | Issuer `mongodb-ca-issuer` chưa Ready vì Secret `mongodb-ca` chưa có; cert-manager webhook không Ready |
| `mongodb-0` kẹt `Init:0/1` hoặc `ContainerCreating` | `kubectl -n three-tier describe pod mongodb-0` (Events) | Secret `mongodb-tls` chưa tồn tại → `FailedMount`; tạo §5 trước §6 |
| `mongodb-0` `Init:Error` | `kubectl -n three-tier logs mongodb-0 -c tls-bundle` | thiếu key `tls.crt`/`tls.key` trong Secret; lệnh ghép lỗi |
| `mongodb-0` `CrashLoopBackOff` sau §6 | `kubectl -n three-tier logs mongodb-0 -c mongodb --previous` | đường dẫn `--tlsCertificateKeyFile`/`--tlsCAFile` sai; file PEM không đọc được bởi user `mongodb`; thiếu `--tlsCAFile` |
| `mongodb-0` `Running` nhưng `0/1` | `kubectl -n three-tier describe pod mongodb-0` (lý do probe fail) | probe TLS lỗi: SAN thiếu `127.0.0.1`, probe trỏ sai `--tlsCAFile` |
| Pod backend mới không Ready ở §7 | log backend (`--prefix`), `kubectl exec deploy/backend -- ls -lL /etc/mongodb-ca/` | ConfigMap CA sai hoặc chưa mount; Secret mới sai hậu tố; host trong URI không có trong SAN |
| Backend mất kết nối ngay sau §8 | `kubectl -n three-tier get deploy backend -o jsonpath='{..secretKeyRef.name}'` | backend vẫn dùng `backend-mongodb-uri` (plaintext) — §7 chưa áp dụng |
| `mongodump`/`mongorestore` lỗi kết nối hoặc certificate | khối §10 | thiếu `--ssl --sslCAFile`; dùng `--host` không có trong SAN |
| Khối Phase 2 §14.2/§14.3 lỗi | — | khối cũ là plaintext; dùng §10 |
| Sau gia hạn, client mới lỗi kiểm tra certificate | §12.2 | CA đã đổi nhưng ConfigMap chưa cập nhật — làm §12.3 |

Lệnh chẩn đoán nhanh, không in Secret:

```bash
kubectl -n three-tier get pod,certificates.cert-manager.io,issuers.cert-manager.io -o wide
kubectl -n three-tier get events --sort-by=.lastTimestamp | tail -40
kubectl -n three-tier logs mongodb-0 -c tls-bundle
kubectl -n three-tier logs mongodb-0 -c mongodb --tail=100
kubectl -n three-tier logs -l app.kubernetes.io/name=backend --tail=100 --prefix
```

---

## 14. Checklist hoàn tất

- [ ] §3.1–§3.3: Phase 2 khỏe; chưa có object Phase 3; backup MongoDB và etcd PASS; đã đếm document; manifest Phase 2 đã lưu ở `~/phase3-rollback/`.
- [ ] §3.4 (tùy chọn): marker thấy được trong gói tin plaintext.
- [ ] §5: hai Issuer và hai Certificate `Ready`; SAN đúng §1.5; `openssl verify` OK; ConfigMap `mongodb-ca-bundle` khớp fingerprint CA.
- [ ] §6: `mongodb-0` chạy `preferTLS` với file PEM `0400` thuộc `mongodb`; `openssl s_client` verify `0 (ok)` với tên `mongodb`; backend plaintext vẫn `200`.
- [ ] §7: Secret `backend-mongodb-uri-tls` có hậu tố TLS; backend dùng Secret mới, mount CA; rollout không gián đoạn; API `200`.
- [ ] §8: `tls.mode=requireTLS`; dương PASS; âm 1 và âm 3 PASS; âm 2 PASS hoặc SKIP; API `200`.
- [ ] §9: CRUD qua Traefik đúng mã trạng thái; (tùy chọn) marker không còn trong gói tin; số document trước = sau; public `200`.
- [ ] §10: `mongodump` và `mongorestore` qua TLS PASS.
- [ ] §12: đã ghi `RENEWAL` của `mongodb-server` và `mongodb-ca` vào lịch.
- [ ] Repository ứng dụng có ba commit Phase 3: `05-mongodb-tls.yaml`, `10-mongodb.yaml` (`preferTLS` rồi `requireTLS`), `20-backend.yaml`; manifest trên master khớp Git.
- [ ] Không có password, private key hay giá trị Secret trong Git, output đã gửi hay file trong `~/phase3`.

---

## 15. Nguồn official

- MongoDB 8.0 — Configure `mongod` and `mongos` for TLS/SSL (CA file bắt buộc, `allowConnectionsWithoutCertificates`, SAN): [https://www.mongodb.com/docs/v8.0/tutorial/configure-ssl/](https://www.mongodb.com/docs/v8.0/tutorial/configure-ssl/)
- MongoDB 8.0 — Configuration file options `net.tls.mode`, `net.tls.CAFile`, `net.tls.allowConnectionsWithoutCertificates`, `net.tls.certificateKeyFile`: [https://www.mongodb.com/docs/v8.0/reference/configuration-options/](https://www.mongodb.com/docs/v8.0/reference/configuration-options/)
- MongoDB 8.0 — `mongod` command-line options: [https://www.mongodb.com/docs/v8.0/reference/program/mongod/](https://www.mongodb.com/docs/v8.0/reference/program/mongod/)
- MongoDB Shell — options `--tls`, `--tlsCAFile`, `--host`: [https://www.mongodb.com/docs/mongodb-shell/reference/options/](https://www.mongodb.com/docs/mongodb-shell/reference/options/)
- MongoDB Database Tools — `mongodump` (`--ssl`, `--sslCAFile`, `--config`): [https://www.mongodb.com/docs/database-tools/mongodump/](https://www.mongodb.com/docs/database-tools/mongodump/)
- MongoDB Database Tools — `mongorestore`: [https://www.mongodb.com/docs/database-tools/mongorestore/](https://www.mongodb.com/docs/database-tools/mongorestore/)
- PyMongo — Configure TLS (`tls`, `tlsCAFile`, kiểm tra hostname mặc định bật, `AsyncMongoClient`): [https://www.mongodb.com/docs/languages/python/pymongo-driver/current/security/tls/](https://www.mongodb.com/docs/languages/python/pymongo-driver/current/security/tls/)
- Entrypoint image MongoDB 8.0 — thêm `mongod` khi đối số bắt đầu bằng `-`, tự thêm `--bind_ip_all`, server init dùng `allowTLS`: [https://github.com/docker-library/mongo/blob/7c24b37b8e53a41b56c450b653c582ff7c3f7fcb/8.0/docker-entrypoint.sh](https://github.com/docker-library/mongo/blob/7c24b37b8e53a41b56c450b653c582ff7c3f7fcb/8.0/docker-entrypoint.sh)
- cert-manager — SelfSigned issuer, bootstrapping CA issuers: [https://cert-manager.io/docs/configuration/selfsigned/](https://cert-manager.io/docs/configuration/selfsigned/)
- cert-manager — CA issuer (đổi CA không tự cấp lại certificate lá): [https://cert-manager.io/docs/configuration/ca/](https://cert-manager.io/docs/configuration/ca/)
- cert-manager — Certificate resource (`duration`, `renewBefore`, `usages`, `ipAddresses`, `rotationPolicy`): [https://cert-manager.io/docs/usage/certificate/](https://cert-manager.io/docs/usage/certificate/)
- Kubernetes — Volumes (`secret`, `configMap`, `emptyDir` với `medium: Memory`): [https://kubernetes.io/docs/concepts/storage/volumes/](https://kubernetes.io/docs/concepts/storage/volumes/)
- Kubernetes — Init containers: [https://kubernetes.io/docs/concepts/workloads/pods/init-containers/](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
