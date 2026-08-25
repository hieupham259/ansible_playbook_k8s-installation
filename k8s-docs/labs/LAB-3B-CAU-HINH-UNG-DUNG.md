# Lab 3b — Cấu hình ứng dụng: ConfigMap, Secret và dữ liệu cho Pod

> **Điểm bắt đầu:** snapshot `01-cluster-ready` — xem
> [Lab 00 — Môi trường](LAB-00-MOI-TRUONG-1.35.7.md).
> **Điểm kết thúc:** cleanup trả cluster về đúng `01-cluster-ready`, không tạo snapshot mới.
> **Lab trước:** [Lab 2 — Container, image, CRI và cgroup](LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md)
> đã cleanup namespace `lab-2`. Lab 3a chưa viết (xem [bản đồ lab](README.md#4-bản-đồ-lab));
> Lab 3b vẫn bắt đầu từ `01-cluster-ready` nên không phụ thuộc kết quả của 3a.
>
> **Cập nhật và đối chiếu phiên bản:** 25/08/2026.

Lab này đi cùng mục
[3b. Cấu hình ứng dụng: ConfigMap, Secret và dữ liệu cho Pod](../00-ALO-TRINH-ADMIN.md#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod).
Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); lab này không chép
lại con số phiên bản nào và **không cài thêm gì**.

Nhóm 3b đứng **trước** giai đoạn 4 (Deployment), giai đoạn 5 (Service) và giai đoạn 6 (volume bền
vững). Vì vậy toàn bộ lab chạy trên **Pod trần**: không Deployment, không Service, không PVC,
không StorageClass. `emptyDir` và volume kiểu `configMap`/`secret` thì được dùng, vì chính bài
[108](../108-configmap-vi.md), [109](../109-secret-vi.md) và
[332](../332-define-env-via-file-vi.md) dạy chúng.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

**PASS:** ba node `Ready`; có dòng `PASS: readyz ok`; lệnh field selector trả `No resources found`;
CoreDNS đủ replica `READY`; namespace `default` không có Pod.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi có thể tự chứng minh và giải thích được:

- ConfigMap **không có `spec`**; nó có `data` (chuỗi UTF-8), `binaryData` (base64) và `immutable`,
  và chỉ ra được key nào rơi vào trường nào.
- Trần **1 MiB** của ConfigMap là ràng buộc do API server thực thi, không phải khuyến nghị.
- Ba cách một Pod tiêu thụ ConfigMap mà kubelet nạp **lúc khởi chạy container**: `command`/`args`,
  biến môi trường, file trong volume.
- **Ranh giới cập nhật**: ConfigMap mount làm volume thì được cập nhật; dùng làm biến môi trường
  thì không, phải khởi động lại Pod.
- Mảng `items` quyết định key nào thành file; bỏ hẳn `items` thì mọi key đều thành file.
- `immutable: true` là thay đổi **một chiều**; muốn sửa phải xóa và tạo lại.
- Pod và ConfigMap/Secret phải **cùng namespace**, và chứng minh được điều đó bằng một Pod hỏng.
- Cú pháp `$(VAR)` trong `command`/`args`, thứ tự phụ thuộc trong danh sách `env`, và cách thoát
  bằng `$$(VAR)`.
- `fileKeyRef` nạp biến môi trường từ file trong `emptyDir` do init container ghi ra — và vì sao
  đó **không** phải chỗ để đặt dữ liệu bí mật.
- Ba đường tạo Secret (`kubectl`, file cấu hình, Kustomize), quan hệ giữa `data` và `stringData`,
  và vì sao Kustomize sinh ra object **mới** chứ không sửa object cũ.
- **Secret chỉ được mã hóa base64**, không phải mã hóa thật: giải ngược được mật khẩu và cả token
  registry ngay trên máy, bằng lệnh có sẵn.
- Cách kubelet đưa Secret tới container: chỉ node nào có Pod cần mới nhận, và bản sao nằm trên
  `tmpfs`.
- Cleanup toàn bộ object lab và đưa cluster về `01-cluster-ready`.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm 3b | Phần lab kiểm chứng |
| --- | --- |
| [108 — ConfigMap](../108-configmap-vi.md) | B1 (`data`/`binaryData`, trần 1 MiB), B2 (bốn cách tiêu thụ), B3 (ranh giới cập nhật, `immutable`, namespace) |
| [109 — Secret](../109-secret-vi.md) | B6 (`data` vs `stringData`, `type`), B7 (base64 không phải mã hóa), B8 (volume, env, `optional`, `tmpfs`) |
| [275 — Cấu hình một Pod để sử dụng ConfigMap](../275-configure-pod-configmap-vi.md) | B1 (bốn nguồn dữ liệu), B2 (`configMapKeyRef`, `envFrom`, volume, `items`), B3 (cập nhật tự động) |
| [326 — Quản lý Secret bằng file cấu hình](../326-secret-config-file-vi.md) | B6.2 — `data` base64, `stringData`, và ai thắng khi trùng key |
| [327 — Quản lý Secret bằng kubectl](../327-secret-kubectl-vi.md) | B6.1 (`--from-literal`, `--from-file`), B7.1 (`describe` giấu giá trị, `jsonpath` + `base64 -d`) |
| [328 — Quản lý Secret bằng Kustomize](../328-secret-kustomize-vi.md) | B6.3 — `secretGenerator`, hậu tố hash, sửa dữ liệu sinh object mới |
| [330 — Định nghĩa command và argument cho container](../330-define-command-argument-vi.md) | B4.1 (`command`/`args` ghi đè image), B4.2 (`args` lấy từ biến môi trường) |
| [331 — Định nghĩa biến môi trường cho một Container](../331-define-environment-variable-vi.md) | B2.2 (`envFrom` và `prefix`), B4.3 (`env` ghi đè image, `$(VAR)` trong config) |
| [332 — Định nghĩa giá trị biến môi trường bằng một Init Container](../332-define-env-via-file-vi.md) | B5 — `fileKeyRef`, `emptyDir`, cú pháp file env |
| [333 — Định nghĩa các biến môi trường phụ thuộc](../333-interdependent-env-variables-vi.md) | B4.4 — thứ tự trong `env`, tham chiếu tiến/lùi, `$$(VAR)` |
| [334 — Phân phối thông tin xác thực an toàn bằng Secret](../334-distribute-credentials-secure-vi.md) | B8 — volume + `items` + `defaultMode`, `secretKeyRef`, `envFrom secretRef`, dotfile |
| [287 — Pull image từ một private registry](../287-pull-image-private-registry-vi.md) | B7.3 — tạo và mổ xẻ Secret `kubernetes.io/dockerconfigjson`; phần pull image thật xem bảng dưới |

Bốn phần dưới đây **không kiểm chứng được trên cluster lab ở giai đoạn này**, đọc để biết:

| Bài / phần | Vì sao không thực hành ở đây |
| --- | --- |
| [107 — Cấu hình](../107-configuration-vi.md) | Là **trang mục lục** của cả mục Configuration: chỉ có một câu giới thiệu và danh sách trang con, không có cơ chế nào để kiểm chứng. Phạm vi mà nó khai báo được phủ bởi B1–B8 |
| [325 — Quản lý Secret](../325-configmap-secret-vi.md) | Là **trang mục lục** của ba bài 326/327/328; cả ba bài con đều được thực hành ở B6 |
| [329 — Đưa dữ liệu vào ứng dụng](../329-inject-data-application-vi.md) | Là **trang mục lục** của nhóm inject-data; các bài con thuộc 3b (330–334) đều được thực hành ở B4, B5 và B8. Hai bài con 335/336 (Downward API) thuộc [nhóm 3a](../00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời), không phải nhóm này |
| [287](../287-pull-image-private-registry-vi.md) — phần pull image thật | Cần một private registry và tài khoản thật; cluster lab không có, và dựng registry là hạ tầng ngoài baseline Lab 00. Phần gắn `imagePullSecrets` vào ServiceAccount cần ServiceAccount và RBAC — [giai đoạn 9](../00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy). B7.3 vẫn kiểm chứng được phần cốt lõi: nội dung thật của Secret registry |

### 1.2. Thời lượng

3–4 giờ. B3.1 phải chờ kubelet đồng bộ lại volume, B3.3 và B8.4 cố ý tạo Pod hỏng nên mất thêm
thời gian chờ.

---

## 2. Quy ước và an toàn

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` trừ khi ghi rõ node khác.
- Lệnh cần `sudo` để đọc cấu hình node chạy **trên chính node đó**; lệnh cho worker đi qua SSH.
- **Fault injection chỉ trên `lab-k8s-worker2`** — B3.3 và B8.4 ghim Pod hỏng vào node này bằng
  `nodeName`.
- Lab chỉ tạo hai Namespace: `lab-3b` và `lab-3b-other`. Không tạo Deployment, Service, PVC,
  StorageClass, ServiceAccount hay RBAC.
- Manifest tạm nằm ở `~/lab-work/3b/`; bằng chứng ghi vào `~/lab-evidence/3b/`.
- **Quy ước giá trị bí mật giả.** Lab thao tác với Secret, nên mọi giá trị dùng trong lab đều là
  giá trị **giả**, cố ý vô hại: `demo-user`, `demo-admin`, `demo-password`, `registry.lab.invalid`.
  Nhờ vậy các bước giải mã ở B7 ghi ra `~/lab-evidence/3b/` không chứa bí mật thật.
  **Không bao giờ chạy lại các bước đó với credential thật**, và không dán mật khẩu thật vào bất
  kỳ manifest nào của lab.
- Lab **chỉ đọc** `/etc/kubernetes/manifests/kube-apiserver.yaml` ở B7.2 để chứng minh cluster
  chưa bật mã hóa at rest. **Không sửa file đó.** Bật Encryption at Rest là
  [nợ #6](README.md#5-sổ-nợ-lab), trả ở
  [giai đoạn 22](../00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu).
- Không truy cập etcd trực tiếp bằng `etcdctl` để "xem Secret nằm thế nào"; kết luận của B7 rút ra
  được hoàn toàn từ API và từ chính bài [109](../109-secret-vi.md).
- Image dùng cho toàn bộ lab là image đã ghi ở
  [bảng A1.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) (`busybox` pin theo
  version). Không dùng tag `latest`, không kéo image mới.
- Dòng bắt đầu bằng `PASS:` là gate. Không đi tiếp khi gate fail.

---

# Phần B — Thực hành kiến thức 3b

## B0. Chuẩn bị workspace và namespace

```bash
mkdir -p ~/lab-work/3b/cm-src ~/lab-work/3b/kust ~/lab-evidence/3b
kubectl config current-context
kubectl get nodes -o wide
kubectl create namespace lab-3b
kubectl get namespace lab-3b -o jsonpath='{.status.phase}'; echo
```

Chốt tên image một lần cho cả lab. **Không gõ con số version từ trí nhớ**: mở dòng *Image dùng cho
gate A5.4* trong [bảng A1.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa), chép
đúng tag ở đó vào chỗ giữ chỗ dưới đây rồi chạy:

```bash
# Thay <tag> bang dung tag ghi o dong "Image dung cho gate A5.4" cua bang A1.3.
IMG='busybox:<tag>'
echo "$IMG" | tee ~/lab-evidence/3b/b0-image.txt
```

```bash
case "$IMG" in
  *'<'*|*'>'*) echo 'FAIL: chua dien tag that vao bien IMG' ;;
  *:latest)    echo 'FAIL: bang A1.3 pin theo version, khong dung tag latest' ;;
  busybox:?*)  echo 'PASS: IMG da duoc dien theo bang A1.3' ;;
  *)           echo 'FAIL: IMG khong phai image cua bang A1.3' ;;
