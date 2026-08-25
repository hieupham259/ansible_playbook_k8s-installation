# Thực hiện rolling update trên một DaemonSet (Perform a Rolling Update on a DaemonSet)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/manage-daemon/update-daemon-set/>
>
> Trang này hướng dẫn cách thực hiện một rolling update trên một DaemonSet.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 29 — DaemonSet, Job nâng cao và thiết bị chuyên dụng](00-ALO-TRINH-ADMIN.md#giai-đoạn-29--daemonset-job-nâng-cao-và-thiết-bị-chuyên-dụng),
bài 3/8 · Kiểm chứng trực tiếp trên cluster lab: đổi `.spec.template` của một DaemonSet do bạn tạo,
theo dõi bằng `kubectl rollout status ds/<tên>` — đây là **nửa đầu của Checkpoint giai đoạn 29**,
nửa sau là bài [387](387-rollback-daemon-set-vi.md).

Đọc bài này **liền mạch với bài [387](387-rollback-daemon-set-vi.md)**: rollout ở đây tạo ra thứ mà
bài kia quay lui. Hai lưu ý khi làm trên cluster lab. Thứ nhất, manifest mẫu đặt DaemonSet vào
namespace `kube-system` — nơi đang chạy DaemonSet của CNI và kube-proxy; mọi lệnh trong bài đều kèm
`-n kube-system`, gõ nhầm tên đối tượng ở namespace đó là chạm vào hạ tầng cluster. An toàn hơn là
tạo namespace riêng cho bài tập. Thứ hai, bạn đã làm rollout và `rollout status` cho **Deployment** ở
[Lab 4a](labs/LAB-4A-REPLICASET-DEPLOYMENT-VA-ROLLOUT.md) phần B4–B5; đọc bài này với câu hỏi thường
trực "DaemonSet khác Deployment ở chỗ nào", vì các núm điều khiển trùng tên nhưng mặc định khác nhau.

**Phải hiểu ở lần đọc này:**

- Hai chiến lược ở mục *Chiến lược cập nhật của DaemonSet* và ranh giới giữa chúng: `RollingUpdate`
  là **mặc định**, tự hủy Pod cũ và tạo Pod mới có kiểm soát; `OnDelete` thì sau khi bạn sửa template
  **không có gì xảy ra** cho tới khi bạn tự tay xóa từng Pod cũ.
- Ba núm điều khiển và mặc định của chúng ở mục *Thực hiện một rolling update*: `maxUnavailable`
  (mặc định `1`), `minReadySeconds` (mặc định `0`), `maxSurge` (mặc định `0`) — cộng câu chốt ở mục
  trước: trong lúc cập nhật, **mỗi node nhiều nhất một Pod** của DaemonSet đang chạy.
- Cái gì kích hoạt rollout, theo mục *Cập nhật template của một DaemonSet*: **bất kỳ thay đổi nào**
  của `.spec.template`. Trong bài, khác biệt duy nhất giữa hai manifest là thêm khối `resources`.
- Ba cách thao tác cùng một việc: `kubectl apply` (khai báo), `kubectl edit` (mệnh lệnh), và
  `kubectl set image ds/...` khi chỉ đổi `.spec.template.spec.containers[*].image`; kiểm chiến lược
  bằng `kubectl get ds ... -o go-template` như mục *Kiểm tra chiến lược cập nhật* chỉ ra, theo dõi
  bằng `kubectl rollout status ds/...`.
- Ba nguyên nhân rollout kẹt ở mục *Khắc phục sự cố* và cách nhận ra từng cái: node cạn tài nguyên
  (so `kubectl get nodes` với `kubectl get pods -l ... -o wide` để tìm node còn thiếu Pod), rollout
  hỏng do crash loop hoặc image sai (**cách thoát là apply template mới**, không phải chờ), và lệch
  đồng hồ giữa master và node làm hỏng phép đo `minReadySeconds`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Trước khi bạn bắt đầu* — minikube và ba playground | Lộ trình cấm minikube, kind và cluster dùng chung | Bỏ hẳn — dùng ba VM của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) |
