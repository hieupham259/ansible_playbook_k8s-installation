# Lab 3a — Pod và vòng đời

> **Điểm bắt đầu:** snapshot `01-cluster-ready` — xem
> [Lab 00 — Môi trường](LAB-00-MOI-TRUONG-1.35.7.md).
> **Điểm kết thúc:** cleanup trả cluster về đúng `01-cluster-ready`, không tạo snapshot mới.
> **Lab trước:** [Lab 2 — Container, image, CRI và cgroup](LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md)
> đã cleanup namespace `lab-2`.
>
> **Cập nhật và đối chiếu phiên bản:** 25/08/2026.

Lab này đi cùng mục [3a. Pod và vòng đời](../00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời) của
[Giai đoạn 3](../00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình). Cluster giữ nguyên baseline
của [bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); lab này không
chép lại con số phiên bản nào và **không cài thêm gì**.

Toàn bộ lab chỉ dùng **Pod trần**. Deployment và ReplicaSet thuộc
[giai đoạn 4](../00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller), Service thuộc
[giai đoạn 5](../00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng), volume bền vững thuộc
[giai đoạn 6](../00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), còn ConfigMap và Secret thuộc nhóm
[3b](../00-ALO-TRINH-ADMIN.md#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod) — không
thứ nào trong số đó xuất hiện ở đây.

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

- Pod trần **không có ai đứng sau nó**: xóa là mất, không có object nào tạo lại; đó chính là lý do
  tồn tại của tài nguyên workload.
- Các container trong một Pod chia sẻ **network namespace và volume**, nên gọi nhau bằng
  `localhost` và **phải tự chia nhau không gian port**; hostname của chúng là tên Pod.
- Pod gần như **bất biến**: chỉ một danh sách ngắn các trường sửa được trên Pod đang chạy.
- Phân biệt **`phase`** với **trường `STATUS` của kubectl**: `CrashLoopBackOff` không phải phase.
- Thứ tự **năm condition vòng đời**, ngoại lệ của `Initialized`, và vì sao `ContainersReady` có thể
  `True` trong khi `Ready` vẫn `False`.
- Hậu quả khác nhau của kết quả `Failure` ở ba loại probe: liveness và startup **giết container**,
  readiness chỉ **cắt Pod khỏi vòng phục vụ**.
- Init container chạy **tuần tự đến khi hoàn thành**, còn sidecar là init container có
  `restartPolicy: Always` — nó **khởi động rồi ở lại**, và bị tắt **sau** container chính.
- Trình tự **kết thúc êm**: `preStop` chạy trước khi tín hiệu dừng tới tiến trình 1, hết ân hạn thì
  `SIGKILL`; và `preStop` **không** được gọi khi container tự hoàn thành.
- Ephemeral container là **hành động do người dùng khởi xướng** để kiểm tra Pod có sẵn: thêm qua
  handler riêng, không sửa, không xóa, không có `resources`, `ports` hay probe.
- `hostUsers: false` ánh xạ root trong container sang một người dùng **không đặc quyền trên host**,
  và cấm mọi namespace khác của host.
- Ranh giới ba nhóm trường của **downward API**: dùng được cả hai cơ chế, chỉ biến môi trường, chỉ
  volume.
- Cleanup toàn bộ object lab và đưa cluster về `01-cluster-ready`.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm 3a | Kiểm chứng ở |
| --- | --- |
| [45 — Workload](../45-workloads-vi.md) | B1.1 — Pod trần không có `ownerReferences` và không ai tạo lại |
| [46 — Pod](../46-pods-vi.md) | B1.2 (chia sẻ mạng và volume), B1.3 (đụng port), B1.4 (tính bất biến) |
| [360 — Giao tiếp giữa các Container bằng Volume dùng chung](../360-containers-shared-volume-vi.md) | B1.2 — hai container đọc chung một file trên `emptyDir` |
| [47 — Vòng đời của Pod](../47-pod-lifecycle-vi.md) | B2.1 (phase), B3 (`restartPolicy`, `CrashLoopBackOff`), B7 (kết thúc êm) |
| [48 — Các Condition của Pod](../48-pod-condition-vi.md) | B2.2 (năm condition), B2.3 (ngoại lệ `Initialized`), B2.4 (`readinessGates`) |
| [49 — Các probe Liveness, Readiness và Startup](../49-probes-vi.md) | B4.1–B4.4 — ba loại probe và ràng buộc `successThreshold` |
| [274 — Cấu hình các probe Liveness, Readiness và Startup](../274-configure-probes-vi.md) | B4.1 (readiness `exec`), B4.2 (liveness `exec`), B4.3 (startup probe) |
| [50 — Container khởi tạo](../50-init-containers-vi.md) | B5.1 (tuần tự), B5.2 (thất bại với `Never`), B5.3 (cấm `readinessProbe`) |
| [276 — Cấu hình khởi tạo Pod](../276-configure-pod-initialization-vi.md) | B5.1 — init container ghi vào volume dùng chung cho app container |
| [51 — Các container sidecar](../51-sidecar-containers-vi.md) | B6.1 (khởi động), B6.2 (thứ tự kết thúc đo bằng timestamp log), B6.3 (bỏ qua `restartPolicy` cấp Pod) |
| [272 — Gắn handler vào các sự kiện vòng đời của Container](../272-attach-handler-lifecycle-event-vi.md) | B7.1 (`preStop` trước TERM), B7.3 (`preStop` không chạy khi container hoàn thành) |
| [52 — Các container tạm thời](../52-ephemeral-containers-vi.md) | B8.2 — `kubectl debug`, tính bất biến và các trường bị cấm |
| [292 — Chia sẻ Process Namespace giữa các Container](../292-share-process-namespace-vi.md) | B8.1 — `shareProcessNamespace`, `/pause`, `/proc/$pid/root` |
| [55 — Không gian tên người dùng](../55-user-namespaces-vi.md) | B9.1 (điều kiện tiên quyết), B9.2 (`uid_map`), B9.3 (hạn chế cứng) |
| [295 — Sử dụng user namespace với Pod](../295-user-namespaces-tasks-vi.md) | B9.2 — `readlink /proc/self/ns/user` và `cat /proc/self/uid_map` |
| [56 — Downward API](../56-downward-api-vi.md) | B10.3 — ranh giới ba nhóm trường, kiểm bằng validation của API server |
| [336 — Expose thông tin Pod qua biến môi trường](../336-env-variable-expose-pod-info-vi.md) | B10.1 — `fieldRef` trong `env` |
| [335 — Expose thông tin Pod qua file](../335-downward-api-volume-vi.md) | B10.2 — volume `downwardAPI` và symlink `..data` |
| [60 — Cấu hình Pod nâng cao](../60-advanced-pod-config-vi.md) | B11.1 (`securityContext` hai cấp), B11.2 (`nodeSelector`), B11.3 (toleration), B11.4 (PriorityClass) |
| [285 — Sử dụng Image Volume với một Pod](../285-image-volumes-vi.md) | B12 — `volumes[].image` và mount chỉ đọc |

Ba bài còn lại của nhóm **không kiểm chứng được trong lab này**, đọc để biết:

| Bài | Vì sao không thực hành ở đây |
| --- | --- |
| [226 — Chạy các thành phần Node dưới người dùng không phải root](../226-kubelet-in-userns-vi.md) | Rootless mode cần feature gate `KubeletInUserNamespace` và dựng lại toàn bộ node bằng kind/minikube/K3s — lệch baseline Lab 00, và lab không cài phần mềm |
| [262 — Cấu hình Pod và Container](../262-configure-pod-container-vi.md) | Là **trang mục lục** của nhánh tác vụ, không có nội dung thực hành riêng; nó chỉ dẫn tới các bài đã nằm trong bảng trên |
| [293 — Tạo static Pod](../293-static-pod-tasks-vi.md) | Khái niệm static Pod là bài [58](../58-static-pods-vi.md), thuộc nhóm [3c](../00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn) — làm ở đây là nhảy cóc; ngoài ra phải ghi file vào `/etc/kubernetes/manifests` của node. **Nợ lab, trả ở Lab 3c** |

Ba phần bị hoãn có chủ đích bên trong các bài đã phủ, cũng là nợ lab:

- **Request/limit hiệu dụng của init container và sidecar** (bài 50 và 51) cần `requests`/`limits`
  của bài [110](../110-manage-resources-containers-vi.md), thuộc nhóm 3c — trả ở Lab 3c.
- **Phase `Unknown`** (bài 47) chỉ xuất hiện khi mất liên lạc với node; việc cắt liên lạc node
  thuộc [giai đoạn 12](../00-ALO-TRINH-ADMIN.md#giai-đoạn-12--quản-trị-cluster-nâng-cao) — trả ở
  Lab 12.
- **Readiness thất bại làm Pod bị gỡ khỏi EndpointSlice** (bài 49) cần Service của giai đoạn 5 —
  trả ở Lab 5a. Ở đây chỉ kiểm chứng tới condition `Ready` của Pod.

### 1.2. Thời lượng

3–4 giờ. B3, B4, B6 và B7 phải chờ kubelet phản ứng nên không rút ngắn được; các mục còn lại chạy
nhanh.

---

## 2. Quy ước và an toàn

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` trừ khi ghi rõ node khác.
- **Fault injection chỉ trên `lab-k8s-worker2`.** Mọi Pod cố ý lỗi trong lab đều ghim vào node này
  bằng `spec.nodeName`, giống cách Lab 2 làm.
- Namespace lab là `lab-3a`. Manifest tạm ghi vào `~/lab-work/3a/`, bằng chứng ghi vào
  `~/lab-evidence/3a/`. Không lưu token, key hay certificate.
- Lab tạo đúng **một object ngoài namespace**: PriorityClass `lab3a-uu-tien-cao` ở B11.4. Nó bị xóa
  ở B13 và gate cuối kiểm lại.
- Lab **không** sửa cấu hình node, không sửa manifest control plane, không cài gói. Lệnh chạy trên
  node qua `ssh` chỉ để **đọc**.
- Lab dùng volume `emptyDir` và `downwardAPI` vì ba bài thực hành của chính nhóm này
  ([276](../276-configure-pod-initialization-vi.md), [335](../335-downward-api-volume-vi.md),
  [360](../360-containers-shared-volume-vi.md)) dựa trên chúng. Không dùng PV, PVC hay
  StorageClass.
- Không gỡ taint của `lab-k8s-master`. B11.3 chỉ **dung thứ** taint đó trong vài phút rồi xóa Pod.
- Lab cần kéo được image `busybox` từ internet ở B0. Nếu môi trường không có internet, xem mục 4.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail.

---

# Phần B — Thực hành kiến thức 3a

## B0. Chuẩn bị workspace, namespace và image

```bash
mkdir -p ~/lab-work/3a ~/lab-evidence/3a
kubectl config current-context
kubectl get nodes -o wide
kubectl create namespace lab-3a
kubectl get namespace lab-3a -o jsonpath='{.status.phase}'; echo
```

Kéo sẵn image dùng cho cả lab về hai worker để các bước sau không lẫn thời gian tải image vào thời
gian phản ứng của kubelet:

```bash
for N in lab-k8s-worker1 lab-k8s-worker2; do
  ssh "$N" 'sudo crictl pull docker.io/library/busybox:1.37' && echo "$N: da co image"
done
```

**Ý nghĩa:** `lab-work` chứa manifest tạm, `lab-evidence` chứa output. Namespace `lab-3a` cô lập
mọi object lab tạo ra. Toàn bộ lab dùng đúng một image, ghim theo tag chứ không dùng `latest` —
lý do đã học ở [Lab 2 B3](LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md).

**PASS:** context trỏ đúng cluster lab; ba node `Ready`; namespace `lab-3a` ở phase `Active`; cả
hai worker in ra dòng `da co image`.

## B1. Pod là gì và Pod chia sẻ những gì

### B1.1. Pod trần không có ai đứng sau nó

Bài [45](../45-workloads-vi.md) nói lỗi node là **chung cuộc** với Pod trên node đó, và đó là lý do
bạn dùng tài nguyên workload thay vì tự quản Pod. Bước này chứng minh nửa đầu của lập luận: Pod
trần không có chủ.

```bash
cat > ~/lab-work/3a/solo.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: solo
  namespace: lab-3a
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
EOF

kubectl apply -f ~/lab-work/3a/solo.yaml
kubectl wait --for=condition=Ready pod/solo -n lab-3a --timeout=180s

kubectl get pod solo -n lab-3a \
  -o jsonpath='{.metadata.uid}{"\n"}{.metadata.ownerReferences}{"\n"}' \
  | tee ~/lab-evidence/3a/b1-solo-owner.txt
```

```bash
OWNER="$(kubectl get pod solo -n lab-3a -o jsonpath='{.metadata.ownerReferences}')"
test -z "$OWNER" && echo 'PASS: Pod tran khong co ownerReferences'

kubectl delete pod solo -n lab-3a --wait=true
for i in $(seq 1 4); do sleep 5; done
if kubectl get pod solo -n lab-3a >/dev/null 2>&1; then
  echo 'FAIL: co thu gi do tao lai Pod solo'
else
  echo 'PASS: khong ai tao lai Pod solo'
fi
```

**Ý nghĩa:** không có `ownerReferences` nghĩa là không controller nào coi Pod này là bản sao của
mình. Bài 45 nói tài nguyên workload **cấu hình controller giữ đúng số lượng Pod đúng loại** —
điều bạn vừa chứng minh là khi không có controller đó thì "tạo Pod" chỉ là một thao tác một lần.
Bài [47](../47-pod-lifecycle-vi.md) bổ sung: kể cả khi bạn tạo lại Pod cùng tên, nó là Pod **khác**
vì `.metadata.uid` khác — hãy giữ file evidence để đối chiếu.

**PASS:** hai dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

### B1.2. Hai container trong một Pod: chung mạng, chung volume

```bash
cat > ~/lab-work/3a/duo.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: duo
  namespace: lab-3a
spec:
  volumes:
  - name: shared-data
    emptyDir: {}
  containers:
  - name: web
    image: busybox:1.37
    command: ["sh", "-c", "echo xin-chao-tu-web > /data/index.html; httpd -f -p 8080 -h /data"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  - name: client
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
EOF

kubectl apply -f ~/lab-work/3a/duo.yaml
kubectl wait --for=condition=Ready pod/duo -n lab-3a --timeout=180s
```

```bash
OUT="$(kubectl exec duo -n lab-3a -c client -- wget -qO- http://127.0.0.1:8080/)"
echo "client goi localhost:8080 -> $OUT"
test "$OUT" = 'xin-chao-tu-web' && echo 'PASS: hai container noi chuyen qua localhost'

FILE="$(kubectl exec duo -n lab-3a -c client -- cat /pod-data/index.html)"
test "$FILE" = "$OUT" && echo 'PASS: hai container doc chung mot file tren volume dung chung'

H1="$(kubectl exec duo -n lab-3a -c web    -- hostname)"
H2="$(kubectl exec duo -n lab-3a -c client -- hostname)"
echo "hostname: web=$H1 client=$H2"
{ test "$H1" = 'duo' && test "$H2" = 'duo'; } && echo 'PASS: hostname cua ca hai container la ten Pod'

IPAPI="$(kubectl get pod duo -n lab-3a -o jsonpath='{.status.podIP}')"
IPIN="$(kubectl exec duo -n lab-3a -c client -- hostname -i | awk '{print $1}')"
echo "podIP theo API = $IPAPI ; theo container = $IPIN"
test "$IPAPI" = "$IPIN" && echo 'PASS: hai container dung chung mot dia chi IP'

kubectl get pod duo -n lab-3a -o wide | tee ~/lab-evidence/3a/b1-duo.txt
```

**Ý nghĩa:** đây là toàn bộ mô hình Pod của bài [46](../46-pods-vi.md) gói trong bốn gate. Mọi
container trong Pod **chung network namespace** nên `localhost` dùng được **và chỉ dùng được** bên
trong Pod; chúng cũng chung volume, đúng mẫu mà bài
[360](../360-containers-shared-volume-vi.md) mô tả: một container ghi file, container kia phục vụ
file đó. Hostname là **tên Pod**, không phải tên container.

**PASS:** đủ bốn dòng `PASS:`.

### B1.3. Chung IP nghĩa là chung không gian port

Fault injection, ghim vào `lab-k8s-worker2`.

```bash
cat > ~/lab-work/3a/port-clash.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: port-clash
  namespace: lab-3a
spec:
  nodeName: lab-k8s-worker2
  restartPolicy: Never
  containers:
  - name: web1
    image: busybox:1.37
    command: ["sh", "-c", "httpd -f -p 8080 -h /tmp"]
  - name: web2
    image: busybox:1.37
    command: ["sh", "-c", "sleep 5; httpd -f -p 8080 -h /tmp"]
EOF

kubectl apply -f ~/lab-work/3a/port-clash.yaml

for i in $(seq 1 24); do
  EC="$(kubectl get pod port-clash -n lab-3a \
        -o jsonpath='{.status.containerStatuses[?(@.name=="web2")].state.terminated.exitCode}' 2>/dev/null)"
  test -n "$EC" && break
  sleep 5
done
echo "exitCode cua web2 = $EC"
kubectl logs port-clash -n lab-3a -c web2 | tee ~/lab-evidence/3a/b1-port-clash.txt
```

```bash
EC="$(kubectl get pod port-clash -n lab-3a \
      -o jsonpath='{.status.containerStatuses[?(@.name=="web2")].state.terminated.exitCode}')"
{ test -n "$EC" && test "$EC" -ne 0; } && echo 'PASS: container thu hai khong bind duoc port 8080'

RUN="$(kubectl get pod port-clash -n lab-3a \
      -o jsonpath='{.status.containerStatuses[?(@.name=="web1")].state.running.startedAt}')"
test -n "$RUN" && echo 'PASS: container thu nhat van chay binh thuong'

kubectl delete pod port-clash -n lab-3a --wait=true
```

**Ý nghĩa:** bài 46 nói các container trong một Pod **phải phối hợp cách dùng tài nguyên mạng dùng
chung, chẳng hạn port**. Đây là mặt trái của việc chung network namespace: hai container không thể
cùng nghe một port, và lỗi không nằm ở Kubernetes mà nằm ở `bind()` của kernel. Với
`restartPolicy: Never`, container hỏng nằm im ở trạng thái `Terminated` trong khi Pod vẫn `Running`
nhờ container còn lại.

**PASS:** hai dòng `PASS:` xuất hiện.

### B1.4. Pod gần như bất biến

Kiểm bằng **server-side dry run**: API server chạy đủ validation và admission nhưng không ghi gì.

```bash
kubectl patch pod duo -n lab-3a --dry-run=server \
  -p '{"spec":{"containers":[{"name":"web","image":"busybox:1.36"}]}}' \
  -o jsonpath='{.spec.containers[?(@.name=="web")].image}{"\n"}'

kubectl patch pod duo -n lab-3a --dry-run=server \
  -p '{"spec":{"nodeSelector":{"disktype":"ssd"}}}' \
  2>&1 | tee ~/lab-evidence/3a/b1-immutable.txt
```

```bash
if kubectl patch pod duo -n lab-3a --dry-run=server \
     -p '{"spec":{"containers":[{"name":"web","image":"busybox:1.36"}]}}' >/dev/null 2>&1; then
  echo 'PASS: doi spec.containers[*].image duoc chap nhan'
else
  echo 'FAIL: khong doi duoc image'
fi

if kubectl patch pod duo -n lab-3a --dry-run=server \
     -p '{"spec":{"nodeSelector":{"disktype":"ssd"}}}' >/dev/null 2>&1; then
  echo 'FAIL: API server chap nhan them nodeSelector cho Pod dang chay'
else
  echo 'PASS: API server tu choi doi nodeSelector'
fi
```

**Ý nghĩa:** bài 46 liệt kê đúng sáu trường mà một cập nhật Pod thông thường được phép đổi, trong
đó có `spec.containers[*].image` nhưng **không** có `nodeSelector`. Đây là nền tảng của cả giai
đoạn 4: muốn đổi thứ ngoài danh sách đó thì phải **thay Pod**, và việc thay Pod là việc của
controller thông qua pod template.

**PASS:** hai dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

## B2. Phase và condition

### B2.1. Bốn phase quan sát được

```bash
cat > ~/lab-work/3a/phase-pending.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: phase-pending
  namespace: lab-3a
spec:
  nodeSelector:
    kubernetes.io/hostname: khong-co-node-nay
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
EOF

cat > ~/lab-work/3a/phase-succeeded.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: phase-succeeded
  namespace: lab-3a
spec:
  restartPolicy: Never
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "echo xong; exit 0"]
EOF

cat > ~/lab-work/3a/phase-failed.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: phase-failed
  namespace: lab-3a
spec:
  nodeName: lab-k8s-worker2
  restartPolicy: Never
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "echo that-bai; exit 1"]
EOF

kubectl apply -f ~/lab-work/3a/phase-pending.yaml
kubectl apply -f ~/lab-work/3a/phase-succeeded.yaml
kubectl apply -f ~/lab-work/3a/phase-failed.yaml

kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/phase-succeeded -n lab-3a --timeout=180s
kubectl wait --for=jsonpath='{.status.phase}'=Failed    pod/phase-failed    -n lab-3a --timeout=180s
```

```bash
for i in $(seq 1 12); do
  R="$(kubectl get pod phase-pending -n lab-3a \
       -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}')"
  test -n "$R" && break
  sleep 5
done

for P in phase-pending phase-succeeded phase-failed duo; do
  echo "$P -> $(kubectl get pod "$P" -n lab-3a -o jsonpath='{.status.phase}')"
done | tee ~/lab-evidence/3a/b2-phase.txt

test "$(kubectl get pod phase-pending   -n lab-3a -o jsonpath='{.status.phase}')" = 'Pending'   && echo 'PASS: phase Pending'
test "$(kubectl get pod duo             -n lab-3a -o jsonpath='{.status.phase}')" = 'Running'   && echo 'PASS: phase Running'
test "$(kubectl get pod phase-succeeded -n lab-3a -o jsonpath='{.status.phase}')" = 'Succeeded' && echo 'PASS: phase Succeeded'
test "$(kubectl get pod phase-failed    -n lab-3a -o jsonpath='{.status.phase}')" = 'Failed'    && echo 'PASS: phase Failed'

R="$(kubectl get pod phase-pending -n lab-3a \
     -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}')"
echo "PodScheduled reason = $R"
test "$R" = 'Unschedulable' && echo 'PASS: Pending vi khong lap lich duoc'
```

**Ý nghĩa:** bốn phase này là bốn giá trị bạn gặp hằng ngày. Phase thứ năm, `Unknown`, chỉ xuất
hiện khi **không lấy được trạng thái Pod vì lỗi giao tiếp với node** — muốn dựng lại tình huống đó
phải cắt liên lạc một node, việc thuộc giai đoạn 12, nên lab này không tạo nó (xem ghi chú hoãn ở
mục 1.1).

Chú ý `phase-pending`: Pod **đã được cluster chấp nhận** nhưng chưa được lập lịch, và `reason` của
condition `PodScheduled` mới là chỗ nói vì sao — đúng luận điểm của bài
[48](../48-pod-condition-vi.md) rằng một giá trị phase đơn lẻ không đủ để chẩn đoán.

**PASS:** đủ năm dòng `PASS:`.

### B2.2. Năm condition vòng đời

```bash
kubectl get pod duo -n lab-3a -o jsonpath='{range .status.conditions[*]}{.type}{"\t"}{.status}{"\t"}{.lastTransitionTime}{"\n"}{end}' \
  | tee ~/lab-evidence/3a/b2-conditions.txt
```

```bash
BAD=0
for T in PodScheduled PodReadyToStartContainers Initialized ContainersReady Ready; do
  S="$(kubectl get pod duo -n lab-3a -o jsonpath="{.status.conditions[?(@.type==\"$T\")].status}")"
  echo "$T = $S"
  test "$S" = 'True' || { echo "FAIL: $T khong phai True"; BAD=1; }
done
test "$BAD" -eq 0 && echo 'PASS: du nam condition vong doi va deu True'
```

**Ý nghĩa:** kubelet đặt cả năm, nhưng người **làm** việc phía sau `PodReadyToStartContainers` là
container runtime và CNI plugin — trên cluster lab là containerd và Flannel. Chỉ sau khi condition
này `True`, kubelet mới kéo image và tạo container.

**PASS:** dòng `PASS: du nam condition vong doi va deu True` xuất hiện.

### B2.3. Ngoại lệ của `Initialized`

Bài 48 nói thứ tự năm condition chỉ đúng **đại thể**: Pod **không có** init container thì
`Initialized` được đặt `True` **trước** khi tạo sandbox. So sánh hai Pod.

```bash
cat > ~/lab-work/3a/init-order.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: init-order
  namespace: lab-3a
spec:
  initContainers:
  - name: cho-mot-chut
    image: busybox:1.37
    command: ["sh", "-c", "sleep 20"]
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
EOF

kubectl apply -f ~/lab-work/3a/init-order.yaml
kubectl wait --for=condition=Ready pod/init-order -n lab-3a --timeout=240s
```

```bash
ct() { kubectl get pod "$1" -n lab-3a \
        -o jsonpath="{.status.conditions[?(@.type==\"$2\")].lastTransitionTime}"; }

A="$(ct duo Initialized)";        B="$(ct duo PodReadyToStartContainers)"
C="$(ct init-order Initialized)"; D="$(ct init-order PodReadyToStartContainers)"
echo "duo        : Initialized=$A PodReadyToStartContainers=$B"
echo "init-order : Initialized=$C PodReadyToStartContainers=$D"

test "$(date -d "$A" +%s)" -le "$(date -d "$B" +%s)" \
  && echo 'PASS: Pod khong co init container — Initialized khong sau PodReadyToStartContainers'
test "$(date -d "$C" +%s)" -gt "$(date -d "$D" +%s)" \
  && echo 'PASS: Pod co init container — Initialized sau khi sandbox va mang da dung xong'
```

**Ý nghĩa:** với Pod có init container, `Initialized` chỉ `True` **sau khi các init container hoàn
tất**, mà việc đó lại xảy ra sau khi runtime dựng sandbox và cấu hình mạng. Đây là chỗ trực giác
hay sai vì danh sách thứ tự trong bài 48 xếp `PodReadyToStartContainers` trước `Initialized`.

**PASS:** hai dòng `PASS:` xuất hiện.

### B2.4. `ContainersReady` là `True` mà `Ready` vẫn `False`

```bash
cat > ~/lab-work/3a/gate-demo.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: gate-demo
  namespace: lab-3a
spec:
  readinessGates:
  - conditionType: "training.example.com/ready"
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
EOF

kubectl apply -f ~/lab-work/3a/gate-demo.yaml

for i in $(seq 1 24); do
  S="$(kubectl get pod gate-demo -n lab-3a \
       -o jsonpath='{.status.conditions[?(@.type=="ContainersReady")].status}')"
  test "$S" = 'True' && break
  sleep 5
done

kubectl get pod gate-demo -n lab-3a -o jsonpath='{range .status.conditions[*]}{.type}{"\t"}{.status}{"\t"}{.reason}{"\n"}{end}' \
  | tee ~/lab-evidence/3a/b2-readiness-gate.txt
```

```bash
CR="$(kubectl get pod gate-demo -n lab-3a -o jsonpath='{.status.conditions[?(@.type=="ContainersReady")].status}')"
RS="$(kubectl get pod gate-demo -n lab-3a -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')"
RR="$(kubectl get pod gate-demo -n lab-3a -o jsonpath='{.status.conditions[?(@.type=="Ready")].reason}')"
echo "ContainersReady=$CR Ready=$RS reason=$RR"
{ test "$CR" = 'True' && test "$RS" = 'False' && test "$RR" = 'ReadinessGatesNotReady'; } \
  && echo 'PASS: container san sang nhung readinessGate chua dat'
```

Đặt condition tùy chỉnh thành `True` bằng `PATCH` lên subresource `status`, đúng cách bài 48 chỉ:

```bash
kubectl patch pod gate-demo -n lab-3a --subresource=status \
  -p '{"status":{"conditions":[{"type":"training.example.com/ready","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-08-25T00:00:00Z"}]}}'

for i in $(seq 1 12); do
  RS="$(kubectl get pod gate-demo -n lab-3a -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')"
  test "$RS" = 'True' && break
  sleep 5
done
test "$RS" = 'True' && echo 'PASS: readinessGate True -> Pod Ready'
```

**Ý nghĩa:** Pod chỉ được coi là sẵn sàng khi **cả hai** điều cùng đúng — mọi container Ready
**và** mọi condition trong `readinessGates` là `True`. Khi chỉ vế đầu đúng, kubelet ghi thẳng lý do
vào `reason`. Trong hệ thống thật, controller của ứng dụng mới là bên đặt condition đó; ở đây bạn
đóng vai controller bằng một lệnh `patch`.

**PASS:** hai dòng `PASS:` xuất hiện.

## B3. `restartPolicy` và `CrashLoopBackOff`

Fault injection, cả ba Pod ghim vào `lab-k8s-worker2`.

```bash
cat > ~/lab-work/3a/restart-never.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: restart-never
  namespace: lab-3a
spec:
  nodeName: lab-k8s-worker2
  restartPolicy: Never
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "exit 1"]
EOF

cat > ~/lab-work/3a/restart-onfailure.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: restart-onfailure
  namespace: lab-3a
spec:
  nodeName: lab-k8s-worker2
  restartPolicy: OnFailure
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "exit 0"]
EOF

cat > ~/lab-work/3a/restart-always.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: restart-always
  namespace: lab-3a
spec:
  nodeName: lab-k8s-worker2
  restartPolicy: Always
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "exit 0"]
EOF