esac

ssh lab-k8s-worker1 "sudo crictl images | grep -q '${IMG%%:*}'" \
  && echo 'image da co san tren node — lab khong phai keo image moi' \
  || echo '(image chua co san; Pod dau tien cua lab se keo ve)'
```

**Ý nghĩa:** `lab-work` chứa manifest tạm, `lab-evidence` chứa output. Namespace `lab-3b` cô lập
mọi object có tác động của lab. Toàn bộ manifest phía sau sinh ra bằng heredoc và nhúng `$IMG`, nên
con số version chỉ tồn tại đúng một chỗ trong lab này — và chỗ đó lấy từ bảng A1.3, không phải từ
lab. Biến `$IMG` chỉ sống trong shell hiện tại; mở shell mới giữa chừng thì đặt lại nó trước khi
chạy tiếp, nếu không manifest sinh ra sẽ có dòng `image:` rỗng.

Dòng `crictl images` là kiểm tra mềm, không phải gate: image có thể đã bị dọn khỏi node từ lab
trước, và Pod đầu tiên sẽ kéo lại.

**PASS:** context trỏ đúng cluster lab; ba node `Ready`; namespace `lab-3b` ở phase `Active`;
dòng `PASS: IMG da duoc dien theo bang A1.3` xuất hiện và không có dòng `FAIL:` nào.

## B1. ConfigMap: nguồn dữ liệu, hai trường lưu trữ và trần kích thước

Bài [275](../275-configure-pod-configmap-vi.md) liệt kê ba nguồn cho `kubectl create configmap`
— thư mục, file, giá trị literal — cộng `--from-env-file`. Bốn bước dưới đây chạy hết cả bốn.

### B1.1. Tạo từ một thư mục

```bash
cat > ~/lab-work/3b/cm-src/game.properties <<'EOF'
enemy.types=aliens,monsters
player.maximum-lives=5
EOF

cat > ~/lab-work/3b/cm-src/user-interface.properties <<'EOF'
color.good=purple
color.bad=yellow
allow.textmode=true
EOF

kubectl create configmap game-config -n lab-3b \
  --from-file="$HOME/lab-work/3b/cm-src/"

kubectl get configmap game-config -n lab-3b \
  -o go-template='{{range $k,$v := .data}}{{$k}}{{"\n"}}{{end}}' \
  | tee ~/lab-evidence/3b/b1-game-config-keys.txt
```

```bash
N="$(kubectl get configmap game-config -n lab-3b -o go-template='{{len .data}}')"
echo "so key = $N"
test "$N" -eq 2 && echo 'PASS: moi file trong thu muc thanh mot key'
```

**Ý nghĩa:** khi tạo từ thư mục, kubectl đóng gói **từng file thông thường** thành một key, key
là basename của file, value là nội dung file. Mọi mục không phải file thông thường bị bỏ qua.

### B1.2. Tạo từ một file và đặt lại tên key

```bash
kubectl create configmap game-config-3 -n lab-3b \
  --from-file=game-special-key="$HOME/lab-work/3b/cm-src/game.properties"

KEYS="$(kubectl get configmap game-config-3 -n lab-3b \
  -o go-template='{{range $k,$v := .data}}{{$k}} {{end}}')"
echo "keys = $KEYS"
test "$KEYS" = 'game-special-key ' && echo 'PASS: key duoc dat lai, khong con la ten file'
```

**Ý nghĩa:** cú pháp `--from-file=<key>=<path>` tách tên key khỏi tên file. Đây là cách duy nhất
để đưa một file có tên không hợp lệ vào ConfigMap dưới một key hợp lệ.

### B1.3. Tạo từ giá trị literal

```bash
kubectl create configmap special-config -n lab-3b \
  --from-literal=special.how=very \
  --from-literal=special.type=charm

kubectl get configmap special-config -n lab-3b -o yaml \
  | tee ~/lab-evidence/3b/b1-special-config.yaml

test "$(kubectl get configmap special-config -n lab-3b -o jsonpath='{.data.special\.how}')" = 'very' \
  && echo 'PASS: literal vao dung key special.how'
```

**Ý nghĩa:** key của ConfigMap được phép chứa dấu `.`, `-`, `_`. Ghi nhớ điều này — B2.2 sẽ cho
thấy chính những key đó lại **không** hợp lệ khi biến thành tên biến môi trường.

### B1.4. Tạo từ env-file

```bash
cat > ~/lab-work/3b/game-env-file.properties <<'EOF'
enemies=aliens
lives=3
allowed="true"

# Dong chu thich nay va dong trong o tren deu bi bo qua
EOF

kubectl create configmap game-config-env-file -n lab-3b \
  --from-env-file="$HOME/lab-work/3b/game-env-file.properties"

kubectl get configmap game-config-env-file -n lab-3b -o yaml \
  | tee ~/lab-evidence/3b/b1-env-file.yaml
```

```bash
N="$(kubectl get configmap game-config-env-file -n lab-3b -o go-template='{{len .data}}')"
ALLOWED="$(kubectl get configmap game-config-env-file -n lab-3b -o jsonpath='{.data.allowed}')"
echo "so key = $N"
echo "allowed = $ALLOWED"
test "$N" -eq 3 && echo 'PASS: comment va dong trong bi bo qua'
test "$ALLOWED" = '"true"' && echo 'PASS: dau nhay tro thanh mot phan cua gia tri'
```

**Ý nghĩa:** env-file dùng cú pháp `VAR=VAL`, bỏ qua dòng trống và dòng bắt đầu bằng `#`, và
**không xử lý đặc biệt dấu nháy** — cặp `"` nằm nguyên trong giá trị. Đây là cái bẫy hay gặp khi
bê thẳng file `.env` của ứng dụng vào cluster.

### B1.5. `data` và `binaryData` là hai trường khác nhau

```bash
printf '\xff\xfe\x00\x01lab3b' > ~/lab-work/3b/blob.bin
kubectl create configmap bin-config -n lab-3b \
  --from-file=blob="$HOME/lab-work/3b/blob.bin"

kubectl get configmap bin-config -n lab-3b -o yaml \
  | tee ~/lab-evidence/3b/b1-bin-config.yaml
```

```bash
BIN="$(kubectl get configmap bin-config -n lab-3b -o jsonpath='{.binaryData.blob}')"
DAT="$(kubectl get configmap bin-config -n lab-3b -o jsonpath='{.data}')"
echo "binaryData.blob = $BIN"
test -n "$BIN" && test -z "$DAT" && echo 'PASS: du lieu nhi phan nam o binaryData, data rong'
```

**Ý nghĩa:** byte `0xff` không hợp lệ trong UTF-8, nên kubectl đẩy nội dung sang `binaryData` dưới
dạng chuỗi base64 thay vì `data`. Bài [108](../108-configmap-vi.md) nói rõ: `data` cho chuỗi
UTF-8, `binaryData` cho dữ liệu nhị phân, và **key của hai trường không được trùng nhau**.

> Base64 ở đây chỉ để biểu diễn byte trong JSON/YAML. Nó không giấu được gì — B7 sẽ chứng minh
> bằng chính Secret.

### B1.6. Trần 1 MiB là ràng buộc, không phải khuyến nghị

```bash
head -c 1500000 /dev/zero | tr '\0' 'a' > ~/lab-work/3b/big.txt
wc -c ~/lab-work/3b/big.txt
```

```bash
if kubectl create configmap too-big -n lab-3b \
     --from-file=big="$HOME/lab-work/3b/big.txt" \
     2>~/lab-evidence/3b/b1-too-big-error.txt; then
  echo 'FAIL: API server chap nhan ConfigMap vuot 1 MiB'
else
  echo 'PASS: API server tu choi ConfigMap vuot 1 MiB'
fi

cat ~/lab-evidence/3b/b1-too-big-error.txt

kubectl get configmap too-big -n lab-3b >/dev/null 2>&1 \
  && echo 'FAIL: ConfigMap too-big van ton tai' \
  || echo 'PASS: ConfigMap too-big khong duoc tao'
```

**Ý nghĩa:** file 1.500.000 byte vượt trần 1 MiB (1.048.576 byte). Bài 108 nêu trần này ở mục
*Động lực*; bước trên cho thấy nó do **API server thực thi**, không phải lời khuyên. Cần lưu cấu
hình lớn hơn thì dùng volume hoặc dịch vụ file riêng — cả hai đều thuộc giai đoạn sau.

**PASS:** tám dòng `PASS:` của B1 xuất hiện, không có dòng `FAIL:` nào.

## B2. Bốn cách Pod tiêu thụ ConfigMap

