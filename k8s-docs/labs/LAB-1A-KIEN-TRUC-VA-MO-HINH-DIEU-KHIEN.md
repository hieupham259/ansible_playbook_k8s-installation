# Lab 1a — Kiến trúc và mô hình điều khiển Kubernetes

> **Điểm bắt đầu:** snapshot `01-cluster-ready` — xem [Lab 00 — Môi trường](LAB-00-MOI-TRUONG.md).
> **Điểm kết thúc:** cleanup trả cluster về đúng `01-cluster-ready`, không tạo snapshot mới.
>
> **Cập nhật và đối chiếu phiên bản:** 05/08/2026.

Lab này đi cùng mục [1a. Kiến trúc và mô hình điều khiển](../LO-TRINH-ADMIN.md#1a-kiến-trúc-và-mô-hình-điều-khiển).

Trước khi bắt đầu, chạy [quy trình mở đầu ở A5.5](LAB-00-MOI-TRUONG.md#a55-quy-trình-mở-đầu-mỗi-lab)
và xác nhận gate `01-cluster-ready` PASS. Toàn bộ phần dựng VM, cài container runtime,
kubeadm và CNI nằm ở Lab 00 và **không** thuộc phạm vi bài này; nội dung cài đặt sẽ được học
tại giai đoạn 2, 5 và 8.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi có thể tự chứng minh và giải thích được:

- Kubernetes tự động hóa việc gì và không thay thế những gì.
- Thành phần nào thuộc control plane, thành phần nào chạy trên node.
- `kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager`, `kubelet`,
  container runtime, CNI, CoreDNS và `kube-proxy` nằm ở đâu.
- Một object có `metadata`, `spec`, `status`; `spec` là trạng thái mong muốn còn `status`
  phản ánh trạng thái quan sát được.
- Cách khám phá core API, named API groups, resource và version qua API server.
- Node đăng ký với cluster như thế nào; `Condition`, `Capacity`, `Allocatable` và heartbeat
  thể hiện ở đâu.
- Chiều `kubectl → API server`, `kubelet → API server` và `API server → kubelet`.
- Controller là vòng lặp quan sát, so sánh và hành động để đưa trạng thái thực tế về trạng
  thái mong muốn.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong 1a | Phần lab kiểm chứng |
| --- | --- |
| Tổng quan | B1 — Từ nhu cầu đến khả năng Kubernetes |
| Các thành phần của Kubernetes | B2 — Kiểm kê component |
| Kiến trúc cluster | B2 và B3 — Vị trí, process và static Pod |
| Các đối tượng trong Kubernetes | B4 — Đọc object và `spec/status` |
| Kubernetes API | B5 — Discovery, group, version và validation |
| Các Node | B6 — Node status, đăng ký và heartbeat |
| Giao tiếp giữa Node và Control Plane | B7 — Quan sát ba chiều giao tiếp |
| Các Controller | B8 — Reconciliation thật |

Lab này **không** dạy label selector, namespace ở mức chi tiết, kubeconfig, quản lý object
imperative/declarative hay field selector. Toàn bộ nhóm đó thuộc
[Lab 1b](README.md#4-bản-đồ-lab).

### 1.2. Thời lượng

Khoảng 2–3 giờ, tính từ lúc cluster đã `Ready`.

---

## 2. Quy ước và an toàn

- Chỉ chạy fault injection trên `lab-k8s-worker2`.
- Không dừng `kube-apiserver`, `etcd` hoặc sửa manifest control plane trong lab 1a.
- Mở ít nhất ba terminal SSH: master, worker 1 và worker 2.
- Các lệnh không ghi rõ node được chạy trên `lab-k8s-master` bằng user quản trị có kubeconfig.
- Dòng bắt đầu bằng `PASS:` mô tả điều kiện phải đạt; không tiếp tục nếu gate tương ứng fail.
- Bằng chứng của lab ghi vào `~/lab-evidence/1a/`.

---

# Phần B — Thực hành kiến thức 1a

## B0. Chuẩn bị terminal và thư mục bằng chứng

Trên master:

```bash
mkdir -p ~/lab-evidence/1a
kubectl config current-context
kubectl cluster-info
kubectl get nodes
```

**PASS:** context trỏ cluster lab, API server phản hồi, cả ba node `Ready`.

## B1. Từ nhu cầu đến khả năng Kubernetes

Chạy:

```bash
kubectl get nodes -o wide | tee ~/lab-evidence/1a/01-nodes.txt
kubectl get pods -A -o wide | tee ~/lab-evidence/1a/01-system-pods.txt
```

Từ output, ghi vào `~/lab-evidence/1a/01-overview.md` câu trả lời cho bốn câu hỏi:

1. Kubernetes đang quản lý những máy nào?
2. Những component nào Kubernetes giữ chạy mà không cần bạn khởi động thủ công từng
   container?
3. Kubernetes tự động hóa deployment/scaling/recovery/configuration ở mức orchestration;
   việc nào vẫn thuộc về người quản trị, ví dụ chọn ứng dụng, policy, backup và capacity?
4. Vì sao ba VM không tự trở thành cluster chỉ vì đã cài container runtime?

**PASS:** câu trả lời phân biệt được container runtime với orchestrator và không tuyên bố
Kubernetes tự xây ứng dụng, tự chọn policy kinh doanh hoặc tự bảo đảm backup.

## B2. Kiểm kê component

Trên master:

```bash
kubectl -n kube-system get pods -o wide \
  | tee ~/lab-evidence/1a/02-kube-system.txt
kubectl -n kube-flannel get pods -o wide \
  | tee ~/lab-evidence/1a/02-flannel.txt
sudo crictl ps \
  | tee ~/lab-evidence/1a/02-master-cri.txt
sudo ls -la /etc/kubernetes/manifests \
  | tee ~/lab-evidence/1a/02-static-manifests.txt
systemctl is-active kubelet containerd
```

Trên worker 1:

```bash
sudo crictl ps
sudo find /etc/kubernetes/manifests -maxdepth 1 -type f -print
systemctl is-active kubelet containerd
```

Điền bảng sau vào ghi chú:

| Component | Nơi quan sát được | Vai trò tối thiểu phải giải thích |
| --- | --- | --- |
| kube-apiserver | Static Pod trên master | Cổng vào duy nhất của Kubernetes API |
| etcd | Static Pod trên master | Lưu state bền vững của API |
| kube-scheduler | Static Pod trên master | Chọn node cho Pod chưa được gán node |
| kube-controller-manager | Static Pod trên master | Chạy nhiều control loop |
| kubelet | systemd service trên mọi node | Agent của node; đăng ký và báo trạng thái |
| containerd | systemd service trên mọi node | Thực thi container qua CRI |
| kube-proxy | DaemonSet/Pod trên mọi node | Cài đặt cơ chế chuyển tiếp Service |
| Flannel | DaemonSet/Pod trên mọi node | CNI/pod network của lab |
| CoreDNS | Deployment/Pod trong cluster | DNS add-on của cluster |

**PASS:** không gọi kubelet/containerd là control plane; không gọi CoreDNS/CNI là thành phần
bắt buộc của chính control plane; biết cloud-controller-manager không xuất hiện vì lab không
tích hợp cloud provider.

## B3. Ghép thành kiến trúc cluster

Trên master:

```bash
sudo grep -n -- '--etcd-servers' /etc/kubernetes/manifests/kube-apiserver.yaml
sudo grep -n -- '--kubeconfig' /etc/kubernetes/manifests/kube-scheduler.yaml
sudo grep -n -- '--kubeconfig' /etc/kubernetes/manifests/kube-controller-manager.yaml
sudo crictl pods --name kube
```

Chứng minh chỉ API server được cấu hình kết nối etcd trực tiếp:

```bash
sudo grep -R -n -- '--etcd-servers' /etc/kubernetes/manifests
```

**PASS:** chỉ manifest `kube-apiserver.yaml` chứa `--etcd-servers`; scheduler và controller
manager dùng kubeconfig để nói chuyện với API server.

Vẽ lại sơ đồ bằng tay hoặc Markdown và lưu thành `~/lab-evidence/1a/03-architecture.md`:

```text
kubectl/admin
     |
     v
kube-apiserver <----> etcd
     ^   ^
     |   +---- scheduler + controller-manager
     |
     +-------- kubelet trên worker 1/2
                   |
                   +---- containerd ---- container
```

Bổ sung Flannel, kube-proxy và CoreDNS vào đúng vị trí. **PASS:** sơ đồ không vẽ kubectl,
kubelet, scheduler hoặc controller-manager kết nối trực tiếp tới etcd.

## B4. Object, desired state và observed state

Lab dùng Namespace chỉ như vùng cô lập; chi tiết namespace sẽ học ở lab 1b.

```bash
kubectl create namespace lab-1a
kubectl get namespace lab-1a -o yaml \
  | tee ~/lab-evidence/1a/04-namespace.yaml

kubectl get node lab-k8s-worker1 -o jsonpath='{"metadata.name: "}{.metadata.name}{"\n"}{"spec.podCIDR: "}{.spec.podCIDR}{"\n"}{"status.capacity.cpu: "}{.status.capacity.cpu}{"\n"}{"status.allocatable.cpu: "}{.status.allocatable.cpu}{"\n"}{"status.nodeInfo.kubeletVersion: "}{.status.nodeInfo.kubeletVersion}{"\n"}'

kubectl get node lab-k8s-worker1 -o jsonpath='{range .status.conditions[*]}{.type}{"\t"}{.status}{"\t"}{.reason}{"\n"}{end}'
```

Trả lời:

- `apiVersion`, `kind`, `metadata` giúp API server nhận diện object ra sao?
- Trường nào ở Node do cấu hình/cluster gán trong `spec`?
- Trường nào do kubelet và control plane quan sát rồi công bố trong `status`?
- Tại sao client không nên tự ghi tùy ý vào `status`?

**PASS:** chỉ đúng `spec.podCIDR` thuộc spec; `capacity`, `allocatable`, `conditions` và
`nodeInfo` thuộc status. Hiểu rằng không phải mọi object đều có nhiều field trong `spec`.

## B5. Kubernetes API: discovery, group, version và validation

### B5.1. API server là cổng vào

```bash
kubectl get --raw /version | tee ~/lab-evidence/1a/05-api-version.json
kubectl get --raw /api | tee ~/lab-evidence/1a/05-core-api.json
kubectl get --raw /apis | head -c 1000
echo
```

`/api/v1` là core group; `/apis/<group>/<version>` là named API group.

### B5.2. Discovery và maturity level

```bash
kubectl api-versions | tee ~/lab-evidence/1a/05-api-versions.txt
kubectl api-resources --api-group='' | head -n 20
kubectl api-resources --api-group=apps
kubectl get --raw /apis/apps/v1 | head -c 1000
echo
kubectl explain node.spec
kubectl explain node.status.conditions
```

Trong output `api-versions`, tìm ví dụ:

```bash
grep -E '(^v1$|/v1$|v1beta[0-9]+$|v1alpha[0-9]+$)' \
  ~/lab-evidence/1a/05-api-versions.txt
```

Giải thích quy ước: `v1alphaN` có thể thay đổi mạnh, `v1betaN` đã chín hơn nhưng vẫn có
thể đổi, còn `v1` là stable/GA. Version là theo từng API group, không phải toàn cluster có
một maturity level duy nhất.

### B5.3. Server-side field validation

Lệnh sau **phải lỗi** và không tạo object:

```bash
kubectl apply --dry-run=server --validate=strict -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: lab-1a-invalid
spec:
  fieldThatDoesNotExist: true
EOF
```

Verify:

```bash
if kubectl get namespace lab-1a-invalid >/dev/null 2>&1; then
  echo 'FAIL: invalid object was created'
else
  echo 'PASS: invalid object does not exist'
fi

kubectl get apiservices | head -n 20
```

**PASS:** strict validation báo unknown field; namespace không tồn tại. Biết APIService là
điểm quan sát aggregation layer, nhưng lab chưa cài extension API server nên chỉ cần nhận
diện cơ chế này.

## B6. Node registration, status, condition và heartbeat

### B6.1. Quan sát Node object

```bash
kubectl describe node lab-k8s-worker2 \
  | tee ~/lab-evidence/1a/06-worker2-describe.txt

kubectl get node lab-k8s-worker2 -o jsonpath='{"UID: "}{.metadata.uid}{"\n"}{"PodCIDR: "}{.spec.podCIDR}{"\n"}{"InternalIP: "}{.status.addresses[?(@.type=="InternalIP")].address}{"\n"}{"Capacity CPU: "}{.status.capacity.cpu}{"\n"}{"Allocatable CPU: "}{.status.allocatable.cpu}{"\n"}{"Kubelet: "}{.status.nodeInfo.kubeletVersion}{"\n"}'

kubectl get node lab-k8s-worker2 -o jsonpath='{range .status.conditions[*]}{.type}{"\t"}{.status}{"\t"}{.lastHeartbeatTime}{"\t"}{.reason}{"\n"}{end}'

kubectl -n kube-node-lease get lease lab-k8s-worker2 -o yaml \
  | tee ~/lab-evidence/1a/06-worker2-lease-before.yaml
```

Node registration do kubelet thực hiện khi join; API server lưu Node object. Heartbeat nhanh
được cập nhật qua Lease, còn Node status mang Conditions và thông tin tài nguyên.

### B6.2. Fault injection: dừng kubelet trên worker 2

Terminal 1 trên master:

```bash
kubectl -n kube-node-lease get lease lab-k8s-worker2 -w
```

Terminal 2 trên worker 2:

```bash
sudo systemctl stop kubelet
systemctl is-active kubelet
```

Terminal 3 trên master:

```bash
for i in {1..18}; do
  READY="$(kubectl get node lab-k8s-worker2 -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')"
  printf '%s Ready=%s\n' "$(date -Is)" "$READY"
  if [ "$READY" != 'True' ]; then
    echo 'PASS: control plane detected missing heartbeat'
    break
  fi
  sleep 10
done

test "$READY" != 'True'
kubectl describe node lab-k8s-worker2 | sed -n '/Conditions:/,/Addresses:/p'
```

**PASS:** `renewTime` của Lease ngừng thay đổi; sau khoảng một phút Node controller đổi
Ready khỏi `True` (thường là `Unknown`, reason `NodeStatusUnknown`). Không dùng đúng số giây
như một cam kết SLA; giá trị phụ thuộc cấu hình controller.

### B6.3. Phục hồi

Trên worker 2:

```bash
sudo systemctl start kubelet
systemctl is-active kubelet
journalctl -u kubelet --since '-5 minutes' --no-pager | tail -n 50
```

Trên master:

```bash
kubectl wait --for=condition=Ready node/lab-k8s-worker2 --timeout=120s
kubectl get node lab-k8s-worker2
kubectl -n kube-node-lease get lease lab-k8s-worker2 -o jsonpath='{.spec.renewTime}{"\n"}'
```

**PASS:** kubelet `active`, worker 2 trở lại `Ready`, `renewTime` tiếp tục tăng.

## B7. Giao tiếp giữa Node và Control Plane

### B7.1. kubectl → API server

```bash
kubectl -v=8 get --raw /version 2>&1 \
  | tee ~/lab-evidence/1a/07-kubectl-to-apiserver.txt
```

**PASS:** log verbose cho thấy request HTTPS tới endpoint port `6443`; response là version
API server. `kubectl` không nói trực tiếp với etcd, scheduler hay kubelet cho thao tác này.

### B7.2. kubelet → API server

Trên worker 1:

```bash
sudo grep -n 'server:' /etc/kubernetes/kubelet.conf
sudo journalctl -u kubelet --since '-15 minutes' --no-pager \
  | grep -E 'node status|lease|registered|apiserver' \
  | tail -n 30 || true
sudo ss -tnp | grep ':6443' || true
```

Trên master, xác nhận Lease của worker 1 đổi theo thời gian:

```bash
kubectl -n kube-node-lease get lease lab-k8s-worker1 \
  -o jsonpath='{.spec.renewTime}{"\n"}'
sleep 12
kubectl -n kube-node-lease get lease lab-k8s-worker1 \
  -o jsonpath='{.spec.renewTime}{"\n"}'
```

**PASS:** kubelet.conf trỏ endpoint API server; `renewTime` lần hai mới hơn lần một. Đây là
chiều node chủ động kết nối tới control plane.

### B7.3. API server → kubelet

Từ master, yêu cầu API server proxy tới health endpoint của kubelet trên worker 1:

```bash
kubectl get --raw '/api/v1/nodes/lab-k8s-worker1/proxy/healthz'
```

**PASS:** output là `ok`. Request đi `kubectl → API server → kubelet`; client không mở
kết nối trực tiếp đến port kubelet. Cùng hướng giao tiếp này được dùng cho các thao tác như
logs, attach, exec và port-forward, sẽ thực hành sâu hơn sau khi học Pod.

### B7.4. Kết luận đường đi

Ghi ba đường đi vào `~/lab-evidence/1a/07-communication.md`:

```text
kubectl ------HTTPS:6443------> kube-apiserver
kubelet ------HTTPS:6443------> kube-apiserver
kube-apiserver --HTTPS:10250--> kubelet
```

Thêm ghi chú: trong control plane mặc định, API server là thành phần nói trực tiếp với etcd;
node và Pod không kết nối trực tiếp tới etcd. Lab không triển khai Konnectivity service nên
chỉ cần biết đó là một lựa chọn tunnel/proxy cho đường API server tới node.

## B8. Controller và reconciliation

Để không đưa Deployment/ReplicaSet của giai đoạn sau vào quá sớm, bài này dùng controller
có sẵn trong `kube-controller-manager`: ServiceAccount controller bảo đảm mỗi namespace
active có một ServiceAccount tên `default`.

### B8.1. Quan sát trạng thái ban đầu

```bash
kubectl get serviceaccount default -n lab-1a -o yaml \
  | tee ~/lab-evidence/1a/08-default-sa-before.yaml

OLD_UID="$(kubectl get serviceaccount default -n lab-1a \
  -o jsonpath='{.metadata.uid}')"
echo "old UID: $OLD_UID"
test -n "$OLD_UID"
```

### B8.2. Tạo sai lệch có chủ đích

```bash
kubectl delete serviceaccount default -n lab-1a

NEW_UID=''
for i in {1..30}; do
  NEW_UID="$(kubectl get serviceaccount default -n lab-1a \
    -o jsonpath='{.metadata.uid}' 2>/dev/null || true)"
  if [ -n "$NEW_UID" ] && [ "$NEW_UID" != "$OLD_UID" ]; then
    break
  fi
  sleep 1
done

echo "old UID: $OLD_UID"
echo "new UID: $NEW_UID"
test -n "$NEW_UID"
test "$NEW_UID" != "$OLD_UID"
```

**PASS:** object `default` xuất hiện lại với UID mới mà người học không chạy lệnh create.

Ánh xạ quan sát vào control loop:

| Bước | Điều vừa xảy ra |
| --- | --- |
| Watch/observe | Controller xem object trong API server |
| Compare | Quy tắc mong muốn có `default`; trạng thái thực tế vừa bị xóa |
| Act | Controller gửi create request tới API server |
| Repeat | Object mới có UID khác; vòng lặp tiếp tục quan sát |

Điểm phải hiểu: controller không phải một script chỉ chạy một lần; nó liên tục làm cho state
hội tụ. Controller thường thao tác qua API server. Một số controller tích hợp hạ tầng ngoài
cluster có thể gọi cloud/provider API, nhưng vẫn dùng Kubernetes API để quan sát/cập nhật
object liên quan.

## B9. Cleanup và gate cuối

Xóa duy nhất namespace của lab:

```bash
kubectl delete namespace lab-1a --wait=true --timeout=120s
kubectl get namespace lab-1a 2>&1 || true
kubectl wait --for=condition=Ready node --all --timeout=120s
kubectl get nodes
kubectl get pods -A
```

**PASS:** namespace `lab-1a` không còn; ba node `Ready`; Pod hệ thống trở lại `Running`.
Cluster đã trở về đúng trạng thái `01-cluster-ready`; không tạo snapshot mới.

---

## 3. Checkpoint 1a

Không đánh dấu lab hoàn tất chỉ vì đã chạy hết lệnh. Tự trả lời không nhìn tài liệu:

- [ ] Vẽ đúng control plane, worker và đường giao tiếp giữa các component.
- [ ] Chỉ ra component duy nhất ghi/đọc etcd trực tiếp trong kiến trúc mặc định.
- [ ] Phân biệt được static Pod control plane với systemd service kubelet/containerd.
- [ ] Mở một Node YAML và chỉ đúng ví dụ của `metadata`, `spec`, `status`.
- [ ] Giải thích core group khác named API group và đọc được `group/version/resource`.
- [ ] Giải thích alpha, beta, stable là maturity của API version, không phải trạng thái của
  toàn cluster.
- [ ] Chỉ ra `Capacity`, `Allocatable`, `Conditions`, `InternalIP`, kubelet version và Lease
  của một Node.
- [ ] Mô tả điều xảy ra từ lúc kubelet ngừng heartbeat tới khi Node không còn `Ready=True`.
- [ ] Phân biệt ba chiều `kubectl→API server`, `kubelet→API server`,
  `API server→kubelet`.
- [ ] Kể lại thí nghiệm ServiceAccount bằng bốn bước observe–compare–act–repeat.
- [ ] Cluster sạch và cả ba node `Ready` sau cleanup.

### Bài giải thích cuối cùng

Trong tối đa 10 phút, nói lại đường đi khái niệm sau:

1. Người dùng gửi một object tới API server.
2. API server xác thực/validate rồi lưu state qua etcd.
3. Controller và scheduler quan sát API, không đọc etcd trực tiếp.
4. Kubelet trên node quan sát assignment qua API server và nhờ runtime thực thi.
5. Kubelet cập nhật status/heartbeat; controller tiếp tục so trạng thái thực tế với mong muốn.

Nếu không giải thích được một mũi tên, quay lại đúng phần B tương ứng. Khi toàn bộ checkbox
được đánh dấu và phần giải thích không còn lẫn vai trò component, lab 1a hoàn tất.

Checkpoint của **cả giai đoạn 1** trong lộ trình chưa đóng ở đây: phần `kubectl apply -f
pod.yaml` chạy tới khi container lên, label selector và `-n` thuộc lab 1b; finalizer, owner
reference và garbage collection thuộc lab 1c.

---

## 4. Troubleshooting của lab này

Sự cố khi dựng môi trường nằm ở [mục 4 của Lab 00](LAB-00-MOI-TRUONG.md#4-troubleshooting-môi-trường).

| Triệu chứng | Kiểm tra | Cách xử lý trong lab |
| --- | --- | --- |
| API proxy kubelet lỗi | kubelet status, 10250, Node Ready | Khởi động kubelet; không bỏ qua lỗi TLS/auth bằng `curl -k` |
| Worker 2 không phục hồi sau B6 | `journalctl -u kubelet` | Start kubelet; nếu cần restore cả ba VM về `01-cluster-ready` |
| ServiceAccount không tạo lại ở B8 | controller-manager Pod/log | Xác nhận controller-manager Running và namespace còn Active |
| Namespace `lab-1a` xóa mãi không xong | `kubectl get ns lab-1a -o yaml` | Xem `status`; cơ chế finalizer sẽ học ở lab 1c |

---

## 5. Nguồn chính thức

- [Kubernetes — Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/)
- [Kubernetes — Components](https://kubernetes.io/docs/concepts/overview/components/)
- [Kubernetes — Objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/)
- [Kubernetes — Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/)
- [Kubernetes — Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/)
- [Kubernetes — Communication between Nodes and the Control Plane](https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/)
- [Kubernetes — Controllers](https://kubernetes.io/docs/concepts/architecture/controller/)
- [Kubernetes — ServiceAccount controller](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#serviceaccount-controller)