kubectl apply -f ~/lab-work/3a/restart-never.yaml
kubectl apply -f ~/lab-work/3a/restart-onfailure.yaml
kubectl apply -f ~/lab-work/3a/restart-always.yaml

kubectl wait --for=jsonpath='{.status.phase}'=Failed    pod/restart-never     -n lab-3a --timeout=180s
kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/restart-onfailure -n lab-3a --timeout=180s

for i in $(seq 1 30); do
  W="$(kubectl get pod restart-always -n lab-3a \
       -o jsonpath='{.status.containerStatuses[0].state.waiting.reason}' 2>/dev/null)"
  test "$W" = 'CrashLoopBackOff' && break
  sleep 5
done

kubectl get pods -n lab-3a -o wide | tee ~/lab-evidence/3a/b3-restart.txt
kubectl describe pod restart-always -n lab-3a | tee ~/lab-evidence/3a/b3-describe.txt
```

```bash
rc() { kubectl get pod "$1" -n lab-3a -o jsonpath='{.status.containerStatuses[0].restartCount}'; }

echo "restart-never     phase=$(kubectl get pod restart-never     -n lab-3a -o jsonpath='{.status.phase}') restarts=$(rc restart-never)"
echo "restart-onfailure phase=$(kubectl get pod restart-onfailure -n lab-3a -o jsonpath='{.status.phase}') restarts=$(rc restart-onfailure)"
echo "restart-always    phase=$(kubectl get pod restart-always    -n lab-3a -o jsonpath='{.status.phase}') restarts=$(rc restart-always)"