Bài [108](../108-configmap-vi.md) liệt kê bốn cách. Ba cách đầu là thứ kubelet nạp **lúc khởi chạy
container**, và B2 chạy đủ ba. Cách thứ tư — Pod tự gọi Kubernetes API để đọc và subscribe — cần
ServiceAccount và RBAC nên thuộc [giai đoạn 9](../00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy);
B2 không làm, và bảng ánh xạ ở mục 1.1 đã ghi nhận điều đó.

### B2.1. `configMapKeyRef` — chọn từng key, đổi được tên biến

```bash
cat > ~/lab-work/3b/env-single.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: env-single
  namespace: lab-3b
spec:
  restartPolicy: Never
  containers:
  - name: demo
    image: $IMG
    command: ["/bin/sh", "-c", "printenv SPECIAL_LEVEL_KEY"]
    env:
    - name: SPECIAL_LEVEL_KEY
      valueFrom:
        configMapKeyRef:
          name: special-config
          key: special.how
EOF

kubectl apply --dry-run=server --validate=strict -f ~/lab-work/3b/env-single.yaml
kubectl apply -f ~/lab-work/3b/env-single.yaml
kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/env-single -n lab-3b --timeout=180s
```

```bash
OUT="$(kubectl logs env-single -n lab-3b)"
echo "SPECIAL_LEVEL_KEY = $OUT"
test "$OUT" = 'very' && echo 'PASS: bien moi truong lay dung gia tri tu key special.how'
```

**Ý nghĩa:** tên biến môi trường (`SPECIAL_LEVEL_KEY`) hoàn toàn độc lập với tên key
(`special.how`). Chỉ key được nêu tên mới vào container; các key khác của `special-config` không
được sao chép.

### B2.2. `envFrom` — nạp cả ConfigMap, và chuyện key không hợp lệ

```bash
cat > ~/lab-work/3b/app-env-cm.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-env
  namespace: lab-3b
data:
  SPECIAL_LEVEL: very
  SPECIAL_TYPE: charm
  bad.key: bi-bo-qua
EOF

kubectl apply -f ~/lab-work/3b/app-env-cm.yaml

cat > ~/lab-work/3b/env-from.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: env-from
  namespace: lab-3b
spec:
  restartPolicy: Never
  containers:
  - name: demo
    image: $IMG
    command: ["/bin/sh", "-c", "env"]
    envFrom:
    - configMapRef:
        name: app-env
    - prefix: CFG_
      configMapRef:
        name: game-config-env-file
EOF

kubectl apply -f ~/lab-work/3b/env-from.yaml
kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/env-from -n lab-3b --timeout=180s
kubectl logs env-from -n lab-3b | sort | tee ~/lab-evidence/3b/b2-envfrom.txt
```

```bash
kubectl logs env-from -n lab-3b | grep -q '^SPECIAL_LEVEL=very$' \
  && echo 'PASS: key hop le thanh bien moi truong'
kubectl logs env-from -n lab-3b | grep -q '^CFG_lives=3$' \
  && echo 'PASS: prefix duoc them vao truoc ten bien'
BAD="$(kubectl logs env-from -n lab-3b | grep -c 'bad' || true)"
echo "so dong chua 'bad' = $BAD"
test "$BAD" -eq 0 && echo 'PASS: key bad.key bi bo qua, Pod van chay xong'

kubectl describe pod env-from -n lab-3b \
  | tee ~/lab-evidence/3b/b2-envfrom-describe.txt \
  | grep -i 'InvalidEnvironmentVariableNames' || true
```

**Ý nghĩa:** `envFrom` lấy **tất cả** cặp key-value và biến key thành tên biến. Nhưng phạm vi ký
tự cho tên biến môi trường hẹp hơn phạm vi ký tự cho key ConfigMap, nên `bad.key` bị **bỏ qua**
— và bài 275 nói rõ hậu quả: **Pod vẫn được phép khởi động**, chỉ có một event cảnh báo
`InvalidEnvironmentVariableNames` được ghi lại. Đây là lý do một biến "biến mất" mà Pod vẫn
`Running`. Lệnh `grep` cuối cùng có thể không in gì nếu event đã bị dọn; bằng chứng chính là ba
dòng `PASS:` ở trên.

### B2.3. Volume — `items` quyết định key nào thành file

```bash
cat > ~/lab-work/3b/game-demo-cm.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
  namespace: lab-3b
data:
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
EOF

kubectl apply -f ~/lab-work/3b/game-demo-cm.yaml
kubectl get configmap game-demo -n lab-3b -o go-template='{{len .data}}'; echo
```

Pod thứ nhất mount **không** khai `items`:

```bash
cat > ~/lab-work/3b/cfg-vol-all.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: cfg-vol-all
  namespace: lab-3b
spec:
  containers:
  - name: demo
    image: $IMG
    command: ["/bin/sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: config
      mountPath: /config
      readOnly: true
  volumes:
  - name: config
    configMap:
      name: game-demo
EOF

kubectl apply -f ~/lab-work/3b/cfg-vol-all.yaml
kubectl wait --for=condition=Ready pod/cfg-vol-all -n lab-3b --timeout=180s
kubectl exec cfg-vol-all -n lab-3b -- ls -1 /config \
  | tee ~/lab-evidence/3b/b2-vol-all.txt
```

Pod thứ hai mount **có** `items`, và đổi tên một file:

```bash
cat > ~/lab-work/3b/cfg-vol-items.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: cfg-vol-items
  namespace: lab-3b
spec:
  containers:
  - name: demo
    image: $IMG
    command: ["/bin/sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: config
      mountPath: /config
      readOnly: true
  volumes:
  - name: config
    configMap:
      name: game-demo
      items:
      - key: "game.properties"
        path: "game.properties"
      - key: "user-interface.properties"
        path: "ui.conf"
EOF

kubectl apply -f ~/lab-work/3b/cfg-vol-items.yaml
kubectl wait --for=condition=Ready pod/cfg-vol-items -n lab-3b --timeout=180s
kubectl exec cfg-vol-items -n lab-3b -- ls -1 /config \
  | tee ~/lab-evidence/3b/b2-vol-items.txt
```

```bash
ALL="$(kubectl exec cfg-vol-all -n lab-3b -- ls -1 /config | wc -l)"
SEL="$(kubectl exec cfg-vol-items -n lab-3b -- ls -1 /config | wc -l)"
echo "khong items = $ALL file; co items = $SEL file"
test "$ALL" -eq 4 && echo 'PASS: bo han items thi moi key thanh mot file'
test "$SEL" -eq 2 && echo 'PASS: co items thi chi key duoc liet ke moi thanh file'
kubectl exec cfg-vol-items -n lab-3b -- test -f /config/ui.conf \
  && echo 'PASS: path doi duoc ten file, khac ten key'
kubectl exec cfg-vol-items -n lab-3b -- test -f /config/player_initial_lives \
  && echo 'FAIL: key khong nam trong items van xuat hien' \
  || echo 'PASS: key ngoai items khong duoc chieu vao'
```

**Ý nghĩa:** ConfigMap `game-demo` có bốn key. Không khai `items` thì cả bốn thành file trùng tên
key. Khai `items` thì **chỉ** những key được liệt kê mới xuất hiện, và `path` quyết định tên file
trong volume. Đây chính là ví dụ của bài 108: bốn key nhưng container chỉ thấy hai file.

**PASS:** tám dòng `PASS:` của B2 xuất hiện, không có dòng `FAIL:` nào.

## B3. Ranh giới cập nhật, tính bất biến và ranh giới namespace

### B3.1. Volume được cập nhật, biến môi trường thì không

Một Pod duy nhất tiêu thụ **cùng một key** theo hai đường, để so sánh công bằng:

```bash
kubectl create configmap app-config -n lab-3b \
  --from-literal=mode=v1 \
  --from-literal=log_level=info

cat > ~/lab-work/3b/cfg-watch.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: cfg-watch
  namespace: lab-3b
spec:
  containers:
  - name: demo
    image: $IMG
    command: ["/bin/sh", "-c", "sleep 3600"]
    env:
    - name: APP_MODE
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: mode
    volumeMounts:
    - name: config
      mountPath: /config
      readOnly: true
  volumes:
  - name: config
    configMap:
      name: app-config
EOF

kubectl apply -f ~/lab-work/3b/cfg-watch.yaml
kubectl wait --for=condition=Ready pod/cfg-watch -n lab-3b --timeout=180s

echo "file  truoc: $(kubectl exec cfg-watch -n lab-3b -- cat /config/mode)"
echo "env   truoc: $(kubectl exec cfg-watch -n lab-3b -- printenv APP_MODE)"
```

Sửa ConfigMap rồi chờ volume hội tụ:

```bash
kubectl patch configmap app-config -n lab-3b --type=merge \
  -p '{"data":{"mode":"v2"}}'

FILE=''
for i in $(seq 1 36); do
  FILE="$(kubectl exec cfg-watch -n lab-3b -- cat /config/mode 2>/dev/null)"
  test "$FILE" = 'v2' && break
  sleep 5
done
ENV="$(kubectl exec cfg-watch -n lab-3b -- printenv APP_MODE)"

echo "file  sau: $FILE"
echo "env   sau: $ENV"
{ echo "file=$FILE"; echo "env=$ENV"; } | tee ~/lab-evidence/3b/b3-update.txt

test "$FILE" = 'v2' && echo 'PASS: ConfigMap mount lam volume duoc cap nhat'
test "$ENV" = 'v1' && echo 'PASS: bien moi truong KHONG duoc cap nhat'
```

**Ý nghĩa:** bài 108 nói kubelet kiểm tra độ mới của ConfigMap đã mount trong **mỗi chu kỳ đồng
bộ định kỳ**, và lấy giá trị từ **cache cục bộ**. Tổng độ trễ vì thế bằng chu kỳ đồng bộ của
kubelet cộng độ trễ lan truyền cache, và cả hai đều **phụ thuộc cấu hình** — vòng lặp 36 lần ở
trên chỉ là trần của phép chờ, không phải cam kết về thời gian hội tụ. Ngược lại, ConfigMap được
tiêu thụ dưới dạng biến môi trường **không** được cập nhật tự động: muốn container thấy giá trị
mới thì phải khởi động lại Pod. Container mount kiểu `subPath` cũng nằm cùng phía "không được cập
nhật" — `subPath` thuộc bài volume của [giai đoạn 6](../00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ),
lab này không dùng.

