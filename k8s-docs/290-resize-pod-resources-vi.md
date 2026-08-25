# Thay đổi kích thước tài nguyên CPU và Memory được gán cho Pod (Resize CPU and Memory Resources assigned to Pods)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/configure-pod-container/resize-pod-resources/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 3 → nhóm [3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn), bài 7/9 ·
**Không kiểm chứng được trên cluster lab** — [Lab 3c](labs/LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md)
ghi rõ lý do ở bảng *Ánh xạ tài liệu sang bài thực hành*; phần gần nhất bạn làm được là B6, nơi
resize diễn ra ở **cấp container**.

Bài là giao điểm của hai bài vừa đọc: [265](265-assign-pod-level-resources-vi.md) cho tài nguyên
cấp Pod và [289](289-resize-container-resources-vi.md) cho cơ chế resize. Đọc bài này **sau** cả
hai, nếu không mọi câu trong bài đều treo lơ lửng. Tính năng đòi bốn feature gate mà baseline của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) không bật, nên đây là bài đọc để hiểu ranh giới, không
phải để chạy.

**Phải hiểu ở lần đọc này:**

- Đối tượng bị resize là `spec.resources` — **khung tài nguyên tổng của Pod**, đóng vai trò giới
  hạn trên cho tổng tiêu thụ của mọi container. Lệnh vẫn là `kubectl patch ... --subresource resize`
  như bài 289, nhưng patch vào `spec.resources` chứ không vào `spec.containers[*]`.
- Mục *Chính sách resize của container và resize cấp Pod*: **không có chính sách khởi động lại ở
  cấp Pod** — thay đổi `spec.resources` luôn được áp dụng tại chỗ, vì tài nguyên cấp Pod chỉ là
  ràng buộc trên cgroup của Pod. Nhưng `resizePolicy` **cấp container** vẫn chi phối, kể cả khi
  thay đổi bắt nguồn từ cấp Pod.
- Ví dụ trong bài chứng minh đúng điều đó: container `nginx-server` **không khai CPU limit riêng**
  nên nó ngầm nhận limit cấp Pod làm trần; patch CPU cấp Pod từ `200m` lên `300m` làm trần ngầm đó
  đổi, và vì container này đặt `resizePolicy` CPU là `RestartContainer` nên kubelet **khởi động lại
  nó** (`restartCount` lên `1`), trong khi container `pause` với `NotRequired` giữ `restartCount`
  bằng `0`.
- Hai luật hợp lệ riêng của cấp Pod, mục *Giới hạn*: `spec.resources.requests` sau resize phải
  **lớn hơn hoặc bằng tổng requests** của tất cả container; limit của **từng** container phải **nhỏ
  hơn hoặc bằng** limit cấp Pod, nhưng **tổng** limit các container **được phép vượt** limit cấp
  Pod — đó là cách các container chia sẻ tài nguyên trong Pod.
- Mục *Trạng thái resize của Pod và logic thử lại*: cơ chế theo dõi dùng **chung** với resize cấp
  container — `PodResizePending` (`Infeasible` hoặc `Deferred`) và `PodResizeInProgress` — và mọi
  hạn chế của bài [289](289-resize-container-resources-vi.md) vẫn áp dụng nguyên vẹn.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Bốn feature gate liệt kê ở *Trước khi bạn bắt đầu* và cách bật chúng | bật gate là sửa apiserver và kubelet của cluster đang chạy, làm lệch baseline Lab 00 | [giai đoạn 20](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) |
