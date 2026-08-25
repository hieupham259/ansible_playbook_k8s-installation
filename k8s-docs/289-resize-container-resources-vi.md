# Thay đổi kích thước tài nguyên CPU và Memory được gán cho Container (Resize CPU and Memory Resources assigned to Containers)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 3 → nhóm [3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn), bài 6/9 ·
Kiểm chứng ở [Lab 3c](labs/LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md) phần B6.

Bài dài và trộn lẫn hai loại nội dung: phần cơ chế bạn phải nắm chắc, và phần chi tiết vận hành
(thứ tự thử lại, `observedGeneration`) chỉ dùng khi gỡ rối. Đọc theo bảng dưới, đừng đọc tuần tự
đều tay từ đầu tới cuối.

Bài nối thẳng vào bài [288](288-quality-service-pod-vi.md) vừa đọc: chính ở đây mới thấy vì sao
QoS class **bất biến** lại trở thành một luật chặn cụ thể khi resize.

**Phải hiểu ở lần đọc này:**

- Ba trường phải phân biệt, mục *Các khái niệm chính*: `spec.containers[*].resources` là tài nguyên
  **mong muốn** và sửa được với CPU/memory; `status.containerStatuses[*].resources` là tài nguyên
  **thực tế đang được cấu hình**; `allocatedResources` là trường nội bộ cho logic lập lịch, không
  phải chỗ để kiểm chứng.
- Cách kích hoạt: sửa `requests`/`limits` qua **subresource `resize`**, ví dụ
  `kubectl patch ... --subresource resize` (cần client `kubectl` từ v1.32; phiên bản cũ báo
  `invalid subresource`).
- Mục *Chính sách resize của container*: `resizePolicy` đặt theo **từng loại tài nguyên** —
  `NotRequired` (mặc định) áp dụng tại chỗ không khởi động lại, `RestartContainer` khởi động lại
  container. Kịch bản ví dụ nói rõ: đổi cả CPU lẫn memory cùng lúc thì **chính sách đòi khởi động
  lại thắng**. Bằng chứng để đọc là `status.containerStatuses[*].restartCount` (Ví dụ 1 giữ `0`,
  Ví dụ 2 lên `1`).
- Mục *Trạng thái resize của Pod* và *Xử lý sự cố*: `PodResizePending` với `reason: Infeasible`
  (node không đủ, không tự hết) hoặc `reason: Deferred` (chưa được nhưng sẽ thử lại);
  `PodResizeInProgress` khi đang áp dụng. Điểm mấu chốt của một lần resize bất khả thi: `spec` đổi
  theo giá trị mong muốn nhưng `status.containerStatuses[*].resources` **vẫn giữ giá trị cũ** và
  `restartCount` không tăng.
- Ranh giới cứng, mục *Các hạn chế*: chỉ resize được **CPU và memory**; **QoS class ban đầu không
  đổi được** (Guaranteed phải giữ request bằng limit, Burstable không được để request bằng limit ở
  cả hai loại, BestEffort không được thêm tài nguyên); request/limit đã đặt thì **không gỡ bỏ được**,
  chỉ đổi giá trị; init container không khởi động lại được và ephemeral container không resize được,
  còn sidecar thì resize được.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Câu mở đầu nói Pod thay thế "thường được quản lý bởi một workload controller" | chưa học controller nào; ở nhóm 3c mọi thứ vẫn là Pod trần | [giai đoạn 4](00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller) |
