# Pod Overhead

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/pod-overhead/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 7 → nhóm [7a](LO-TRINH-ADMIN.md#7a-scheduling-và-eviction), bài 9/13 ·
Kiểm chứng ở Lab 7a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài này chỉ có tác dụng khi cluster dùng một container runtime "nặng" — bài lấy ví dụ Kata
Containers chạy trên máy ảo Firecracker, tốn khoảng 120MiB cho mỗi Pod. Cluster lab dùng
containerd với runc nên bạn **sẽ không thấy `spec.overhead` xuất hiện**. Đọc bài để hiểu
**cách một khoản tài nguyên vô hình được cộng vào phép tính lập lịch**, chứ không phải để cấu
hình gì.

Bài nối trực tiếp với [43 — RuntimeClass](43-runtime-class-vi.md) đã đọc ở giai đoạn 2:
overhead là một trường của RuntimeClass, không phải một trường bạn tự viết trong Pod.

**Phải hiểu ở lần đọc này:**

- Pod Overhead là tài nguyên mà **hạ tầng của chính Pod** tiêu thụ, **cộng thêm** vào request
  và limit của các container bên trong.
- Nguồn của nó là `overhead.podFixed` khai báo trong **RuntimeClass**. Admission controller
  RuntimeClass điền `spec.overhead` vào PodSpec tại thời điểm admission; nếu PodSpec **đã tự
  định nghĩa** trường này thì **Pod bị từ chối**.
- Ba nơi overhead thực sự có hiệu lực: **lập lịch** (kube-scheduler cộng overhead vào tổng
  request của container khi tìm node), **kích thước cgroup của Pod** do kubelet đặt, và **xếp
  hạng eviction**. Ngoài ra nó cũng được tính vào ResourceQuota.
- Ví dụ số của bài phải nhớ được đường đi: container xin tổng `2000m` CPU và `200MiB` bộ nhớ,
  overhead `250m` / `120MiB`, node báo cáo request là **`2250m` CPU và `320MiB`**, và cgroup
  cấp Pod cũng bị giới hạn ở 320MiB.
- Overhead **chỉ tồn tại khi Pod chỉ định một RuntimeClass có trường `overhead`**; không có
  RuntimeClass như vậy thì không có khoản cộng thêm nào.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Kiểm tra giới hạn cgroup của Pod* bằng `crictl` | chính bài nói đây là ví dụ nâng cao và người dùng thường không cần | không cần |
| `cpu.cfs_quota_us`, `memory.limit_in_bytes`, `cpu.shares` | là chi tiết cgroup | giai đoạn 2, bài [33](33-cgroups-vi.md) |
| *Khả năng quan sát* — metric `kube_pod_overhead_*` | cần kube-state-metrics | giai đoạn 11, bài [163](163-kube-state-metrics-vi.md) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.24 [stable]`

Khi bạn chạy một Pod trên một Node, bản thân Pod chiếm một lượng tài nguyên hệ thống. Các
tài nguyên này là phần bổ sung so với các tài nguyên cần thiết để chạy (các) container bên trong Pod.
Trong Kubernetes, _Pod Overhead_ là cách để tính đến các tài nguyên mà hạ tầng của Pod
tiêu thụ, cộng thêm vào các request và limit của container.

Trong Kubernetes, overhead của Pod được thiết lập tại thời điểm
[admission](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#what-are-admission-webhooks)
theo overhead gắn với
[RuntimeClass](./43-runtime-class-vi.md) của Pod.

Overhead của pod được xét cộng thêm vào tổng các request tài nguyên của container khi
lập lịch một Pod. Tương tự, kubelet sẽ tính cả Pod overhead khi xác định kích thước cgroup của Pod,
và khi thực hiện xếp hạng eviction (thu hồi) của Pod.

## Cấu hình Pod overhead (Configuring Pod overhead) {#set-up}

Bạn cần đảm bảo sử dụng một `RuntimeClass` có định nghĩa trường `overhead`.

## Ví dụ sử dụng (Usage example)

Để làm việc với Pod overhead, bạn cần một RuntimeClass có định nghĩa trường `overhead`. Ví dụ,
bạn có thể dùng định nghĩa RuntimeClass dưới đây với một container runtime ảo hóa
(trong ví dụ này là Kata Containers kết hợp với trình giám sát máy ảo Firecracker)
sử dụng khoảng 120MiB cho mỗi Pod dành cho máy ảo và hệ điều hành khách (guest OS):

```yaml
# Bạn cần thay đổi ví dụ này cho khớp với tên runtime thực tế, và overhead
# tài nguyên theo từng Pod mà container runtime đang thêm vào trong cluster của bạn.
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata-fc
handler: kata-fc
overhead:
  podFixed:
    memory: "120Mi"
    cpu: "250m"
```

Các workload được tạo có chỉ định RuntimeClass handler `kata-fc` sẽ tính overhead bộ nhớ và
cpu vào các phép tính resource quota, việc lập lịch node, cũng như việc xác định kích thước cgroup của Pod.

Xét việc chạy workload ví dụ sau, test-pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  runtimeClassName: kata-fc
  containers:
  - name: busybox-ctr
    image: busybox:1.28
    stdin: true
    tty: true
    resources:
      limits:
        cpu: 500m
        memory: 100Mi
  - name: nginx-ctr
    image: nginx
    resources:
      limits:
        cpu: 1500m
        memory: 100Mi
```

> **Ghi chú:**
> Nếu chỉ có `limits` được chỉ định trong định nghĩa pod, kubelet sẽ suy ra `requests` từ các limit đó và đặt chúng bằng với `limits` đã định nghĩa.

Tại thời điểm admission, [admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) RuntimeClass
cập nhật PodSpec của workload để bao gồm `overhead` như được mô tả trong RuntimeClass. Nếu PodSpec đã định nghĩa sẵn trường này,
Pod sẽ bị từ chối. Trong ví dụ đã cho, vì chỉ có tên RuntimeClass được chỉ định, admission controller biến đổi (mutate) Pod
để bao gồm `overhead`.

Sau khi admission controller RuntimeClass đã thực hiện các thay đổi, bạn có thể kiểm tra
giá trị Pod overhead đã cập nhật:

```bash
kubectl get pod test-pod -o jsonpath='{.spec.overhead}'
```

Kết quả là:

```
map[cpu:250m memory:120Mi]
```

Nếu một [ResourceQuota](./134-resource-quotas-vi.md) được định nghĩa, tổng các request của container cũng như
trường `overhead` đều được tính vào.

Khi kube-scheduler quyết định node nào sẽ chạy một Pod mới, bộ lập lịch xét `overhead` của
Pod đó cũng như tổng các request container của Pod. Với ví dụ này, bộ lập lịch cộng
các request và overhead, rồi tìm một node có sẵn 2.25 CPU và 320 MiB bộ nhớ.

Khi một Pod đã được lập lịch lên một node, kubelet trên node đó tạo một
cgroup mới cho Pod. Chính bên trong pod này, container runtime
bên dưới sẽ tạo các container.

Nếu tài nguyên có limit được định nghĩa cho mỗi container (QoS Guaranteed hoặc QoS Burstable có định nghĩa limit),
kubelet sẽ đặt một giới hạn trên cho cgroup của pod gắn với tài nguyên đó (cpu.cfs_quota_us cho CPU
và memory.limit_in_bytes cho bộ nhớ). Giới hạn trên này dựa trên tổng các limit của container cộng với `overhead`
được định nghĩa trong PodSpec.

Với CPU, nếu Pod có QoS Guaranteed hoặc Burstable, kubelet sẽ đặt `cpu.shares` dựa trên
tổng các request của container cộng với `overhead` được định nghĩa trong PodSpec.

Nhìn vào ví dụ của chúng ta, hãy kiểm tra các request container của workload:

```bash
kubectl get pod test-pod -o jsonpath='{.spec.containers[*].resources.limits}'
```

Tổng các request container là 2000m CPU và 200MiB bộ nhớ:

```
map[cpu: 500m memory:100Mi] map[cpu:1500m memory:100Mi]
```

Đối chiếu điều này với những gì node quan sát được:

```bash
kubectl describe node | grep test-pod -B2
```

Kết quả cho thấy các request là 2250m CPU và 320MiB bộ nhớ. Các request này bao gồm cả Pod overhead:

```
  Namespace    Name       CPU Requests  CPU Limits   Memory Requests  Memory Limits  AGE
  ---------    ----       ------------  ----------   ---------------  -------------  ---
  default      test-pod   2250m (56%)   2250m (56%)  320Mi (1%)       320Mi (1%)     36m
```

## Kiểm tra giới hạn cgroup của Pod (Verify Pod cgroup limits)

Kiểm tra các cgroup bộ nhớ của Pod trên node nơi workload đang chạy. Trong ví dụ sau,
[`crictl`](https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md)
được sử dụng trên node, công cụ này cung cấp CLI cho các container runtime tương thích CRI. Đây là
ví dụ nâng cao để minh họa hành vi của Pod overhead, và người dùng thường không cần phải
kiểm tra cgroup trực tiếp trên node.

Đầu tiên, trên node cụ thể đó, xác định định danh (identifier) của Pod:

```bash
# Chạy lệnh này trên node nơi Pod được lập lịch
POD_ID="$(sudo crictl pods --name test-pod -q)"
```

Từ đó, bạn có thể xác định đường dẫn cgroup của Pod:

```bash
# Chạy lệnh này trên node nơi Pod được lập lịch
sudo crictl inspectp -o=json $POD_ID | grep cgroupsPath
```

Đường dẫn cgroup thu được bao gồm container `pause` của Pod. Cgroup cấp Pod nằm ở thư mục ngay phía trên.

```
  "cgroupsPath": "/kubepods/podd7f4b509-cf94-4951-9417-d1087c92a5b2/7ccf55aee35dd16aca4189c952d83487297f3cd760f1bbf09620e206e7d0c27a"
```

Trong trường hợp cụ thể này, đường dẫn cgroup của pod là `kubepods/podd7f4b509-cf94-4951-9417-d1087c92a5b2`.
Kiểm tra thiết lập cgroup cấp Pod cho bộ nhớ:

```bash
# Chạy lệnh này trên node nơi Pod được lập lịch.
# Ngoài ra, hãy đổi tên cgroup cho khớp với cgroup được cấp phát cho pod của bạn.
 cat /sys/fs/cgroup/memory/kubepods/podd7f4b509-cf94-4951-9417-d1087c92a5b2/memory.limit_in_bytes
```

Giá trị là 320 MiB, đúng như mong đợi:

```
335544320
```

### Khả năng quan sát (Observability)

Một số metric `kube_pod_overhead_*` có sẵn trong [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)
để giúp nhận biết khi nào Pod overhead đang được sử dụng và giúp quan sát tính ổn định
của các workload chạy với overhead đã được định nghĩa.

## Tiếp theo (What's next)

* Tìm hiểu thêm về [RuntimeClass](./43-runtime-class-vi.md)
* Đọc đề xuất cải tiến [PodOverhead Design](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/688-pod-overhead)
  để có thêm bối cảnh

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 7:

1. Ai điền `spec.overhead` vào Pod, và lấy giá trị từ đâu? Nếu bạn tự viết trường `overhead`
   trong manifest thì sao?
2. Overhead có được cộng thẳng vào `resources.requests` của container không? Trong ví dụ của
   bài, `kubectl get pod -o jsonpath='{.spec.containers[*].resources.limits}'` và
   `kubectl describe node` cho ra hai con số khác nhau — vì sao?
3. Trên cluster lab, mỗi worker có 2 vCPU. Bạn tạo một Pod xin tổng `1800m` CPU và dùng một
   RuntimeClass khai `podFixed.cpu: 250m`. Bộ lập lịch đi tìm node còn trống bao nhiêu CPU, và
   Pod này có chạy được trên `k8s-worker1` hay `k8s-worker2` không?
4. Kể ba chỗ mà Pod overhead thực sự thay đổi hành vi của hệ thống.

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Admission controller RuntimeClass** điền nó, tại thời điểm admission, **theo trường
   `overhead` của RuntimeClass mà Pod chỉ định**. Nếu PodSpec đã tự định nghĩa sẵn trường này,
   **Pod bị từ chối** — đây không phải trường dành cho người viết manifest.
2. **Không cộng vào requests của container.** Overhead nằm riêng ở `spec.overhead`; các
   container vẫn khai đúng những gì bạn viết. Vì vậy lệnh thứ nhất trả về tổng của các container
   (`500m + 1500m` CPU, `100Mi + 100Mi` bộ nhớ), còn `kubectl describe node` báo **`2250m` CPU
   và `320Mi` bộ nhớ** — vì node cộng cả overhead vào. Đây là chỗ dễ nhầm: hai con số khác nhau
   không có nghĩa là cluster tính sai, mà là hai lăng kính khác nhau lên cùng một Pod.
3. Bộ lập lịch cộng request của container với overhead và đi tìm node còn trống **`2050m`
   CPU**. Hai worker chỉ có **2 vCPU, tức `2000m`**, nên **không worker nào khả thi** kể cả khi
   chúng hoàn toàn rỗng — Pod sẽ nằm chờ. Bài nói rõ: khi quyết định node nào chạy Pod mới, bộ
   lập lịch xét `overhead` **cũng như** tổng các request container.
4. **(1) Lập lịch** — scheduler cộng overhead vào tổng request khi tìm node; **(2) kích thước
   cgroup của Pod** — kubelet đặt giới hạn trên cgroup cấp Pod bằng tổng limit của container
   **cộng** overhead (trong ví dụ là 320MiB), và đặt `cpu.shares` theo tổng request cộng
   overhead; **(3) xếp hạng eviction** của Pod. Ngoài ba chỗ đó, overhead còn được tính vào
   **ResourceQuota** nếu namespace có định nghĩa quota.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
