# Lab 2 — Container, image, CRI và cgroup

> **Điểm bắt đầu:** snapshot `01-cluster-ready` — xem
> [Lab 00 — Môi trường](LAB-00-MOI-TRUONG-1.35.7.md).
> **Điểm kết thúc:** cleanup trả cluster về đúng `01-cluster-ready`, không tạo snapshot mới.
> **Lab trước:** [Lab 1c — Vòng đời và cơ chế nền của object](LAB-1C-VONG-DOI-VA-CO-CHE-NEN-CUA-OBJECT.md)
> đã cleanup namespace `lab-1c`.
>
> **Cập nhật và đối chiếu phiên bản:** 25/08/2026.

Lab này đi cùng [Giai đoạn 2 — Container và runtime](../00-ALO-TRINH-ADMIN.md#giai-đoạn-2--container-và-runtime).
Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); lab này không chép
lại con số phiên bản nào và **không cài thêm gì**.

Lab dùng Pod làm vật chứa nhỏ nhất để quan sát container. Chi tiết về Pod — phase, probe,
restartPolicy — thuộc [giai đoạn 3](../00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình);
ở đây chỉ cần biết Pod là thứ mang container.

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

- kubelet là **client**, container runtime là **server**, nói chuyện với nhau qua gRPC theo CRI;
  chỉ ra được endpoint thật trên node của mình.
- Node đang chạy **cgroup v2**, và kubelet lẫn containerd **cùng dùng cgroup driver `systemd`**;
  nói được hậu quả nếu hai bên lệch nhau.
- Kubernetes **tự suy ra `imagePullPolicy` từ tag** khi bạn không khai báo, và giá trị đó chỉ
  được chốt **lúc object được tạo lần đầu**.
- Phân biệt tham chiếu image bằng **tag** (có thể bị dời) và bằng **digest** (bất biến).
- Đọc được `ImagePullBackOff` và chỉ ra nguyên nhân từ `kubectl describe`.
- Container biết **hostname là tên Pod**, và nhận danh sách Service cùng namespace qua **biến
  môi trường** tại thời điểm nó được tạo.
- Hook **`PostStart` chạy đồng thời** với tiến trình chính, còn **`PreStop` chạy trước** khi
  tín hiệu dừng được gửi.
- RuntimeClass chọn cấu hình runtime qua **`handler`**, và handler không tồn tại thì container
  không khởi động được.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong giai đoạn 2 | Kiểm chứng ở |
| --- | --- |
| [39 — Các Container](../39-containers-vi.md) | B1: xác định ai thực sự chạy container trên node |
| [44 — Container Runtime Interface (CRI)](../44-cri-vi.md) | B1: endpoint gRPC, phiên bản CRI, quan hệ client/server |
| [33 — Giới thiệu về cgroup v2](../33-cgroups-vi.md) | B2: `stat -fc %T /sys/fs/cgroup/` |
| [00 — Các container runtime](../00-container-runtimes-vi.md) | B2: đối chiếu cgroup driver của kubelet và containerd |
| [40 — Các Image](../40-images-vi.md) | B3 (tag, digest, `imagePullPolicy`) và B4 (`ImagePullBackOff`) |
| [41 — Môi trường Container](../41-container-environment-vi.md) | B5: hostname và biến môi trường Service |
| [42 — Các hook vòng đời của Container](../42-container-lifecycle-hooks-vi.md) | B6: `PostStart` và `PreStop` |
| [43 — Runtime Class](../43-runtime-class-vi.md) | B7: `handler` và giới hạn trên cluster một runtime |

Hai bài thực hành của giai đoạn 2 **không kiểm chứng được trên cluster lab**, đọc để biết:

| Bài | Vì sao không thực hành ở đây |
| --- | --- |
| [225 — Cấu hình kubelet image credential provider](../225-kubelet-credential-provider-vi.md) | Cần private registry và Secret — Secret thuộc [giai đoạn 3b](../00-ALO-TRINH-ADMIN.md#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod) |
| [257 — Chuyển sang cập nhật trạng thái container dựa trên sự kiện CRI](../257-switch-to-evented-pleg-vi.md) | Đổi feature gate của kubelet làm lệch baseline Lab 00; thuộc [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) |

### 1.2. Thời lượng

2–3 giờ. B4 và B7 cố ý tạo lỗi nên mất thêm thời gian chờ.

## 2. Quy ước và an toàn

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` trừ khi ghi rõ node khác.
- Lệnh cần `sudo` để đọc cấu hình node chạy **trên chính node đó** qua SSH.
- **Fault injection chỉ trên `lab-k8s-worker2`** — B4 và B7 ghim Pod lỗi vào node này.
- Bằng chứng ghi vào `~/lab-evidence/2/`. Không lưu token, key hay certificate.
- Lab **không** sửa `/etc/containerd/config.toml`, không sửa cấu hình kubelet, không cài gói.
  Mọi thao tác lên node chỉ là **đọc**.
- Lab cần kéo được image từ internet ở B5 và B6. Nếu môi trường không có internet, xem mục 4.

---

# Phần B — Thực hành kiến thức giai đoạn 2

## B0. Chuẩn bị workspace và namespace

```bash
mkdir -p ~/lab-work/2 ~/lab-evidence/2
kubectl config current-context
kubectl get nodes -o wide
kubectl create namespace lab-2
kubectl get namespace lab-2 -o jsonpath='{.status.phase}'; echo
```

**Ý nghĩa:** `lab-work` chứa manifest tạm, `lab-evidence` chứa output. Namespace `lab-2` cô lập
mọi object lab tạo ra.

**PASS:** context trỏ đúng cluster lab; ba node `Ready`; namespace `lab-2` ở phase `Active`.

## B1. kubelet nói chuyện với container runtime bằng gì

### B1.1. Ai thực sự chạy container

Bài [39](../39-containers-vi.md) nói Kubernetes không tự chạy container. Xác minh trên node.

```bash
ssh lab-k8s-worker1 'sudo crictl version'
ssh lab-k8s-worker1 'sudo crictl version' | tee ~/lab-evidence/2/b1-crictl-version.txt
```

Đọc output: dòng `RuntimeName` cho biết runtime nào đang chạy container, dòng `RuntimeApiVersion`
cho biết phiên bản CRI API đang dùng.

```bash
RT="$(ssh lab-k8s-worker1 'sudo crictl version' | awk -F': *' '/RuntimeName/{print $2}')"
API="$(ssh lab-k8s-worker1 'sudo crictl version' | awk -F': *' '/RuntimeApiVersion/{print $2}')"
echo "RuntimeName=$RT"
echo "RuntimeApiVersion=$API"

test "$RT" = 'containerd' && echo 'PASS: runtime la containerd'
case "$API" in v1*) echo 'PASS: CRI API v1' ;; *) echo "FAIL: CRI API khong phai v1 ($API)" ;; esac
```

**Ý nghĩa:** bài [44](../44-cri-vi.md) nói từ Kubernetes v1.26 kubelet **chỉ làm việc với CRI
API `v1`**; runtime không hỗ trợ `v1` thì kubelet không đăng ký được node. Node của bạn đang
`Ready`, nên điều kiện này phải thỏa — bước trên biến điều đó thành bằng chứng.

**PASS:** hai dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

### B1.2. Endpoint gRPC nằm ở đâu

```bash
ssh lab-k8s-worker1 'cat /etc/crictl.yaml'
ssh lab-k8s-worker1 'sudo grep -i container-runtime-endpoint /var/lib/kubelet/kubeadm-flags.env' || \
  echo '(khong co trong kubeadm-flags.env — kubelet dung endpoint mac dinh cua containerd)'
ssh lab-k8s-worker1 'sudo ls -l /run/containerd/containerd.sock'
```

```bash
SOCK="$(ssh lab-k8s-worker1 'sudo ls /run/containerd/containerd.sock 2>/dev/null')"
test -n "$SOCK" && echo 'PASS: unix socket cua containerd ton tai'
```

**Ý nghĩa:** giá trị `runtime-endpoint` trong `/etc/crictl.yaml` mà bạn viết ở
[Lab 00 A4.3](LAB-00-MOI-TRUONG-1.35.7.md#a43-cài-kubeadm-kubelet-kubectl-và-crictl) chính là
endpoint gRPC mà bài 44 mô tả. `crictl` và kubelet là **hai client khác nhau** cùng nói chuyện
với **một server** là containerd — đó là lý do `crictl` thấy đúng những container mà kubelet tạo.

**PASS:** dòng `PASS: unix socket cua containerd ton tai` xuất hiện.

### B1.3. Chứng minh crictl và kubectl nhìn cùng một thứ

```bash
kubectl -n kube-system get pods -o wide --field-selector spec.nodeName=lab-k8s-worker1 \
  | tee ~/lab-evidence/2/b1-kubectl-pods-worker1.txt

