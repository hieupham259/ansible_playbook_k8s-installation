# Runbook Phase 2: Triển khai ứng dụng 3-tier React → FastAPI → MongoDB trên Kubernetes VMware

> **Phụ thuộc:** hoàn thành Phase 1 trong [`runbook-k8s-vmware.md`](runbook-k8s-vmware.md), tối thiểu đến §10: cụm có 3 node `Ready`, StorageClass `local-path`, Traefik hoạt động và app mẫu đi qua Ingress thành công.
>
> **Phạm vi Phase 2:** chuẩn bị, tạo source theo prompt, build image, triển khai và kiểm thử một ứng dụng CRUD nhỏ. File này **không chứa source code frontend/backend**; §5 chỉ định contract và prompt để tạo source ở bước sau.
>
> **Cách chạy bắt buộc:** thực hiện từng checkpoint theo thứ tự. Sau mỗi khối có nhãn **DỪNG — GỬI OUTPUT**, gửi nguyên output để kiểm tra; chỉ sang checkpoint tiếp theo khi kết quả được xác nhận **PASS**.
>
> **Môi trường:** mọi lệnh `kubectl` chạy trên `k8s-master` bằng user `ubuntu`, trừ khi tiêu đề ghi rõ worker hoặc máy build.

---

## Mục lục