| Mục *Cách kubelet thử lại các resize bị Deferred* — thứ tự ưu tiên theo PriorityClass rồi tới QoS | cần biết PriorityClass trước mới hiểu vế đầu của thứ tự | bài [141](141-pod-priority-preemption-vi.md) ở [giai đoạn 7a](00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction) |
| Mục *Tận dụng các trường `observedGeneration`* | công cụ đối chiếu chi tiết khi một lần resize treo; Lab 3c B6 chỉ dùng `restartCount` và `status...resources` | [giai đoạn 24](00-ALO-TRINH-ADMIN.md#giai-đoạn-24--xử-lý-sự-cố) |
| Trong *Các hạn chế*: chính sách static của CPU/Memory manager, swap, Pod Windows | phụ thuộc cấu hình kubelet ở mức node mà baseline Lab 00 không bật | bài [74](74-resource-managers-vi.md) ở [giai đoạn 7b](00-ALO-TRINH-ADMIN.md#7b-chính-sách-giới-hạn-tài-nguyên) và bài [200](200-cpu-management-policies-vi.md) ở [giai đoạn 25](00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [stable]`

Trang này giải thích cách thay đổi các request và limit tài nguyên CPU và memory được gán cho
một container *mà không cần tạo lại Pod*.

Theo cách truyền thống, việc thay đổi yêu cầu tài nguyên của một Pod đòi hỏi phải xóa Pod hiện
có và tạo một Pod thay thế, thường được quản lý bởi một
[workload controller](./62-controllers-index-vi.md). Tính năng thay đổi kích thước Pod tại chỗ
(In-place Pod Resize) cho phép thay đổi lượng CPU/memory cấp phát cho các container bên trong
một Pod đang chạy, đồng thời có khả năng tránh được gián đoạn ứng dụng. Quy trình thay đổi kích
thước tài nguyên ở cấp Pod được trình bày trong
[Thay đổi kích thước tài nguyên CPU và Memory được gán cho Pod](290-resize-pod-resources-vi.md).

**Các khái niệm chính:**

* **Tài nguyên mong muốn (Desired Resources):** trường `spec.containers[*].resources` của một
  container biểu diễn tài nguyên *mong muốn* cho container đó, và có thể thay đổi được
  (mutable) đối với CPU và memory.
* **Tài nguyên thực tế (Actual Resources):** trường `status.containerStatuses[*].resources`
  phản ánh tài nguyên *hiện đang được cấu hình* cho một container đang chạy. Với các container
  chưa khởi động hoặc đã bị khởi động lại, trường này phản ánh tài nguyên được cấp phát cho lần
  khởi động kế tiếp của chúng.
* **Kích hoạt một lần resize (Triggering a Resize):** Bạn có thể yêu cầu thay đổi kích thước
  bằng cách cập nhật `requests` và `limits` mong muốn trong đặc tả (specification) của Pod.
  Việc này thường được thực hiện bằng `kubectl patch`, `kubectl apply` hoặc `kubectl edit`
  nhắm vào subresource `resize` của Pod. Khi tài nguyên mong muốn không khớp với tài nguyên đã
  cấp phát, Kubelet sẽ cố gắng thay đổi kích thước container.
* **Tài nguyên đã cấp phát (Allocated Resources — nâng cao):** trường
  `status.containerStatuses[*].allocatedResources` theo dõi các giá trị tài nguyên đã được
  Kubelet xác nhận, chủ yếu dùng cho logic lập lịch nội bộ. Với hầu hết mục đích giám sát và
  kiểm chứng, hãy tập trung vào `status.containerStatuses[*].resources`.

Nếu một node có các Pod với một lần resize đang chờ hoặc chưa hoàn tất (xem
[Trạng thái resize của Pod](#pod-resize-status) bên dưới), scheduler sẽ dùng giá trị *lớn nhất*
trong số request mong muốn, request đã cấp phát và request thực tế lấy từ status của container
khi đưa ra quyết định lập lịch.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản 1.33 trở lên. Để kiểm tra phiên bản, hãy nhập
`kubectl version`.

[Feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`InPlacePodVerticalScaling` phải được bật cho control plane và cho tất cả các node trong
cluster của bạn.

Phiên bản client `kubectl` phải tối thiểu là v1.32 để dùng được cờ `--subresource=resize`.

## Trạng thái resize của Pod (Pod resize status) {#pod-resize-status}

Kubelet cập nhật các condition trong status của Pod để chỉ ra trạng thái của một yêu cầu
resize:

* `type: PodResizePending`: Kubelet không thể đáp ứng yêu cầu ngay lập tức. Trường `message`
  giải thích lý do vì sao.
    * `reason: Infeasible`: yêu cầu resize là bất khả thi trên node hiện tại (ví dụ: yêu cầu
      nhiều tài nguyên hơn mức node đang có).
    * `reason: Deferred`: yêu cầu resize hiện chưa thể thực hiện, nhưng có thể trở nên khả thi
      sau này (ví dụ khi một Pod khác bị xóa). Kubelet sẽ thử lại việc resize.
* `type: PodResizeInProgress`: Kubelet đã chấp nhận resize và đã cấp phát tài nguyên, nhưng các
  thay đổi vẫn đang được áp dụng. Trạng thái này thường rất ngắn nhưng cũng có thể kéo dài hơn
  tùy loại tài nguyên và hành vi của runtime. Mọi lỗi trong quá trình thực thi được báo cáo
  trong trường `message` (kèm theo `reason: Error`).

### Cách kubelet thử lại các resize bị Deferred (How kubelet retries Deferred resizes)

Nếu yêu cầu resize bị _Deferred_ (hoãn lại), kubelet sẽ định kỳ thử lại việc resize, ví dụ khi
một Pod khác bị xóa hoặc bị thu nhỏ. Nếu có nhiều resize bị hoãn, chúng được thử lại theo thứ
tự ưu tiên sau:

* Pod có Priority cao hơn (dựa trên PriorityClass) sẽ được thử lại yêu cầu resize trước.
* Nếu hai Pod có cùng Priority, resize của các Pod thuộc lớp Guaranteed sẽ được thử lại trước
  resize của các Pod thuộc lớp Burstable.
* Nếu mọi yếu tố khác đều như nhau, Pod nào ở trạng thái Deferred lâu hơn sẽ được ưu tiên.

Một resize có độ ưu tiên cao hơn bị đánh dấu là pending sẽ không chặn các resize pending còn
lại được thử; tất cả các resize pending còn lại vẫn sẽ được thử lại ngay cả khi một resize có
độ ưu tiên cao hơn tiếp tục bị hoãn.

### Tận dụng các trường `observedGeneration` (Leveraging `observedGeneration` Fields)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [stable]`

* Trường `status.observedGeneration` ở cấp cao nhất cho biết `metadata.generation` tương ứng
  với đặc tả Pod mới nhất mà kubelet đã ghi nhận. Bạn có thể dùng trường này để xác định yêu
  cầu resize gần nhất mà kubelet đã xử lý.
* Trong condition `PodResizeInProgress`, trường `conditions[].observedGeneration` cho biết
  `metadata.generation` của podSpec tại thời điểm lần resize đang diễn ra được khởi động.
* Trong condition `PodResizePending`, trường `conditions[].observedGeneration` cho biết
  `metadata.generation` của podSpec tại thời điểm việc cấp phát cho lần resize đang chờ được
  thử lần gần nhất.

## Chính sách resize của container (Container resize policies) {#container-resize-policies}

Bạn có thể kiểm soát việc một container có bị khởi động lại khi resize hay không bằng cách đặt
`resizePolicy` trong đặc tả của container. Điều này cho phép kiểm soát chi tiết theo từng loại
tài nguyên (CPU hoặc memory).

```yaml
    resizePolicy:
    - resourceName: cpu
      restartPolicy: NotRequired
    - resourceName: memory
      restartPolicy: RestartContainer
```

* `NotRequired`: (Mặc định) Áp dụng thay đổi tài nguyên cho container đang chạy mà không khởi
  động lại nó.
* `RestartContainer`: Khởi động lại container để áp dụng các giá trị tài nguyên mới. Điều này
  thường cần thiết đối với các thay đổi về memory vì nhiều ứng dụng và runtime không thể điều
  chỉnh lượng memory đã cấp phát của chúng một cách động.

Nếu `resizePolicy[*].restartPolicy` không được chỉ định cho một loại tài nguyên, giá trị mặc
định là `NotRequired`.

> **Ghi chú:** Nếu `restartPolicy` chung của Pod là `Never`, thì `resizePolicy` của mọi
> container phải là `NotRequired` cho tất cả các loại tài nguyên. Bạn không thể cấu hình một
> chính sách resize đòi hỏi khởi động lại trong những Pod như vậy.

**Kịch bản ví dụ:**

Xét một container được cấu hình `restartPolicy: NotRequired` cho CPU và
`restartPolicy: RestartContainer` cho memory.
* Nếu chỉ thay đổi tài nguyên CPU, container được resize tại chỗ.
* Nếu chỉ thay đổi tài nguyên memory, container bị khởi động lại.
* Nếu thay đổi *cả* CPU lẫn memory cùng lúc, container bị khởi động lại (do chính sách của
  memory).

## Các hạn chế (Limitations) {#limitations}

Đối với Kubernetes v1.36, việc thay đổi kích thước tài nguyên Pod tại chỗ có các hạn chế sau:

* **Loại tài nguyên:** Chỉ có thể resize tài nguyên CPU và memory.
* **Giảm memory:** Nếu chính sách khởi động lại khi resize memory là `NotRequired` (hoặc không
  được chỉ định), kubelet sẽ cố gắng ở mức tốt nhất có thể (best-effort) để tránh oom-kill khi
  giảm memory limit, nhưng không đưa ra bất kỳ bảo đảm nào. Trước khi giảm memory limit của
  container, nếu mức sử dụng memory vượt quá limit được yêu cầu, việc resize sẽ bị bỏ qua và
  trạng thái sẽ giữ nguyên là "In Progress". Đây được coi là best-effort vì nó vẫn chịu ảnh
  hưởng của tình huống tranh chấp (race condition), khi mức sử dụng memory có thể tăng vọt ngay
  sau thời điểm kiểm tra.
* **QoS Class:** [Lớp Chất lượng dịch vụ (QoS)](./54-pod-qos-vi.md) ban đầu của Pod
  (Guaranteed, Burstable hoặc BestEffort) được xác định lúc tạo và **không thể** bị thay đổi
  bởi một lần resize. Các giá trị tài nguyên sau khi resize vẫn phải tuân theo quy tắc của QoS
  class ban đầu:
    * *Guaranteed*: Request phải tiếp tục bằng limit cho cả CPU lẫn memory sau khi resize.
    * *Burstable*: Request và limit không được trở nên bằng nhau cho *cả* CPU lẫn memory cùng
      lúc (vì điều đó sẽ biến Pod thành Guaranteed).
    * *BestEffort*: Không được thêm yêu cầu tài nguyên (`requests` hoặc `limits`) (vì điều đó
      sẽ biến Pod thành Burstable hoặc Guaranteed).
* **Loại container:** Các init container không thể khởi động lại (non-restartable) và các
  ephemeral container không thể được resize.
  [Sidecar container](./51-sidecar-containers-vi.md) có thể được resize.
* **Gỡ bỏ tài nguyên:** Request và limit tài nguyên không thể bị gỡ bỏ hoàn toàn một khi đã
  đặt; chúng chỉ có thể được đổi sang giá trị khác.
* **Hệ điều hành:** Các Pod trên Windows không hỗ trợ resize tại chỗ.
* **Chính sách của node:** Các Pod được quản lý bởi
  [chính sách static của CPU hoặc Memory manager](./200-cpu-management-policies-vi.md)
  không thể được resize tại chỗ.
* **Swap:** Các Pod sử dụng
  [swap memory](https://kubernetes.io/docs/concepts/architecture/nodes#swap-memory)
  không thể resize memory request trừ khi `resizePolicy` cho memory là `RestartContainer`.

Các hạn chế này có thể được nới lỏng trong các phiên bản Kubernetes tương lai.

## Tạo một namespace (Create a namespace)

Tạo một namespace để các tài nguyên bạn tạo trong bài thực hành này được cô lập khỏi phần còn
lại của cluster.

```shell
kubectl create namespace qos-example
```

## Ví dụ 1: Resize CPU không cần khởi động lại (Example 1: Resizing CPU without restart)

Trước tiên, tạo một Pod được thiết kế để resize CPU tại chỗ và resize memory có yêu cầu khởi
động lại.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resize-demo
  namespace: qos-example
spec:
  containers:
  - name: pause
    image: registry.k8s.io/pause:3.8
    resizePolicy:
    - resourceName: cpu
      restartPolicy: NotRequired # Mặc định, nhưng ghi tường minh ở đây
    - resourceName: memory
      restartPolicy: RestartContainer
    resources:
      limits:
        memory: "200Mi"
        cpu: "700m"
      requests:
        memory: "200Mi"
        cpu: "700m"
```

Tạo Pod:

```shell
kubectl create -f pod-resize.yaml -n qos-example
```

Pod này khởi đầu trong QoS class Guaranteed. Xác minh trạng thái ban đầu của nó:

```shell
# Chờ một lát để Pod chuyển sang trạng thái chạy
kubectl get pod resize-demo --output=yaml -n qos-example
```

Quan sát `spec.containers[0].resources` và `status.containerStatuses[0].resources`. Chúng phải
khớp với manifest (700m CPU, 200Mi memory). Ghi nhận giá trị
`status.containerStatuses[0].restartCount` (phải là 0).

Bây giờ, tăng CPU request và limit lên `800m`. Bạn dùng `kubectl patch` với đối số dòng lệnh
`--subresource resize`.

```shell
kubectl patch pod resize-demo -n qos-example --subresource resize --patch \
  '{"spec":{"containers":[{"name":"pause", "resources":{"requests":{"cpu":"800m"}, "limits":{"cpu":"800m"}}}]}}'

# Các cách khác:
# kubectl -n qos-example edit pod resize-demo --subresource resize
# kubectl -n qos-example apply -f <updated-manifest> --subresource resize --server-side
```

> **Ghi chú:** Đối số dòng lệnh `--subresource resize` yêu cầu client `kubectl` phiên bản
> v1.32.0 trở lên. Các phiên bản cũ hơn sẽ báo lỗi `invalid subresource`.

Kiểm tra lại trạng thái Pod sau khi patch:

```shell
kubectl get pod resize-demo --output=yaml --namespace=qos-example
```

Bạn sẽ thấy:
* `spec.containers[0].resources` giờ hiển thị `cpu: 800m`.
* `status.containerStatuses[0].resources` cũng hiển thị `cpu: 800m`, cho thấy việc resize đã
  thành công trên node.
* `status.containerStatuses[0].restartCount` vẫn là `0`, vì `resizePolicy` của CPU là
  `NotRequired`.

## Ví dụ 2: Resize memory kèm khởi động lại (Example 2: Resizing memory with restart)

Bây giờ, resize memory cho *chính* Pod đó bằng cách tăng nó lên `300Mi`. Vì `resizePolicy` của
memory là `RestartContainer`, container được kỳ vọng sẽ khởi động lại.

```shell
kubectl patch pod resize-demo -n qos-example --subresource resize --patch \
  '{"spec":{"containers":[{"name":"pause", "resources":{"requests":{"memory":"300Mi"}, "limits":{"memory":"300Mi"}}}]}}'
```

Kiểm tra trạng thái Pod ngay sau khi patch:

```shell
kubectl get pod resize-demo --output=yaml --namespace=qos-example
```

Lúc này bạn sẽ quan sát thấy:
* `spec.containers[0].resources` hiển thị `memory: 300Mi`.
* `status.containerStatuses[0].resources` cũng hiển thị `memory: 300Mi`.
* `status.containerStatuses[0].restartCount` đã tăng lên `1` (hoặc nhiều hơn, nếu trước đó đã
  có các lần khởi động lại), cho thấy container đã bị khởi động lại để áp dụng thay đổi memory.

## Xử lý sự cố: Yêu cầu resize bất khả thi (Troubleshooting: Infeasible resize request)

Tiếp theo, thử yêu cầu một lượng CPU phi lý, chẳng hạn 1000 core đầy đủ (viết là `"1000"` thay
vì `"1000m"` theo đơn vị millicore), lượng này nhiều khả năng vượt quá dung lượng của node.

```shell
# Thử patch với một yêu cầu CPU lớn quá mức
kubectl patch pod resize-demo -n qos-example --subresource resize --patch \
  '{"spec":{"containers":[{"name":"pause", "resources":{"requests":{"cpu":"1000"}, "limits":{"cpu":"1000"}}}]}}'
```

Truy vấn thông tin chi tiết của Pod:

```shell
kubectl get pod resize-demo --output=yaml --namespace=qos-example
```

Bạn sẽ thấy các thay đổi chỉ ra vấn đề:

* `spec.containers[0].resources` phản ánh trạng thái *mong muốn* (`cpu: "1000"`).
* Một condition với `type: PodResizePending` và `reason: Infeasible` đã được thêm vào Pod.
* Trường `message` của condition sẽ giải thích lý do
  (`Node didn't have enough capacity: cpu, requested: 800000, capacity: ...`)
* Điểm quan trọng nhất: `status.containerStatuses[0].resources` sẽ *vẫn hiển thị các giá trị
  trước đó* (`cpu: 800m`, `memory: 300Mi`), vì lần resize bất khả thi này không được Kubelet
  áp dụng.
* `restartCount` sẽ không thay đổi do lần thử thất bại này.

Để khắc phục, bạn cần patch Pod một lần nữa với các giá trị tài nguyên khả thi.

## Dọn dẹp (Clean up)

Xóa namespace của bạn. Thao tác này xóa tất cả các Pod mà bạn đã tạo cho bài thực hành này:

```shell
kubectl delete namespace qos-example
```

## Tiếp theo (What's next)

### Dành cho nhà phát triển ứng dụng (For application developers)

* [Gán tài nguyên Memory cho Container và Pod](./264-assign-memory-resource-vi.md)

* [Gán tài nguyên CPU cho Container và Pod](./263-assign-cpu-resource-vi.md)

* [Gán tài nguyên CPU và memory ở cấp Pod](./265-assign-pod-level-resources-vi.md)

### Dành cho quản trị viên cluster (For cluster administrators)

* [Cấu hình Memory Request và Limit mặc định cho một Namespace](./232-memory-default-namespace-vi.md)

* [Cấu hình CPU Request và Limit mặc định cho một Namespace](./230-cpu-default-namespace-vi.md)

* [Cấu hình ràng buộc Memory tối thiểu và tối đa cho một Namespace](./231-memory-constraint-namespace-vi.md)

* [Cấu hình ràng buộc CPU tối thiểu và tối đa cho một Namespace](./229-cpu-constraint-namespace-vi.md)

* [Cấu hình Quota Memory và CPU cho một Namespace](./233-quota-memory-cpu-namespace-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở nhóm 3c:

1. `lab-k8s-worker2` có 2 vCPU. Một Pod đang chạy trên đó và bạn patch container của nó lên
   `cpu: "1000"`. So sánh `spec.containers[0].resources` với
   `status.containerStatuses[0].resources` sau lệnh patch — hai chỗ đó nói gì, `restartCount` thay
   đổi ra sao, và condition nào xuất hiện kèm `reason` gì?
2. **Câu bẫy.** `resizePolicy` của memory là `RestartContainer`, bạn patch memory và container khởi
   động lại. Vậy Pod có bị tạo lại, đổi tên hay đổi node không?
3. Một Pod `Burstable` có `requests.cpu: 100m` / `limits.cpu: 500m` và
   `requests.memory: 100Mi` / `limits.memory: 200Mi`. Bạn muốn patch cho request bằng limit ở **cả
   hai** loại. Yêu cầu đó có được chấp nhận không, và vì sao?
4. `Infeasible` và `Deferred` khác nhau ở chỗ nào? Cái nào tự khỏi theo thời gian?
5. Container đặt `NotRequired` cho CPU và `RestartContainer` cho memory. Bạn patch **cả** CPU lẫn
   memory trong cùng một lệnh. Container có bị khởi động lại không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. `spec` **đổi theo giá trị mong muốn** (`cpu: "1000"`), nhưng
   `status.containerStatuses[0].resources` **vẫn giữ nguyên giá trị cũ** vì kubelet không áp dụng
   được lần resize này. `restartCount` **không đổi** — lần thử thất bại không khởi động lại gì.
   Condition thêm vào là **`PodResizePending` với `reason: Infeasible`**, và `message` giải thích
   node không đủ dung lượng. Muốn thoát, phải patch lại bằng giá trị khả thi.
2. **Không.** Cả bài xoay quanh việc đổi tài nguyên **mà không cần tạo lại Pod**: object Pod giữ
   nguyên, tên giữ nguyên, node giữ nguyên; chỉ **container bên trong** được khởi động lại để nhận
   giá trị memory mới, và dấu vết duy nhất là `restartCount` tăng thêm 1. Đây là chỗ dễ nhầm giữa
   "khởi động lại container" và "thay Pod mới".
3. **Không được chấp nhận.** Mục *Các hạn chế* nói QoS class ban đầu **không thể bị thay đổi bởi
   một lần resize**, và với Pod `Burstable` thì luật cụ thể là: request và limit **không được trở
   nên bằng nhau cho cả CPU lẫn memory cùng lúc**, vì như thế Pod sẽ thành `Guaranteed`. Muốn đổi
   class thì phải tạo Pod mới, không phải resize.
4. **`Infeasible` là bất khả thi trên node hiện tại** — ví dụ xin nhiều tài nguyên hơn mức node có;
   nó **không tự khỏi**, bạn phải patch lại bằng giá trị khả thi. **`Deferred` là chưa thể lúc này
   nhưng có thể sau** — ví dụ khi một Pod khác bị xóa — và **kubelet sẽ tự thử lại**, nên nó có thể
   tự khỏi.
5. **Có.** Kịch bản ví dụ trong mục *Chính sách resize của container* nêu đúng trường hợp này: đổi
   riêng CPU thì resize tại chỗ, đổi riêng memory thì khởi động lại, còn đổi **cả hai cùng lúc** thì
   container **bị khởi động lại** — chính sách của memory quyết định.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