| Thứ tự thử lại các resize `Deferred` theo PriorityClass rồi tới lớp QoS | cần biết PriorityClass trước | bài [141](141-pod-priority-preemption-vi.md) ở [giai đoạn 7a](00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction) |
| Các trường `observedGeneration` dùng để truy vết yêu cầu resize | công cụ đối chiếu chi tiết khi một lần resize treo | mô tả đầy đủ ở bài [289](289-resize-container-resources-vi.md), dùng tới khi gỡ rối ở [giai đoạn 24](00-ALO-TRINH-ADMIN.md#giai-đoạn-24--xử-lý-sự-cố) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [beta]`

Trang này giải thích cách thay đổi tài nguyên CPU và memory được đặt ở cấp Pod (Pod level)
mà không cần tạo lại Pod.

Tính năng thay đổi kích thước Pod tại chỗ (In-place Pod Resize) cho phép chỉnh sửa mức cấp phát
tài nguyên cho một Pod đang chạy, tránh làm gián đoạn ứng dụng. Quy trình thay đổi kích thước
tài nguyên của từng container riêng lẻ được trình bày trong
[Thay đổi kích thước tài nguyên CPU và Memory được gán cho Container](289-resize-container-resources-vi.md).

Trang này tập trung vào việc thay đổi kích thước tài nguyên cấp Pod tại chỗ (In-place Pod-level
resources resize). Tài nguyên cấp Pod được định nghĩa trong `spec.resources` và đóng vai trò là
giới hạn trên (upper bound) đối với tổng tài nguyên mà tất cả các container trong Pod tiêu thụ.
Tính năng thay đổi kích thước tài nguyên cấp Pod tại chỗ cho phép bạn thay đổi trực tiếp các mức
cấp phát CPU và memory tổng hợp này cho một Pod đang chạy.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản v1.35 hoặc mới hơn. Để kiểm tra phiên bản,
nhập `kubectl version`.

Các [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
sau phải được bật cho control plane và cho tất cả các node trong cluster của bạn:

* [`InPlacePodLevelResourcesVerticalScaling`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#InPlacePodLevelResourcesVerticalScaling)
* [`PodLevelResources`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#PodLevelResources)
* [`InPlacePodVerticalScaling`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#InPlacePodVerticalScaling)
* [`NodeDeclaredFeatures`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#NodeDeclaredFeatures)

Phiên bản kubectl client phải tối thiểu là v1.32 để dùng được flag `--subresource=resize`.

## Trạng thái resize của Pod và logic thử lại (Pod Resize Status and Retry Logic)

Cơ chế mà `kubelet` dùng để theo dõi và thử lại (retry) các thay đổi tài nguyên được dùng chung
cho cả yêu cầu resize cấp container lẫn cấp Pod.

Các trạng thái, lý do và độ ưu tiên khi thử lại giống hệt với những gì được định nghĩa cho
resize cấp container:

* Điều kiện trạng thái (Status Conditions): `kubelet` dùng PodResizePending (với các lý do như
  Infeasible hoặc Deferred) và PodResizeInProgress để truyền đạt trạng thái của yêu cầu.

* Độ ưu tiên thử lại (Retry Priority): Các yêu cầu resize bị hoãn (Deferred) được thử lại theo
  thứ tự dựa trên PriorityClass, sau đó đến lớp QoS (Guaranteed trước Burstable), và cuối cùng
  là theo khoảng thời gian chúng đã bị hoãn.

* Theo dõi (Tracking): Bạn có thể dùng các field `observedGeneration` để theo dõi xem đặc tả Pod
  nào (metadata.generation) tương ứng với trạng thái của yêu cầu resize mới nhất đã được xử lý.

Để có mô tả đầy đủ về các điều kiện và logic thử lại này, vui lòng tham khảo mục
[Trạng thái resize của Pod](289-resize-container-resources-vi.md#pod-resize-status)
trong tài liệu về resize container.

## Chính sách resize của container và resize cấp Pod (Container Resize Policy and Pod-Level Resize)

Việc thay đổi kích thước tài nguyên cấp Pod không hỗ trợ và cũng không yêu cầu chính sách
khởi động lại (restart policy) riêng của nó.

* Không có chính sách cấp Pod (No Pod-Level Policy): Các thay đổi đối với tài nguyên tổng hợp
  của Pod (spec.resources) luôn được áp dụng tại chỗ mà không kích hoạt khởi động lại. Lý do là
  tài nguyên cấp Pod đóng vai trò như một ràng buộc tổng thể trên cgroup của Pod và không trực
  tiếp quản lý runtime của ứng dụng bên trong các container.

* [Chính sách cấp container](289-resize-container-resources-vi.md#container-resize-policies)
  vẫn chi phối: `resizePolicy` vẫn phải được cấu hình ở cấp container
  (spec.containers[*].resizePolicy). Chính sách này quyết định liệu một container riêng lẻ có bị
  khởi động lại khi resource requests hoặc limits của nó thay đổi hay không, bất kể thay đổi đó
  bắt nguồn từ một lần resize trực tiếp ở cấp container hay từ một cập nhật lên khung tài nguyên
  (resource envelope) tổng thể ở cấp Pod.

## Giới hạn (Limitations)

Đối với Kubernetes v1.36, việc thay đổi kích thước tài nguyên cấp Pod tại chỗ chịu tất cả các
giới hạn được mô tả cho resize tài nguyên cấp container, mà bạn có thể xem tại đây:
[Thay đổi kích thước tài nguyên CPU và Memory được gán cho Container: Giới hạn](289-resize-container-resources-vi.md#limitations).

Ngoài ra, ràng buộc sau đây là đặc thù của việc resize tài nguyên cấp Pod:
* Kiểm tra hợp lệ requests của container (Container Requests Validation): Một lần resize chỉ
  được phép nếu resource requests cấp Pod thu được (spec.resources.requests) lớn hơn hoặc bằng
  tổng các resource requests tương ứng của tất cả các container riêng lẻ bên trong Pod. Điều này
  duy trì mức sẵn sàng tài nguyên tối thiểu được bảo đảm cho Pod.

* Kiểm tra hợp lệ limits của container (Container Limits Validation): Một lần resize được phép
  nếu limits của từng container riêng lẻ nhỏ hơn hoặc bằng resource limits cấp Pod
  (spec.resources.limits). Limit cấp Pod đóng vai trò là ranh giới mà không một container đơn lẻ
  nào được vượt qua, nhưng tổng các limits của các container được phép vượt quá limit cấp Pod,
  cho phép chia sẻ tài nguyên giữa các container bên trong Pod.

## Ví dụ: Thay đổi kích thước tài nguyên cấp Pod (Example: Resizing Pod-Level Resources)

Trước tiên, tạo một Pod được thiết kế để resize CPU tại chỗ và resize memory có yêu cầu
khởi động lại.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-level-resize-demo
spec:
  containers:
  - name: pause
    image: registry.k8s.io/pause:3.9
    resizePolicy:
    - resourceName: cpu
      restartPolicy: NotRequired # Mặc định, nhưng ghi tường minh ở đây
    - resourceName: memory
      restartPolicy: RestartContainer
    resources:
      requests:
        cpu: 100m
        memory: 100Mi
  - name: nginx-server
    image: registry.k8s.io/nginx:latest
    resizePolicy:
    - resourceName: cpu
      restartPolicy: RestartContainer
    - resourceName: memory
      restartPolicy: RestartContainer
  resources: # Tài nguyên cấp Pod
    requests:
      cpu: 200m
      memory: 200Mi
    limits:
      cpu: 200m
      memory: 200Mi
```

Tạo pod:

```shell
kubectl create -f pod-level-resize.yaml
```

Pod này khởi đầu trong lớp QoS Guaranteed vì requests cấp Pod bằng với limits. Kiểm tra trạng
thái ban đầu của nó:

```shell
# Chờ một lát để pod chuyển sang trạng thái running
kubectl get pod pod-level-resize-demo --output=yaml
```

Quan sát `spec.resources` (200m CPU, 200Mi memory). Lưu ý
`status.containerStatuses[0].restartCount` (phải là 0) và
`status.containerStatuses[1].restartCount` (phải là 0).

Bây giờ, tăng CPU request và limit cấp Pod lên `300m`. Bạn dùng `kubectl patch` với đối số
dòng lệnh `--subresource resize`.

```shell
kubectl patch pod pod-level-resize-demo --subresource resize --patch \
  '{"spec":{"resources":{"requests":{"cpu":"300m"}, "limits":{"cpu":"300m"}}}}'

# Các cách thay thế:
# kubectl edit pod pod-level-resize-demo --subresource resize
# kubectl apply -f <updated-manifest> --subresource resize --server-side
```

> **Ghi chú:** Đối số dòng lệnh `--subresource resize` yêu cầu `kubectl` client phiên bản
> v1.32.0 trở lên. Các phiên bản cũ hơn sẽ báo lỗi `invalid subresource`.

Kiểm tra lại trạng thái pod sau khi patch:

```shell
kubectl get pod pod-level-resize-demo --output=yaml
```

Bạn sẽ thấy:
* `spec.resources.requests` và `spec.resources.limits` giờ hiển thị `cpu: 300m`.
* `status.containerStatuses[0].restartCount` vẫn là `0`, vì `resizePolicy` cho CPU là
  `NotRequired`.
* `status.containerStatuses[1].restartCount` tăng lên `1`, cho biết container đã bị khởi động
  lại để áp dụng thay đổi CPU. Việc khởi động lại xảy ra ở Container 1 mặc dù lần resize được
  áp dụng ở cấp Pod, do mối quan hệ phức tạp giữa limits cấp Pod và các chính sách cấp
  container. Vì Container 1 không chỉ định CPU limit tường minh, cấu hình tài nguyên bên dưới
  của nó (ví dụ: cgroups) mặc nhiên nhận limit CPU tổng thể của Pod làm ranh giới tiêu thụ tối
  đa thực tế của mình. Khi CPU limit cấp Pod được patch từ 200m lên 300m, hành động này kéo theo
  thay đổi limit ngầm định đang được áp lên Container 1. Vì Container 1 đã đặt tường minh
  resizePolicy là RestartContainer cho CPU, `kubelet` buộc phải khởi động lại container để áp
  dụng đúng thay đổi này trong cơ chế thực thi tài nguyên bên dưới — qua đó xác nhận rằng việc
  thay đổi limits cấp Pod có thể kích hoạt chính sách khởi động lại của container ngay cả khi
  limits của container không được định nghĩa trực tiếp.

## Dọn dẹp (Clean up)

Xóa pod:

```shell
kubectl delete pod pod-level-resize-demo
```

## Tiếp theo (What's next)

### Dành cho người phát triển ứng dụng (For application developers)

* [Gán tài nguyên memory cho Container và Pod](264-assign-memory-resource-vi.md)

* [Gán tài nguyên CPU cho Container và Pod](263-assign-cpu-resource-vi.md)

* [Gán tài nguyên CPU và memory ở cấp Pod](265-assign-pod-level-resources-vi.md)

### Dành cho quản trị viên cluster (For cluster administrators)

* [Cấu hình memory request và limit mặc định cho một Namespace](232-memory-default-namespace-vi.md)

* [Cấu hình CPU request và limit mặc định cho một Namespace](230-cpu-default-namespace-vi.md)

* [Cấu hình ràng buộc bộ nhớ tối thiểu và tối đa cho một Namespace](231-memory-constraint-namespace-vi.md)

* [Cấu hình ràng buộc CPU tối thiểu và tối đa cho một Namespace](229-cpu-constraint-namespace-vi.md)

* [Cấu hình Quota Memory và CPU cho một Namespace](233-quota-memory-cpu-namespace-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở nhóm 3c:

1. Một Pod có hai container: A đặt `resizePolicy` CPU là `NotRequired`, B đặt `RestartContainer`
   cho CPU và **không** khai CPU limit riêng. Bạn patch CPU ở **cấp Pod**. Container nào bị khởi
   động lại, và vì sao thay đổi ở cấp Pod lại chạm tới nó?
2. **Câu bẫy.** Bài nói thay đổi tài nguyên cấp Pod "luôn được áp dụng tại chỗ mà không kích hoạt
   khởi động lại". Vậy có thể kết luận rằng một lần resize cấp Pod không bao giờ làm container nào
   khởi động lại không?
3. `spec.resources.requests.cpu` đang là `200m` trong khi requests của các container cộng lại là
   `300m`. Lần resize đó có hợp lệ không? Còn trường hợp **tổng** limits của các container vượt quá
   limit cấp Pod thì sao?
4. Giả sử baseline lab đã bật đủ bốn gate. `lab-k8s-worker2` có 2 vCPU và bạn patch
   `spec.resources.limits.cpu` của một Pod trên đó lên `4`. Condition nào xuất hiện, với `reason`
   gì, và giá trị trong `status` của các container có đổi theo không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Container B.** Vì B không khai CPU limit tường minh, cấu hình tài nguyên bên dưới của nó
   **ngầm lấy limit CPU của cả Pod làm trần thực tế**. Khi limit cấp Pod bị patch, cái trần ngầm đó
   đổi theo — tức tài nguyên của B thay đổi — và B đã đặt `resizePolicy` CPU là `RestartContainer`,
   nên kubelet **buộc phải khởi động lại B**. Container A với `NotRequired` giữ `restartCount` bằng
   `0`.
2. **Không, kết luận đó sai** — và ví dụ ngay trong bài chứng minh điều ngược lại. Câu "không kích
   hoạt khởi động lại" chỉ nói rằng **bản thân cấp Pod không có chính sách khởi động lại nào**;
   `resizePolicy` **cấp container vẫn chi phối**, bất kể thay đổi bắt nguồn từ một lần resize trực
   tiếp ở cấp container hay từ cập nhật khung tài nguyên ở cấp Pod.
3. **Không hợp lệ.** Luật kiểm tra requests của cấp Pod đòi `spec.resources.requests` thu được phải
   **lớn hơn hoặc bằng tổng requests** của tất cả container — đó là cách giữ mức tài nguyên tối
   thiểu được bảo đảm cho Pod; `200m < 300m` nên bị chặn. Trường hợp thứ hai thì **được phép**: luật
   chỉ đòi limit của **từng** container không vượt limit cấp Pod, còn **tổng** limits các container
   **được phép vượt**, để các container chia sẻ tài nguyên với nhau.
4. **`PodResizePending` với `reason: Infeasible`** — node không đủ dung lượng, đúng cơ chế dùng
   chung với resize cấp container. Và **không**: giá trị trong `status` giữ nguyên như trước, vì
   kubelet không áp dụng được lần resize này; chỉ `spec` phản ánh trạng thái mong muốn.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