ssh lab-k8s-worker1 'sudo crictl pods --state Ready' \
  | tee ~/lab-evidence/2/b1-crictl-pods-worker1.txt
```

```bash
K="$(kubectl -n kube-system get pods --field-selector spec.nodeName=lab-k8s-worker1,status.phase=Running -o name | wc -l)"
C="$(ssh lab-k8s-worker1 "sudo crictl pods --state Ready --namespace kube-system -q" | wc -l)"
echo "kubectl thay $K Pod kube-system tren worker1"
echo "crictl thay  $C pod sandbox kube-system tren worker1"
test "$K" -eq "$C" && echo 'PASS: hai goc nhin khop nhau' \
  || echo 'FAIL: lech — doc lai muc 4'
```

**Ý nghĩa:** `kubectl` hỏi API server, `crictl` hỏi thẳng containerd qua CRI. Hai đường khác nhau
nhưng ra cùng một tập container, vì kubelet dịch desired state từ API server thành lời gọi CRI.

**PASS:** dòng `PASS: hai goc nhin khop nhau` xuất hiện.

## B2. cgroup v2 và cgroup driver

### B2.1. Node đang dùng cgroup phiên bản nào

Bài [33](../33-cgroups-vi.md) cho đúng một lệnh để xác định.

```bash
for N in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  V="$(ssh "$N" 'stat -fc %T /sys/fs/cgroup/')"
  echo "$N: $V"
done | tee ~/lab-evidence/2/b2-cgroup-version.txt
```

```bash
BAD=0
for N in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  V="$(ssh "$N" 'stat -fc %T /sys/fs/cgroup/')"
  test "$V" = 'cgroup2fs' || { echo "FAIL: $N dung $V"; BAD=1; }
done
test "$BAD" -eq 0 && echo 'PASS: ca ba node dung cgroup v2'
```

**Ý nghĩa:** `cgroup2fs` là cgroup v2, `tmpfs` là cgroup v1. Bài 33 ghi cgroup v1 đã
**deprecated từ v1.35** và kubelet mặc định **không khởi động** trên node cgroup v1.

**PASS:** dòng `PASS: ca ba node dung cgroup v2` xuất hiện.

### B2.2. Hai bên có dùng cùng một cgroup driver không

Bài [00](../00-container-runtimes-vi.md#cgroup-drivers) nói **tối quan trọng** là kubelet và
container runtime dùng cùng driver. Đọc cả hai phía.

```bash
echo '--- kubelet ---'
ssh lab-k8s-worker1 'sudo grep -i cgroupDriver /var/lib/kubelet/config.yaml'

echo '--- containerd ---'
ssh lab-k8s-worker1 'sudo grep -i SystemdCgroup /etc/containerd/config.toml'
```

```bash
KD="$(ssh lab-k8s-worker1 "sudo awk -F': *' '/cgroupDriver/{print \$2}' /var/lib/kubelet/config.yaml")"
CD="$(ssh lab-k8s-worker1 "sudo awk -F'= *' '/SystemdCgroup/{gsub(/ /,\"\",\$2); print \$2}' /etc/containerd/config.toml")"
echo "kubelet cgroupDriver = $KD"
echo "containerd SystemdCgroup = $CD"

test "$KD" = 'systemd' && echo 'PASS: kubelet dung systemd'
test "$CD" = 'true'    && echo 'PASS: containerd dung systemd'
{ test "$KD" = 'systemd' && test "$CD" = 'true'; } \
  && echo 'PASS: hai ben khop nhau' \
  || echo 'FAIL: hai ben lech — cluster se khong on dinh khi chiu ap luc tai nguyen'