test "$(rc restart-never)" -eq 0     && echo 'PASS: Never — thoat loi van khong khoi dong lai'
test "$(rc restart-onfailure)" -eq 0 && echo 'PASS: OnFailure — thoat 0 la thanh cong, khong khoi dong lai'

W="$(kubectl get pod restart-always -n lab-3a -o jsonpath='{.status.containerStatuses[0].state.waiting.reason}')"
PH="$(kubectl get pod restart-always -n lab-3a -o jsonpath='{.status.phase}')"
echo "restart-always: waiting.reason=$W phase=$PH"
test "$W" = 'CrashLoopBackOff' && echo 'PASS: STATUS hien CrashLoopBackOff'
test "$PH" = 'Running'         && echo 'PASS: nhung phase van la Running'
test "$(rc restart-always)" -ge 1 && echo 'PASS: bo dem RESTARTS da tang'

kubectl delete pod restart-never restart-onfailure restart-always -n lab-3a --wait=true
```

**Ý nghĩa:** đây là chỗ dễ nhầm nhất của bài [47](../47-pod-lifecycle-vi.md).
`CrashLoopBackOff` **không phải một phase** — nó là giá trị của **trường Status hiển thị của
kubectl**. Pod đã gắn với node và container đang trong vòng khởi động lại, nên phase đúng là
`Running`. Cơ chế đứng sau là backoff theo hàm mũ giữa các lần thử; bộ đếm chỉ được đặt lại sau khi
container chạy êm một khoảng, và cả hai con số đó **phụ thuộc cấu hình kubelet** nên đừng học
thuộc chúng như một cam kết.

Bảng so sánh trong bài 47 vừa được kiểm chứng ở hai ô: `Never` không khởi động lại dù mã thoát khác
0, `OnFailure` không khởi động lại khi mã thoát 0. Ô còn lại — sidecar **luôn** khởi động lại, kể
cả khi Pod đặt `Never` — được kiểm ở B6.3.

**PASS:** năm dòng `PASS:` xuất hiện.

## B4. Ba loại probe

### B4.1. Readiness thất bại: container vẫn chạy

```bash
cat > ~/lab-work/3a/probe-readiness.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: probe-readiness
  namespace: lab-3a
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    readinessProbe:
      exec:
        command: ["cat", "/tmp/san-sang"]
      periodSeconds: 5
EOF

kubectl apply -f ~/lab-work/3a/probe-readiness.yaml

for i in $(seq 1 24); do
  R="$(kubectl get pod probe-readiness -n lab-3a \
       -o jsonpath='{.status.containerStatuses[0].state.running.startedAt}')"
  test -n "$R" && break
  sleep 5
done
sleep 15
kubectl get pod probe-readiness -n lab-3a | tee ~/lab-evidence/3a/b4-readiness.txt
```

```bash
PH="$(kubectl get pod probe-readiness -n lab-3a -o jsonpath='{.status.phase}')"
RD="$(kubectl get pod probe-readiness -n lab-3a -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')"
RC="$(kubectl get pod probe-readiness -n lab-3a -o jsonpath='{.status.containerStatuses[0].restartCount}')"
echo "phase=$PH Ready=$RD restarts=$RC"
{ test "$PH" = 'Running' && test "$RD" = 'False' && test "$RC" -eq 0; } \
  && echo 'PASS: readiness that bai — container van chay, chi bi danh dau chua san sang'

kubectl exec probe-readiness -n lab-3a -- touch /tmp/san-sang
kubectl wait --for=condition=Ready pod/probe-readiness -n lab-3a --timeout=120s \
  && echo 'PASS: probe thanh cong -> Pod Ready'
```

**Ý nghĩa:** kubelet **tiếp tục chạy container không vượt qua readiness probe** và cũng tiếp tục
probe; nó chỉ đặt condition `Ready` thành `false`. Hệ quả thật của việc đó — Pod bị controller
EndpointSlice gỡ khỏi Service — thuộc giai đoạn 5, nên ở đây bạn dừng ở condition (xem ghi chú hoãn
ở mục 1.1).

**PASS:** hai dòng `PASS:` xuất hiện.

### B4.2. Liveness thất bại: container bị giết và khởi động lại

Fault injection, ghim vào `lab-k8s-worker2`. Manifest theo đúng ví dụ `exec-liveness` của bài
[274](../274-configure-probes-vi.md).

```bash
cat > ~/lab-work/3a/probe-liveness.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: probe-liveness
  namespace: lab-3a
  labels:
    test: liveness
spec:
  nodeName: lab-k8s-worker2
  containers:
  - name: app
    image: busybox:1.37
    args: ["/bin/sh", "-c", "touch /tmp/healthy; sleep 20; rm -f /tmp/healthy; sleep 600"]
    livenessProbe:
      exec:
        command: ["cat", "/tmp/healthy"]
      initialDelaySeconds: 5
      periodSeconds: 5
EOF

kubectl apply -f ~/lab-work/3a/probe-liveness.yaml

for i in $(seq 1 30); do
  RC="$(kubectl get pod probe-liveness -n lab-3a \
        -o jsonpath='{.status.containerStatuses[0].restartCount}' 2>/dev/null)"
  test -n "$RC" && test "$RC" -ge 1 && break
  sleep 5
done
kubectl describe pod probe-liveness -n lab-3a | tee ~/lab-evidence/3a/b4-liveness.txt
```

```bash
RC="$(kubectl get pod probe-liveness -n lab-3a -o jsonpath='{.status.containerStatuses[0].restartCount}')"
echo "restartCount = $RC"
test "$RC" -ge 1 && echo 'PASS: liveness that bai -> container bi khoi dong lai'

kubectl describe pod probe-liveness -n lab-3a | grep -q 'Liveness probe failed' \
  && echo 'PASS: su kien ghi ro Liveness probe failed'

kubectl delete pod probe-liveness -n lab-3a --wait=true
```

**Ý nghĩa:** cùng một kết quả `Failure`, hai loại probe cho hai hậu quả hoàn toàn khác nhau:
readiness chỉ rút Pod khỏi vòng phục vụ, liveness **kill container** rồi để nó chịu chi phối của
`restartPolicy`. Đó là lý do bài [49](../49-probes-vi.md) dành hẳn một khối *Thận trọng* cho
liveness: cấu hình quá nhạy sẽ giết container **đúng lúc tải cao**, và tải dồn sang các Pod còn lại
làm chúng cũng trượt probe.

**PASS:** hai dòng `PASS:` xuất hiện.

### B4.3. Startup probe thất bại cũng giết container

Fault injection, ghim vào `lab-k8s-worker2`.

```bash
cat > ~/lab-work/3a/probe-startup.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: probe-startup
  namespace: lab-3a
spec:
  nodeName: lab-k8s-worker2
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    startupProbe:
      exec:
        command: ["cat", "/tmp/da-khoi-dong"]
      periodSeconds: 5
      failureThreshold: 3
    livenessProbe:
      exec:
        command: ["cat", "/tmp/khong-bao-gio-co"]
      periodSeconds: 5
EOF

kubectl apply -f ~/lab-work/3a/probe-startup.yaml

for i in $(seq 1 24); do
  RC="$(kubectl get pod probe-startup -n lab-3a \
        -o jsonpath='{.status.containerStatuses[0].restartCount}' 2>/dev/null)"
  test -n "$RC" && test "$RC" -ge 1 && break
  sleep 5