Khởi động lại để thấy biến đổi theo:

```bash
kubectl delete pod cfg-watch -n lab-3b --wait=true --timeout=120s
kubectl apply -f ~/lab-work/3b/cfg-watch.yaml
kubectl wait --for=condition=Ready pod/cfg-watch -n lab-3b --timeout=180s

ENV2="$(kubectl exec cfg-watch -n lab-3b -- printenv APP_MODE)"
echo "env sau khi tao lai Pod: $ENV2"
test "$ENV2" = 'v2' && echo 'PASS: bien moi truong chi doi khi Pod duoc tao lai'
```

### B3.2. `immutable: true` là thay đổi một chiều

```bash
kubectl create configmap frozen-config -n lab-3b --from-literal=level=one
kubectl patch configmap frozen-config -n lab-3b --type=merge -p '{"immutable":true}'
kubectl get configmap frozen-config -n lab-3b -o jsonpath='{.immutable}'; echo
```

```bash
if kubectl patch configmap frozen-config -n lab-3b --type=merge \
     -p '{"data":{"level":"two"}}' 2>~/lab-evidence/3b/b3-immutable-data.txt; then
  echo 'FAIL: sua duoc data cua ConfigMap bat bien'
else
  echo 'PASS: khong sua duoc data khi immutable=true'
fi

if kubectl patch configmap frozen-config -n lab-3b --type=merge \
     -p '{"immutable":false}' 2>~/lab-evidence/3b/b3-immutable-revert.txt; then
  echo 'FAIL: go duoc co immutable'
else
  echo 'PASS: khong go duoc co immutable — thay doi mot chieu'
fi

cat ~/lab-evidence/3b/b3-immutable-data.txt ~/lab-evidence/3b/b3-immutable-revert.txt
```

**Ý nghĩa:** một khi đã đánh dấu bất biến thì **không** hoàn tác được và **không** sửa được `data`
hay `binaryData`; đường duy nhất là xóa và tạo lại. Bài 108 kèm một cảnh báo vận hành: Pod hiện có
vẫn giữ mount point trỏ tới ConfigMap đã bị xóa, nên sau khi tạo lại phải tạo lại cả các Pod đó.

### B3.3. Pod và ConfigMap phải cùng namespace

Bước này cố ý tạo một Pod hỏng, nên nó chạy trên `lab-k8s-worker2`.

```bash
kubectl create namespace lab-3b-other

cat > ~/lab-work/3b/cross-ns.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: cross-ns
  namespace: lab-3b-other
spec:
  nodeName: lab-k8s-worker2
  restartPolicy: Never
  containers:
  - name: demo
    image: $IMG
    command: ["/bin/sh", "-c", "printenv APP_MODE"]
    env:
    - name: APP_MODE
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: mode
EOF

kubectl apply -f ~/lab-work/3b/cross-ns.yaml

REASON=''
for i in $(seq 1 24); do
  REASON="$(kubectl get pod cross-ns -n lab-3b-other \
    -o jsonpath='{.status.containerStatuses[0].state.waiting.reason}' 2>/dev/null)"
  test -n "$REASON" && break
  sleep 5
done
echo "waiting.reason = $REASON"
kubectl describe pod cross-ns -n lab-3b-other | tee ~/lab-evidence/3b/b3-cross-ns.txt | tail -20
```

```bash
test "$REASON" = 'CreateContainerConfigError' \
  && echo 'PASS: khong tham chieu duoc ConfigMap o namespace khac'
kubectl describe pod cross-ns -n lab-3b-other | grep -qi 'app-config' \
  && echo 'PASS: event chi dung ten ConfigMap khong tim thay'
```

**Ý nghĩa:** ConfigMap `app-config` tồn tại ở `lab-3b`, còn Pod ở `lab-3b-other`. Tham chiếu
`configMapKeyRef.name` chỉ tìm **trong namespace của chính Pod**, nên kubelet không có gì để nạp
và container không được tạo. Cách duy nhất bài 108 nêu để đọc ConfigMap ở namespace khác là cách
thứ tư — Pod tự gọi Kubernetes API — và khi đó vấn đề chuyển thành phân quyền, thuộc giai đoạn 9.
Cùng logic đó, **static Pod không dùng được ConfigMap lẫn Secret**, vì spec của chúng không tham
chiếu được object API nào; static Pod là bài của
[nhóm 3c](../00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn).

**PASS:** bảy dòng `PASS:` của B3 xuất hiện, không có dòng `FAIL:` nào.

## B4. `command`, `args` và biến môi trường

### B4.1. `command`/`args` ghi đè ENTRYPOINT và CMD của image

```bash
cat > ~/lab-work/3b/command-demo.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: command-demo
  namespace: lab-3b
  labels:
    purpose: demonstrate-command
spec:
  restartPolicy: Never
  containers:
  - name: demo
    image: $IMG
    command: ["printenv"]
    args: ["HOSTNAME", "KUBERNETES_PORT"]
EOF

kubectl apply -f ~/lab-work/3b/command-demo.yaml
kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/command-demo -n lab-3b --timeout=180s
kubectl logs command-demo -n lab-3b | tee ~/lab-evidence/3b/b4-command.txt
```

```bash
kubectl logs command-demo -n lab-3b | sed -n '1p' | grep -q '^command-demo$' \
  && echo 'PASS: HOSTNAME chinh la ten Pod'
kubectl logs command-demo -n lab-3b | sed -n '2p' | grep -q '^tcp://' \
  && echo 'PASS: KUBERNETES_PORT do kubelet dua vao'
```

**Ý nghĩa:** `command` tương ứng `ENTRYPOINT`, `args` tương ứng `CMD`. Khai `command` thì
`ENTRYPOINT` của image bị thay hẳn; chỉ khai `args` thì `ENTRYPOINT` mặc định vẫn chạy với argument
mới. Cả hai **không đổi được sau khi Pod đã tạo**.

### B4.2. Dùng biến môi trường làm argument

```bash
cat > ~/lab-work/3b/args-from-env.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: args-from-env
  namespace: lab-3b
spec:
  restartPolicy: Never
  containers:
  - name: demo
    image: $IMG
    env:
    - name: MESSAGE
      value: "hello lab-3b"
    command: ["/bin/echo"]
    args: ["\$(MESSAGE)"]
EOF

kubectl apply -f ~/lab-work/3b/args-from-env.yaml
kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/args-from-env -n lab-3b --timeout=180s

OUT="$(kubectl logs args-from-env -n lab-3b)"
echo "output = $OUT"
test "$OUT" = 'hello lab-3b' && echo 'PASS: $(VAR) duoc khai trien trong args'
```

> Dấu `\$` trong heredoc ở trên là để **shell của bạn** không nuốt mất `$(MESSAGE)`. Trong file
> YAML sinh ra, giá trị đúng là `$(MESSAGE)` — mở file bằng `cat ~/lab-work/3b/args-from-env.yaml`
> để đối chiếu trước khi đi tiếp.

**Ý nghĩa:** biến môi trường phải nằm trong cặp ngoặc `"$(VAR)"` mới được khai triển trong
`command`/`args`. Vì `$(VAR)` chấp nhận mọi kỹ thuật định nghĩa biến môi trường, bạn có thể lấy
giá trị từ ConfigMap hoặc Secret rồi đưa thẳng vào dòng lệnh.

### B4.3. `env` ghi đè biến môi trường của image

```bash
cat > ~/lab-work/3b/path-default.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: path-default
  namespace: lab-3b
spec:
  restartPolicy: Never
  containers:
  - name: demo
    image: $IMG
    command: ["/bin/sh", "-c", "printenv PATH"]
EOF

cat > ~/lab-work/3b/path-override.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: path-override
  namespace: lab-3b
spec:
  restartPolicy: Never
  containers:
  - name: demo
    image: $IMG
    command: ["/bin/sh", "-c", "printenv PATH"]
    env:
    - name: PATH
      value: "/lab-3b-path:/bin"
EOF

kubectl apply -f ~/lab-work/3b/path-default.yaml -f ~/lab-work/3b/path-override.yaml
kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/path-default -n lab-3b --timeout=180s
kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/path-override -n lab-3b --timeout=180s
```

```bash
P1="$(kubectl logs path-default  -n lab-3b)"
P2="$(kubectl logs path-override -n lab-3b)"
echo "PATH tu image : $P1"
echo "PATH do env   : $P2"
{ echo "image=$P1"; echo "env=$P2"; } | tee ~/lab-evidence/3b/b4-path.txt

test "$P2" = '/lab-3b-path:/bin' && echo 'PASS: env ghi de gia tri cua image'
test "$P1" != "$P2" && echo 'PASS: hai gia tri khac nhau — phep so sanh co nghia'
```

**Ý nghĩa:** bài [331](../331-define-environment-variable-vi.md) ghi rõ biến đặt bằng `env` hoặc
`envFrom` **ghi đè** mọi biến môi trường đã có trong container image. Hai Pod ở trên chỉ khác nhau
đúng khối `env`, nên chênh lệch output là bằng chứng trực tiếp, không phải suy đoán.

### B4.4. Thứ tự trong danh sách `env` và cách thoát `$$(VAR)`

```bash
cat > ~/lab-work/3b/dependent-envars.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: dependent-envars
  namespace: lab-3b
spec:
  restartPolicy: Never
  containers:
  - name: demo
    image: IMAGE_PLACEHOLDER
    command: ["/bin/sh", "-c"]
    args:
    - printf 'UNCHANGED_REFERENCE=%s\nSERVICE_ADDRESS=%s\nESCAPED_REFERENCE=%s\n' "$UNCHANGED_REFERENCE" "$SERVICE_ADDRESS" "$ESCAPED_REFERENCE"
    env:
    - name: SERVICE_PORT
      value: "80"
    - name: SERVICE_IP
      value: "172.17.0.1"
    - name: UNCHANGED_REFERENCE
      value: "$(PROTOCOL)://$(SERVICE_IP):$(SERVICE_PORT)"
    - name: PROTOCOL
      value: "https"
    - name: SERVICE_ADDRESS
      value: "$(PROTOCOL)://$(SERVICE_IP):$(SERVICE_PORT)"
    - name: ESCAPED_REFERENCE
      value: "$$(PROTOCOL)://$(SERVICE_IP):$(SERVICE_PORT)"
EOF

sed -i "s|IMAGE_PLACEHOLDER|$IMG|" ~/lab-work/3b/dependent-envars.yaml
grep -n 'image:' ~/lab-work/3b/dependent-envars.yaml

kubectl apply -f ~/lab-work/3b/dependent-envars.yaml
kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/dependent-envars -n lab-3b --timeout=180s
kubectl logs dependent-envars -n lab-3b | tee ~/lab-evidence/3b/b4-dependent-envars.txt
```