```

**Ý nghĩa:** khi systemd là init system mà kubelet lại dùng `cgroupfs`, hệ thống có **hai trình
quản lý cgroup** với hai cách nhìn khác nhau về tài nguyên. Bài 00 nói rõ hệ quả: node trở nên
**không ổn định khi chịu áp lực tài nguyên**. Đây cũng là lý do kubeadm từ v1.22 mặc định đặt
`cgroupDriver: systemd`.

**PASS:** ba dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

### B2.3. Vì sao không đổi driver trên cluster đang chạy

Chỉ đọc, **không thực hiện**. Bài 00 có khối *Thận trọng*: đổi cgroup driver của node đã tham gia
cluster có thể làm **lỗi khi tạo lại Pod sandbox** cho các Pod hiện có, và restart kubelet không
cứu được. Cách đúng là thay node hoặc cài lại node.

Ghi lại kết luận để đối chiếu ở giai đoạn 20:

```bash
cat > ~/lab-evidence/2/b2-ket-luan.txt <<'EOF'
cgroup version: v2 (cgroup2fs) tren ca ba node
cgroup driver : kubelet=systemd, containerd SystemdCgroup=true -> khop
Khong doi driver tren node dang chay: rui ro loi tao lai Pod sandbox.
Quy trinh doi driver dung cach nam o bai 218, giai doan 20.
EOF
cat ~/lab-evidence/2/b2-ket-luan.txt
```

**PASS:** file `b2-ket-luan.txt` tồn tại và có ba dòng trên.

## B3. Tag, digest và quy tắc tự đặt `imagePullPolicy`

Bài [40](../40-images-vi.md) nói Kubernetes **suy ra `imagePullPolicy` từ tag** khi bạn bỏ trống.
Kiểm chứng bằng **server-side dry run** — API server áp defaulting nhưng không tạo object và
không kéo image.

### B3.1. Ba cách tham chiếu image, ba kết quả mặc định

```bash
cat > ~/lab-work/2/pull-latest.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pull-latest
  namespace: lab-2
spec:
  containers:
  - name: app
    image: busybox:latest
EOF

cat > ~/lab-work/2/pull-tag.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pull-tag
  namespace: lab-2
spec:
  containers:
  - name: app
    image: busybox:1.36
EOF

cat > ~/lab-work/2/pull-notag.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pull-notag
  namespace: lab-2
spec:
  containers:
  - name: app
    image: busybox
EOF
```

```bash
for F in pull-latest pull-tag pull-notag; do
  P="$(kubectl create -f ~/lab-work/2/$F.yaml --dry-run=server \
        -o jsonpath='{.spec.containers[0].imagePullPolicy}')"
  echo "$F -> imagePullPolicy=$P"
done | tee ~/lab-evidence/2/b3-pullpolicy.txt
```

```bash
L="$(kubectl create -f ~/lab-work/2/pull-latest.yaml --dry-run=server -o jsonpath='{.spec.containers[0].imagePullPolicy}')"
T="$(kubectl create -f ~/lab-work/2/pull-tag.yaml    --dry-run=server -o jsonpath='{.spec.containers[0].imagePullPolicy}')"
N="$(kubectl create -f ~/lab-work/2/pull-notag.yaml  --dry-run=server -o jsonpath='{.spec.containers[0].imagePullPolicy}')"
test "$L" = 'Always'       && echo 'PASS: :latest -> Always'
test "$N" = 'Always'       && echo 'PASS: khong tag -> Always (vi khong tag nghia la :latest)'
test "$T" = 'IfNotPresent' && echo 'PASS: tag cu the -> IfNotPresent'
```

**Ý nghĩa:** không tag nghĩa là `:latest`, nên hai trường hợp đầu cho cùng kết quả. Đây là quy
tắc gây bất ngờ nhiều nhất của bài 40.

**PASS:** đủ ba dòng `PASS:`.

### B3.2. Digest là bất biến, tag thì không

```bash
kubectl run digest-probe --image=busybox:1.36 --restart=Never -n lab-2 \
  --command -- sh -c 'sleep 3600'
kubectl wait --for=condition=Ready pod/digest-probe -n lab-2 --timeout=180s

kubectl get pod digest-probe -n lab-2 \
  -o jsonpath='{.spec.containers[0].image}{"\n"}{.status.containerStatuses[0].imageID}{"\n"}' \
  | tee ~/lab-evidence/2/b3-digest.txt