1. [Mục tiêu, kiến trúc và giới hạn](#1-mục-tiêu-kiến-trúc-và-giới-hạn)
2. [Quy hoạch tài nguyên và phiên bản](#2-quy-hoạch-tài-nguyên-và-phiên-bản)
3. [Gate đầu vào từ Phase 1](#3-gate-đầu-vào-từ-phase-1)
4. [Chốt biến và contract triển khai](#4-chốt-biến-và-contract-triển-khai)
5. [Gate tạo source code — prompt dùng ở bước sau](#5-gate-tạo-source-code--prompt-dùng-ở-bước-sau)
6. [Build và push image](#6-build-và-push-image)
7. [Tạo namespace, ConfigMap và Secret](#7-tạo-namespace-configmap-và-secret)
8. [Triển khai MongoDB](#8-triển-khai-mongodb)
9. [Triển khai backend FastAPI](#9-triển-khai-backend-fastapi)
10. [Triển khai frontend React](#10-triển-khai-frontend-react)
11. [Tạo Ingress — chỉ frontend nhận traffic](#11-tạo-ingress--chỉ-frontend-nhận-traffic)
12. [Test CRUD end-to-end](#12-test-crud-end-to-end)
13. [Publish qua Cloudflare Tunnel](#13-publish-qua-cloudflare-tunnel)
14. [Quan sát, backup, restore và rollback](#14-quan-sát-backup-restore-và-rollback)
15. [Security và giới hạn của baseline](#15-security-và-giới-hạn-của-baseline)
16. [Troubleshooting](#16-troubleshooting)
17. [Checklist hoàn tất](#17-checklist-hoàn-tất)
18. [Nguồn official](#18-nguồn-official)

---

## 1. Mục tiêu, kiến trúc và giới hạn

### 1.1. Kết quả cần đạt

- Frontend React cung cấp giao diện Create/Read/Update/Delete.
- Backend FastAPI cung cấp REST API và là thành phần duy nhất kết nối MongoDB.
- MongoDB lưu dữ liệu trên PVC; restart Pod không làm mất dữ liệu.
- Client bên ngoài chỉ đi vào hostname của frontend qua Cloudflare Tunnel và Traefik.
- Backend và MongoDB dùng Service `ClusterIP`, không có Ingress, `NodePort` hay `LoadBalancer`.
- Frontend Nginx proxy `/api/*` tới Service backend. Browser không biết và không gọi DNS nội bộ `backend`.

### 1.2. Luồng request

```text
Internet client
  → Cloudflare Edge / Tunnel
  → Service traefik:80 (namespace traefik)
  → Ingress host=crud.example.com
  → Service frontend:80 (ClusterIP)
  → Pod frontend, Nginx
       ├─ /, assets/* → React static files
       └─ /api/*      → Service backend:8000 (ClusterIP)
                           → Pod FastAPI
                           → Service mongodb:27017 (ClusterIP, headless)
                           → Pod mongodb-0 + PVC 10 GiB
```

Ingress **không** trỏ trực tiếp tới backend. `/api` vẫn là API công khai đối với client, nhưng đi qua cùng hostname frontend và lớp reverse proxy; vì vậy production cần authentication, authorization và rate limiting.

### 1.3. Vì sao MongoDB dùng StatefulSet

MongoDB là workload **có trạng thái**: dữ liệu phải còn sau khi container hoặc Pod bị restart/tạo lại. `Deployment` phù hợp với frontend/backend stateless; `StatefulSet` phù hợp hơn cho database vì cung cấp đồng thời danh tính Pod, volume và DNS ổn định.

**Danh tính Pod ổn định:** Pod trong runbook luôn có tên `mongodb-0`. Khi Pod bị xóa, StatefulSet tạo lại một Pod mới nhưng vẫn dùng tên `mongodb-0`. Pod do Deployment quản lý thường có hậu tố ngẫu nhiên và thay đổi sau mỗi lần tạo lại, ví dụ `mongodb-6f79488b7c-x8p2k`.

**PVC ổn định cho từng Pod:** `volumeClaimTemplates` tạo PVC `data-mongodb-0`, sau đó mount vào `/data/db`:

```text
Pod mongodb-0
  → /data/db
  → PVC data-mongodb-0
  → PersistentVolume local-path
  → disk của worker
```

Khi `mongodb-0` bị recreate, StatefulSet gắn lại đúng PVC `data-mongodb-0`; dữ liệu không nằm trong writable layer tạm thời của container. Deployment vẫn có thể gắn một PVC thủ công, nhưng khi tăng nhiều replica sẽ khó bảo đảm mỗi Pod nhận đúng volume riêng. StatefulSet giải quyết mapping Pod ↔ PVC theo danh tính có thứ tự.

**DNS ổn định:** StatefulSet kết hợp Service headless `clusterIP: None` tạo hostname ổn định:

```text
mongodb-0.mongodb.three-tier.svc.cluster.local
```

Backend hiện gọi tên Service ngắn `mongodb:27017`. Nếu sau này chuyển sang replica set, các member có thể nhận diện nhau bằng các hostname ổn định như `mongodb-0.mongodb`, `mongodb-1.mongodb`, `mongodb-2.mongodb`.

StatefulSet **không tự biến MongoDB thành HA**. Baseline này khai `replicas: 1`, vì vậy vẫn là một MongoDB standalone:

- không có bản sao dữ liệu;
- không có bầu chọn primary;
- không tự failover khi worker hoặc disk bị mất;
- `local-path` vẫn ràng buộc dữ liệu với worker chứa volume.

Baseline này chạy **một MongoDB standalone** vì mục tiêu là homelab CRUD nhỏ và tài nguyên chỉ có hai worker 2 vCPU/6 GB. MongoDB xác định standalone phù hợp development/test, không có replication và automatic failover. Production cần ít nhất replica set ba member, storage HA, backup ngoài cluster và quy trình restore đã diễn tập.

### 1.4. Vì sao frontend cần Nginx proxy

React/Vite sau khi build tạo các static files như `index.html`, JavaScript và CSS. Production không cần chạy Vite development server; Nginx trong container frontend đảm nhiệm hai vai trò.

**Vai trò 1 — phục vụ React static files và SPA routing:**

```text
GET /             → index.html
GET /assets/*.js  → JavaScript bundle
GET /assets/*.css → CSS
GET /items/123    → index.html (SPA fallback, React Router xử lý tiếp)
```

Nếu không có SPA fallback, truy cập trực tiếp hoặc refresh `/items/123` có thể trả `404` dù route đó tồn tại trong React.

**Vai trò 2 — reverse proxy `/api` tới backend nội bộ:** browser của client nằm ngoài Kubernetes nên không thể resolve hoặc gọi trực tiếp DNS Service `http://backend:8000`. Tên `backend` chỉ có ý nghĩa với CoreDNS bên trong cluster. Vì Nginx chạy trong Pod frontend, nó có thể gọi Service này:

```text
Browser gọi https://crud.example.com/api/items
  → Traefik
  → Service frontend:80
  → Nginx trong Pod frontend
  → proxy /api/items tới http://backend:8000/api/items
  → Pod FastAPI
```

Frontend JavaScript vì vậy chỉ dùng URL tương đối:

```javascript
fetch("/api/items")
```

Không hard-code `backend:8000`, ClusterIP hay MongoDB URI vào JavaScript bundle. Cách này có các lợi ích:

- frontend và API dùng cùng scheme/hostname/port, nên browser coi là cùng origin và không cần thiết kế CORS cho hai domain riêng;
- chỉ có một Ingress trỏ tới `frontend:80`; backend và MongoDB không cần hostname public riêng;
- Nginx cung cấp một điểm cấu hình thống nhất cho SPA fallback, proxy headers, timeout và giới hạn request.

Nginx proxy **không phải biện pháp authentication**. Client vẫn có thể gọi `https://crud.example.com/api/*`; production phải bổ sung xác thực, phân quyền và rate limiting. Ý nghĩa của thiết kế này là backend không có Ingress/public Service riêng và chỉ nhận kết nối Kubernetes từ lớp frontend/proxy.

### 1.5. Giới hạn quan trọng của `local-path`

StorageClass `local-path` trong Phase 1 cấp volume nằm trên filesystem của một node. Dữ liệu sống qua việc restart/recreate Pod trên cùng node, nhưng không trở thành bản sao HA:

- worker chết hoặc mất disk → Pod MongoDB không tự mang dữ liệu sang worker còn lại;
- xóa PVC/PV có thể làm mất dữ liệu;
- snapshot VMware không thay thế backup ứng dụng nhất quán.

---

## 2. Quy hoạch tài nguyên và phiên bản

### 2.1. Baseline đã ghim

| Thành phần | Baseline | Lý do |
| --- | --- | --- |
| Kubernetes | Phase 1: v1.35.6 | Không thay đổi trong Phase 2 |
| Ingress | Traefik chart 41.0.2 / Proxy v3.7.6 | Tái sử dụng Phase 1 |
| Storage | `local-path` | Đã verify ở Phase 1; homelab, không HA |
| MongoDB | `docker.io/library/mongo:8.0.28-noble` | Major release 8.0, ghim patch và OS variant |
| Frontend | React + Vite, build thành static files; Nginx unprivileged | Nhẹ, browser chỉ gọi path tương đối `/api` |
| Backend | FastAPI + official PyMongo Async API | REST CRUD, health probes, connection pooling; không dùng Motor đã deprecated |

> Trước một lần cài mới trong tương lai, kiểm tra MongoDB 8.0 có patch bảo mật mới hơn hay không. Nếu đổi version, cập nhật **đồng thời** bảng này, manifest §8, gate pull image và checklist; không đổi riêng một chỗ.

### 2.2. Sizing cho lab

| Workload | Replica | Request mỗi Pod | Limit mỗi Pod | Storage |
| --- | ---: | --- | --- | --- |
| frontend | 2 | 50m CPU / 64Mi RAM | 250m / 128Mi | Không |
| backend | 2 | 100m CPU / 128Mi RAM | 500m / 384Mi | Không |
| MongoDB | 1 | 250m CPU / 512Mi RAM | 1000m / 1536Mi | PVC 10Gi |

Tổng request ứng dụng khoảng **550m CPU / 896Mi RAM**; phù hợp với hai worker tổng 4 vCPU/12 GB nếu Phase 1 vẫn còn headroom. Limit là trần khi có tải, không phải lượng được giữ trước.

---

## 3. Gate đầu vào từ Phase 1

### 3.1. Cluster, tài nguyên và add-on

**Mục đích:** xác nhận không triển khai database lên một cluster đang thiếu node, tài nguyên, ingress hoặc storage.

Chạy trên `k8s-master`:

```bash
kubectl get nodes -o wide
kubectl top nodes
kubectl get storageclass
kubectl -n local-path-storage rollout status deploy/local-path-provisioner --timeout=180s
kubectl -n traefik rollout status deploy/traefik --timeout=180s
kubectl -n traefik get deploy,pod,svc,endpointslice -o wide
kubectl auth can-i '*' '*' --all-namespaces
```

PASS khi:

- cả 3 node `Ready`; hai worker không có `MemoryPressure`, `DiskPressure`;
- `kubectl top nodes` có số liệu và còn headroom theo §2.2;
- `local-path` có `(default)` và provisioner rolled out;
- Traefik `1/1`, Pod `Running/Ready`, Service `ClusterIP`;
- quyền trả `yes`.

> **DỪNG — GỬI OUTPUT CHECKPOINT 3.1.** Không tiếp tục nếu thiếu bất kỳ điều kiện PASS nào.

### 3.2. Kiểm tra dung lượng disk thực trên hai worker

**Mục đích:** PVC 10Gi chỉ là yêu cầu Kubernetes; cần chắc worker còn đủ disk cho MongoDB, container images và log.

Chạy trên `k8s-worker1`, sau đó lặp lại trên `k8s-worker2`:

```bash
hostname
df -h /
sudo du -sh /var/lib/containerd /opt/local-path-provisioner 2>/dev/null
```

PASS khi đúng hostname, filesystem `/` không gần đầy và còn tối thiểu **15 GiB** trống trên mỗi worker trước khi bắt đầu.

> **DỪNG — GỬI OUTPUT CHECKPOINT 3.2 CỦA CẢ HAI WORKER.**

### 3.3. Gate pull image MongoDB

**Mục đích:** tách lỗi registry/DNS/TLS khỏi lỗi StatefulSet trước khi apply; scheduler có thể đặt MongoDB lên worker bất kỳ.

Chạy trên `k8s-worker1`, rồi `k8s-worker2`:

```bash
sudo crictl pull docker.io/library/mongo:8.0.28-noble
```

PASS khi cả hai node trả image reference/ID, không có lỗi DNS, TLS, timeout hay rate-limit.

> **DỪNG — GỬI OUTPUT CHECKPOINT 3.3 CỦA CẢ HAI WORKER.**

---

## 4. Chốt biến và contract triển khai

### 4.1. Các giá trị duy nhất được dùng trong toàn runbook

| Biến | Giá trị baseline | Được đổi? |
| --- | --- | --- |
| Namespace | `three-tier` | Không đổi giữa chừng |
| App hostname | `crud.example.com` | Bắt buộc đổi thành domain thật trước §11/§13 |
| MongoDB database | `cruddb` | Có, nhưng phải đổi mọi chỗ |
| Collection | `items` | Có, nhưng phải đổi contract API/test |
| Mongo root user | `root` | Chỉ dùng quản trị DB |
| Mongo app user | `crudapp` | Backend dùng user này với role `readWrite` trên `cruddb` |
| Frontend Service | `frontend:80` | Không |
| Backend Service | `backend:8000` | Không |
| MongoDB Service | `mongodb:27017` | Không |

### 4.2. Contract dữ liệu và API

Document `items`:

```json
{
  "id": "phase2-smoke-001",
  "name": "Keyboard",
  "description": "Mechanical keyboard",
  "created_at": "ISO-8601 UTC",
  "updated_at": "ISO-8601 UTC"
}
```

`id` là string duy nhất và có unique index. API bắt buộc:

| Method | Path | Kết quả |
| --- | --- | --- |
| `GET` | `/api/health/live` | `200` khi process FastAPI/event loop còn phản hồi; **không gọi MongoDB** |
| `GET` | `/api/health/ready` | `200` khi process và MongoDB sẵn sàng; non-200 nếu ping DB lỗi/timeout |
| `POST` | `/api/items` | `201`; tạo item, conflict trả `409` |
| `GET` | `/api/items` | `200`; danh sách có phân trang đơn giản |
| `GET` | `/api/items/{id}` | `200` hoặc `404` |
| `PUT` | `/api/items/{id}` | `200`; sửa name/description, không đổi id |
| `DELETE` | `/api/items/{id}` | `204` hoặc `404` |

Frontend luôn gọi URL tương đối `/api/...`; **không** hard-code IP, ClusterIP, hostname backend hay MongoDB URI vào JavaScript bundle.

---

## 5. Gate tạo source code — prompt dùng ở bước sau

Phase 2 hiện tại chỉ lưu yêu cầu. Khi sẵn sàng tạo source, mở một task Codex mới tại repository ứng dụng và gửi nguyên prompt dưới đây. Không tiếp tục §6 cho đến khi source gate cuối mục này PASS.

### 5.1. Prompt tạo source

```text
Hãy tạo một monorepo nhỏ tên three-tier-crud, production-oriented cho homelab Kubernetes.

Kiến trúc bắt buộc:
- frontend/: React + Vite + TypeScript; CRUD items gồm id, name, description.
- frontend build thành static files và chạy bằng Nginx unprivileged trên port 8080.
- Nginx phục vụ SPA fallback, GET /healthz trả 200, và reverse proxy /api/ tới
  http://backend:8000/api/; giữ đúng path /api, forward các header chuẩn.
- frontend chỉ gọi relative URL /api; không có MongoDB URI/credential hay backend ClusterIP trong bundle.
- backend/: FastAPI + official PyMongo Async API (AsyncMongoClient), không dùng Motor; REST contract đúng bảng ở
  runbook-k8s-vmware-phase2.md §4.2.
- Backend đọc MONGODB_URI, MONGODB_DATABASE=cruddb, MONGODB_COLLECTION=items từ environment.
- GET /api/health/live chỉ kiểm tra process FastAPI/event loop còn phản hồi, không gọi MongoDB.
- GET /api/health/ready phải ping MongoDB với timeout hữu hạn và trả non-200 nếu DB chưa sẵn sàng.
- Unit test phải chứng minh khi giả lập MongoDB down: /api/health/live vẫn 200 nhưng /api/health/ready non-200.
- Tạo unique index cho field id theo cách idempotent khi startup.
- Validate input: id/name không rỗng, giới hạn độ dài; description có giới hạn; trả status code rõ ràng.
- Không log credential hoặc toàn bộ MONGODB_URI.
- Xử lý SIGTERM/graceful shutdown và đóng MongoDB client.

Supply chain/build:
- Tạo Dockerfile riêng cho frontend/backend, multi-stage build, pin base image bằng version cụ thể.
- Container frontend và backend chạy non-root, không cần privileged, không ghi vào root filesystem
  ngoài /tmp nếu framework cần.
- Tạo .dockerignore, package lockfile và Python lock/requirements được pin đầy đủ.
- Không chạy npm/pip install ngoài build stage; không commit secrets.
- Image backend listen 0.0.0.0:8000; image frontend listen 0.0.0.0:8080.

Quality:
- Unit tests backend cho create/read/update/delete, validation, duplicate và not-found.
- Frontend tests tối thiểu cho API client và một CRUD flow.
- README nêu rõ lệnh test/build, biến môi trường và API examples.
- Chỉ tạo thư mục k8s/ rỗng (có thể giữ bằng .gitkeep); không tự thiết kế manifest Kubernetes.
- Các manifest tại §8–§11 của runbook-k8s-vmware-phase2.md là source of truth. Sau khi source/tests
  hoàn tất, operator sẽ chép nguyên văn các manifest đó vào k8s/ và chỉ thay placeholder image/domain.
- Không tự push image, không deploy cluster và không cài package trên máy của tôi nếu chưa được cho phép.

Trước khi sửa file: đọc AGENTS.md và repository hiện có. Sau khi tạo, chạy các test/build có thể chạy bằng
toolchain đã cài; báo rõ phần nào chưa verify. Không đổi contract nếu chưa hỏi tôi.
```

### 5.2. Cấu trúc source bắt buộc sau khi generate

```text
three-tier-crud/
├── frontend/
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── package.json
│   ├── lockfile
│   └── src/...
├── backend/
│   ├── Dockerfile
│   ├── dependency lock/requirements
│   ├── app/...
│   └── tests/...
├── k8s/
│   └── .gitkeep (manifest sẽ được operator chép từ source of truth §8–§11)
└── README.md
```

### 5.3. Source gate

Từ root repository ứng dụng, chạy các lệnh verify đúng theo README vừa tạo. Tối thiểu phải có:

```bash
git status --short
git ls-files | grep -Ei '(^|/)(\.env|.*secret.*|.*credential.*)$' || true
grep -RInE 'mongodb://[^[:space:]]+:[^[:space:]@]+@|MONGO_INITDB_ROOT_PASSWORD=' . \
  --exclude-dir=.git --exclude-dir=node_modules --exclude-dir=dist --exclude='*.md' || true
```

Mục đích: xem toàn bộ file mới và phát hiện credential/MongoDB URI bị commit. Hai lệnh grep phải không trả secret thật. Sau đó gửi:

- cây file;
- output test frontend/backend;
- output build frontend/backend;
- `git status --short`;
- kết quả scan secret (che giá trị nếu công cụ in ra).

> **DỪNG — GỬI OUTPUT CHECKPOINT 5.3.** Chỉ sang §6 khi source, tests và hai Docker build đều PASS.

---

## 6. Build và push image

> Chưa chạy mục này khi source chưa được tạo. Lệnh chạy trên **máy build** đã có Docker và đăng nhập registry. Runbook không tự cài Docker hay tạo tài khoản registry.

### 6.1. Verify toolchain máy build

**Mục đích:** chắc Docker daemon/buildx hoạt động trước khi build.

```bash
docker version
docker buildx version
docker info
```

> **DỪNG — GỬI OUTPUT CHECKPOINT 6.1.** Không gửi token/password registry.

### 6.2. Chốt image coordinates

Thay `<dockerhub-user>` bằng namespace thật. Dùng tag immutable theo Git commit, không dùng `latest`:

```bash
export REGISTRY_NAMESPACE='<dockerhub-user>'
export APP_VERSION="$(git rev-parse --short=12 HEAD)"
export FRONTEND_IMAGE="docker.io/${REGISTRY_NAMESPACE}/three-tier-frontend:${APP_VERSION}"
export BACKEND_IMAGE="docker.io/${REGISTRY_NAMESPACE}/three-tier-backend:${APP_VERSION}"
printf 'APP_VERSION=%s\nFRONTEND_IMAGE=%s\nBACKEND_IMAGE=%s\n' \
  "$APP_VERSION" "$FRONTEND_IMAGE" "$BACKEND_IMAGE"
```

PASS khi không còn dấu `< >`, tag là commit hiện tại và đúng repository của bạn.

Baseline giả định hai repository Docker Hub ở chế độ **public** để worker pull không cần credential. Nếu repository private, dừng tại đây và bổ sung `imagePullSecret` vào namespace cùng `spec.template.spec.imagePullSecrets` của cả hai Deployment; không paste registry token vào manifest hoặc chat.

> **DỪNG — GỬI OUTPUT CHECKPOINT 6.2.**

### 6.3. Build và smoke test image

**Mục đích:** tạo image linux/amd64 cho worker VMware và bắt lỗi container khởi động trước khi push.

```bash
docker buildx build --platform linux/amd64 --load \
  -t "$FRONTEND_IMAGE" ./frontend
docker buildx build --platform linux/amd64 --load \
  -t "$BACKEND_IMAGE" ./backend
docker image inspect "$FRONTEND_IMAGE" --format '{{.Id}} {{.Architecture}} {{.Config.User}}'
docker image inspect "$BACKEND_IMAGE" --format '{{.Id}} {{.Architecture}} {{.Config.User}}'
```

PASS khi build thành công, architecture `amd64` và `Config.User` không phải rỗng/`root`/`0`.

> **DỪNG — GỬI OUTPUT CHECKPOINT 6.3.**

### 6.4. Push và verify digest

```bash
docker push "$FRONTEND_IMAGE"
docker push "$BACKEND_IMAGE"
docker buildx imagetools inspect "$FRONTEND_IMAGE"
docker buildx imagetools inspect "$BACKEND_IMAGE"
```

Mục đích: registry trở thành nguồn image cho containerd trên worker. PASS khi cả hai có manifest `linux/amd64` và digest `sha256:...`.

> **DỪNG — GỬI OUTPUT CHECKPOINT 6.4.**

---

## 7. Tạo namespace, ConfigMap và Secret

### 7.1. Namespace

**Mục đích:** cô lập resource của app khỏi namespace `default` và add-on Phase 1.

```bash
kubectl create namespace three-tier --dry-run=client -o yaml | kubectl apply -f -
kubectl get namespace three-tier
```

PASS khi namespace `Active`.

> **DỪNG — GỬI OUTPUT CHECKPOINT 7.1.**

### 7.2. Tạo credential không ghi plaintext vào YAML

MongoDB cần hai credential:

- root: chỉ dùng quản trị/backup;
- app user `crudapp`: backend chỉ có `readWrite` trên `cruddb`.

Nhập password ẩn trên `k8s-master`; không gửi password/screenshot cho người kiểm tra:

```bash
read -rsp 'Mongo root password: ' MONGO_ROOT_PASSWORD; echo
read -rsp 'Mongo app password: ' MONGO_APP_PASSWORD; echo
test -n "$MONGO_ROOT_PASSWORD" && test -n "$MONGO_APP_PASSWORD" && echo 'password variables: set'
```

Password được đọc theo một dòng nên không dùng newline; các ký tự đặc biệt khác như `"`, `\`, `$`, `@`, `:` được hỗ trợ. Tạo thêm `MONGO_TOOLS_CONFIG` ở dạng YAML một dòng với password được JSON-quote an toàn để `mongodump`/`mongorestore` đọc bằng `--config` mà không đưa password vào command line:

```bash
MONGO_TOOLS_CONFIG=$(printf '%s' "$MONGO_ROOT_PASSWORD" | python3 -c \
  'import json, sys; print("password: " + json.dumps(sys.stdin.read()))')
{
  printf 'MONGO_INITDB_ROOT_USERNAME=root\n'
  printf 'MONGO_INITDB_ROOT_PASSWORD=%s\n' "$MONGO_ROOT_PASSWORD"
  printf 'MONGO_APP_USERNAME=crudapp\n'
  printf 'MONGO_APP_PASSWORD=%s\n' "$MONGO_APP_PASSWORD"
  printf 'MONGO_TOOLS_CONFIG=%s\n' "$MONGO_TOOLS_CONFIG"
} | kubectl -n three-tier create secret generic mongodb-credentials \
  --from-env-file=/dev/stdin \
  --dry-run=client -o yaml | kubectl apply -f -
unset MONGO_ROOT_PASSWORD MONGO_APP_PASSWORD MONGO_TOOLS_CONFIG
kubectl -n three-tier get secret mongodb-credentials \
  -o go-template='name={{.metadata.name}}{{"\n"}}type={{.type}}{{"\n"}}keys={{range $k,$v := .data}}{{$k}} {{end}}{{"\n"}}'
```

Mục đích: Secret lưu credential dưới Kubernetes API thay vì file Git. Dữ liệu đi vào `kubectl` qua stdin, không xuất hiện trong arguments của process `kubectl`. PASS khi có năm key; **không** chạy `kubectl get secret -o yaml` và không gửi base64 values.

> **DỪNG — GỬI OUTPUT CHECKPOINT 7.2, CHỈ OUTPUT DÒNG METADATA/KEY NAME.**

### 7.3. ConfigMap ứng dụng

```bash
kubectl -n three-tier create configmap app-config \
  --from-literal=MONGODB_DATABASE=cruddb \
  --from-literal=MONGODB_COLLECTION=items \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl -n three-tier get configmap app-config -o yaml
```

Mục đích: tách config không nhạy cảm khỏi image. PASS khi database/collection đúng §4.

> **DỪNG — GỬI OUTPUT CHECKPOINT 7.3.**

---

## 8. Triển khai MongoDB

### 8.1. Tạo manifest database

Tạo `k8s/10-mongodb.yaml` trong repository ứng dụng bằng cách chép đúng manifest dưới đây; đây là **source of truth**, không hợp nhất với manifest do task tạo source tự sinh. Manifest tạo Service headless, script init app user và StatefulSet. Script trong `/docker-entrypoint-initdb.d` chỉ chạy khi data directory còn rỗng.

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
      containers:
        - name: mongodb
          image: docker.io/library/mongo:8.0.28-noble
          imagePullPolicy: IfNotPresent
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
          startupProbe:
            exec:
              command: ["mongosh", "--quiet", "--eval", "db.adminCommand('ping').ok"]
            periodSeconds: 5
            timeoutSeconds: 5
            failureThreshold: 30
          readinessProbe:
            exec:
              command: ["mongosh", "--quiet", "--eval", "db.adminCommand('ping').ok"]
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
            defaultMode: 0550
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

> **Kiểm tra trước apply:** trong container MongoDB chỉ được có **một** block `volumeMounts`. Server-side dry-run dưới đây kiểm tra schema Kubernetes; khi review phải đồng thời kiểm tra không có key YAML bị lặp.
>
> `startupProbe` và `readinessProbe` dùng `mongosh`, nên mỗi lần probe phải spawn một process Node.js. Baseline giữ command-level check ở startup/readiness nhưng giảm readiness xuống bốn lần/phút; liveness dùng TCP nhẹ hơn và chỉ kiểm tra `mongod` còn listen. Readiness mới là tín hiệu xác nhận database thực sự trả command.
>
> Runbook cố ý không khai `fsGroup`. StorageClass `local-path` mặc định tạo PV `hostPath` và setup directory mode `0777`; không được dựa vào `fsGroup` để sửa ownership hay coi đây là storage hardening. Verify loại volume thực tế ở §8.3.

```bash
kubectl apply --dry-run=server -f k8s/10-mongodb.yaml
```

PASS khi server trả `service/mongodb`, `configmap/mongodb-init`, `statefulset.apps/mongodb` và không có validation error.

> **DỪNG — GỬI OUTPUT CHECKPOINT 8.1.**

### 8.2. Apply và verify MongoDB

```bash
kubectl apply -f k8s/10-mongodb.yaml
kubectl -n three-tier rollout status statefulset/mongodb --timeout=300s
kubectl -n three-tier get statefulset,pod,svc,pvc,pv -o wide
kubectl -n three-tier logs mongodb-0 --tail=80
```

Mục đích: tạo database, đợi Ready, xác nhận PVC `Bound` và xem init/auth error. PASS khi:

- `mongodb-0` `1/1 Running`;
- PVC `data-mongodb-0` `Bound`, StorageClass `local-path`, capacity 10Gi;
- Service `mongodb` có `CLUSTER-IP=None`, không có external IP;
- log không có `AuthenticationFailed`, `permission denied`, crash/restart loop.

> Warning WiredTiger hoặc khuyến nghị kernel có thể cần đánh giá riêng; không bỏ qua `ERROR`, `FATAL` hay restart tăng.

> **DỪNG — GỬI OUTPUT CHECKPOINT 8.2.**

### 8.3. Verify app user và persistence

Không lấy password về master và không truyền password qua `--password`. `mongosh` đọc credential từ environment mà Pod đã nhận qua Secret:

```bash
kubectl -n three-tier exec mongodb-0 -- mongosh --quiet --host 127.0.0.1 --eval '
  const appDb = db.getSiblingDB(process.env.MONGO_APP_DATABASE)
  if (!appDb.auth(process.env.MONGO_APP_USERNAME, process.env.MONGO_APP_PASSWORD)) {
    throw new Error("MongoDB app-user authentication failed")
  }
  printjson(appDb.runCommand({ping: 1}))
  printjson(appDb.items.getIndexes())
'
```

PASS khi ping `ok: 1` và index `id_1` có `unique: true`.

Kiểm tra Pod gắn đúng PVC và node:

```bash
kubectl -n three-tier get pod mongodb-0 \
  -o custom-columns='POD:.metadata.name,NODE:.spec.nodeName,PVC:.spec.volumes[*].persistentVolumeClaim.claimName'
kubectl -n three-tier get pvc data-mongodb-0
PV_NAME=$(kubectl -n three-tier get pvc data-mongodb-0 -o jsonpath='{.spec.volumeName}')
kubectl get pv "$PV_NAME" \
  -o custom-columns='PV:.metadata.name,HOSTPATH:.spec.hostPath.path,LOCAL:.spec.local.path,NODE:.spec.nodeAffinity.required.nodeSelectorTerms[*].matchExpressions[*].values[*]'
unset PV_NAME
```

Baseline mong đợi cột `HOSTPATH` có đường dẫn dưới `/opt/local-path-provisioner`, cột `LOCAL` rỗng và node khớp Pod `mongodb-0`. Điều này xác nhận vì sao runbook không dựa vào `fsGroup`.

> **DỪNG — GỬI OUTPUT CHECKPOINT 8.3.**

---

## 9. Triển khai backend FastAPI

### 9.1. Tạo MongoDB URI Secret cho backend

URI chứa password nên không đặt trong ConfigMap/YAML/Git. Nhập lại app password ẩn:

```bash
read -rsp 'Mongo app password: ' MONGO_APP_PASSWORD; echo
MONGO_APP_PASSWORD_ENCODED=$(printf '%s' "$MONGO_APP_PASSWORD" | python3 -c \
  'import sys, urllib.parse; print(urllib.parse.quote(sys.stdin.read(), safe=""))')
MONGODB_URI="mongodb://crudapp:${MONGO_APP_PASSWORD_ENCODED}@mongodb:27017/cruddb?authSource=cruddb"
printf 'MONGODB_URI=%s\n' "$MONGODB_URI" | \
  kubectl -n three-tier create secret generic backend-mongodb-uri \
  --from-env-file=/dev/stdin \
  --dry-run=client -o yaml | kubectl apply -f -
unset MONGO_APP_PASSWORD MONGO_APP_PASSWORD_ENCODED MONGODB_URI
kubectl -n three-tier get secret backend-mongodb-uri \
  -o go-template='name={{.metadata.name}}{{"\n"}}keys={{range $k,$v := .data}}{{$k}} {{end}}{{"\n"}}'
```

Mục đích: backend kết nối bằng user quyền hẹp, không dùng root. PASS khi Secret có key `MONGODB_URI`; không gửi value.

> Lệnh Python dùng `urllib.parse.quote` để percent-encode mọi ký tự reserved trong password trước khi ghép URI. Source không được tự nối username/password chưa encode.

> **DỪNG — GỬI OUTPUT CHECKPOINT 9.1, KHÔNG GỬI SECRET.**

### 9.2. Manifest backend

Tạo `k8s/20-backend.yaml`, thay `CHANGE_ME_BACKEND_IMAGE` bằng giá trị `BACKEND_IMAGE` đã PASS ở §6.2:

Readiness và liveness cố ý dùng hai endpoint khác nhau. Readiness phụ thuộc MongoDB để loại Pod khỏi EndpointSlice khi database chưa phục vụ được request. Liveness chỉ kiểm tra process; MongoDB down không được làm Kubernetes restart một process FastAPI vẫn khỏe, vì restart backend không sửa được sự cố database.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: three-tier
  labels:
    app.kubernetes.io/name: backend
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: backend
  template:
    metadata:
      labels:
        app.kubernetes.io/name: backend
    spec:
      automountServiceAccountToken: false
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: backend
      securityContext:
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: backend
          image: CHANGE_ME_BACKEND_IMAGE
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8000
          envFrom:
            - configMapRef:
                name: app-config
          env:
            - name: MONGODB_URI
              valueFrom:
                secretKeyRef:
                  name: backend-mongodb-uri
                  key: MONGODB_URI
          readinessProbe:
            httpGet: { path: /api/health/ready, port: http }
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 6
          livenessProbe:
            httpGet: { path: /api/health/live, port: http }
            periodSeconds: 20
            timeoutSeconds: 3
            failureThreshold: 6
          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits: { cpu: 500m, memory: 384Mi }
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
            readOnlyRootFilesystem: true
            runAsNonRoot: true
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: three-tier
  labels:
    app.kubernetes.io/name: backend
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: backend
  ports:
    - name: http
      port: 8000
      targetPort: http
```

Kiểm tra placeholder và server-side validation:

```bash
grep -n 'CHANGE_ME' k8s/20-backend.yaml || true
kubectl apply --dry-run=server -f k8s/20-backend.yaml
```

PASS khi grep không có output và dry-run không lỗi.

> **DỪNG — GỬI OUTPUT CHECKPOINT 9.2.**

### 9.3. Apply và verify backend

```bash
kubectl apply -f k8s/20-backend.yaml
kubectl -n three-tier rollout status deploy/backend --timeout=300s
kubectl -n three-tier get deploy,pod,svc -l app.kubernetes.io/name=backend -o wide
kubectl -n three-tier get endpointslice \
  -l kubernetes.io/service-name=backend -o wide
kubectl -n three-tier logs \
  -l app.kubernetes.io/name=backend --tail=80 --prefix
```

PASS khi Deployment `2/2`, hai Pod Ready, EndpointSlice có hai endpoint, Service là `ClusterIP`, log của **cả hai Pod** không lộ URI/password và không có connection/auth error. `topologySpreadConstraints` với `ScheduleAnyway` khiến scheduler ưu tiên đặt hai replica trên hai worker khác nhau nhưng vẫn cho phép co-locate khi chỉ còn một worker; `maxUnavailable: 0` chỉ bảo vệ rolling update, không tự bảo vệ khỏi node failure.

> **DỪNG — GỬI OUTPUT CHECKPOINT 9.3.**

### 9.4. Test backend tạm thời bằng port-forward

**Mục đích:** xác nhận API ↔ MongoDB trước khi thêm frontend/Ingress. Port-forward chỉ mở trên `127.0.0.1` của master và không expose backend ra LAN/Internet.

```bash
kubectl -n three-tier port-forward svc/backend 18000:8000 \
  >/tmp/backend-port-forward.log 2>&1 & BACKEND_PF_PID=$!
trap 'kill "$BACKEND_PF_PID" 2>/dev/null' EXIT
curl -sS --retry 10 --retry-connrefused --retry-delay 1 \
  -o /dev/null -w 'backend-live=%{http_code}\n' \
  http://127.0.0.1:18000/api/health/live
curl -sS -o /dev/null -w 'backend-ready=%{http_code}\n' \
  http://127.0.0.1:18000/api/health/ready
```

PASS khi cả `backend-live` và `backend-ready` là `200`; kết quả thứ hai chứng minh backend ping MongoDB thành công. Cleanup:

```bash
kill "$BACKEND_PF_PID"; wait "$BACKEND_PF_PID" 2>/dev/null
trap - EXIT; unset BACKEND_PF_PID
```

> **DỪNG — GỬI OUTPUT CHECKPOINT 9.4.**

---

## 10. Triển khai frontend React

### 10.1. Manifest frontend

Tạo `k8s/30-frontend.yaml`, thay `CHANGE_ME_FRONTEND_IMAGE` bằng image §6.2:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: three-tier
  labels:
    app.kubernetes.io/name: frontend
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: frontend
  template:
    metadata:
      labels:
        app.kubernetes.io/name: frontend
    spec:
      automountServiceAccountToken: false
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: frontend
      securityContext:
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: frontend
          image: CHANGE_ME_FRONTEND_IMAGE
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
          readinessProbe:
            httpGet: { path: /healthz, port: http }
            periodSeconds: 10
            timeoutSeconds: 3
          livenessProbe:
            httpGet: { path: /healthz, port: http }
            periodSeconds: 20
            timeoutSeconds: 3
          resources:
            requests: { cpu: 50m, memory: 64Mi }
            limits: { cpu: 250m, memory: 128Mi }
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
            readOnlyRootFilesystem: true
            runAsNonRoot: true
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: three-tier
  labels:
    app.kubernetes.io/name: frontend
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: frontend
  ports:
    - name: http
      port: 80
      targetPort: http
```

```bash
grep -n 'CHANGE_ME' k8s/30-frontend.yaml || true
kubectl apply --dry-run=server -f k8s/30-frontend.yaml
```

PASS khi không còn placeholder và dry-run không lỗi.

> **DỪNG — GỬI OUTPUT CHECKPOINT 10.1.**

### 10.2. Apply và verify frontend/proxy

```bash
kubectl apply -f k8s/30-frontend.yaml
kubectl -n three-tier rollout status deploy/frontend --timeout=300s
kubectl -n three-tier get deploy,pod,svc -l app.kubernetes.io/name=frontend -o wide
kubectl -n three-tier get endpointslice \
  -l kubernetes.io/service-name=frontend -o wide
kubectl -n three-tier logs \
  -l app.kubernetes.io/name=frontend --tail=50 --prefix
```

PASS khi Deployment `2/2`, hai Pod Ready, Service `ClusterIP`, EndpointSlice có hai endpoint và output log có prefix của cả hai Pod. Scheduler nên ưu tiên tách hai replica sang hai worker; vì dùng `ScheduleAnyway`, co-location là fallback hợp lệ khi tài nguyên hoặc số worker không cho phép spread.

Test Nginx và reverse proxy qua frontend Service:

```bash
kubectl -n three-tier port-forward svc/frontend 18081:80 \
  >/tmp/frontend-port-forward.log 2>&1 & FRONTEND_PF_PID=$!
trap 'kill "$FRONTEND_PF_PID" 2>/dev/null' EXIT
curl -sS -o /dev/null -w 'frontend=%{http_code}\n' \
  http://127.0.0.1:18081/
curl -sS -o /dev/null -w 'api-live=%{http_code}\n' \
  http://127.0.0.1:18081/api/health/live
curl -sS -o /dev/null -w 'api-ready=%{http_code}\n' \
  http://127.0.0.1:18081/api/health/ready
```

PASS khi cả ba code là `200`. Cleanup:

```bash
kill "$FRONTEND_PF_PID"; wait "$FRONTEND_PF_PID" 2>/dev/null
trap - EXIT; unset FRONTEND_PF_PID
```

> **DỪNG — GỬI OUTPUT CHECKPOINT 10.2.**

---

## 11. Tạo Ingress — chỉ frontend nhận traffic

### 11.1. Manifest Ingress

Trước khi tạo file, thay `crud.example.com` bằng domain thật. Nếu chưa có domain vẫn có thể dùng placeholder để test nội bộ bằng Host header, nhưng không làm §13.

Tạo `k8s/40-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: three-tier
  namespace: three-tier
spec:
  ingressClassName: traefik
  rules:
    - host: crud.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

Điểm kiểm soát quan trọng: backend duy nhất của Ingress phải là `frontend:80`; file không được nhắc `backend:8000` hay `mongodb:27017`.

```bash
kubectl apply --dry-run=server -f k8s/40-ingress.yaml
grep -nE 'name: (backend|mongodb)|number: (8000|27017)' k8s/40-ingress.yaml || true
```

PASS khi dry-run thành công và grep không có output.

> **DỪNG — GỬI OUTPUT CHECKPOINT 11.1.**

### 11.2. Apply và kiểm tra bề mặt expose

```bash
kubectl apply -f k8s/40-ingress.yaml
kubectl -n three-tier get ingress three-tier
kubectl -n three-tier describe ingress three-tier
kubectl -n three-tier get svc \
  -o custom-columns='NAME:.metadata.name,TYPE:.spec.type,CLUSTER-IP:.spec.clusterIP,PORTS:.spec.ports[*].port'
kubectl get ingress -A
```

PASS khi:

- Ingress class `traefik`, host đúng, backend `frontend:80`;
- `frontend`, `backend` là `ClusterIP`; `mongodb` headless (`None`);
- không có Ingress nào trỏ backend/MongoDB;
- cột `ADDRESS` của Ingress có thể trống với Traefik Service `ClusterIP`; đây là expectation của Phase 1, không phải fail.

> **DỪNG — GỬI OUTPUT CHECKPOINT 11.2.**

### 11.3. Test nội bộ end-to-end qua Traefik

Đặt hostname đúng như manifest:

```bash
export APP_HOST='crud.example.com'
ING_IP=$(kubectl -n traefik get svc traefik -o jsonpath='{.spec.clusterIP}')
printf 'APP_HOST=%s\nTRAEFIK_CLUSTER_IP=%s\n' "$APP_HOST" "$ING_IP"
curl -sS -o /dev/null -w 'frontend-via-traefik=%{http_code}\n' \
  -H "Host: $APP_HOST" "http://$ING_IP/"
curl -sS -o /dev/null -w 'api-ready-via-traefik=%{http_code}\n' \
  -H "Host: $APP_HOST" "http://$ING_IP/api/health/ready"
```

Mục đích: chứng minh chuỗi master → Traefik → Ingress → frontend → backend → MongoDB. PASS khi cả hai code `200`.

> **DỪNG — GỬI OUTPUT CHECKPOINT 11.3.**

---

## 12. Test CRUD end-to-end

Các request dưới đây vẫn đi qua Traefik và frontend Nginx, không gọi trực tiếp backend. Dùng ID cố định để không cần `jq`.

### 12.1. Create và duplicate protection

```bash
curl -sS -i -X POST -H "Host: $APP_HOST" \
  -H 'Content-Type: application/json' \
  "http://$ING_IP/api/items" \
  -d '{"id":"phase2-smoke-001","name":"Keyboard","description":"Mechanical keyboard"}'
```

PASS lần đầu: `201 Created`, response có đúng `id`. Chạy lại cùng lệnh phải trả `409 Conflict`, chứng minh unique ID được enforce.

> **DỪNG — GỬI OUTPUT CHECKPOINT 12.1.**

### 12.2. Read/list

```bash
curl -sS -i -H "Host: $APP_HOST" \
  "http://$ING_IP/api/items/phase2-smoke-001"
curl -sS -i -H "Host: $APP_HOST" \
  "http://$ING_IP/api/items"
```

PASS khi cả hai trả `200` và item xuất hiện trong list.

> **DỪNG — GỬI OUTPUT CHECKPOINT 12.2.**

### 12.3. Update

```bash
curl -sS -i -X PUT -H "Host: $APP_HOST" \
  -H 'Content-Type: application/json' \
  "http://$ING_IP/api/items/phase2-smoke-001" \
  -d '{"name":"Keyboard v2","description":"Updated through Traefik and frontend proxy"}'
curl -sS -H "Host: $APP_HOST" \
  "http://$ING_IP/api/items/phase2-smoke-001"
```

PASS khi update trả `200`, lần GET sau có name/description mới và `id` không đổi.

> **DỪNG — GỬI OUTPUT CHECKPOINT 12.3.**

### 12.4. Delete và not-found

```bash
curl -sS -i -X DELETE -H "Host: $APP_HOST" \
  "http://$ING_IP/api/items/phase2-smoke-001"
curl -sS -i -H "Host: $APP_HOST" \
  "http://$ING_IP/api/items/phase2-smoke-001"
```

PASS khi DELETE trả `204` và GET sau đó trả `404`.

> **DỪNG — GỬI OUTPUT CHECKPOINT 12.4.**

### 12.5. Persistence qua restart MongoDB Pod

Tạo lại một record, restart Pod rồi đọc lại:

```bash
curl -sS -X POST -H "Host: $APP_HOST" -H 'Content-Type: application/json' \
  "http://$ING_IP/api/items" \
  -d '{"id":"phase2-persist-001","name":"Persistent item","description":"must survive pod restart"}'
OLD_MONGO_UID=$(kubectl -n three-tier get pod mongodb-0 -o jsonpath='{.metadata.uid}')
kubectl -n three-tier delete pod mongodb-0
kubectl -n three-tier wait --for=create pod/mongodb-0 --timeout=120s
kubectl -n three-tier wait --for=condition=Ready pod/mongodb-0 --timeout=300s
NEW_MONGO_UID=$(kubectl -n three-tier get pod mongodb-0 -o jsonpath='{.metadata.uid}')
printf 'OLD_UID=%s\nNEW_UID=%s\n' "$OLD_MONGO_UID" "$NEW_MONGO_UID"
test "$OLD_MONGO_UID" != "$NEW_MONGO_UID"
kubectl -n three-tier get pod mongodb-0 -o wide
curl -sS -i -H "Host: $APP_HOST" \
  "http://$ING_IP/api/items/phase2-persist-001"
unset OLD_MONGO_UID NEW_MONGO_UID
```

Mục đích: `--for=create` loại race `NotFound` giữa lúc Pod cũ biến mất và StatefulSet tạo Pod mới; UID khác nhau chứng minh đây thực sự là Pod mới. PASS khi Pod mới Ready, vẫn dùng PVC `data-mongodb-0` và GET trả `200`.

> **DỪNG — GỬI OUTPUT CHECKPOINT 12.5.**

---

## 13. Publish qua Cloudflare Tunnel

Chỉ làm khi đã có domain/zone Cloudflare và tunnel Phase 1 đang healthy.

### 13.1. Tạo Published application

Trong Cloudflare Zero Trust → Networks → Tunnels → tunnel Phase 1 → Published application routes:

- Hostname: domain thật đã dùng trong `k8s/40-ingress.yaml`;
- Service type: `HTTP`;
- URL: `traefik.traefik.svc.cluster.local:80`;
- không trỏ tunnel trực tiếp tới `frontend`, `backend` hoặc `mongodb`.

Lý do: Traefik cần nhận đúng Host header để chọn Ingress; giữ một điểm vào chung như Phase 1.

### 13.2. Verify tunnel và public endpoint

```bash
kubectl -n cloudflare rollout status deploy/cloudflared --timeout=180s
kubectl -n cloudflare logs deploy/cloudflared --tail=80
curl -sS -o /dev/null -w 'public-frontend=%{http_code}\n' "https://$APP_HOST/"
curl -sS -o /dev/null -w 'public-api-ready=%{http_code}\n' "https://$APP_HOST/api/health/ready"
```

PASS khi cloudflared healthy và cả hai public URL trả `200` qua HTTPS.

> **DỪNG — GỬI OUTPUT CHECKPOINT 13.2.**

---

## 14. Quan sát, backup, restore và rollback

### 14.1. Health snapshot sau triển khai

```bash
kubectl -n three-tier get deploy,statefulset,pod,svc,ingress,pvc -o wide
kubectl -n three-tier top pod
kubectl -n three-tier get events --sort-by=.lastTimestamp | tail -30
kubectl -n three-tier logs \
  -l app.kubernetes.io/name=backend --tail=100 --prefix
kubectl -n three-tier logs \
  -l app.kubernetes.io/name=frontend --tail=50 --prefix
kubectl -n three-tier logs mongodb-0 --tail=100
```

PASS khi workload Ready, PVC Bound, restart thấp, resource không sát limit, không có Warning lặp lại.

> **DỪNG — GỬI OUTPUT CHECKPOINT 14.1.**

### 14.2. Backup logic bằng `mongodump`

Không coi PVC là backup. `mongodump` đọc password từ YAML config chứa trong `MONGO_TOOLS_CONFIG`; remote shell ghi config vào file tạm mode `0600` và xóa bằng `trap`. Password không được lấy về master hoặc đặt trong CLI arguments:

```bash
BACKUP_TS=$(date -u +%Y%m%dT%H%M%SZ)
kubectl -n three-tier exec mongodb-0 -- env BACKUP_TS="$BACKUP_TS" bash -ec '
  umask 077
  CONFIG_FILE=$(mktemp /tmp/mongodb-tools.XXXXXX.yaml)
  trap "rm -f \"$CONFIG_FILE\"" EXIT
  printf "%s\n" "$MONGO_TOOLS_CONFIG" > "$CONFIG_FILE"
  mongodump --config="$CONFIG_FILE" \
    --host 127.0.0.1 \
    --username "$MONGO_INITDB_ROOT_USERNAME" \
    --authenticationDatabase admin \
    --db cruddb \
    --archive="/tmp/cruddb-${BACKUP_TS}.archive" \
    --gzip
'
kubectl -n three-tier cp \
  "mongodb-0:/tmp/cruddb-${BACKUP_TS}.archive" "./cruddb-${BACKUP_TS}.archive"
ls -lh "./cruddb-${BACKUP_TS}.archive"
```

PASS khi file archive tồn tại trên master và size lớn hơn 0. Sau khi copy/verify, xóa bản tạm trong Pod:

```bash
kubectl -n three-tier exec mongodb-0 -- rm -f "/tmp/cruddb-${BACKUP_TS}.archive"
```

> Archive trên master vẫn cùng site VMware; sao chép mã hóa sang nơi lưu backup được kiểm soát là bước vận hành riêng. Không upload lên dịch vụ ngoài khi chưa được phép.

> **DỪNG — GỬI OUTPUT CHECKPOINT 14.2; KHÔNG GỬI FILE BACKUP.**

### 14.3. Rehearse restore vào database tạm

Backup chưa từng restore thì chưa được coi là backup đã kiểm chứng. Bài test này restore archive mới nhất vào `cruddb_restore_test`, không ghi đè `cruddb`:

```bash
BACKUP_FILE=$(ls -1t ./cruddb-*.archive | head -1)
test -s "$BACKUP_FILE" && echo "restore source: $BACKUP_FILE"
kubectl -n three-tier cp "$BACKUP_FILE" mongodb-0:/tmp/restore-test.archive
kubectl -n three-tier exec mongodb-0 -- bash -ec '
  umask 077
  CONFIG_FILE=$(mktemp /tmp/mongodb-tools.XXXXXX.yaml)
  trap "rm -f \"$CONFIG_FILE\"" EXIT
  printf "%s\n" "$MONGO_TOOLS_CONFIG" > "$CONFIG_FILE"
  mongorestore --config="$CONFIG_FILE" \
    --host 127.0.0.1 \
    --username "$MONGO_INITDB_ROOT_USERNAME" \
    --authenticationDatabase admin \
    --archive=/tmp/restore-test.archive \
    --gzip \
    --nsFrom="cruddb.*" \
    --nsTo="cruddb_restore_test.*" \
    --drop
'
kubectl -n three-tier exec mongodb-0 -- mongosh --quiet --host 127.0.0.1 --eval '
  const adminDb = db.getSiblingDB("admin")
  if (!adminDb.auth(
    process.env.MONGO_INITDB_ROOT_USERNAME,
    process.env.MONGO_INITDB_ROOT_PASSWORD
  )) {
    throw new Error("MongoDB root authentication failed")
  }
  print(db.getSiblingDB("cruddb_restore_test").items.countDocuments({}))
'
```

PASS khi `mongorestore` không lỗi và count phù hợp dữ liệu tại thời điểm backup. Cleanup chỉ database test và archive tạm trong Pod; giữ archive gốc trên master:

```bash
kubectl -n three-tier exec mongodb-0 -- mongosh --quiet --host 127.0.0.1 --eval '
  const adminDb = db.getSiblingDB("admin")
  if (!adminDb.auth(
    process.env.MONGO_INITDB_ROOT_USERNAME,
    process.env.MONGO_INITDB_ROOT_PASSWORD
  )) {
    throw new Error("MongoDB root authentication failed")
  }
  printjson(db.getSiblingDB("cruddb_restore_test").dropDatabase())
'
kubectl -n three-tier exec mongodb-0 -- rm -f /tmp/restore-test.archive
```

> **DỪNG — GỬI OUTPUT CHECKPOINT 14.3; KHÔNG GỬI FILE BACKUP HOẶC SECRET.**

### 14.4. Rollback frontend/backend

Xem revision rồi rollback từng Deployment nếu rollout mới lỗi:

```bash
kubectl -n three-tier rollout history deploy/frontend
kubectl -n three-tier rollout history deploy/backend
kubectl -n three-tier rollout undo deploy/frontend
kubectl -n three-tier rollout undo deploy/backend
kubectl -n three-tier rollout status deploy/frontend --timeout=300s
kubectl -n three-tier rollout status deploy/backend --timeout=300s
```

Không rollback MongoDB bằng `rollout undo` một cách máy móc. Upgrade/downgrade database phải theo compatibility/FCV docs và có backup đã test restore.

---

## 15. Security và giới hạn của baseline

### 15.1. Những gì baseline đã làm

- Chỉ frontend có Ingress; backend/MongoDB không có public route.
- MongoDB bật authentication qua root init variables của official image.
- Backend dùng user `crudapp` với `readWrite` chỉ trên `cruddb`, không dùng root.
- Credential nằm trong Secret, không trong source/image/manifest Git.
- Secret được đưa vào `kubectl` qua stdin; `mongosh` đọc `process.env`; Database Tools đọc config tạm `0600`, tránh password trong process arguments.
- Frontend/backend chạy non-root, drop Linux capabilities và dùng seccomp RuntimeDefault.
- Frontend/backend có topology spread mềm theo hostname để ưu tiên tách hai replica sang hai worker.
- Image và MongoDB patch được pin; không dùng `latest`.
- Resource request/limit và health probes được khai báo.

### 15.2. Những gì baseline chưa giải quyết

- Kubernetes Secret mặc định được lưu base64 và có thể chưa được encrypt at rest trong etcd. Production cần encryption at rest, RBAC tối thiểu và secret manager/external secrets.
- MongoDB traffic trong cluster chưa bật TLS. Cluster đáng tin cậy/homelab chấp nhận tạm; production cần TLS giữa backend và MongoDB.
- Một MongoDB replica + local-path không HA.
- API public chưa có đăng nhập/phân quyền. Cần OIDC/session, CSRF strategy phù hợp, rate limit và Cloudflare Access/WAF tùy đối tượng dùng.
- Không có vulnerability scanning/signing/SBOM trong baseline build.

### 15.3. Lưu ý NetworkPolicy với Flannel Phase 1

Service `ClusterIP` ngăn truy cập trực tiếp từ ngoài cluster, nhưng không mặc nhiên chặn Pod khác trong cluster gọi backend/MongoDB. Phase 1 dùng Flannel thuần; tài liệu Flannel hướng người dùng cần NetworkPolicy sang một policy engine như Calico.

Vì vậy runbook **không thêm một NetworkPolicy tạo cảm giác an toàn giả**. Nếu yêu cầu isolation east-west nghiêm ngặt, phải thiết kế/migrate CNI hoặc policy engine, verify enforcement bằng test allow/deny rồi mới thêm policy:

```text
Traefik/cloudflared → chỉ frontend
frontend           → chỉ backend:8000 + DNS
backend            → chỉ mongodb:27017 + DNS
mongodb            → không egress ngoài nhu cầu vận hành
```

---

## 16. Troubleshooting

| Triệu chứng | Kiểm tra | Nguyên nhân thường gặp |
| --- | --- | --- |
| PVC `Pending` | `kubectl describe pvc -n three-tier data-mongodb-0` | provisioner lỗi; StorageClass sai; scheduler chưa chọn node |
| MongoDB `CrashLoopBackOff` | `kubectl logs -n three-tier mongodb-0 --previous` | permission volume, init script lỗi, password/Secret thiếu |
| App user auth fail | kiểm tra `authSource=cruddb`, key Secret, app user | dùng nhầm `admin`; password có ký tự chưa URL-encode; PVC cũ khiến init script không chạy lại |
| Backend `/api/health/live` fail | `kubectl logs -l app.kubernetes.io/name=backend --prefix`; port-forward §9.4 | process/event loop lỗi hoặc endpoint path không khớp source |
| Backend live 200 nhưng `/api/health/ready` fail | log backend, MongoDB và kiểm tra URI/Service | MongoDB chưa Ready, auth/URI/DNS sai hoặc DB timeout; không restart backend để chữa lỗi DB |
| Frontend `/` 200 nhưng `/api/health/ready` 502 | log Nginx và EndpointSlice backend | Nginx upstream/path sai; backend không Ready |
| Traefik trả 404 | `kubectl describe ingress -n three-tier three-tier` | Host header/domain/class không khớp |
| Public 502/1033 | log cloudflared | route tunnel sai; trỏ trực tiếp app thay vì Traefik; tunnel unhealthy |
| Pod `ImagePullBackOff` | `kubectl describe pod`; registry permissions | image/tag sai, private registry thiếu imagePullSecret |
| Record mất sau restart | kiểm tra PVC name/PV/node | ghi nhầm filesystem ngoài `/data/db`; PVC bị xóa; storage node bị mất |

Lệnh chẩn đoán nhanh, không in Secret:

```bash
kubectl -n three-tier get all,ingress,pvc -o wide
kubectl -n three-tier get endpointslice
kubectl -n three-tier get events --sort-by=.lastTimestamp | tail -50
kubectl -n three-tier describe pod mongodb-0
kubectl -n three-tier logs mongodb-0 --tail=100
kubectl -n three-tier logs \
  -l app.kubernetes.io/name=backend --tail=100 --prefix
kubectl -n three-tier logs \
  -l app.kubernetes.io/name=frontend --tail=100 --prefix
```

> Đổi giá trị trong Kubernetes Secret **không tự đổi password đã lưu trong MongoDB**. Rotation app user phải đổi password bằng MongoDB user-management command trước, cập nhật URI Secret, rollout backend và verify. Khi rotate root password, phải tái tạo đồng bộ cả `MONGO_INITDB_ROOT_PASSWORD` lẫn `MONGO_TOOLS_CONFIG`; không chỉ `kubectl apply` một key mới.

---

## 17. Checklist hoàn tất

- [ ] Checkpoint 3.1–3.3: cluster, disk, storage, Traefik và pull image PASS.
- [ ] Checkpoint 5.3: source đã được tạo ở task sau, tests/build PASS, không lộ secret.
- [ ] Hai image dùng tag immutable và registry digest đã verify.
- [ ] MongoDB `Running`, PVC 10Gi `Bound`, app user ping được, unique index tồn tại.
- [ ] Backend `2/2`, frontend `2/2`, probes và EndpointSlice PASS.
- [ ] Mọi Service là `ClusterIP`/headless; chỉ frontend xuất hiện trong Ingress.
- [ ] Test qua Traefik: frontend và `/api/health/ready` đều 200.
- [ ] CRUD: create/read/list/update/delete, duplicate và not-found đúng status code.
- [ ] Record sống qua restart `mongodb-0`.
- [ ] Nếu publish: HTTPS public 200 qua Cloudflare Tunnel.
- [ ] Có ít nhất một `mongodump` archive > 0 và đã định vị nơi lưu backup ngoài node.
- [ ] Không có credential trong Git, output chat, frontend bundle hay container logs.

---

## 18. Nguồn official

- Kubernetes — StatefulSets: [https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- Kubernetes — Persistent Volumes: [https://kubernetes.io/docs/concepts/storage/persistent-volumes/](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- Kubernetes — Services: [https://kubernetes.io/docs/concepts/services-networking/service/](https://kubernetes.io/docs/concepts/services-networking/service/)
- Kubernetes — Ingress: [https://kubernetes.io/docs/concepts/services-networking/ingress/](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- Kubernetes — Liveness, readiness và startup probes: [https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- Kubernetes — `kubectl wait`: [https://kubernetes.io/docs/reference/kubectl/generated/kubectl_wait/](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_wait/)
- Kubernetes — `kubectl logs`: [https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/)
- Kubernetes — Pod topology spread constraints: [https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- Kubernetes — Secrets: [https://kubernetes.io/docs/concepts/configuration/secret/](https://kubernetes.io/docs/concepts/configuration/secret/)
- Kubernetes — Good practices for Secrets: [https://kubernetes.io/docs/concepts/security/secrets-good-practices/](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)
- MongoDB — Versioning (major release có lifecycle dự đoán được): [https://www.mongodb.com/docs/v8.0/reference/versioning/](https://www.mongodb.com/docs/v8.0/reference/versioning/)
- MongoDB — Release notes 8.0: [https://www.mongodb.com/docs/v8.0/release-notes/8.0/](https://www.mongodb.com/docs/v8.0/release-notes/8.0/)
- MongoDB — Authentication và RBAC: [https://www.mongodb.com/docs/manual/core/authentication/](https://www.mongodb.com/docs/manual/core/authentication/), [https://www.mongodb.com/docs/manual/core/authorization/](https://www.mongodb.com/docs/manual/core/authorization/)
- MongoDB — Backup/restore tools: [https://www.mongodb.com/docs/manual/tutorial/backup-and-restore-tools/](https://www.mongodb.com/docs/manual/tutorial/backup-and-restore-tools/)
- MongoDB — Database Tools `--config` và cảnh báo password trong `ps`: [https://www.mongodb.com/docs/database-tools/mongodump/](https://www.mongodb.com/docs/database-tools/mongodump/)
- MongoDB — Dùng `process.env` trong mongosh scripts: [https://www.mongodb.com/docs/mongodb-shell/write-scripts/env-variables/](https://www.mongodb.com/docs/mongodb-shell/write-scripts/env-variables/)
- MongoDB — PyMongo Async API / Motor migration: [https://www.mongodb.com/docs/languages/python/pymongo-driver/current/reference/migration/](https://www.mongodb.com/docs/languages/python/pymongo-driver/current/reference/migration/)
- Docker Official Image — MongoDB: [https://hub.docker.com/_/mongo](https://hub.docker.com/_/mongo)
- MongoDB Kubernetes Controllers — topology guidance: [https://www.mongodb.com/docs/kubernetes/current/tutorial/configure-mdb-cr/](https://www.mongodb.com/docs/kubernetes/current/tutorial/configure-mdb-cr/)
- Flannel — NetworkPolicy guidance: [https://github.com/flannel-io/flannel](https://github.com/flannel-io/flannel)
- Traefik — Kubernetes Ingress provider: [https://doc.traefik.io/traefik/reference/install-configuration/providers/kubernetes/kubernetes-ingress/](https://doc.traefik.io/traefik/reference/install-configuration/providers/kubernetes/kubernetes-ingress/)