```bash
kubectl logs dependent-envars -n lab-3b \
  | grep -q '^SERVICE_ADDRESS=https://172.17.0.1:80$' \
  && echo 'PASS: tham chieu toi bien da dinh nghia truoc do duoc phan giai'
kubectl logs dependent-envars -n lab-3b \
  | grep -q '^UNCHANGED_REFERENCE=\$(PROTOCOL)://172.17.0.1:80$' \
  && echo 'PASS: tham chieu toi bien nam duoi trong danh sach KHONG duoc phan giai'
kubectl logs dependent-envars -n lab-3b \
  | grep -q '^ESCAPED_REFERENCE=\$(PROTOCOL)://172.17.0.1:80$' \
  && echo 'PASS: $$(VAR) khong bao gio duoc khai trien'
```

> Heredoc ở bước này dùng `<<'EOF'` (có nháy đơn) nên shell **không** đụng vào `$(...)` và `$$(...)`;
> đổi lại tên image phải thay bằng `sed`. Đây là lý do bước trên in lại dòng `image:` để đối chiếu.

**Ý nghĩa:** bài [333](../333-interdependent-env-variables-vi.md) nói thứ tự trong danh sách `env`
là yếu tố quyết định — một biến chỉ được coi là "đã định nghĩa" nếu nó đứng **trước** chỗ tham
chiếu. `PROTOCOL` nằm dưới `UNCHANGED_REFERENCE` nên chuỗi `$(PROTOCOL)` ở đó được giữ nguyên như
văn bản thường, và điều đó **không ngăn container khởi động**. `ESCAPED_REFERENCE` cho kết quả
giống hệt nhưng vì lý do khác hẳn: `$$(VAR)` là cú pháp thoát, không bao giờ khai triển, bất kể
biến có tồn tại hay không.

**PASS:** tám dòng `PASS:` của B4 xuất hiện.

## B5. Biến môi trường lấy từ file trong `emptyDir`

Bài [332](../332-define-env-via-file-vi.md) là tính năng **beta** ở baseline của Lab 00 và được bật
mặc định. Kiểm tra trước khi viết manifest — hỏng ở đây thì dừng, đừng sửa cấu hình kubelet:

```bash
kubectl explain pods.spec.containers.env.valueFrom.fileKeyRef \
  | tee ~/lab-evidence/3b/b5-explain.txt \
  && echo 'PASS: baseline co field fileKeyRef'
```

**STOP:** nếu lệnh trên báo không tìm thấy field, dừng B5 và ghi lại kết quả. Không bật feature
gate, không sửa cấu hình kubelet hay apiserver — làm vậy là lệch baseline Lab 00.

### B5.1. Init container ghi file, container chính đọc bằng `fileKeyRef`

Nội dung file env đặt trong một ConfigMap để tránh mọi rắc rối về dấu nháy khi sinh file:

```bash
cat > ~/lab-work/3b/envfile-cm.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: envfile-source
  namespace: lab-3b
data:
  config.env: |
    # dong nay la chu thich, bi bo qua
    DB_ADDRESS='address'
    REST_ENDPOINT='endpoint'
    HASH_INSIDE='a#b'
EOF

kubectl apply -f ~/lab-work/3b/envfile-cm.yaml

cat > ~/lab-work/3b/envfile-pod.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: envfile-test-pod
  namespace: lab-3b
spec:
  restartPolicy: Never
  initContainers:
  - name: setup-envfile
    image: $IMG
    command: ["/bin/sh", "-c", "cp /source/config.env /data/config.env"]
    volumeMounts:
    - name: source
      mountPath: /source
      readOnly: true
    - name: config
      mountPath: /data
  containers:
  - name: use-envfile
    image: $IMG
    command: ["/bin/sh", "-c", "env"]
    env:
    - name: DB_ADDRESS
      valueFrom:
        fileKeyRef:
          path: config.env
          volumeName: config
          key: DB_ADDRESS
          optional: false
    - name: HASH_INSIDE
      valueFrom:
        fileKeyRef:
          path: config.env
          volumeName: config
          key: HASH_INSIDE
          optional: false
  volumes:
  - name: source
    configMap:
      name: envfile-source
  - name: config
    emptyDir: {}
EOF

kubectl apply --dry-run=server --validate=strict -f ~/lab-work/3b/envfile-pod.yaml
kubectl apply -f ~/lab-work/3b/envfile-pod.yaml
kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/envfile-test-pod -n lab-3b --timeout=180s
kubectl logs envfile-test-pod -n lab-3b -c use-envfile | sort \
  | tee ~/lab-evidence/3b/b5-envfile.txt
```

```bash
kubectl logs envfile-test-pod -n lab-3b -c use-envfile | grep -q '^DB_ADDRESS=address$' \
  && echo 'PASS: bien lay tu file trong emptyDir'
kubectl logs envfile-test-pod -n lab-3b -c use-envfile | grep -q '^HASH_INSIDE=a#b$' \
  && echo 'PASS: dau # trong nhay don khong phai chu thich'
kubectl logs envfile-test-pod -n lab-3b -c use-envfile | grep -q '^REST_ENDPOINT=' \
  && echo 'FAIL: key khong duoc khai trong env lai xuat hien' \
  || echo 'PASS: chi key duoc khai bang fileKeyRef moi thanh bien'
```

**Ý nghĩa:** volume `config` **chỉ được mount vào init container** — container tiêu thụ không mount
nó, mà tham chiếu qua `volumeName` + `path` + `key`. Với `optional: false`, key phải tồn tại trong
file, nếu không container không khởi động được. Cú pháp file env mà Kubernetes chấp nhận là tập
con chặt của bash POSIX: giá trị **bắt buộc** nằm trong nháy đơn, không có nội suy hay nối chuỗi,
và `#` bên trong nháy đơn là ký tự thường chứ không mở chú thích.

> **Ranh giới an toàn phải nhớ.** Bài 332 cảnh báo thẳng: `emptyDir` **không** có các cơ chế bảo vệ
> của object Secret, nên đưa biến môi trường bí mật vào container theo đường này **không** phải
> thực hành bảo mật tốt. Từ B6 trở đi, dữ liệu nhạy cảm đi bằng Secret.

**PASS:** bốn dòng `PASS:` của B5 xuất hiện, không có dòng `FAIL:` nào.

## B6. Ba đường tạo Secret

Từ đây trở đi mọi giá trị đều là giá trị giả theo quy ước ở mục 2.

### B6.1. Bằng `kubectl`

```bash
kubectl create secret generic db-cred -n lab-3b \
  --from-literal=username=demo-user \
  --from-literal=password='demo-password'

echo -n 'demo-user'     > ~/lab-work/3b/username.txt
echo -n 'demo-password' > ~/lab-work/3b/password.txt
kubectl create secret generic db-cred-file -n lab-3b \
  --from-file=username="$HOME/lab-work/3b/username.txt" \
  --from-file=password="$HOME/lab-work/3b/password.txt"

kubectl get secrets -n lab-3b | tee ~/lab-evidence/3b/b6-secret-list.txt
```

```bash
T1="$(kubectl get secret db-cred      -n lab-3b -o jsonpath='{.type}')"
T2="$(kubectl get secret db-cred-file -n lab-3b -o jsonpath='{.type}')"
N1="$(kubectl get secret db-cred      -n lab-3b -o go-template='{{len .data}}')"
echo "type = $T1 / $T2 ; so muc du lieu = $N1"
test "$T1" = 'Opaque' && test "$T2" = 'Opaque' \
  && echo 'PASS: subcommand generic tao Secret loai Opaque'
test "$N1" -eq 2 && echo 'PASS: hai muc du lieu duoc luu'
```

**Ý nghĩa:** `kubectl create secret generic` luôn cho ra loại `Opaque` — loại mặc định, dành cho
dữ liệu tùy ý do người dùng định nghĩa. Cờ `-n` của `echo` là chi tiết bắt buộc: thiếu nó, ký tự
xuống dòng cũng bị mã hóa vào giá trị và mật khẩu sẽ sai một ký tự mà không ai nhìn ra.

### B6.2. Bằng file cấu hình: `data` và `stringData`

```bash
test "$(echo -n 'demo-user' | base64)" = 'ZGVtby11c2Vy' \
  && echo 'PASS: base64 cua demo-user dung nhu manifest sap dung'

cat > ~/lab-work/3b/file-secret.yaml <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: file-secret
  namespace: lab-3b
type: Opaque
data:
  # ZGVtby11c2Vy la base64 cua chuoi demo-user
  username: ZGVtby11c2Vy
stringData:
  username: demo-admin
  config.yaml: |
    apiUrl: "https://api.lab.invalid/v1"
    username: demo-admin
    password: demo-password
EOF

kubectl apply -f ~/lab-work/3b/file-secret.yaml
kubectl get secret file-secret -n lab-3b -o yaml \
  | tee ~/lab-evidence/3b/b6-file-secret.yaml
```

```bash
U="$(kubectl get secret file-secret -n lab-3b -o jsonpath='{.data.username}' | base64 -d)"
echo "username sau khi giai ma = $U"
test "$U" = 'demo-admin' && echo 'PASS: trung key thi stringData thang data'

kubectl get secret file-secret -n lab-3b -o jsonpath='{.stringData}' \
  | tee ~/lab-evidence/3b/b6-stringdata-readback.txt; echo
test -z "$(kubectl get secret file-secret -n lab-3b -o jsonpath='{.stringData}')" \
  && echo 'PASS: stringData khong ton tai khi doc lai — da gop vao data'
```