```

```bash
SPEC="$(kubectl get pod digest-probe -n lab-2 -o jsonpath='{.spec.containers[0].image}')"
RESOLVED="$(kubectl get pod digest-probe -n lab-2 -o jsonpath='{.status.containerStatuses[0].imageID}')"
echo "spec.image  = $SPEC"
echo "imageID     = $RESOLVED"
case "$SPEC"     in *:1.36)    echo 'PASS: spec ghi bang tag' ;; esac
case "$RESOLVED" in *@sha256:*) echo 'PASS: runtime ghi lai bang digest' ;; esac
```

**Ý nghĩa:** bạn khai báo bằng **tag**, nhưng runtime ghi lại thứ nó thực sự chạy bằng **digest**.
Tag có thể bị dời sang image khác; digest thì không. Đây là bằng chứng cụ thể cho khuyến nghị
ghim `@sha256:...` của bài 40.

**PASS:** hai dòng `PASS:` xuất hiện.

## B4. `ImagePullBackOff` — đọc lỗi thay vì đoán

Fault injection, chạy trên `lab-k8s-worker2`.

```bash
cat > ~/lab-work/2/bad-image.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: bad-image
  namespace: lab-2
spec:
  nodeName: lab-k8s-worker2
  containers:
  - name: app
    image: registry.example.invalid/khong-ton-tai:v1
EOF

kubectl apply -f ~/lab-work/2/bad-image.yaml
```

Chờ trạng thái ổn định rồi đọc:

```bash
for i in $(seq 1 30); do
  R="$(kubectl get pod bad-image -n lab-2 \
        -o jsonpath='{.status.containerStatuses[0].state.waiting.reason}' 2>/dev/null)"
  case "$R" in ImagePullBackOff|ErrImagePull) break ;; esac
  sleep 5
done
echo "waiting.reason = $R"

kubectl describe pod bad-image -n lab-2 | tee ~/lab-evidence/2/b4-describe.txt
```

```bash
R="$(kubectl get pod bad-image -n lab-2 -o jsonpath='{.status.containerStatuses[0].state.waiting.reason}')"
case "$R" in
  ImagePullBackOff|ErrImagePull) echo "PASS: reason = $R" ;;
  *) echo "FAIL: reason khong nhu mong doi ($R)" ;;
esac
kubectl describe pod bad-image -n lab-2 | grep -q -i 'Failed to pull image' \
  && echo 'PASS: describe co su kien Failed to pull image'
```

**Ý nghĩa:** `ErrImagePull` là lần thất bại đầu; `ImagePullBackOff` là trạng thái chờ giữa các
lần thử lại, với back-off tăng dần. Bài 40 nêu hai nguyên nhân thường gặp: **tên image sai** và
**private registry thiếu `imagePullSecrets`**. Ở đây là nguyên nhân thứ nhất, và `describe` chỉ
thẳng ra nó.

**PASS:** hai dòng `PASS:` xuất hiện.

```bash
kubectl delete pod bad-image -n lab-2 --wait=true
```

## B5. Môi trường mà container nhìn thấy

### B5.1. Hostname chính là tên Pod

```bash
kubectl run envcheck --image=busybox:1.36 --restart=Never -n lab-2 \
  --command -- sh -c 'sleep 3600'
kubectl wait --for=condition=Ready pod/envcheck -n lab-2 --timeout=180s

HN="$(kubectl exec envcheck -n lab-2 -- hostname)"
echo "hostname trong container = $HN"
test "$HN" = 'envcheck' && echo 'PASS: hostname = ten Pod'
```

**Ý nghĩa:** bài [41](../41-container-environment-vi.md) nói hostname của container là **tên Pod**,
không phải tên container và không phải tên node.

### B5.2. Biến môi trường của Service cùng namespace

```bash
kubectl exec envcheck -n lab-2 -- env | sort | tee ~/lab-evidence/2/b5-env.txt
kubectl exec envcheck -n lab-2 -- env | grep -E '^KUBERNETES_(SERVICE|PORT)' | sort
```

```bash
kubectl exec envcheck -n lab-2 -- env | grep -q '^KUBERNETES_SERVICE_HOST=' \
  && echo 'PASS: co bien KUBERNETES_SERVICE_HOST'
kubectl exec envcheck -n lab-2 -- env | grep -q '^HOSTNAME=envcheck$' \
  && echo 'PASS: bien HOSTNAME khop ten Pod'