done
kubectl describe pod probe-startup -n lab-3a | tee ~/lab-evidence/3a/b4-startup.txt
```

```bash
RC="$(kubectl get pod probe-startup -n lab-3a -o jsonpath='{.status.containerStatuses[0].restartCount}')"
echo "restartCount = $RC"
test "$RC" -ge 1 && echo 'PASS: startup that bai -> container bi khoi dong lai'

kubectl describe pod probe-startup -n lab-3a | grep -q 'Startup probe failed' \
  && echo 'PASS: su kien ghi ro Startup probe failed'

kubectl describe pod probe-startup -n lab-3a | grep -q 'Liveness probe failed' \
  && echo 'FAIL: liveness da chay trong khi startup chua thanh cong' \
  || echo 'PASS: khong co su kien liveness — startup chan liveness va readiness'

kubectl delete pod probe-startup -n lab-3a --wait=true
```

**Ý nghĩa:** startup probe chỉ chạy **lúc khởi động** và **chặn** liveness lẫn readiness cho tới
khi nó thành công. Liveness probe ở đây cố tình **cũng sai** — nó `cat` một file không tồn tại —
nhưng trong log sự kiện **không hề có** dòng `Liveness probe failed`, vì kubelet chưa từng chạy nó:
startup probe chưa bao giờ thành công. Đây là công cụ dành cho ứng dụng khởi động lâu: bạn nới thời
gian khởi động bằng `failureThreshold` của startup probe mà **không phải nới các giá trị mặc định
của liveness probe**.

**PASS:** ba dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

### B4.4. Ràng buộc `successThreshold`

```bash
cat > ~/lab-work/3a/probe-invalid.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: probe-invalid
  namespace: lab-3a
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    livenessProbe:
      exec:
        command: ["true"]
      successThreshold: 2
EOF

kubectl create -f ~/lab-work/3a/probe-invalid.yaml --dry-run=server \
  2>&1 | tee ~/lab-evidence/3a/b4-invalid.txt
```

```bash
if kubectl create -f ~/lab-work/3a/probe-invalid.yaml --dry-run=server >/dev/null 2>&1; then
  echo 'FAIL: API server chap nhan successThreshold khac 1 cho liveness probe'
else
  echo 'PASS: successThreshold phai la 1 voi liveness va startup probe'
fi
```

**Ý nghĩa:** bài 49 ghi `successThreshold` **phải là 1** với liveness và startup probe. Đây không
phải khuyến nghị mà là ràng buộc được thực thi ở tầng validation — sai thì manifest bị từ chối chứ
không âm thầm chạy sai.

**PASS:** dòng `PASS: successThreshold phai la 1...` xuất hiện.

## B5. Init container

### B5.1. Tuần tự, chạy đến khi hoàn thành

```bash
cat > ~/lab-work/3a/init-demo.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
  namespace: lab-3a
spec:
  volumes:
  - name: workdir
    emptyDir: {}
  initContainers:
  - name: init-1
    image: busybox:1.37
    command: ["sh", "-c", "sleep 20; echo tu-init-1 > /work/data.txt; echo init-1 xong"]
    volumeMounts:
    - name: workdir
      mountPath: /work
  - name: init-2
    image: busybox:1.37
    command: ["sh", "-c", "cat /work/data.txt; echo tu-init-2 >> /work/data.txt; echo init-2 xong"]
    volumeMounts:
    - name: workdir
      mountPath: /work
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "cat /work/data.txt; sleep 3600"]
    volumeMounts:
    - name: workdir
      mountPath: /work
EOF

kubectl apply -f ~/lab-work/3a/init-demo.yaml

for i in $(seq 1 15); do
  ST="$(kubectl get pod init-demo -n lab-3a --no-headers | awk '{print $3}')"
  case "$ST" in
    Init:*) echo "PASS: STATUS trong luc khoi tao = $ST"; break ;;
  esac
  sleep 2
done

PH="$(kubectl get pod init-demo -n lab-3a -o jsonpath='{.status.phase}')"
IN="$(kubectl get pod init-demo -n lab-3a -o jsonpath='{.status.conditions[?(@.type=="Initialized")].status}')"
echo "trong luc khoi tao: phase=$PH Initialized=$IN"
{ test "$PH" = 'Pending' && test "$IN" = 'False'; } \
  && echo 'PASS: dang khoi tao thi Pod o Pending va Initialized la False'

kubectl wait --for=condition=Ready pod/init-demo -n lab-3a --timeout=240s
kubectl logs init-demo -n lab-3a -c init-1 | tee ~/lab-evidence/3a/b5-init1.txt
kubectl logs init-demo -n lab-3a -c init-2 | tee ~/lab-evidence/3a/b5-init2.txt
kubectl get pod init-demo -n lab-3a -o jsonpath='{range .status.initContainerStatuses[*]}{.name}{"\t"}{.state.terminated.exitCode}{"\n"}{end}' \
  | tee ~/lab-evidence/3a/b5-init-status.txt
```

```bash
for C in init-1 init-2; do
  EC="$(kubectl get pod init-demo -n lab-3a \
        -o jsonpath="{.status.initContainerStatuses[?(@.name==\"$C\")].state.terminated.exitCode}")"
  echo "$C exitCode=$EC"
  test "$EC" -eq 0 || echo "FAIL: $C khong thoat 0"
done

L1="$(kubectl exec init-demo -n lab-3a -- sed -n '1p' /work/data.txt)"
L2="$(kubectl exec init-demo -n lab-3a -- sed -n '2p' /work/data.txt)"
echo "noi dung volume: [$L1] [$L2]"
{ test "$L1" = 'tu-init-1' && test "$L2" = 'tu-init-2'; } \
  && echo 'PASS: hai init container chay tuan tu, dung thu tu trong spec'

kubectl logs init-demo -n lab-3a -c init-2 | grep -q 'tu-init-1' \
  && echo 'PASS: init-2 doc duoc thu init-1 de lai tren volume dung chung'
```

**Ý nghĩa:** đây là mẫu của bài [276](../276-configure-pod-initialization-vi.md): init container
chuẩn bị dữ liệu vào một volume dùng chung, app container chỉ việc dùng. Ba điều bài
[50](../50-init-containers-vi.md) nhấn mạnh đều hiện ra ở gate trên — **tuần tự**, **chạy đến khi
hoàn thành**, và app container **chưa hề được tạo** cho tới khi init xong; cột `STATUS` hiện
`Init:<đã xong>/<tổng>` chứ không phải `Running`.

Log của init container lấy bằng `kubectl logs <pod> -c <tên-init-container>` — không có tùy chọn
này thì `kubectl logs` chỉ trả log của app container.

**PASS:** bốn dòng `PASS:` xuất hiện — hai dòng lúc Pod còn đang khởi tạo, hai dòng ở khối gate
cuối — và không có dòng `FAIL:`.

### B5.2. Init container thất bại với `restartPolicy: Never`

Fault injection, ghim vào `lab-k8s-worker2`.

```bash
cat > ~/lab-work/3a/init-fail.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: init-fail
  namespace: lab-3a
spec:
  nodeName: lab-k8s-worker2
  restartPolicy: Never
  initContainers:
  - name: init-hong
    image: busybox:1.37
    command: ["sh", "-c", "echo init that bai; exit 1"]
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
EOF

kubectl apply -f ~/lab-work/3a/init-fail.yaml
kubectl wait --for=jsonpath='{.status.phase}'=Failed pod/init-fail -n lab-3a --timeout=180s
kubectl describe pod init-fail -n lab-3a | tee ~/lab-evidence/3a/b5-init-fail.txt
```

```bash
EC="$(kubectl get pod init-fail -n lab-3a -o jsonpath='{.status.initContainerStatuses[0].state.terminated.exitCode}')"
RUN="$(kubectl get pod init-fail -n lab-3a -o jsonpath='{.status.containerStatuses[0].state.running}')"
echo "init exitCode=$EC ; trang thai running cua app container=[$RUN]"
test "$EC" -eq 1 && echo 'PASS: init container thoat 1'
test -z "$RUN"   && echo 'PASS: app container chua bao gio chay'
test "$(kubectl get pod init-fail -n lab-3a -o jsonpath='{.status.phase}')" = 'Failed' \
  && echo 'PASS: restartPolicy Never — mot init container hong lam ca Pod that bai'

kubectl delete pod init-fail -n lab-3a --wait=true
```

**Ý nghĩa:** bình thường kubelet **liên tục khởi động lại** init container hỏng cho tới khi nó
thành công; riêng với Pod đặt `restartPolicy: Never`, Kubernetes coi **toàn bộ Pod là thất bại**.
Ghi nhớ thêm chi tiết dễ sai: với Pod đặt `Always`, init container dùng chính sách `OnFailure`,
không phải `Always`.

**PASS:** ba dòng `PASS:` xuất hiện.

### B5.3. Init container thông thường bị cấm `readinessProbe`

```bash
cat > ~/lab-work/3a/init-invalid.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: init-invalid
  namespace: lab-3a
spec:
  initContainers:
  - name: init-1
    image: busybox:1.37
    command: ["sh", "-c", "true"]
    readinessProbe:
      exec:
        command: ["true"]
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
EOF

kubectl create -f ~/lab-work/3a/init-invalid.yaml --dry-run=server \
  2>&1 | tee ~/lab-evidence/3a/b5-init-invalid.txt
```

```bash
if kubectl create -f ~/lab-work/3a/init-invalid.yaml --dry-run=server >/dev/null 2>&1; then
  echo 'FAIL: API server chap nhan readinessProbe tren init container thong thuong'
else
  echo 'PASS: readinessProbe bi cam o tang validation'
fi
```

**Ý nghĩa:** init container **không thể định nghĩa trạng thái sẵn sàng tách biệt với việc hoàn
thành** — với nó, "xong" và "sẵn sàng" là một. Lệnh cấm này được thực thi trong quá trình kiểm tra
hợp lệ chứ không phải bị bỏ qua âm thầm. Ngoại lệ duy nhất là sidecar, và B6 sẽ cho thấy vì sao.

**PASS:** dòng `PASS: readinessProbe bi cam o tang validation` xuất hiện.

## B6. Sidecar container

### B6.1. Sidecar khởi động rồi ở lại

```bash
cat > ~/lab-work/3a/sc-demo.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: sc-demo
  namespace: lab-3a
spec:
  terminationGracePeriodSeconds: 30
  volumes:
  - name: workdir
    emptyDir: {}
  initContainers:
  - name: logshipper
    image: busybox:1.37
    restartPolicy: Always
    command: ["sh", "-c", "trap 'echo SIDECAR-NHAN-TERM; exit 0' TERM; touch /work/sidecar-len; while true; do sleep 1; done"]
    volumeMounts:
    - name: workdir
      mountPath: /work
  - name: init-sau-sidecar
    image: busybox:1.37
    command: ["sh", "-c", "test -f /work/sidecar-len && echo init-sau-sidecar thay sidecar da chay"]
    volumeMounts:
    - name: workdir
      mountPath: /work
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "trap 'echo APP-NHAN-TERM; exit 0' TERM; while true; do sleep 1; done"]
EOF

kubectl apply -f ~/lab-work/3a/sc-demo.yaml
kubectl wait --for=condition=Ready pod/sc-demo -n lab-3a --timeout=240s

kubectl get pod sc-demo -n lab-3a -o jsonpath='{range .status.initContainerStatuses[*]}{.name}{"\tstarted="}{.started}{"\trunning="}{.state.running.startedAt}{"\tterminated="}{.state.terminated.exitCode}{"\n"}{end}' \
  | tee ~/lab-evidence/3a/b6-sidecar-status.txt
```

```bash
SR="$(kubectl get pod sc-demo -n lab-3a \
      -o jsonpath='{.status.initContainerStatuses[?(@.name=="logshipper")].state.running.startedAt}')"
SS="$(kubectl get pod sc-demo -n lab-3a \
      -o jsonpath='{.status.initContainerStatuses[?(@.name=="logshipper")].started}')"
IE="$(kubectl get pod sc-demo -n lab-3a \
      -o jsonpath='{.status.initContainerStatuses[?(@.name=="init-sau-sidecar")].state.terminated.exitCode}')"
echo "logshipper: started=$SS running tu $SR ; init-sau-sidecar exitCode=$IE"

{ test -n "$SR" && test "$SS" = 'true'; } \
  && echo 'PASS: sidecar van chay sau khi Pod da Ready'
test "$IE" -eq 0 \
  && echo 'PASS: init container ke tiep chay sau khi sidecar started=true, khong cho sidecar ket thuc'

kubectl logs sc-demo -n lab-3a -c init-sau-sidecar | grep -q 'thay sidecar da chay' \
  && echo 'PASS: init container ke tiep thay dau vet cua sidecar'
```

**Ý nghĩa:** sidecar của Kubernetes **không phải một loại container mới** — nó là một mục trong
`initContainers` có `restartPolicy: Always` ở **cấp container**. Vì nó không bao giờ kết thúc,
kubelet chuyển sang mục kế tiếp ngay khi trạng thái `started` của nó thành `true`, chứ không chờ nó
hoàn thành. Đó là lý do một sidecar không làm treo luồng khởi tạo.

**PASS:** ba dòng `PASS:` xuất hiện.

### B6.2. Sidecar tắt sau container chính

Đo bằng dấu thời gian trong log của hai container, không phải bằng cảm nhận.

```bash
kubectl logs -f -n lab-3a sc-demo -c app        --timestamps > ~/lab-evidence/3a/b6-app.log 2>&1 &
APID=$!
kubectl logs -f -n lab-3a sc-demo -c logshipper --timestamps > ~/lab-evidence/3a/b6-sidecar.log 2>&1 &
SPID=$!

kubectl delete pod sc-demo -n lab-3a --wait=false
kubectl wait --for=delete pod/sc-demo -n lab-3a --timeout=180s
kill "$APID" "$SPID" 2>/dev/null

grep -h 'NHAN-TERM' ~/lab-evidence/3a/b6-app.log ~/lab-evidence/3a/b6-sidecar.log
```

```bash
TA="$(awk '/APP-NHAN-TERM/{print $1; exit}'     ~/lab-evidence/3a/b6-app.log)"
TS="$(awk '/SIDECAR-NHAN-TERM/{print $1; exit}' ~/lab-evidence/3a/b6-sidecar.log)"
echo "app     nhan TERM luc $TA"
echo "sidecar nhan TERM luc $TS"

{ test -n "$TA" && test -n "$TS" && \
  test "$(date -d "$TA" +%s%N)" -lt "$(date -d "$TS" +%s%N)"; } \
  && echo 'PASS: kubelet tri hoan TERM cho sidecar toi khi app container da dung'
```

**Ý nghĩa:** kubelet **trì hoãn** việc gửi TERM cho sidecar cho tới khi container ứng dụng chính
dừng hoàn toàn, rồi tắt các sidecar theo **thứ tự ngược** với thứ tự khai báo. Hệ quả vận hành: một
app container tắt chậm sẽ kéo dài luôn việc tắt của sidecar, và nếu hết thời gian ân hạn thì toàn
bộ container còn lại bị kết thúc đồng thời. Cũng vì vậy, **mã thoát khác 0 của sidecar khi Pod kết
thúc là bình thường** và công cụ giám sát bên ngoài nên bỏ qua.

**PASS:** dòng `PASS: kubelet tri hoan TERM cho sidecar...` xuất hiện.

### B6.3. Sidecar bỏ qua `restartPolicy` cấp Pod

Fault injection, ghim vào `lab-k8s-worker2`.

```bash
cat > ~/lab-work/3a/sc-restart.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: sc-restart
  namespace: lab-3a
spec:
  nodeName: lab-k8s-worker2
  restartPolicy: Never
  initContainers:
  - name: sidecar-tu-thoat
    image: busybox:1.37
    restartPolicy: Always
    command: ["sh", "-c", "echo sidecar chay mot vong; sleep 10; exit 0"]
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
EOF

kubectl apply -f ~/lab-work/3a/sc-restart.yaml

for i in $(seq 1 24); do
  RC="$(kubectl get pod sc-restart -n lab-3a \
        -o jsonpath='{.status.initContainerStatuses[?(@.name=="sidecar-tu-thoat")].restartCount}' 2>/dev/null)"
  test -n "$RC" && test "$RC" -ge 1 && break
  sleep 5
done
kubectl get pod sc-restart -n lab-3a -o jsonpath='{.status.phase}{"\t"}{.status.initContainerStatuses[0].restartCount}{"\n"}' \
  | tee ~/lab-evidence/3a/b6-sidecar-restart.txt
```

```bash
RC="$(kubectl get pod sc-restart -n lab-3a \
      -o jsonpath='{.status.initContainerStatuses[?(@.name=="sidecar-tu-thoat")].restartCount}')"
PH="$(kubectl get pod sc-restart -n lab-3a -o jsonpath='{.status.phase}')"
echo "restartPolicy cap Pod = Never ; sidecar restartCount = $RC ; phase = $PH"
test "$RC" -ge 1 && echo 'PASS: sidecar khoi dong lai du Pod dat restartPolicy Never'
test "$PH" = 'Running' && echo 'PASS: Pod van Running, app container khong bi anh huong'

kubectl delete pod sc-restart -n lab-3a --wait=true
```

**Ý nghĩa:** đây là ô cuối cùng của bảng so sánh trong bài 47. Sidecar **bỏ qua `restartPolicy` cấp
Pod** vì nó mang `restartPolicy: Always` của riêng mình ở **cấp container**; nó khởi động lại kể cả
khi thoát với mã 0, và **vòng đời của nó độc lập** với container ứng dụng chính — app container
không hề bị khởi động lại theo.

**PASS:** hai dòng `PASS:` xuất hiện.

## B7. Kết thúc êm và hook vòng đời

### B7.1. `preStop` chạy trước khi TERM tới tiến trình 1

```bash
cat > ~/lab-work/3a/term-demo.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: term-demo
  namespace: lab-3a
spec:
  terminationGracePeriodSeconds: 20
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "trap 'echo NHAN-TERM' TERM; while true; do sleep 1; done"]
    lifecycle:
      preStop:
        exec:
          command: ["sh", "-c", "echo PRESTOP-CHAY > /proc/1/fd/1; sleep 2"]