**Ý nghĩa:** `data` bắt buộc là chuỗi base64; `stringData` nhận chuỗi thuần và được **gộp nội bộ
vào `data`** khi object được tạo. Trùng key thì `stringData` thắng, và đọc lại object thì
`stringData` biến mất — đúng như bài [326](../326-secret-config-file-vi.md) mô tả. Ghi nhớ thêm
một câu bài lặp lại nhiều lần: `stringData` **không hoạt động tốt với server-side apply**.

### B6.3. Bằng Kustomize

```bash
cat > ~/lab-work/3b/kust/kustomization.yaml <<'EOF'
namespace: lab-3b
secretGenerator:
- name: database-creds
  literals:
  - username=demo-admin
  - password=demo-password
EOF

kubectl apply -k ~/lab-work/3b/kust
NAME1="$(kubectl get -k ~/lab-work/3b/kust -o jsonpath='{.metadata.name}')"
echo "ten lan 1 = $NAME1"
```

```bash
sed -i 's/password=demo-password/password=demo-password-2/' ~/lab-work/3b/kust/kustomization.yaml
kubectl apply -k ~/lab-work/3b/kust
NAME2="$(kubectl get -k ~/lab-work/3b/kust -o jsonpath='{.metadata.name}')"
echo "ten lan 2 = $NAME2"

kubectl get secret -n lab-3b -o name | grep 'database-creds-' \
  | tee ~/lab-evidence/3b/b6-kustomize-names.txt
CNT="$(kubectl get secret -n lab-3b -o name | grep -c 'secret/database-creds-')"

case "$NAME1" in
  database-creds-?*) echo 'PASS: ten mang hau to hash cua noi dung' ;;
  *)                 echo "FAIL: ten sinh ra khong dung dang mong doi ($NAME1)" ;;
esac
test "$NAME1" != "$NAME2" && echo 'PASS: doi du lieu thi hash doi theo'
test "$CNT" -eq 2 && echo 'PASS: Kustomize tao object MOI, khong sua object cu'
```

**Ý nghĩa:** `secretGenerator` băm dữ liệu rồi nối giá trị băm vào tên, nên mỗi lần dữ liệu đổi là
một Secret mới ra đời còn Secret cũ **vẫn nằm đó**. Hệ quả vận hành mà bài
[328](../328-secret-kustomize-vi.md) nhắc: các Pod đang trỏ tới tên cũ không tự đổi theo, bạn phải
cập nhật tham chiếu — và phải tự dọn những Secret cũ không còn ai dùng.

**PASS:** tám dòng `PASS:` của B6 xuất hiện.

## B7. Secret chỉ được mã hóa base64

Đây là bằng chứng cốt lõi của cả nhóm bài 3b. Bài [109](../109-secret-vi.md) mở đầu bằng một khối
*Thận trọng*: Secret được lưu **không mã hóa** trong kho dữ liệu nền của API server.

### B7.1. `get` và `describe` giấu giá trị, `jsonpath` thì không

```bash
kubectl describe secret db-cred -n lab-3b | tee ~/lab-evidence/3b/b7-describe.txt
```

```bash
kubectl describe secret db-cred -n lab-3b | grep -q 'bytes' \
  && echo 'PASS: describe chi in do dai, khong in noi dung'
kubectl describe secret db-cred -n lab-3b | grep -q 'demo-password' \
  && echo 'FAIL: describe in ca mat khau' \
  || echo 'PASS: describe khong in mat khau'

kubectl get secret db-cred -n lab-3b -o jsonpath='{.data}' \
  | tee ~/lab-evidence/3b/b7-data-base64.txt; echo

P="$(kubectl get secret db-cred -n lab-3b -o jsonpath='{.data.password}' | base64 -d)"
echo "password sau khi giai ma = $P"
test "$P" = 'demo-password' \
  && echo 'PASS: base64 giai nguoc duoc — day KHONG phai ma hoa'
```

**Ý nghĩa:** `kubectl get` và `kubectl describe` **cố ý** không in nội dung Secret, để giá trị
không lọt vào log terminal một cách vô tình. Nhưng đó là biện pháp của công cụ, không phải của
kho dữ liệu: chỉ một lệnh `jsonpath` cộng `base64 -d` là ra mật khẩu. Ai đọc được object qua API
thì đọc được giá trị thật, và bài 109 nói thêm hai điều nặng hơn: ai đọc được etcd cũng vậy, và
**ai được phép tạo Pod trong namespace đều có thể lợi dụng quyền đó để đọc mọi Secret trong
namespace** — kể cả gián tiếp qua quyền tạo Deployment.

> Giá trị vừa in ra là `demo-password`, một chuỗi giả theo quy ước ở mục 2. Đừng lặp lại bước này
> với credential thật, và đừng đẩy `~/lab-evidence/` lên nơi dùng chung.

### B7.2. Vì sao chỉ có base64 — và nợ #6

Chỉ **đọc** manifest apiserver trên `lab-k8s-master`:

```bash
sudo grep -n -- '--encryption-provider-config' \
  /etc/kubernetes/manifests/kube-apiserver.yaml \
  | tee ~/lab-evidence/3b/b7-encryption-flag.txt || true

ENC="$(sudo grep -c -- '--encryption-provider-config' \
  /etc/kubernetes/manifests/kube-apiserver.yaml || true)"
echo "so dong khai --encryption-provider-config = $ENC"
test "$ENC" = '0' \
  && echo 'PASS: cluster chua bat Encryption at Rest — dung ky vong cua no #6'
```