```

**Ý nghĩa:** bài 41 nói danh sách Service **tại thời điểm container được tạo** được đưa vào dưới
dạng biến môi trường, giới hạn trong Service **cùng namespace** cộng Service của control plane.
`KUBERNETES_SERVICE_HOST` là Service `kubernetes` của control plane — đó là lý do nó có mặt dù
namespace `lab-2` chưa có Service nào.

**Điểm cần nhớ cho giai đoạn 5:** vì biến chỉ được đặt **lúc tạo**, một Service tạo ra **sau**
container sẽ không xuất hiện trong biến môi trường của container đó. DNS không có hạn chế này.

**PASS:** hai dòng `PASS:` xuất hiện.

## B6. Hook `PostStart` và `PreStop`

### B6.1. `PostStart` chạy đồng thời với tiến trình chính

```bash
cat > ~/lab-work/2/hooks.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: hooks
  namespace: lab-2
spec:
  terminationGracePeriodSeconds: 30
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
    lifecycle:
      postStart:
        exec:
          command: ["sh", "-c", "echo poststart-da-chay > /tmp/poststart.txt"]
      preStop:
        exec:
          command: ["sh", "-c", "echo prestop-da-chay > /tmp/prestop.txt; sleep 5"]
EOF

kubectl apply -f ~/lab-work/2/hooks.yaml
kubectl wait --for=condition=Ready pod/hooks -n lab-2 --timeout=180s
```

```bash
OUT="$(kubectl exec hooks -n lab-2 -- cat /tmp/poststart.txt 2>/dev/null)"
echo "noi dung file do PostStart tao: $OUT"
test "$OUT" = 'poststart-da-chay' && echo 'PASS: PostStart da chay'
```

**Ý nghĩa:** bài [42](../42-container-lifecycle-hooks-vi.md) nói `PostStart` chạy **đồng thời**
với `ENTRYPOINT`, nên không có bảo đảm nó chạy trước tiến trình chính. Bài cũng cảnh báo hook này
**có thể làm chậm** việc container chuyển sang `Running`.

### B6.2. `PreStop` chạy trước khi tín hiệu dừng được gửi

```bash
kubectl delete pod hooks -n lab-2 --wait=false
sleep 2
kubectl exec hooks -n lab-2 -- cat /tmp/prestop.txt 2>/dev/null \
  | tee ~/lab-evidence/2/b6-prestop.txt || echo '(container da dung — chay lai nhanh hon hoac xem ghi chu ben duoi)'
kubectl wait --for=delete pod/hooks -n lab-2 --timeout=120s
echo 'Pod hooks da bien mat'
```

**Ý nghĩa:** `PreStop` phải **hoàn tất trước** khi tín hiệu TERM được gửi. Nhưng đồng hồ
`terminationGracePeriodSeconds` **bắt đầu chạy trước** khi hook được gọi, nên hook dài không kéo
dài thêm thời gian gia hạn — container vẫn bị chấm dứt trong khoảng đó.

> **Ghi chú:** cửa sổ để `exec` vào container trong lúc `PreStop` đang chạy chỉ dài bằng
> `sleep 5` trong hook. Nếu lệnh `exec` ở trên báo container không còn, đó **không phải lỗi** —
> nó chỉ nghĩa là bạn gõ chậm hơn 5 giây. Bằng chứng vẫn đạt nếu Pod biến mất đúng cách.

**PASS:** dòng `PASS: PostStart da chay` ở B6.1 xuất hiện, và `Pod hooks da bien mat` xuất hiện
ở B6.2.

## B7. RuntimeClass và giới hạn của cluster một runtime

Bài [43](../43-runtime-class-vi.md) nói RuntimeClass chỉ có hai trường quan trọng: `metadata.name`
và `handler`. Handler phải khớp một cấu hình có thật trong container runtime.

### B7.1. Cluster lab có handler nào

```bash
kubectl get runtimeclass -o wide || echo '(chua co RuntimeClass nao — dung nhu mong doi)'
ssh lab-k8s-worker2 'sudo grep -n "runtimes" -A6 /etc/containerd/config.toml' \
  | tee ~/lab-evidence/2/b7-containerd-runtimes.txt
```

**Ý nghĩa:** cluster lab dựng theo Lab 00 chỉ có một cấu hình runtime mặc định (`runc`) và
**không có RuntimeClass nào**. Đó là trạng thái bình thường.

### B7.2. Handler không tồn tại thì container không chạy

```bash
cat > ~/lab-work/2/rc-khong-ton-tai.yaml <<'EOF'
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: lab2-khong-ton-tai
handler: handler-khong-co-that
EOF