EOF

kubectl apply -f ~/lab-work/3a/term-demo.yaml
kubectl wait --for=condition=Ready pod/term-demo -n lab-3a --timeout=180s

kubectl logs -f -n lab-3a term-demo > ~/lab-evidence/3a/b7-term.log 2>&1 &
LPID=$!
T0=$(date +%s)
kubectl delete pod term-demo -n lab-3a --wait=false
kubectl wait --for=delete pod/term-demo -n lab-3a --timeout=180s
T1=$(date +%s)
kill "$LPID" 2>/dev/null

cat ~/lab-evidence/3a/b7-term.log
echo "Pod bien mat sau $((T1-T0)) giay"
```

```bash
LP="$(grep -n 'PRESTOP-CHAY' ~/lab-evidence/3a/b7-term.log | head -1 | cut -d: -f1)"
LT="$(grep -n 'NHAN-TERM'    ~/lab-evidence/3a/b7-term.log | head -1 | cut -d: -f1)"
echo "dong PRESTOP-CHAY = $LP ; dong NHAN-TERM = $LT"
{ test -n "$LP" && test -n "$LT" && test "$LP" -lt "$LT"; } \
  && echo 'PASS: preStop chay xong truoc khi tin hieu dung toi tien trinh 1'

D=$((T1-T0))
test "$D" -ge 15 \
  && echo "PASS: tien trinh khong tu thoat nen Pod song het thoi gian an han da dat ($D giay)"
echo "$D" > ~/lab-evidence/3a/b7-graceful-seconds.txt
```

**Ý nghĩa:** đúng trình tự bài 47 mô tả — Pod bị đánh dấu đang kết thúc, kubelet chạy `preStop`
**bên trong container**, rồi mới kích hoạt container runtime gửi TERM tới tiến trình 1. Tiến trình
ở đây bắt TERM nhưng không thoát, nên nó sống tới khi hết `terminationGracePeriodSeconds` **mà bạn
tự đặt trong manifest** rồi nhận `SIGKILL`.

Nhớ hai chi tiết đi kèm: đồng hồ ân hạn **bắt đầu trước khi hook được gọi**, và nếu `preStop` chưa
xong khi hết ân hạn thì kubelet chỉ xin **một khoảng gia hạn nhỏ, một lần duy nhất**. Hook dài hơn
ân hạn thì phải nới `terminationGracePeriodSeconds`, đừng trông vào khoảng gia hạn đó.

**PASS:** hai dòng `PASS:` xuất hiện.

### B7.2. Buộc xóa bỏ qua ân hạn

```bash
sed 's/name: term-demo/name: term-force/' ~/lab-work/3a/term-demo.yaml \
  > ~/lab-work/3a/term-force.yaml
kubectl apply -f ~/lab-work/3a/term-force.yaml
kubectl wait --for=condition=Ready pod/term-force -n lab-3a --timeout=180s

T0=$(date +%s)
kubectl delete pod term-force -n lab-3a --grace-period=0 --force
kubectl wait --for=delete pod/term-force -n lab-3a --timeout=120s
T1=$(date +%s)
echo "buoc xoa mat $((T1-T0)) giay"
```

```bash
DF=$((T1-T0))
DG="$(cat ~/lab-evidence/3a/b7-graceful-seconds.txt)"
echo "an han binh thuong: $DG giay ; buoc xoa: $DF giay"
test "$DF" -lt "$DG" && echo 'PASS: buoc xoa loai Pod khoi API ngay, khong cho het an han'
```

**Ý nghĩa:** khi buộc xóa, API server **không chờ xác nhận từ kubelet** rằng Pod đã kết thúc trên
node; nó gỡ Pod khỏi API ngay để một Pod mới có thể mang cùng tên. Trên node, tiến trình vẫn được
cho một khoảng ân hạn rất ngắn trước khi bị giết. Đây là lý do bài 47 gắn khối *Thận trọng* cho
thao tác này: tài nguyên đang chạy có thể còn sống trong khi API đã nói là hết.

**PASS:** dòng `PASS: buoc xoa loai Pod khoi API ngay...` xuất hiện.

### B7.3. `preStop` không chạy khi container tự hoàn thành

```bash
cat > ~/lab-work/3a/hook-complete.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: hook-complete
  namespace: lab-3a
spec:
  restartPolicy: Never
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "echo XONG-BINH-THUONG; exit 0"]
    lifecycle:
      preStop:
        exec:
          command: ["sh", "-c", "echo PRESTOP-KHONG-NEN-CHAY > /proc/1/fd/1"]
EOF

kubectl apply -f ~/lab-work/3a/hook-complete.yaml
kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/hook-complete -n lab-3a --timeout=180s
kubectl logs hook-complete -n lab-3a | tee ~/lab-evidence/3a/b7-complete.log
```

```bash
grep -q 'XONG-BINH-THUONG' ~/lab-evidence/3a/b7-complete.log \
  && echo 'PASS: container da chay va tu hoan thanh'
if grep -q 'PRESTOP-KHONG-NEN-CHAY' ~/lab-evidence/3a/b7-complete.log; then
  echo 'FAIL: preStop da chay khi container hoan thanh'
else
  echo 'PASS: preStop khong duoc goi khi container hoan thanh'
fi
```

**Ý nghĩa:** bài [272](../272-attach-handler-lifecycle-event-vi.md) ghi rõ Kubernetes **chỉ gửi sự
kiện `preStop` khi Pod hoặc container bị *chấm dứt***, không gửi khi nó *hoàn thành*. Đây là cái
bẫy kinh điển với các workload chạy-rồi-thoát: mọi việc dọn dẹp đặt trong `preStop` sẽ không bao
giờ chạy.

**PASS:** hai dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

## B8. Chia sẻ process namespace và ephemeral container

### B8.1. `shareProcessNamespace`

```bash
cat > ~/lab-work/3a/psns.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: psns
  namespace: lab-3a
spec:
  shareProcessNamespace: true
  containers:
  - name: web
    image: busybox:1.37
    command: ["sh", "-c", "echo dau-an-cua-web > /tmp/dau-an.txt; httpd -f -p 8080 -h /tmp"]
  - name: shell
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    securityContext:
      capabilities:
        add: ["SYS_PTRACE"]
EOF

kubectl apply -f ~/lab-work/3a/psns.yaml
kubectl wait --for=condition=Ready pod/psns -n lab-3a --timeout=180s

kubectl exec psns -n lab-3a -c shell -- ps ax | tee ~/lab-evidence/3a/b8-ps.txt
```

```bash
kubectl exec psns -n lab-3a -c shell -- ps ax | grep -q '/pause' \
  && echo 'PASS: thay tien trinh 1 la pod sandbox /pause'
kubectl exec psns -n lab-3a -c shell -- ps ax | grep -q 'httpd' \
  && echo 'PASS: thay tien trinh cua container khac trong cung Pod'

PID="$(kubectl exec psns -n lab-3a -c shell -- pidof httpd | awk '{print $1}')"
OUT="$(kubectl exec psns -n lab-3a -c shell -- cat "/proc/$PID/root/tmp/dau-an.txt")"
echo "doc qua /proc/$PID/root -> $OUT"
test "$OUT" = 'dau-an-cua-web' \
  && echo 'PASS: doc duoc filesystem cua container khac qua /proc/PID/root'

if kubectl exec duo -n lab-3a -c client -- ps ax | grep -q 'httpd'; then
  echo 'FAIL: Pod khong bat shareProcessNamespace ma van thay tien trinh container khac'
else
  echo 'PASS: khong bat shareProcessNamespace thi moi container chi thay tien trinh cua minh'