**STOP:** không sửa file này trong Lab 3b. Bật *Encryption at Rest* phải thay cấu hình
kube-apiserver và khởi động lại control plane, tức là làm lệch baseline của Lab 00. Đó là
[nợ #6](README.md#5-sổ-nợ-lab), trả ở
[giai đoạn 22 — Audit và mã hóa dữ liệu](../00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu).

**Ý nghĩa:** bốn bước tối thiểu mà bài 109 yêu cầu để dùng Secret an toàn là: bật Encryption at
Rest, cấu hình RBAC theo quyền tối thiểu, giới hạn quyền truy cập Secret cho những container cụ
thể, và cân nhắc kho Secret bên ngoài. Ở giai đoạn 3 bạn mới làm được bước thứ ba (B8 sẽ làm);
bước 1 là nợ #6, bước 2 thuộc giai đoạn 9, bước 4 nằm ngoài phạm vi lộ trình.

### B7.3. Secret registry: `auth` chỉ là base64 của `user:password`

```bash
kubectl create secret docker-registry regcred-demo -n lab-3b \
  --docker-server=registry.lab.invalid:5000 \
  --docker-username=demo-user \
  --docker-password='demo-password' \
  --docker-email=demo@lab.invalid

T="$(kubectl get secret regcred-demo -n lab-3b -o jsonpath='{.type}')"
echo "type = $T"
test "$T" = 'kubernetes.io/dockerconfigjson' \
  && echo 'PASS: subcommand docker-registry tao dung loai Secret'

kubectl get secret regcred-demo -n lab-3b \
  -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d \
  | tee ~/lab-evidence/3b/b7-dockerconfigjson.txt; echo
```

```bash
AUTH="$(kubectl get secret regcred-demo -n lab-3b \
  -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d \
  | tr ',' '\n' | grep '"auth":' | cut -d'"' -f4)"
echo "auth (base64) = $AUTH"
DEC="$(echo "$AUTH" | base64 -d)"
echo "auth (giai ma) = $DEC"
echo "auth-decoded=$DEC" | tee ~/lab-evidence/3b/b7-auth-decoded.txt

test "$DEC" = 'demo-user:demo-password' \
  && echo 'PASS: auth chi la base64 cua user:password — bi che mo, khong he bi mat'
```

**Ý nghĩa:** loại `kubernetes.io/dockerconfigjson` lưu nguyên nội dung file `~/.docker/config.json`
dưới dạng base64. Bài [287](../287-pull-image-private-registry-vi.md) và khối *Thận trọng* của bài
109 nói cùng một câu: giá trị `auth` "bị che mờ nhưng không hề bí mật" — bất kỳ ai đọc được Secret
đó đều biết bearer token truy cập registry. Trường `type` chỉ quyết định **validation và quy ước
tên key**, không hề quyết định mức bảo vệ; các loại built-in "chỉ được cung cấp để thuận tiện".

Phần còn lại của bài 287 — pull thật một image riêng tư bằng `imagePullSecrets`, và gắn secret đó
vào ServiceAccount — không kiểm chứng được ở đây; lý do đã ghi ở bảng thứ hai của mục 1.1.

**PASS:** sáu dòng `PASS:` của B7 xuất hiện, không có dòng `FAIL:` nào.

## B8. Pod tiêu thụ Secret

### B8.1. Volume: file, `items`, `defaultMode` và dotfile

Pod này ghim vào `lab-k8s-worker1` để B8.3 biết chỗ mà tìm; đây **không** phải fault injection.

```bash
kubectl create secret generic dotfile-secret -n lab-3b \
  --from-literal=.secret-file=demo-hidden-value

cat > ~/lab-work/3b/secret-vol.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: secret-vol
  namespace: lab-3b
spec:
  nodeName: lab-k8s-worker1
  containers:
  - name: demo
    image: $IMG
    command: ["/bin/sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secret-volume
      readOnly: true
    - name: picked
      mountPath: /etc/picked
      readOnly: true
    - name: dotfiles
      mountPath: /etc/dotfiles
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-cred
      defaultMode: 0400
  - name: picked
    secret:
      secretName: db-cred
      items:
      - key: username
        path: my-group/my-username
  - name: dotfiles
    secret:
      secretName: dotfile-secret
EOF

kubectl apply -f ~/lab-work/3b/secret-vol.yaml
kubectl wait --for=condition=Ready pod/secret-vol -n lab-3b --timeout=180s
kubectl exec secret-vol -n lab-3b -- ls -1 /etc/secret-volume \
  | tee ~/lab-evidence/3b/b8-secret-vol.txt
```

```bash
kubectl exec secret-vol -n lab-3b -- test -f /etc/secret-volume/username \
  && kubectl exec secret-vol -n lab-3b -- test -f /etc/secret-volume/password \
  && echo 'PASS: moi key thanh mot file trong mountPath'

MODE="$(kubectl exec secret-vol -n lab-3b -- stat -L -c %a /etc/secret-volume/password)"
echo "quyen file = $MODE"
test "$MODE" = '400' && echo 'PASS: defaultMode duoc ap dung cho file trong volume'

kubectl exec secret-vol -n lab-3b -- test -f /etc/picked/my-group/my-username \
  && echo 'PASS: items chieu key vao dung duong dan da chi dinh'
kubectl exec secret-vol -n lab-3b -- test -e /etc/picked/password \
  && echo 'FAIL: key ngoai items van xuat hien' \
  || echo 'PASS: key khong nam trong items khong duoc chieu vao'

kubectl exec secret-vol -n lab-3b -- ls    /etc/dotfiles | grep -q 'secret-file' \
  && echo 'FAIL: ls thuong da nhin thay dotfile' \
  || echo 'PASS: ls thuong khong liet ke .secret-file'
kubectl exec secret-vol -n lab-3b -- ls -a /etc/dotfiles | grep -q '^\.secret-file$' \
  && echo 'PASS: ls -a moi thay .secret-file'
```

**Ý nghĩa:** mỗi key trong `data` của Secret thành một file dưới `mountPath`. `defaultMode` đặt bit
quyền POSIX cho toàn bộ volume (mặc định `0644`), `items` chọn key và đổi đường dẫn đích — và khi
đã liệt kê `items` thì **chỉ** key trong danh sách được chiếu vào. Key bắt đầu bằng dấu chấm tạo
ra dotfile: nó không mất đi, chỉ bị ẩn khỏi `ls` thường.

### B8.2. Biến môi trường: `secretKeyRef` và `envFrom secretRef`

```bash
cat > ~/lab-work/3b/secret-env.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: secret-env
  namespace: lab-3b
spec:
  restartPolicy: Never
  containers:
  - name: demo
    image: $IMG
    command: ["/bin/sh", "-c", "env"]
    env:
    - name: SECRET_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-cred
          key: username
    envFrom:
    - prefix: DB_
      secretRef:
        name: db-cred-file
EOF

kubectl apply -f ~/lab-work/3b/secret-env.yaml
kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/secret-env -n lab-3b --timeout=180s
kubectl logs secret-env -n lab-3b | grep -E '^(SECRET_USERNAME|DB_)' | sort \
  | tee ~/lab-evidence/3b/b8-secret-env.txt
```

```bash
kubectl logs secret-env -n lab-3b | grep -q '^SECRET_USERNAME=demo-user$' \
  && echo 'PASS: secretKeyRef lay dung mot key'
kubectl logs secret-env -n lab-3b | grep -q '^DB_username=demo-user$' \
  && echo 'PASS: envFrom secretRef lay ca Secret, co prefix'
```

**Ý nghĩa:** đường env của Secret giống hệt đường env của ConfigMap về cú pháp, và **giống luôn về
giới hạn**: bài 109 nói nếu container đã dùng Secret qua biến môi trường thì một lần cập nhật
Secret sẽ không được container đó nhìn thấy trừ khi nó khởi động lại. Bạn đã chứng minh cơ chế này
ở B3.1 với ConfigMap; kết luận áp dụng nguyên vẹn cho Secret.

Chú ý bằng chứng: file `b8-secret-env.txt` chứa giá trị giả `demo-user` theo quy ước mục 2. Với
credential thật, **không** ghi output `env` ra file.

### B8.3. Secret nằm trên `tmpfs` của node

Chạy trên `lab-k8s-master`, hỏi sang `lab-k8s-worker1` — nơi Pod `secret-vol` đang chạy:

```bash
kubectl get pod secret-vol -n lab-3b -o wide

ssh lab-k8s-worker1 'sudo findmnt -n -o FSTYPE,TARGET' \
  | grep 'kubernetes.io~secret' \
  | tee ~/lab-evidence/3b/b8-tmpfs.txt
```

```bash
TMPFS="$(ssh lab-k8s-worker1 'sudo findmnt -n -o FSTYPE,TARGET' \
  | grep 'kubernetes.io~secret' | grep -c '^tmpfs')"
OTHER="$(ssh lab-k8s-worker1 'sudo findmnt -n -o FSTYPE,TARGET' \
  | grep 'kubernetes.io~secret' | grep -vc '^tmpfs')"
echo "mount tmpfs = $TMPFS ; mount khac tmpfs = $OTHER"
test "$TMPFS" -ge 1 && test "$OTHER" -eq 0 \
  && echo 'PASS: bansao Secret nam tren tmpfs, khong ghi xuong luu tru ben vung'

ssh lab-k8s-worker2 'sudo findmnt -n -o FSTYPE,TARGET' \
  | grep -c 'kubernetes.io~secret' \
  | tee ~/lab-evidence/3b/b8-worker2-secret-mounts.txt
```

**Ý nghĩa:** bài 109 nói một Secret **chỉ được gửi tới node nào có Pod cần nó**, kubelet lưu bản
sao vào `tmpfs` để dữ liệu không rơi xuống bộ nhớ bền vững, và xóa bản sao khi Pod phụ thuộc bị
xóa. Cùng logic đó: một Pod chỉ nhìn thấy Secret mà **chính nó** yêu cầu, Pod khác trên cùng node
không đọc được — trừ đúng một ngoại lệ mà bài nêu là container chạy với `privileged: true`, thứ có
thể truy cập mọi Secret đang dùng trên node đó.

> Số ở `b8-worker2-secret-mounts.txt` chỉ là bằng chứng đối chiếu, không phải gate: `lab-k8s-worker2`
> vẫn có Secret của các Pod hệ thống, nên con số không nhất thiết bằng `0`.

### B8.4. Secret phải tồn tại trước Pod, trừ khi `optional: true`

Hai Pod dưới đây cố ý hỏng, nên chúng chạy trên `lab-k8s-worker2`.

```bash
cat > ~/lab-work/3b/secret-missing.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: secret-missing
  namespace: lab-3b
spec:
  nodeName: lab-k8s-worker2
  restartPolicy: Never
  containers:
  - name: demo
    image: $IMG
    command: ["/bin/sh", "-c", "env"]
    env:
    - name: MISSING
      valueFrom:
        secretKeyRef:
          name: khong-ton-tai
          key: password
EOF

cat > ~/lab-work/3b/secret-optional.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: secret-optional
  namespace: lab-3b
spec:
  nodeName: lab-k8s-worker2
  restartPolicy: Never
  containers:
  - name: demo
    image: $IMG
    command: ["/bin/sh", "-c", "env"]
    env:
    - name: OPT_VALUE
      valueFrom:
        secretKeyRef:
          name: khong-ton-tai
          key: password
          optional: true
EOF

kubectl apply -f ~/lab-work/3b/secret-missing.yaml -f ~/lab-work/3b/secret-optional.yaml

REASON=''
for i in $(seq 1 24); do
  REASON="$(kubectl get pod secret-missing -n lab-3b \
    -o jsonpath='{.status.containerStatuses[0].state.waiting.reason}' 2>/dev/null)"
  test -n "$REASON" && break
  sleep 5
done
echo "secret-missing waiting.reason = $REASON"
kubectl describe pod secret-missing -n lab-3b | tee ~/lab-evidence/3b/b8-missing.txt | tail -20

kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/secret-optional -n lab-3b --timeout=180s
```

```bash
test "$REASON" = 'CreateContainerConfigError' \
  && echo 'PASS: Secret bat buoc khong ton tai thi container khong duoc tao'
kubectl describe pod secret-missing -n lab-3b | grep -qi 'khong-ton-tai' \
  && echo 'PASS: event chi dung ten Secret khong tim thay'

OPT="$(kubectl logs secret-optional -n lab-3b | grep -c '^OPT_VALUE=' || true)"
echo "so dong OPT_VALUE = $OPT"
test "$OPT" -eq 0 && echo 'PASS: Secret optional khong ton tai thi bi bo qua, Pod van chay xong'
```

**Ý nghĩa:** mặc định Secret là **bắt buộc**, và không container nào của Pod được khởi động cho tới
khi mọi Secret không-tùy-chọn sẵn sàng; kubelet định kỳ thử lại và ghi Event kèm chi tiết. Đánh dấu
`optional: true` thì Kubernetes bỏ qua Secret thiếu và Pod chạy bình thường — tiện, nhưng nó biến
một lỗi cấu hình ồn ào thành một biến rỗng im lặng, nên chỉ dùng khi ứng dụng thật sự xử lý được
trường hợp thiếu giá trị.

**PASS:** mười hai dòng `PASS:` của B8 xuất hiện, không có dòng `FAIL:` nào.

## B9. Cleanup và gate cuối

**Mục đích:** xóa mọi object lab tạo ra và chứng minh cluster trở về `01-cluster-ready`.

```bash
kubectl delete namespace lab-3b lab-3b-other --wait=true --timeout=180s

rm -f "$HOME"/lab-work/3b/cm-src/*
rmdir "$HOME/lab-work/3b/cm-src"
rm -f "$HOME"/lab-work/3b/kust/*
rmdir "$HOME/lab-work/3b/kust"
rm -f "$HOME"/lab-work/3b/*
rmdir "$HOME/lab-work/3b"
```

```bash
kubectl get namespace lab-3b >/dev/null 2>&1 \
  && echo 'FAIL: namespace lab-3b van con' \
  || echo 'PASS: namespace lab-3b da xoa'

kubectl get namespace lab-3b-other >/dev/null 2>&1 \
  && echo 'FAIL: namespace lab-3b-other van con' \
  || echo 'PASS: namespace lab-3b-other da xoa'

test ! -e "$HOME/lab-work/3b" && echo 'PASS: manifest tam da xoa'

kubectl get secret,configmap -A --no-headers \
  -o custom-columns='NS:.metadata.namespace,NAME:.metadata.name' \
  | grep -E '^lab-3b' \
  && echo 'FAIL: con object cua lab o namespace lab-3b*' \
  || echo 'PASS: khong con Secret hay ConfigMap nao cua lab'

ENC="$(sudo grep -c -- '--encryption-provider-config' \
  /etc/kubernetes/manifests/kube-apiserver.yaml || true)"
test "$ENC" = '0' && echo 'PASS: manifest apiserver van nguyen — lab khong sua control plane'

kubectl wait --for=condition=Ready node --all --timeout=180s
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

`rmdir` cố ý không dùng `-rf`: nó fail nếu thư mục còn file lạ, và `test ! -e` biến điều đó thành
gate thay vì im lặng bỏ qua. Thư mục `~/lab-evidence/3b/` **giữ lại** — đó là bằng chứng. Nếu bạn
định chia sẻ nó, đọc lại quy ước giá trị bí mật giả ở mục 2 trước.

**PASS:** không có dòng `FAIL:` nào; năm dòng `PASS:` xuất hiện; ba node `Ready`; lệnh field
selector trả `No resources found`; CoreDNS đủ replica; namespace `default` không có Pod. Cluster
trở về `01-cluster-ready`; **không tạo snapshot mới**.

---

## 3. Checkpoint 3b

Không đánh dấu lab hoàn tất chỉ vì command chạy hết. Tự trả lời, không nhìn tài liệu:

- [ ] ConfigMap có `spec` không? Kể ba trường thật của nó và nói key nào rơi vào trường nào.
- [ ] Kể bốn cách một Pod tiêu thụ ConfigMap. Với ba cách đầu, kubelet nạp dữ liệu vào lúc nào?
- [ ] Bạn sửa một key. Pod A mount ConfigMap làm volume, Pod B nạp bằng `envFrom`. Pod nào thấy
      giá trị mới, và phải làm gì với Pod còn lại?
- [ ] Mảng `items` quyết định điều gì? Bỏ hẳn nó thì container thấy bao nhiêu file?
- [ ] `immutable: true` đã đặt rồi, giờ cần đổi một giá trị. Làm thế nào, và phải lưu ý gì với Pod
      đang mount?
- [ ] Vì sao Pod ở namespace khác không mount được ConfigMap của bạn, và vì sao static Pod cũng
      không dùng được ConfigMap lẫn Secret?
- [ ] `data` và `stringData` khác nhau chỗ nào? Trùng key thì cái nào thắng, và đọc lại object thì
      thấy gì?
- [ ] Bạn thấy `password: ZGVtby1wYXNzd29yZA==` trong Secret. Giá trị đó đã được mã hóa chưa? Ai
      đọc được nó? Kể bốn bước tối thiểu để dùng Secret an toàn.
- [ ] Trong lab này bạn mới làm được bước nào trong bốn bước đó? Nêu tên món nợ còn lại, số hiệu
      của nó, và giai đoạn sẽ trả.
- [ ] Trường `type` của Secret quyết định điều gì — và **không** quyết định điều gì?
- [ ] Một Pod trên `lab-k8s-worker1` mount Secret. Dữ liệu nằm ở đâu trên node, Pod khác cùng node
      có đọc được không, và ngoại lệ duy nhất là gì?
- [ ] `$(VAR)` và `$$(VAR)` khác nhau thế nào? Vì sao thứ tự trong danh sách `env` lại quan trọng?

### Bài giải thích cuối cùng

Trong tối đa 10 phút, kể lại luồng sau bằng lời của bạn:

1. Một giá trị cấu hình rời khỏi container image và trở thành object trong cluster — qua
   `--from-literal`, `--from-file`, `--from-env-file` hay một manifest.
2. Pod tuyên bố nó cần giá trị đó theo một trong ba đường: dòng lệnh, biến môi trường, file trong
   volume. Kubelet nạp dữ liệu **lúc khởi chạy container**.
3. Bạn sửa object. Volume hội tụ theo chu kỳ đồng bộ của kubelet; biến môi trường thì không, và
   Pod phải được tạo lại.
4. Dữ liệu nhạy cảm đi đường Secret thay vì ConfigMap, để có `tmpfs`, có phạm vi node hẹp lại và
   có quy ước `type`.
5. Nhưng Secret ở cluster này **mới chỉ được base64**: bạn đã tự tay giải ngược cả mật khẩu lẫn
   token registry.

Đóng lại bằng một câu nói rõ **nợ #6 chưa được trả**: cluster lab chưa bật Encryption at Rest, việc
đó phải sửa cấu hình kube-apiserver nên được để tới
[giai đoạn 22](../00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu); và cho tới khi đó,
mọi kết luận về "Secret an toàn" đều chỉ đúng ở mức RBAC và phạm vi node, không phải ở mức lưu trữ.

Khi toàn bộ checkbox được đánh dấu và không còn nhầm base64 với mã hóa, `data` với `stringData`,
volume với biến môi trường, hay `items` với `envFrom`, Lab 3b hoàn tất.

---

## 4. Troubleshooting của lab này

Sự cố khi dựng môi trường nằm ở [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Bảng dưới chỉ liệt kê sự cố phát sinh từ nội dung bài học 3b.

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| B1.1 đếm ra nhiều hơn 2 key | `ls -a ~/lab-work/3b/cm-src/` | Thư mục có file lạ; xóa file thừa rồi tạo lại ConfigMap. Không sửa gate |
| B1.6 báo lỗi khác "too long" | `cat ~/lab-evidence/3b/b1-too-big-error.txt` | Gate chỉ đòi lệnh **fail** và ConfigMap không tồn tại; đọc thông điệp thật rồi ghi vào evidence, không đoán câu chữ |
| Pod báo lỗi image, hoặc manifest có dòng `image:` trống | `echo "$IMG"`; `grep -n 'image:' ~/lab-work/3b/*.yaml` | `IMG` chưa điền hoặc bạn đã mở shell mới. Đặt lại theo B0 — lấy tag từ bảng A1.3, không gõ từ trí nhớ — rồi **sinh lại** manifest |
| B3.1 chờ hết vòng lặp mà file chưa đổi | `kubectl get cm app-config -n lab-3b -o yaml`; `kubectl describe pod cfg-watch -n lab-3b` | Xác nhận patch đã vào object; nếu có, chạy lại **riêng vòng lặp chờ** — độ trễ phụ thuộc chu kỳ đồng bộ và loại cache của kubelet, không phải lỗi |
| B3.1 biến môi trường cũng đổi theo | `kubectl get pod cfg-watch -n lab-3b -o jsonpath='{.metadata.uid}'` | Bạn đang xem một Pod **mới** được tạo lại. So sánh phải làm trên đúng một Pod, đối chiếu UID |
| B3.3 Pod ở `Pending` thay vì `CreateContainerConfigError` | `kubectl get pod cross-ns -n lab-3b-other -o wide` | `nodeName` sai chính tả nên Pod chưa được gán node; sửa cho khớp tên node thật |
| B4.2 output in ra `$(MESSAGE)` | `cat ~/lab-work/3b/args-from-env.yaml` | Heredoc đã nuốt mất dấu `\$`; sinh lại file và kiểm tra dòng `args` trước khi apply |
| B4.4 cả ba biến đều được phân giải | `grep -n 'PROTOCOL' ~/lab-work/3b/dependent-envars.yaml` | Thứ tự khối `env` bị đảo. Giữ nguyên thứ tự trong lab: `PROTOCOL` phải nằm **dưới** `UNCHANGED_REFERENCE` |
| B5 `fileKeyRef` bị từ chối lúc `--dry-run=server` | `kubectl explain pods.spec.containers.env.valueFrom.fileKeyRef` | Dừng B5 và ghi lại. Không bật feature gate, không sửa cấu hình kubelet/apiserver — đó là lệch baseline |
| B6.3 `kubectl get -k` trả nhiều object | `kubectl kustomize ~/lab-work/3b/kust` | File kustomization có thêm resource ngoài `secretGenerator`; đưa nó về đúng nội dung của B6.3 |
| B8.1 `stat -L` báo không hiểu tùy chọn | `kubectl exec secret-vol -n lab-3b -- stat --help` | Đổi sang `ls -lL /etc/secret-volume/password` và đọc cột quyền; đừng bỏ `-L`, vì file trong secret volume là symlink trỏ vào `..data` |
| B8.3 không có dòng `kubernetes.io~secret` nào | `kubectl get pod secret-vol -n lab-3b -o wide` | Pod không nằm trên `lab-k8s-worker1`; kiểm tra `nodeName` rồi hỏi đúng node đang chạy Pod |
| B9 `rmdir` báo thư mục không rỗng | `ls -a ~/lab-work/3b` | Còn file bạn tự thêm. Xem nội dung, xóa thủ công, rồi chạy lại gate — không dùng `rm -rf` để né gate |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Configuration](https://v1-35.docs.kubernetes.io/docs/concepts/configuration/)
- [Kubernetes v1.35 — ConfigMaps](https://v1-35.docs.kubernetes.io/docs/concepts/configuration/configmap/)
- [Kubernetes v1.35 — Secrets](https://v1-35.docs.kubernetes.io/docs/concepts/configuration/secret/)
- [Kubernetes v1.35 — Configure a Pod to Use a ConfigMap](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
- [Kubernetes v1.35 — Pull an Image from a Private Registry](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)
- [Kubernetes v1.35 — Managing Secrets](https://v1-35.docs.kubernetes.io/docs/tasks/configmap-secret/)
- [Kubernetes v1.35 — Managing Secrets using kubectl](https://v1-35.docs.kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kubectl/)
- [Kubernetes v1.35 — Managing Secrets using Configuration File](https://v1-35.docs.kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-config-file/)
- [Kubernetes v1.35 — Managing Secrets using Kustomize](https://v1-35.docs.kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kustomize/)
- [Kubernetes v1.35 — Inject Data Into Applications](https://v1-35.docs.kubernetes.io/docs/tasks/inject-data-application/)
- [Kubernetes v1.35 — Define a Command and Arguments for a Container](https://v1-35.docs.kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/)
- [Kubernetes v1.35 — Define Environment Variables for a Container](https://v1-35.docs.kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/)
- [Kubernetes v1.35 — Define Environment Variable Values Using An Init Container](https://v1-35.docs.kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-via-file/)
- [Kubernetes v1.35 — Define Dependent Environment Variables](https://v1-35.docs.kubernetes.io/docs/tasks/inject-data-application/define-interdependent-environment-variables/)
- [Kubernetes v1.35 — Distribute Credentials Securely Using Secrets](https://v1-35.docs.kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/)