kubectl apply -f ~/lab-work/2/rc-khong-ton-tai.yaml
kubectl get runtimeclass lab2-khong-ton-tai -o jsonpath='{.handler}{"\n"}'
```

```bash
cat > ~/lab-work/2/rc-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: rc-pod
  namespace: lab-2
spec:
  nodeName: lab-k8s-worker2
  runtimeClassName: lab2-khong-ton-tai
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
EOF

kubectl apply -f ~/lab-work/2/rc-pod.yaml

for i in $(seq 1 24); do
  PH="$(kubectl get pod rc-pod -n lab-2 -o jsonpath='{.status.phase}' 2>/dev/null)"
  RE="$(kubectl get pod rc-pod -n lab-2 -o jsonpath='{.status.containerStatuses[0].state.waiting.reason}' 2>/dev/null)"
  test -n "$RE" && break
  test "$PH" = 'Running' && break
  sleep 5
done
kubectl describe pod rc-pod -n lab-2 | tee ~/lab-evidence/2/b7-describe.txt
```

```bash
PH="$(kubectl get pod rc-pod -n lab-2 -o jsonpath='{.status.phase}')"
echo "phase = $PH"
test "$PH" != 'Running' && echo 'PASS: Pod khong chay duoc vi handler khong ton tai'
kubectl describe pod rc-pod -n lab-2 | grep -qi 'runtime' \
  && echo 'PASS: su kien co nhac toi runtime handler'
```

**Ý nghĩa:** RuntimeClass **không tự tạo ra** khả năng chạy runtime khác. Nó chỉ **chọn** một
cấu hình đã được thiết lập sẵn trong container runtime trên node. Đây là lý do bài 43 đặt bước
"cấu hình phần hiện thực CRI trên các node" **trước** bước tạo object RuntimeClass.

**PASS:** dòng `PASS: Pod khong chay duoc vi handler khong ton tai` xuất hiện.

## B8. Cleanup và gate cuối

**Mục đích:** xóa mọi object lab tạo ra và chứng minh cluster trở về `01-cluster-ready`.

```bash
kubectl delete namespace lab-2 --wait=true --timeout=180s
kubectl delete runtimeclass lab2-khong-ton-tai --ignore-not-found=true

rm -f ~/lab-work/2/pull-latest.yaml ~/lab-work/2/pull-tag.yaml ~/lab-work/2/pull-notag.yaml \
      ~/lab-work/2/bad-image.yaml ~/lab-work/2/hooks.yaml \
      ~/lab-work/2/rc-khong-ton-tai.yaml ~/lab-work/2/rc-pod.yaml
rmdir ~/lab-work/2
```

```bash
kubectl get namespace lab-2 >/dev/null 2>&1 \
  && echo 'FAIL: namespace lab-2 van con' \
  || echo 'PASS: namespace lab-2 da xoa'

kubectl get runtimeclass lab2-khong-ton-tai >/dev/null 2>&1 \
  && echo 'FAIL: RuntimeClass van con' \
  || echo 'PASS: RuntimeClass da xoa'

test ! -e ~/lab-work/2 && echo 'PASS: manifest tam da xoa'

kubectl wait --for=condition=Ready node --all --timeout=180s
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

`rmdir` cố ý không dùng `-rf`: nó fail nếu thư mục còn file lạ, và `test ! -e` biến điều đó thành
gate thay vì im lặng bỏ qua. Thư mục `~/lab-evidence/2/` **giữ lại** — đó là bằng chứng.

**PASS:** không có dòng `FAIL:` nào; ba dòng `PASS:` xuất hiện; ba node `Ready`; lệnh field
selector trả `No resources found`; CoreDNS đủ replica; namespace `default` không có Pod. Cluster
trở về `01-cluster-ready`; **không tạo snapshot mới**.

---

## 3. Checkpoint 2

Không đánh dấu lab hoàn tất chỉ vì command chạy hết. Tự trả lời, không nhìn tài liệu:

- [ ] Trong quan hệ CRI, ai là client, ai là server, và giao thức là gì?
- [ ] Trên node của bạn, endpoint mà kubelet dùng để gọi container runtime nằm ở đường dẫn nào?
- [ ] Vì sao `crictl` và `kubectl` nhìn thấy cùng một tập container dù đi hai đường khác nhau?
- [ ] Lệnh nào cho biết node đang chạy cgroup v1 hay v2, và output nào ứng với v2?
- [ ] Kubelet dùng `systemd` còn containerd dùng `cgroupfs` thì hậu quả cụ thể là gì?
- [ ] Bạn khai báo `image: myapp` không tag. `imagePullPolicy` được đặt thành gì, và vì sao?
- [ ] Sau khi Pod đã tồn tại, bạn sửa tag thành `:latest`. `imagePullPolicy` có đổi theo không?
- [ ] `spec.containers[0].image` và `status.containerStatuses[0].imageID` khác nhau ở điểm nào,
      và điều đó nói gì về tag so với digest?
- [ ] Phân biệt `ErrImagePull` và `ImagePullBackOff`. Hai nguyên nhân thường gặp là gì?
- [ ] Hostname bên trong container bằng tên gì? Một Service tạo **sau** container có xuất hiện
      trong biến môi trường của container đó không?
- [ ] `PostStart` chạy trước hay chạy đồng thời với tiến trình chính?
- [ ] Đồng hồ `terminationGracePeriodSeconds` bắt đầu chạy trước hay sau khi `PreStop` được gọi?
- [ ] Tạo RuntimeClass trỏ tới handler chưa được cấu hình trên node thì chuyện gì xảy ra?

### Bài giải thích cuối cùng

Trong vài phút, kể lại bằng lời: từ lúc bạn `kubectl apply` một Pod có image `busybox:1.36` cho
tới lúc container chạy trên `lab-k8s-worker1` — kubelet nhận desired state từ đâu, nó gọi ai và
bằng giao thức nào, ai kéo image về, ai đặt giới hạn tài nguyên, và vì sao cgroup driver của hai
bên phải giống nhau.

## 4. Troubleshooting của lab này

Sự cố dựng môi trường xem [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md). Dưới đây chỉ là sự cố
phát sinh trong nội dung bài học.

| Triệu chứng | Nguyên nhân thường gặp | Xử lý |
| --- | --- | --- |
| B1.3 báo `FAIL: lech` | Có Pod `kube-system` đang khởi động lại đúng lúc đếm, hoặc còn pod sandbox đã dừng | Chờ 30 giây rồi đếm lại; `crictl pods` mặc định chỉ đếm sandbox `Ready` |
| B3.2 hoặc B5 treo ở `kubectl wait` | Không kéo được image `busybox` từ internet | Kiểm tra mạng của VM; nếu môi trường cô lập, thay bằng image đã có sẵn trên node (`sudo crictl images`) và sửa lại manifest |
| B4 không bao giờ đạt `ImagePullBackOff` | Môi trường có proxy trả về lỗi khác, hoặc DNS trả IP cho tên miền `.invalid` | Đọc `kubectl describe` để lấy `reason` thật; gate chấp nhận cả `ErrImagePull` |
| B6.2 không đọc được `/tmp/prestop.txt` | Bạn `exec` sau khi container đã dừng | Không phải lỗi — xem ghi chú trong B6.2 |
| B7.2 Pod ở `Pending` thay vì lỗi runtime | Pod chưa được gán node vì `nodeName` sai chính tả | Kiểm tra `kubectl get pod rc-pod -n lab-2 -o wide`, sửa `nodeName` cho khớp tên node thật |
| Xóa namespace `lab-2` treo ở `Terminating` | Pod còn trong grace period của `PreStop` | Chờ hết `terminationGracePeriodSeconds`; nếu vẫn treo, xem `kubectl get pod -n lab-2 -o yaml` |

## 5. Nguồn chính thức

- [Containers](https://kubernetes.io/docs/concepts/containers/)
- [Images](https://kubernetes.io/docs/concepts/containers/images/)
- [Container Environment](https://kubernetes.io/docs/concepts/containers/container-environment/)
- [Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)
- [Container Runtime Interface (CRI)](https://kubernetes.io/docs/concepts/architecture/cri/)
- [About cgroup v2](https://kubernetes.io/docs/concepts/architecture/cgroups/)
- [Runtime Class](https://kubernetes.io/docs/concepts/containers/runtime-class/)
- [Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