fi
```

**Ý nghĩa:** ba hệ quả mà bài [292](../292-share-process-namespace-vi.md) cảnh báo đều hiện ra:
tiến trình của container **không còn mang PID 1** — PID 1 giờ là `/pause` của sandbox, nên
`kill -HUP 1` sẽ bắn nhầm vào sandbox; toàn bộ `/proc` của các container lộ ra nhau, kể cả tham số
dòng lệnh và biến môi trường; và filesystem của container này đọc được từ container kia qua
`/proc/$pid/root`, nên bí mật trên filesystem chỉ còn được bảo vệ bằng quyền Unix. Đối chứng với
`duo` cho thấy đây là hành vi **phải bật mới có**.

**PASS:** bốn dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

### B8.2. Ephemeral container

```bash
kubectl debug -n lab-3a pod/duo --image=busybox:1.37 --target=web -c dbg -- sh -c 'sleep 3600'

for i in $(seq 1 24); do
  R="$(kubectl get pod duo -n lab-3a \
       -o jsonpath='{.status.ephemeralContainerStatuses[?(@.name=="dbg")].state.running.startedAt}')"
  test -n "$R" && break
  sleep 5
done

kubectl get pod duo -n lab-3a -o jsonpath='{.spec.ephemeralContainers[*].name}{"\n"}' \
  | tee ~/lab-evidence/3a/b8-ephemeral.txt
kubectl exec duo -n lab-3a -c dbg -- ps ax | tee -a ~/lab-evidence/3a/b8-ephemeral.txt
```

```bash
N="$(kubectl get pod duo -n lab-3a -o jsonpath='{.spec.ephemeralContainers[?(@.name=="dbg")].name}')"
test "$N" = 'dbg' && echo 'PASS: ephemeral container da duoc them vao Pod dang chay'

kubectl exec duo -n lab-3a -c dbg -- ps ax | grep -q 'httpd' \
  && echo 'PASS: --target cho dbg nhin thay tien trinh cua container dich'
```

Ba ràng buộc của bài [52](../52-ephemeral-containers-vi.md), kiểm bằng dry run nên không tạo gì:

```bash
if kubectl patch pod duo -n lab-3a --dry-run=server \
     -p '{"spec":{"ephemeralContainers":[{"name":"them-bang-tay","image":"busybox:1.37"}]}}' >/dev/null 2>&1; then
  echo 'FAIL: them duoc ephemeral container bang cap nhat Pod thong thuong'
else
  echo 'PASS: cap nhat Pod thong thuong khong them duoc ephemeral container'
fi

if kubectl patch pod duo -n lab-3a --subresource=ephemeralcontainers --dry-run=server --type=json \
     -p '[{"op":"remove","path":"/spec/ephemeralContainers/0"}]' >/dev/null 2>&1; then
  echo 'FAIL: xoa duoc ephemeral container da them'
else
  echo 'PASS: da them roi thi khong xoa duoc'
fi

if kubectl patch pod duo -n lab-3a --subresource=ephemeralcontainers --dry-run=server \
     -p '{"spec":{"ephemeralContainers":[{"name":"dbg2","image":"busybox:1.37","livenessProbe":{"exec":{"command":["true"]}}}]}}' >/dev/null 2>&1; then
  echo 'FAIL: dat duoc livenessProbe cho ephemeral container'
else
  echo 'PASS: ephemeral container khong duoc co probe'
fi
```

**Ý nghĩa:** ephemeral container được tạo qua **handler riêng `ephemeralcontainers`** chứ không
thêm thẳng vào `pod.spec` — đó là lý do `kubectl edit` không làm được và `kubectl debug` phải tồn
tại. Nó **không có port** nên `ports`, `livenessProbe`, `readinessProbe` đều bị cấm; việc phân bổ
tài nguyên của Pod là **bất biến** nên `resources` cũng bị cấm. Và vì thiếu bảo đảm về tài nguyên
lẫn việc thực thi, lại không bao giờ được tự động khởi động lại, nó chỉ dùng để **kiểm tra**, không
dùng để xây dựng ứng dụng.

**PASS:** năm dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

## B9. User namespace

### B9.1. Điều kiện tiên quyết nằm ở tầng hạ tầng

```bash
ssh lab-k8s-worker1 'uname -r; runc --version | head -1; containerd --version' \
  | tee ~/lab-evidence/3a/b9-prereq.txt
ssh lab-k8s-worker1 'stat -f -c %T /var/lib/kubelet' | tee -a ~/lab-evidence/3a/b9-prereq.txt
```

```bash
ssh lab-k8s-worker1 'uname -r' | awk -F'[.-]' \
  '{ if ($1>6 || ($1==6 && $2>=3)) print "PASS: kernel " $0 " >= 6.3, du de idmap mount tmpfs";
     else print "FAIL: kernel " $0 " < 6.3" }'

FS="$(ssh lab-k8s-worker1 'stat -f -c %T /var/lib/kubelet')"
echo "filesystem cua /var/lib/kubelet = $FS"
case "$FS" in
  ext2/ext3|ext4|btrfs|xfs|tmpfs|overlayfs) echo 'PASS: filesystem nam trong danh sach ho tro idmap mount' ;;
  *) echo "FAIL: $FS khong nam trong danh sach bai 55 liet ke" ;;
esac
```

**Ý nghĩa:** bài [55](../55-user-namespaces-vi.md) đặt ba điều kiện, và **không điều kiện nào thuộc
Kubernetes**: kernel hỗ trợ idmap mount cho `/var/lib/kubelet/pods/` lẫn mọi filesystem dùng trong
volume của Pod, OCI runtime đủ mới (`runc` từ 1.2 hoặc `crun` từ 1.9), CRI runtime đủ mới
(containerd từ 2.0). Phiên bản thật đang cài trên cluster lab tra ở
[bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) — lab này không chép lại con số đó,
bạn chỉ đối chiếu output vừa in với bảng.

**PASS:** hai dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

### B9.2. Root trong container không phải root trên host

```bash
cat > ~/lab-work/3a/userns.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: userns
  namespace: lab-3a
spec:
  nodeName: lab-k8s-worker1
  hostUsers: false
  containers:
  - name: shell
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
EOF

kubectl apply -f ~/lab-work/3a/userns.yaml
kubectl wait --for=condition=Ready pod/userns -n lab-3a --timeout=240s

{
  echo "--- trong Pod userns ---"
  kubectl exec userns -n lab-3a -- sh -c 'id -u; readlink /proc/self/ns/user; cat /proc/self/uid_map'
  echo "--- trong Pod duo (khong bat) ---"
  kubectl exec duo -n lab-3a -c client -- sh -c 'id -u; cat /proc/self/uid_map'
  echo "--- tren lab-k8s-worker1 ---"
  ssh lab-k8s-worker1 'readlink /proc/self/ns/user'
} | tee ~/lab-evidence/3a/b9-userns.txt
```

```bash
UID_IN="$(kubectl exec userns -n lab-3a -- id -u)"
MAP_HOST="$(kubectl exec userns -n lab-3a -- awk '{print $2}' /proc/self/uid_map)"
MAP_LEN="$(kubectl exec userns -n lab-3a -- awk '{print $3}' /proc/self/uid_map)"
echo "trong container: uid=$UID_IN, anh xa toi UID host $MAP_HOST, dai $MAP_LEN"
{ test "$UID_IN" -eq 0 && test "$MAP_HOST" -ne 0 && test "$MAP_LEN" -eq 65536; } \
  && echo 'PASS: root trong container duoc anh xa sang UID khong dac quyen tren host'

MAP_DUO="$(kubectl exec duo -n lab-3a -c client -- awk '{print $2}' /proc/self/uid_map)"
test "$MAP_DUO" -eq 0 \
  && echo 'PASS: Pod khong bat user namespace thi root trong container la root tren host'

NSIN="$(kubectl exec userns -n lab-3a -- readlink /proc/self/ns/user)"
NSHOST="$(ssh lab-k8s-worker1 'readlink /proc/self/ns/user')"
echo "user namespace: trong Pod=$NSIN ; tren node=$NSHOST"
test "$NSIN" != "$NSHOST" && echo 'PASS: Pod nam trong mot user namespace khac cua node'
```

**Ý nghĩa:** đây đúng hai lệnh mà bài [295](../295-user-namespaces-tasks-vi.md) yêu cầu chạy.
Tiến trình **nghĩ** nó là root (`id -u` bằng 0) nhưng `uid_map` cho thấy nó được ánh xạ sang một
dải UID khác trên host, dài 65536, và **kubelet bảo đảm không hai Pod nào trên cùng node dùng chung
một ánh xạ**. Vì vậy capability cấp cho Pod chỉ có hiệu lực trong namespace của nó:
`CAP_SYS_MODULE` mất tác dụng hoàn toàn, `CAP_SYS_ADMIN` bị giới hạn trong user namespace của Pod.

Ghi nhớ ranh giới dễ nhầm: `runAsUser`, `runAsGroup`, `fsGroup` **luôn trỏ tới người dùng bên trong
container**, nên bật hay tắt user namespace **không đổi quyền sở hữu file trên volume**.

**PASS:** ba dòng `PASS:` xuất hiện.

### B9.3. Hạn chế cứng đi kèm

```bash
cat > ~/lab-work/3a/userns-invalid.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: userns-invalid
  namespace: lab-3a
spec:
  hostUsers: false
  hostNetwork: true
  containers:
  - name: shell
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
EOF

kubectl create -f ~/lab-work/3a/userns-invalid.yaml --dry-run=server \
  2>&1 | tee ~/lab-evidence/3a/b9-invalid.txt
```

```bash
if kubectl create -f ~/lab-work/3a/userns-invalid.yaml --dry-run=server >/dev/null 2>&1; then
  echo 'FAIL: API server chap nhan hostUsers false di kem hostNetwork true'
else
  echo 'PASS: hostUsers false thi khong duoc dung namespace khac cua host'
fi
```

**Ý nghĩa:** mục đích của tính năng là **cắt đường tới host**, còn `hostNetwork`, `hostIPC`,
`hostPID` là **mở đường tới host** — hai thứ loại trừ nhau, và Kubernetes chặn ngay ở validation.
Pod cần biết địa chỉ IP của node thì dùng downward API ở B10, không dùng `hostNetwork`.

**PASS:** dòng `PASS: hostUsers false thi khong duoc dung namespace khac cua host` xuất hiện.

## B10. Downward API

### B10.1. Qua biến môi trường

```bash
cat > ~/lab-work/3a/dapi-env.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: dapi-env
  namespace: lab-3a
  labels:
    nhom: "3a"
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    env:
    - name: MY_POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: MY_NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    - name: MY_POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: MY_NHOM
      valueFrom:
        fieldRef:
          fieldPath: metadata.labels['nhom']
EOF

kubectl apply -f ~/lab-work/3a/dapi-env.yaml
kubectl wait --for=condition=Ready pod/dapi-env -n lab-3a --timeout=180s
kubectl exec dapi-env -n lab-3a -- printenv MY_POD_NAME MY_NODE_NAME MY_POD_IP MY_NHOM \
  | tee ~/lab-evidence/3a/b10-env.txt
```

```bash
test "$(kubectl exec dapi-env -n lab-3a -- printenv MY_POD_NAME)" = 'dapi-env' \
  && echo 'PASS: metadata.name'
test "$(kubectl exec dapi-env -n lab-3a -- printenv MY_NODE_NAME)" \
   = "$(kubectl get pod dapi-env -n lab-3a -o jsonpath='{.spec.nodeName}')" \
  && echo 'PASS: spec.nodeName khop voi node that'
test "$(kubectl exec dapi-env -n lab-3a -- printenv MY_POD_IP)" \
   = "$(kubectl get pod dapi-env -n lab-3a -o jsonpath='{.status.podIP}')" \
  && echo 'PASS: status.podIP khop voi API'
test "$(kubectl exec dapi-env -n lab-3a -- printenv MY_NHOM)" = '3a' \
  && echo 'PASS: lay duoc mot label cu the bang metadata.labels[...]'
```

**Ý nghĩa:** container biết tên mình, node mình đang chạy và IP của mình **mà không cần Kubernetes
client hay API server** — đúng mục tiêu gắn kết lỏng của bài [56](../56-downward-api-vi.md).

**PASS:** bốn dòng `PASS:` xuất hiện.

### B10.2. Qua file trong volume `downwardAPI`

```bash
cat > ~/lab-work/3a/dapi-vol.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: dapi-vol
  namespace: lab-3a
  labels:
    nhom: "3a"
    lab: "pod-va-vong-doi"
  annotations:
    ghi-chu: "vi du cua lab 3a"
spec:
  volumes:
  - name: podinfo
    downwardAPI:
      items:
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels
      - path: "annotations"
        fieldRef:
          fieldPath: metadata.annotations
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: podinfo
      mountPath: /etc/podinfo
EOF

kubectl apply -f ~/lab-work/3a/dapi-vol.yaml
kubectl wait --for=condition=Ready pod/dapi-vol -n lab-3a --timeout=180s