| Bản thân workload fluentd/Elasticsearch và volume `/var/lib/docker/containers` | Chỉ là ví dụ minh họa; đường dẫn kia là di sản của runtime cũ | [Giai đoạn 27 — Di chuyển khỏi dockershim](00-ALO-TRINH-ADMIN.md#giai-đoạn-27--di-chuyển-khỏi-dockershim-cluster-cũ) đã kết luận cluster lab dùng containerd, nên đường dẫn đó không phải chỗ chứa log container của bạn |
| Lời khuyên "xóa bớt Pod không thuộc DaemonSet để lấy chỗ" và ghi chú kèm theo ở mục *Một số node cạn kiệt tài nguyên* | Đây là thao tác gây gián đoạn và **không** tôn trọng PodDisruptionBudget | Cơ chế PDB đã học ở bài [53](53-disruptions-vi.md) và [339](339-configure-pdb-vi.md); cách rút Pod khỏi node đúng quy trình ở bài [255](255-safely-drain-node-vi.md), giai đoạn 16 |

---

Trang này hướng dẫn cách thực hiện một rolling update trên một DaemonSet.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Chiến lược cập nhật của DaemonSet (DaemonSet Update Strategy)

DaemonSet có hai kiểu chiến lược cập nhật (update strategy):

* `OnDelete`: Với chiến lược cập nhật `OnDelete`, sau khi bạn cập nhật template của một
  DaemonSet, các pod mới của DaemonSet *chỉ* được tạo ra khi bạn xóa thủ công các pod cũ của
  DaemonSet. Đây cũng chính là hành vi của DaemonSet trong Kubernetes phiên bản 1.5 trở về
  trước.
* `RollingUpdate`: Đây là chiến lược cập nhật mặc định.
  Với chiến lược cập nhật `RollingUpdate`, sau khi bạn cập nhật template của một DaemonSet,
  các pod cũ của DaemonSet sẽ bị hủy, và các pod mới của DaemonSet sẽ được tạo ra tự động,
  theo một cách có kiểm soát. Trong suốt quá trình cập nhật, trên mỗi node sẽ có nhiều nhất
  một pod của DaemonSet đang chạy.

## Thực hiện một rolling update (Performing a Rolling Update)

Để bật tính năng rolling update của một DaemonSet, bạn phải đặt `.spec.updateStrategy.type`
của nó thành `RollingUpdate`.

Bạn có thể cũng muốn đặt thêm
[`.spec.updateStrategy.rollingUpdate.maxUnavailable`](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/daemon-set-v1/#DaemonSetSpec)
(mặc định là 1),
[`.spec.minReadySeconds`](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/daemon-set-v1/#DaemonSetSpec)
(mặc định là 0) và
[`.spec.updateStrategy.rollingUpdate.maxSurge`](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/daemon-set-v1/#DaemonSetSpec)
(mặc định là 0).

### Tạo một DaemonSet với chiến lược cập nhật `RollingUpdate` (Creating a DaemonSet with `RollingUpdate` update strategy)

File YAML sau đây khai báo một DaemonSet có chiến lược cập nhật là 'RollingUpdate'

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # các toleration này để daemonset có thể chạy được trên các node control plane
      # hãy xóa chúng nếu các node control plane của bạn không nên chạy pod
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v5.0.1
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

Sau khi đã kiểm tra chiến lược cập nhật trong manifest của DaemonSet, hãy tạo DaemonSet:

```shell
kubectl create -f https://k8s.io/examples/controllers/fluentd-daemonset.yaml
```

Ngoài ra, bạn có thể dùng `kubectl apply` để tạo cùng DaemonSet đó nếu bạn dự định sẽ cập nhật
DaemonSet bằng `kubectl apply`.

```shell
kubectl apply -f https://k8s.io/examples/controllers/fluentd-daemonset.yaml
```

### Kiểm tra chiến lược cập nhật `RollingUpdate` của DaemonSet (Checking DaemonSet `RollingUpdate` update strategy)

Hãy kiểm tra chiến lược cập nhật của DaemonSet, và đảm bảo rằng nó được đặt thành
`RollingUpdate`:

```shell
kubectl get ds/fluentd-elasticsearch -o go-template='{{.spec.updateStrategy.type}}{{"\n"}}' -n kube-system
```

Nếu bạn chưa tạo DaemonSet trong hệ thống, hãy dùng lệnh sau để kiểm tra manifest của
DaemonSet thay thế:

```shell
kubectl apply -f https://k8s.io/examples/controllers/fluentd-daemonset.yaml --dry-run=client -o go-template='{{.spec.updateStrategy.type}}{{"\n"}}'
```

Kết quả của cả hai lệnh đều phải là:

```
RollingUpdate
```

Nếu kết quả không phải là `RollingUpdate`, hãy quay lại và sửa đối tượng DaemonSet hoặc
manifest cho phù hợp.


### Cập nhật template của một DaemonSet (Updating a DaemonSet template)

Bất kỳ thay đổi nào đối với `.spec.template` của một DaemonSet dùng `RollingUpdate` đều sẽ
kích hoạt một rolling update. Hãy cập nhật DaemonSet bằng cách apply một file YAML mới. Việc
này có thể thực hiện bằng vài lệnh `kubectl` khác nhau.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # các toleration này để daemonset có thể chạy được trên các node control plane
      # hãy xóa chúng nếu các node control plane của bạn không nên chạy pod
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v5.0.1
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

#### Lệnh khai báo (Declarative commands)

Nếu bạn cập nhật DaemonSet bằng [file cấu hình](319-declarative-config-vi.md), hãy dùng
`kubectl apply`:

```shell
kubectl apply -f https://k8s.io/examples/controllers/fluentd-daemonset-update.yaml
```

#### Lệnh mệnh lệnh (Imperative commands)

Nếu bạn cập nhật DaemonSet bằng
[lệnh mệnh lệnh (imperative commands)](320-imperative-command-vi.md), hãy dùng `kubectl edit`:

```shell
kubectl edit ds/fluentd-elasticsearch -n kube-system
```

##### Chỉ cập nhật container image (Updating only the container image)

Nếu bạn chỉ cần cập nhật container image trong template của DaemonSet, tức là
`.spec.template.spec.containers[*].image`, hãy dùng `kubectl set image`:

```shell
kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=quay.io/fluentd_elasticsearch/fluentd:v2.6.0 -n kube-system
```

### Theo dõi trạng thái của rolling update (Watching the rolling update status)

Cuối cùng, hãy theo dõi trạng thái rollout của lần rolling update DaemonSet mới nhất:

```shell
kubectl rollout status ds/fluentd-elasticsearch -n kube-system
```

Khi rollout hoàn tất, kết quả sẽ tương tự như sau:

```shell
daemonset "fluentd-elasticsearch" successfully rolled out
```

## Khắc phục sự cố (Troubleshooting)

### Rolling update của DaemonSet bị kẹt (DaemonSet rolling update is stuck)

Đôi khi, một rolling update của DaemonSet có thể bị kẹt. Dưới đây là một số nguyên nhân có
thể xảy ra:

#### Một số node cạn kiệt tài nguyên (Some nodes run out of resources)

Rollout bị kẹt vì các pod mới của DaemonSet không thể được lập lịch (schedule) trên ít nhất
một node. Điều này có thể xảy ra khi node đang
[cạn kiệt tài nguyên](142-node-pressure-eviction-vi.md).

Khi điều này xảy ra, hãy tìm ra các node chưa có pod của DaemonSet được lập lịch lên bằng
cách so sánh kết quả của `kubectl get nodes` với kết quả của lệnh:

```shell
kubectl get pods -l name=fluentd-elasticsearch -o wide -n kube-system
```

Khi đã tìm ra các node đó, hãy xóa bớt một số pod không thuộc DaemonSet khỏi node để tạo chỗ
trống cho các pod mới của DaemonSet.

> **Ghi chú:** Việc này sẽ gây gián đoạn dịch vụ khi các pod bị xóa không được điều khiển bởi
> controller nào hoặc các pod đó không được nhân bản (replicate). Nó cũng không tôn trọng
> [PodDisruptionBudget](339-configure-pdb-vi.md).

#### Rollout hỏng (Broken rollout)

Nếu bản cập nhật template DaemonSet gần đây bị lỗi, ví dụ container rơi vào vòng lặp crash
(crash looping), hoặc container image không tồn tại (thường là do gõ sai), rollout của
DaemonSet sẽ không tiến triển.

Để khắc phục, hãy cập nhật lại template của DaemonSet một lần nữa. Rollout mới sẽ không bị
chặn bởi các rollout không lành mạnh trước đó.

#### Lệch đồng hồ (Clock skew)

Nếu `.spec.minReadySeconds` được chỉ định trong DaemonSet, việc lệch đồng hồ (clock skew)
giữa master và các node sẽ khiến DaemonSet không thể phát hiện đúng tiến độ rollout.

## Dọn dẹp (Clean up)

Xóa DaemonSet khỏi một namespace:

```shell
kubectl delete ds fluentd-elasticsearch -n kube-system
```

## Tiếp theo (What's next)

* Xem [Thực hiện rollback trên một DaemonSet (Performing a rollback on a DaemonSet)](387-rollback-daemon-set-vi.md)
* Xem [Tạo một DaemonSet để nhận (adopt) các pod DaemonSet đã có](66-daemonset-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 29:

1. **Câu bẫy.** Trong lúc rolling update một DaemonSet, trên **một node** có thể có nhiều nhất bao
   nhiêu Pod của DaemonSet đó cùng chạy? Giá trị mặc định của `maxSurge` là gì, và nó dẫn tới hệ quả
   nào cho tính liên tục của dịch vụ trên node đang được cập nhật?
2. Bạn apply manifest `fluentd-elasticsearch` của bài lên cluster lab ba node.
   `kubectl get pods -n kube-system -l name=fluentd-elasticsearch -o wide` cho mấy dòng? Vì sao con
   số này khác con số bạn thấy với `example-daemonset` ở bài [385](385-create-daemon-set-vi.md)?
3. Hai manifest trong bài khác nhau đúng một chỗ: bản sau thêm khối `resources` cho container. Việc
   đó có kích hoạt rolling update không? Còn nếu bạn chỉ sửa `metadata.labels` của chính DaemonSet
   (`k8s-app: fluentd-logging`) thì sao?
4. Bạn đổi image sang một tag gõ sai. `kubectl rollout status` đứng im. Bạn chờ, hay làm gì — và làm
   sao phân biệt tình huống này với tình huống "node cạn kiệt tài nguyên" cũng làm rollout kẹt?
5. DaemonSet của bạn đang để `updateStrategy.type: OnDelete`. Bạn sửa `.spec.template` rồi chạy
   `kubectl rollout status`. Điều gì xảy ra với các Pod, và bạn phải làm gì để bản mới lên node?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Nhiều nhất một** — bài nói thẳng điều đó ở mục *Chiến lược cập nhật của DaemonSet*, và
   `maxSurge` **mặc định là `0`** chính là cách nói cùng một việc bằng con số: không có Pod thứ hai
   được dựng trước trên cùng node. Hệ quả là chỗ dễ nhầm: rolling update của DaemonSet **không phải
   là không gián đoạn** trên từng node — Pod cũ phải bị hủy trước, Pod mới mới được tạo, nên node đó
   có một khoảng trống không có Pod của DaemonSet. Trực giác "rolling update thì luôn còn bản cũ
   chạy đỡ" đến từ Deployment, nơi nhiều replica nằm rải trên nhiều node để bù cho nhau; DaemonSet
   chỉ có đúng một Pod cho mỗi node nên không có gì để bù.
2. **Ba dòng** — cả `lab-k8s-master`, `lab-k8s-worker1` và `lab-k8s-worker2`. Khác với
   `example-daemonset` (hai Pod) vì manifest ở bài này **có khai `tolerations`** cho
   `node-role.kubernetes.io/control-plane` và `node-role.kubernetes.io/master` với
   `effect: NoSchedule`, nên taint của control plane không loại `lab-k8s-master` ra nữa. Chính
   comment trong manifest nói rõ: xóa hai toleration đó đi nếu control plane của bạn không nên chạy
   Pod.
3. **Có.** Bài nêu quy tắc gọn: **bất kỳ thay đổi nào đối với `.spec.template`** của một DaemonSet
   dùng `RollingUpdate` đều kích hoạt một rolling update — thêm `resources` là sửa `.spec.template`,
   nên nó tính. Còn `metadata.labels` của chính đối tượng DaemonSet **nằm ngoài `.spec.template`**,
   nên nó không thuộc phạm vi quy tắc đó; muốn Pod đổi thì phải đụng vào template.
4. **Sửa template một lần nữa rồi apply lại** — bài nói rõ: rollout mới **không bị chặn** bởi các
   rollout không lành mạnh trước đó, nên không cần dọn dẹp gì trước. Chờ là vô ích vì image không tồn
   tại thì Pod không bao giờ chạy được. Phân biệt bằng **chỗ Pod bị kẹt**: rollout hỏng thì Pod mới
   **đã được lập lịch** lên node nhưng crash hoặc không kéo được image; cạn tài nguyên thì Pod mới
   **không lập lịch được** trên ít nhất một node — so `kubectl get nodes` với
   `kubectl get pods -l name=fluentd-elasticsearch -o wide -n kube-system` sẽ lộ ra node nào đang
   thiếu hẳn Pod.
5. **Không Pod nào đổi.** Với `OnDelete`, sau khi template được cập nhật, Pod mới *chỉ* được tạo khi
   bạn **xóa thủ công** Pod cũ — đây đúng là hành vi DaemonSet của Kubernetes 1.5 trở về trước. Muốn
   bản mới lên node nào thì xóa Pod của DaemonSet trên node đó; đổi lại, bạn kiểm soát hoàn toàn thứ
   tự và thời điểm.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