kubectl exec dapi-vol -n lab-3a -- sh -c 'cat /etc/podinfo/labels; echo; cat /etc/podinfo/annotations; echo; ls -la /etc/podinfo' \
  | tee ~/lab-evidence/3a/b10-vol.txt
```

```bash
kubectl exec dapi-vol -n lab-3a -- cat /etc/podinfo/labels | grep -q 'nhom="3a"' \
  && echo 'PASS: file labels chua tat ca label, moi label mot dong'
kubectl exec dapi-vol -n lab-3a -- cat /etc/podinfo/annotations | grep -q 'ghi-chu="vi du cua lab 3a"' \
  && echo 'PASS: file annotations chua annotation cua Pod'
kubectl exec dapi-vol -n lab-3a -- ls -la /etc/podinfo | grep -q '\.\.data' \
  && echo 'PASS: co symlink ..data — co che lam moi metadata nguyen tu'
```

**Ý nghĩa:** `labels` và `annotations` trong `/etc/podinfo` là **symlink** trỏ qua `..data` sang một
thư mục con tạm. Nhờ vậy bản cập nhật được ghi vào thư mục mới rồi đổi symlink bằng `rename(2)` —
container không bao giờ đọc phải file dở dang. Đây cũng là lý do bài
[335](../335-downward-api-volume-vi.md) cảnh báo mount kiểu `subPath` sẽ **không** nhận cập nhật.

**PASS:** ba dòng `PASS:` xuất hiện.

### B10.3. Ranh giới ba nhóm trường

```bash
cat > ~/lab-work/3a/dapi-sai-env.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: dapi-sai-env
  namespace: lab-3a
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    env:
    - name: MOI_LABEL
      valueFrom:
        fieldRef:
          fieldPath: metadata.labels
EOF

cat > ~/lab-work/3a/dapi-sai-vol.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: dapi-sai-vol
  namespace: lab-3a
spec:
  volumes:
  - name: podinfo
    downwardAPI:
      items:
      - path: "nodename"
        fieldRef:
          fieldPath: spec.nodeName
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: podinfo
      mountPath: /etc/podinfo
EOF

kubectl create -f ~/lab-work/3a/dapi-sai-env.yaml --dry-run=server 2>&1 \
  | tee ~/lab-evidence/3a/b10-bien-gioi.txt
kubectl create -f ~/lab-work/3a/dapi-sai-vol.yaml --dry-run=server 2>&1 \
  | tee -a ~/lab-evidence/3a/b10-bien-gioi.txt
```

```bash
if kubectl create -f ~/lab-work/3a/dapi-sai-env.yaml --dry-run=server >/dev/null 2>&1; then
  echo 'FAIL: bien moi truong lay duoc metadata.labels day du'
else
  echo 'PASS: metadata.labels day du chi lay duoc qua volume'
fi

if kubectl create -f ~/lab-work/3a/dapi-sai-vol.yaml --dry-run=server >/dev/null 2>&1; then
  echo 'FAIL: volume downwardAPI lay duoc spec.nodeName'
else
  echo 'PASS: spec.nodeName chi lay duoc qua bien moi truong'
fi
```

**Ý nghĩa:** hai cơ chế trông như thay thế được cho nhau nhưng **ba nhóm trường của chúng chỉ giao
nhau một phần**. Nhóm chung gồm `metadata.name`, `metadata.namespace`, `metadata.uid`, và từng
label/annotation theo key. `spec.nodeName`, `spec.serviceAccountName`, `status.hostIP`,
`status.podIP` **chỉ có ở biến môi trường**. Ngược lại `metadata.labels` và `metadata.annotations`
ở dạng đầy đủ **chỉ có ở volume**. Nhớ sai ranh giới này là lỗi manifest thường gặp, và API server
bắt nó ngay.

**PASS:** hai dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

## B11. Cấu hình Pod nâng cao

### B11.1. `securityContext` hai cấp

```bash
cat > ~/lab-work/3a/secctx.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: secctx
  namespace: lab-3a
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
  containers:
  - name: theo-cap-pod
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
  - name: ghi-de-cap-container
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    securityContext:
      runAsUser: 2000
      allowPrivilegeEscalation: false
EOF

kubectl apply -f ~/lab-work/3a/secctx.yaml
kubectl wait --for=condition=Ready pod/secctx -n lab-3a --timeout=180s

for C in theo-cap-pod ghi-de-cap-container; do
  echo "$C -> uid=$(kubectl exec secctx -n lab-3a -c "$C" -- id -u) gid=$(kubectl exec secctx -n lab-3a -c "$C" -- id -g)"
done | tee ~/lab-evidence/3a/b11-secctx.txt
```

```bash
U1="$(kubectl exec secctx -n lab-3a -c theo-cap-pod -- id -u)"
G1="$(kubectl exec secctx -n lab-3a -c theo-cap-pod -- id -g)"
U2="$(kubectl exec secctx -n lab-3a -c ghi-de-cap-container -- id -u)"
G2="$(kubectl exec secctx -n lab-3a -c ghi-de-cap-container -- id -g)"
{ test "$U1" -eq 1000 && test "$G1" -eq 3000; } \
  && echo 'PASS: securityContext cap Pod ap cho container khong khai rieng'
{ test "$U2" -eq 2000 && test "$G2" -eq 3000; } \
  && echo 'PASS: cap container ghi de runAsUser nhung van thua ke runAsGroup cua Pod'
```

**Ý nghĩa:** `securityContext` cấp Pod dùng cho hai việc — những khía cạnh vốn thuộc cả Pod, và
những giá trị bạn muốn đặt làm **mặc định khi container không ghi đè**. Cấp container chỉ áp cho
đúng container đó. Bài [60](../60-advanced-pod-config-vi.md) cũng cảnh báo chế độ đặc quyền **ghi
đè nhiều thiết lập bảo mật khác**, nên lab không dùng nó.

**PASS:** hai dòng `PASS:` xuất hiện.

### B11.2. `nodeSelector` nhìn label của node

```bash
cat > ~/lab-work/3a/pin-worker1.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pin-worker1
  namespace: lab-3a
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker1
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
EOF

kubectl apply -f ~/lab-work/3a/pin-worker1.yaml
kubectl wait --for=condition=Ready pod/pin-worker1 -n lab-3a --timeout=180s
```

```bash
N="$(kubectl get pod pin-worker1 -n lab-3a -o jsonpath='{.spec.nodeName}')"
echo "pin-worker1 chay tren $N"
test "$N" = 'lab-k8s-worker1' && echo 'PASS: nodeSelector rang buoc Pod theo label cua node'
```

**Ý nghĩa:** `nodeSelector` là dạng ràng buộc chọn node đơn giản nhất và nó nhìn **label của
node**. Khác với `nodeName` mà lab dùng cho các Pod fault injection, `nodeSelector` vẫn đi qua
scheduler — đó là lý do B2.1 dùng nó để tạo một Pod `Pending` thật.

**PASS:** dòng `PASS: nodeSelector rang buoc Pod theo label cua node` xuất hiện.

### B11.3. Toleration gỡ rào do node dựng lên

```bash
kubectl get node lab-k8s-master -o jsonpath='{range .spec.taints[*]}{.key}{"="}{.value}{":"}{.effect}{"\n"}{end}' \
  | tee ~/lab-evidence/3a/b11-taint.txt

cat > ~/lab-work/3a/khong-tolerate.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: khong-tolerate
  namespace: lab-3a
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-master
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
EOF

cat > ~/lab-work/3a/co-tolerate.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: co-tolerate
  namespace: lab-3a
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-master
  tolerations:
  - key: "node-role.kubernetes.io/control-plane"
    operator: "Exists"
    effect: "NoSchedule"
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
EOF

kubectl apply -f ~/lab-work/3a/khong-tolerate.yaml
kubectl apply -f ~/lab-work/3a/co-tolerate.yaml

for i in $(seq 1 12); do
  R="$(kubectl get pod khong-tolerate -n lab-3a \
       -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}')"
  test -n "$R" && break
  sleep 5
done
kubectl describe pod khong-tolerate -n lab-3a | tee -a ~/lab-evidence/3a/b11-taint.txt
kubectl wait --for=condition=Ready pod/co-tolerate -n lab-3a --timeout=180s
```

```bash
PH="$(kubectl get pod khong-tolerate -n lab-3a -o jsonpath='{.status.phase}')"
R="$(kubectl get pod khong-tolerate -n lab-3a -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}')"
{ test "$PH" = 'Pending' && test "$R" = 'Unschedulable'; } \
  && echo 'PASS: khong co toleration thi taint cua control plane chan Pod'
kubectl describe pod khong-tolerate -n lab-3a | grep -qi 'taint' \
  && echo 'PASS: su kien noi ro ly do la taint'

N="$(kubectl get pod co-tolerate -n lab-3a -o jsonpath='{.spec.nodeName}')"
test "$N" = 'lab-k8s-master' && echo 'PASS: co toleration thi Pod len duoc node co taint'

kubectl delete pod khong-tolerate co-tolerate -n lab-3a --wait=true
```

**Ý nghĩa:** hai cơ chế khác nhau về bản chất. `nodeSelector` **chọn** node; toleration **không
chọn gì cả**, nó chỉ **gỡ một rào cản** do node dựng lên. Có toleration không có nghĩa Pod sẽ lên
node đó — ở đây Pod lên `lab-k8s-master` là nhờ `nodeSelector`, còn toleration chỉ khiến taint
không chặn nữa. Xóa ngay hai Pod này: chuỗi lab không để workload nằm trên control plane.

**PASS:** ba dòng `PASS:` xuất hiện.

### B11.4. PriorityClass đặt `.spec.priority` thay bạn

```bash
cat > ~/lab-work/3a/priorityclass.yaml <<'EOF'
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: lab3a-uu-tien-cao
value: 10000
globalDefault: false
description: "PriorityClass tam cua Lab 3a"
EOF

cat > ~/lab-work/3a/pod-uu-tien.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-uu-tien
  namespace: lab-3a
spec:
  priorityClassName: lab3a-uu-tien-cao
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
EOF

cat > ~/lab-work/3a/pod-khong-uu-tien.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-khong-uu-tien
  namespace: lab-3a
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
EOF

kubectl apply -f ~/lab-work/3a/priorityclass.yaml
kubectl get priorityclass lab3a-uu-tien-cao -o jsonpath='{.value}{"\n"}' \
  | tee ~/lab-evidence/3a/b11-priority.txt
```

Hai lệnh dưới dùng `--dry-run=server` nên hai Pod này chỉ đi qua admission chứ không được tạo:

```bash
P1="$(kubectl create -f ~/lab-work/3a/pod-uu-tien.yaml       --dry-run=server -o jsonpath='{.spec.priority}')"
P0="$(kubectl create -f ~/lab-work/3a/pod-khong-uu-tien.yaml --dry-run=server -o jsonpath='{.spec.priority}')"
echo "co priorityClassName -> spec.priority=$P1 ; khong co -> spec.priority=$P0"
test "$P1" = '10000' && echo 'PASS: Kubernetes tu dat spec.priority tu PriorityClass'
test "$P0" = '0'     && echo 'PASS: khong khai priorityClassName thi priority la 0'
```

**Ý nghĩa:** PriorityClass là object **ở phạm vi cluster**, ánh xạ một cái tên sang một số nguyên.
Bạn gán bằng `priorityClassName` và Kubernetes tự đặt `.spec.priority` — **bạn không đặt
`.spec.priority` trực tiếp**. Độ ưu tiên chỉ phát huy tác dụng khi Pod không lập lịch được vì thiếu
tài nguyên, lúc đó scheduler chiếm chỗ Pod ưu tiên thấp hơn; cơ chế chiếm chỗ đó thuộc
[giai đoạn 7](../00-ALO-TRINH-ADMIN.md#giai-đoạn-7--lập-lịch-và-chính-sách-tài-nguyên) nên lab này
dừng ở chỗ chứng minh trường được đặt.

Lệnh trên dùng `--dry-run=server` nên không Pod nào được tạo; PriorityClass thì tạo thật và sẽ bị
xóa ở B13.

**PASS:** hai dòng `PASS:` xuất hiện.

## B12. Image volume

```bash
cat > ~/lab-work/3a/img-vol.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: img-vol
  namespace: lab-3a
spec:
  volumes:
  - name: goi-image
    image:
      reference: busybox:1.37
      pullPolicy: IfNotPresent
  containers:
  - name: shell
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: goi-image
      mountPath: /img
EOF

R="$(kubectl create -f ~/lab-work/3a/img-vol.yaml --dry-run=server \
     -o jsonpath='{.spec.volumes[?(@.name=="goi-image")].image.reference}')"
echo "API server giu reference = $R"
test "$R" = 'busybox:1.37' && echo 'PASS: cluster nhan truong volumes[].image'

kubectl apply -f ~/lab-work/3a/img-vol.yaml
kubectl wait --for=condition=Ready pod/img-vol -n lab-3a --timeout=240s
kubectl exec img-vol -n lab-3a -- ls /img | tee ~/lab-evidence/3a/b12-img-vol.txt
```

```bash
kubectl exec img-vol -n lab-3a -- test -e /img/bin/busybox \
  && echo 'PASS: noi dung image duoc mount vao /img'

if kubectl exec img-vol -n lab-3a -- sh -c 'touch /img/thu-ghi' >/dev/null 2>&1; then
  echo 'FAIL: ghi duoc vao image volume'
else
  echo 'PASS: image volume la mount chi doc'
fi
```

**Ý nghĩa:** `volumes[].image` mount **nội dung của một object trong OCI registry** vào container,
tách hẳn khỏi image mà container đang chạy. Hai điều kiện phải cùng đúng thì nó chạy được: API
server chấp nhận trường (gate đầu), và **container runtime hỗ trợ tính năng** (hai gate sau). Đối
chiếu runtime đang dùng ở [bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) nếu gate
thứ hai fail.

**PASS:** ba dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

## B13. Cleanup và gate cuối

**Mục đích:** xóa mọi object lab tạo ra và chứng minh cluster trở về `01-cluster-ready`.

```bash
kubectl delete namespace lab-3a --wait=true --timeout=300s
kubectl delete priorityclass lab3a-uu-tien-cao --ignore-not-found=true

rm -f ~/lab-work/3a/solo.yaml ~/lab-work/3a/duo.yaml ~/lab-work/3a/port-clash.yaml \
      ~/lab-work/3a/phase-pending.yaml ~/lab-work/3a/phase-succeeded.yaml \
      ~/lab-work/3a/phase-failed.yaml ~/lab-work/3a/init-order.yaml \
      ~/lab-work/3a/gate-demo.yaml ~/lab-work/3a/restart-never.yaml \
      ~/lab-work/3a/restart-onfailure.yaml ~/lab-work/3a/restart-always.yaml \
      ~/lab-work/3a/probe-readiness.yaml ~/lab-work/3a/probe-liveness.yaml \
      ~/lab-work/3a/probe-startup.yaml ~/lab-work/3a/probe-invalid.yaml \
      ~/lab-work/3a/init-demo.yaml ~/lab-work/3a/init-fail.yaml \
      ~/lab-work/3a/init-invalid.yaml ~/lab-work/3a/sc-demo.yaml \
      ~/lab-work/3a/term-demo.yaml ~/lab-work/3a/term-force.yaml \
      ~/lab-work/3a/hook-complete.yaml ~/lab-work/3a/psns.yaml \
      ~/lab-work/3a/userns.yaml ~/lab-work/3a/userns-invalid.yaml \
      ~/lab-work/3a/dapi-env.yaml ~/lab-work/3a/dapi-vol.yaml \
      ~/lab-work/3a/dapi-sai-env.yaml ~/lab-work/3a/dapi-sai-vol.yaml \
      ~/lab-work/3a/secctx.yaml ~/lab-work/3a/pin-worker1.yaml \
      ~/lab-work/3a/khong-tolerate.yaml ~/lab-work/3a/co-tolerate.yaml \
      ~/lab-work/3a/priorityclass.yaml ~/lab-work/3a/pod-uu-tien.yaml \
      ~/lab-work/3a/pod-khong-uu-tien.yaml ~/lab-work/3a/sc-restart.yaml \
      ~/lab-work/3a/img-vol.yaml
rmdir ~/lab-work/3a
```

```bash
kubectl get namespace lab-3a >/dev/null 2>&1 \
  && echo 'FAIL: namespace lab-3a van con' \
  || echo 'PASS: namespace lab-3a da xoa'

kubectl get priorityclass lab3a-uu-tien-cao >/dev/null 2>&1 \
  && echo 'FAIL: PriorityClass van con' \
  || echo 'PASS: PriorityClass da xoa'

test ! -e ~/lab-work/3a && echo 'PASS: manifest tam da xoa'

kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
kubectl get pods -A -o wide --field-selector spec.nodeName=lab-k8s-master
```

`rmdir` cố ý không dùng `-rf`: nó fail nếu thư mục còn file lạ, và `test ! -e` biến điều đó thành
gate thay vì im lặng bỏ qua. Thư mục `~/lab-evidence/3a/` **giữ lại** — đó là bằng chứng.

**PASS:** không có dòng `FAIL:` nào; bốn dòng `PASS:` xuất hiện; ba node `Ready`; lệnh field
selector trả `No resources found`; CoreDNS đủ replica; namespace `default` không có Pod; trên
`lab-k8s-master` chỉ còn Pod của `kube-system` và `kube-flannel`. Cluster trở về
`01-cluster-ready`; **không tạo snapshot mới**.

---

## 3. Checkpoint 3a

Không đánh dấu lab hoàn tất chỉ vì command chạy hết. Tự trả lời, không nhìn tài liệu:

- [ ] Node chạy Pod trần của bạn gặp sự cố nghiêm trọng rồi khỏe lại. Chính Pod đó có chạy tiếp
      không, và điều gì trong `metadata` chứng minh Pod thay thế là một Pod khác?
- [ ] Hai container trong một Pod chia sẻ những gì, và thứ gì chúng buộc phải tự thỏa thuận với
      nhau? Trên Pod đang chạy, trường nào sửa được và trường nào không?
- [ ] `kubectl get pod` hiện `STATUS: CrashLoopBackOff`. Lúc đó `phase` là gì, và vì sao? Với
      `restartPolicy: OnFailure`, container thoát mã 0 có được khởi động lại không?
- [ ] Kể thứ tự năm condition vòng đời. Pod **không có** init container thì `Initialized` được đặt
      `True` trước hay sau khi sandbox và mạng được dựng?
- [ ] `ContainersReady` là `True` mà `Ready` vẫn `False`: nguyên nhân là gì và `reason` ghi gì?
- [ ] Readiness thất bại, liveness thất bại, startup thất bại — kubelet làm gì trong từng trường
      hợp? Startup probe chặn hai loại kia tới khi nào?
- [ ] Init container thất bại trên Pod đặt `restartPolicy: Never` thì chuyện gì xảy ra? Vì sao
      Kubernetes cấm hẳn `readinessProbe` trên init container thông thường?
- [ ] Sidecar được khai ở đâu và bằng trường nào? Kubelet chờ điều gì ở sidecar trước khi chạy mục
      kế tiếp, và các sidecar bị tắt theo thứ tự nào so với container chính?
- [ ] Kể trình tự kết thúc êm từ lúc bạn gõ `kubectl delete pod` tới lúc object biến mất khỏi API
      server. `preStop` có được gọi khi container tự hoàn thành không?
- [ ] Vì sao không thêm được ephemeral container bằng `kubectl edit`, và ba nhóm trường nào bị cấm
      trên nó?
- [ ] `hostUsers: false` đổi điều gì đối với tiến trình root trong container, và nó cấm kèm ba
      trường nào? `runAsUser` trỏ tới người dùng ở đâu?
- [ ] Ba nhóm trường của downward API khác nhau thế nào? Muốn lấy **tất cả** label của Pod thì dùng
      cơ chế nào, và muốn biết tên node thì dùng cơ chế nào?

Nợ chưa trả sau lab này, phải nói được là **vì sao chưa trả**: thực hành tạo static Pod (bài
[293](../293-static-pod-tasks-vi.md)), request/limit hiệu dụng của init container và sidecar, phase
`Unknown`, và ảnh hưởng của readiness lên EndpointSlice — xem mục 1.1.

### Bài giải thích cuối cùng

Trong vài phút, kể lại bằng lời vòng đời đầy đủ của một Pod có một init container thông thường, một
sidecar, một readiness probe và một hook `preStop`: từ lúc `kubectl apply` cho tới lúc object biến
mất khỏi API server. Nói rõ ở mỗi mốc: condition nào vừa đổi, container nào đang chạy, ai quyết
định Pod có `Ready` hay không, thứ tự gửi tín hiệu khi bạn xóa Pod, và điều gì xảy ra nếu tiến trình
chính không thoát trước khi hết `terminationGracePeriodSeconds`.

## 4. Troubleshooting của lab này

Sự cố dựng môi trường xem [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Dưới đây chỉ là sự cố phát sinh trong nội dung bài học.

| Triệu chứng | Nguyên nhân thường gặp | Xử lý |
| --- | --- | --- |
| B1.3 `web2` không có `exitCode` | `web1` chưa kịp bind port khi `web2` thử | Xóa Pod, tăng `sleep` của `web2` trong manifest rồi chạy lại |
| B2.3 so sánh timestamp fail | Init container quá nhanh nên hai mốc rơi vào cùng một giây | Tăng `sleep` của init container `cho-mot-chut` rồi tạo lại Pod |
| B2.4 `Ready` không lên `True` sau khi patch | Tên condition không khớp `readinessGates`, hoặc patch không vào subresource `status` | So lại chuỗi `training.example.com/ready` ở cả hai chỗ; giữ nguyên `--subresource=status` |
| B3 `restart-always` mãi không hiện `CrashLoopBackOff` | Backoff đang ở khoảng chờ dài; giá trị phụ thuộc cấu hình kubelet | Chờ thêm vài chu kỳ; gate phụ `restartCount >= 1` vẫn là bằng chứng |
| B4.2 `restartCount` không tăng | Probe chưa đạt `failureThreshold`, hoặc image chưa kéo xong nên đồng hồ probe chưa bắt đầu | Chờ thêm vài chu kỳ `periodSeconds`; kiểm `kubectl describe` xem container đã `Started` chưa |
| B5.1 không bắt được `STATUS` dạng `Init:` | Init container chạy xong trước vòng lặp đầu tiên | Xóa Pod, tăng `sleep` của `init-1`, tạo lại |
| B6.2 thiếu dòng `NHAN-TERM` trong một file log | Luồng `kubectl logs -f` đóng trước khi container kịp in | Tạo lại Pod và chạy lại nguyên khối B6.2; mở luồng log **trước** khi `delete` |
| B7.1 không có dòng `PRESTOP-CHAY` | Tiến trình 1 trong container không phải shell nên `/proc/1/fd/1` không phải stdout của log | Giữ nguyên `command` dạng `sh -c`; không đổi sang entrypoint khác |
| B8.2 `kubectl debug` báo không hỗ trợ `--target` | Container runtime không cho chia sẻ process namespace theo container đích | Bỏ `--target`, ephemeral container vẫn thêm được nhưng chỉ thấy tiến trình của chính nó |
| B9.2 Pod `userns` kẹt ở `ContainerCreating` với lỗi `MOUNT_ATTR_IDMAP` | Filesystem hoặc kernel không hỗ trợ idmap mount | Đọc lại gate B9.1; nếu B9.1 fail thì bỏ qua B9.2 và ghi vào evidence |
| B12 Pod `img-vol` không khởi động được | Container runtime chưa hỗ trợ image volume | Đối chiếu runtime ở [A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); gate đầu của B12 vẫn chứng minh phần API |
| Xóa namespace `lab-3a` treo ở `Terminating` | Còn Pod trong thời gian ân hạn của B7 | Chờ hết `terminationGracePeriodSeconds` đã khai; kiểm `kubectl get pod -n lab-3a` |

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Workloads](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/)
- [Kubernetes v1.35 — Pods](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/)
- [Kubernetes v1.35 — Pod Lifecycle](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Kubernetes v1.35 — Pod Conditions](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/pod-condition/)
- [Kubernetes v1.35 — Liveness, Readiness and Startup Probes](https://v1-35.docs.kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)
- [Kubernetes v1.35 — Init Containers](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/init-containers/)
- [Kubernetes v1.35 — Sidecar Containers](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)
- [Kubernetes v1.35 — Ephemeral Containers](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/)
- [Kubernetes v1.35 — User Namespaces](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/user-namespaces/)
- [Kubernetes v1.35 — Downward API](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/downward-api/)
- [Kubernetes v1.35 — Advanced Pod Configuration](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/advanced-pod-config/)
- [Kubernetes v1.35 — Attach Handlers to Container Lifecycle Events](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/attach-handler-lifecycle-event/)
- [Kubernetes v1.35 — Configure Liveness, Readiness and Startup Probes](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Kubernetes v1.35 — Configure Pod Initialization](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/configure-pod-initialization/)
- [Kubernetes v1.35 — Use an Image Volume With a Pod](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/image-volumes/)
- [Kubernetes v1.35 — Share Process Namespace between Containers in a Pod](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/)
- [Kubernetes v1.35 — Use a User Namespace With a Pod](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/user-namespaces/)
- [Kubernetes v1.35 — Expose Pod Information to Containers Through Files](https://v1-35.docs.kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/)
- [Kubernetes v1.35 — Expose Pod Information to Containers Through Environment Variables](https://v1-35.docs.kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information/)
- [Kubernetes v1.35 — Communicate Between Containers in the Same Pod Using a Shared Volume](https://v1-35.docs.kubernetes.io/docs/tasks/access-application-cluster/communicate-containers-same-pod-shared-volume/)
- [Kubernetes v1.35 — Create static Pods](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/static-pod/)
