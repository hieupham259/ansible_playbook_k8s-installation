# Runtime Class

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/containers/runtime-class/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 2](LO-TRINH-ADMIN.md#giai-đoạn-2--container-và-runtime), bài 7/8 ·
Kiểm chứng ở Lab 2 (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Cluster lab của bạn chỉ có một runtime handler nên sẽ không tạo RuntimeClass nào. Đọc để biết
**cơ chế tồn tại** và nhận ra nó khi gặp cluster có workload cần cô lập mạnh.

**Phải hiểu ở lần đọc này:**

- RuntimeClass là cách **chọn cấu hình container runtime cho từng Pod**, đánh đổi giữa hiệu
  năng và mức cô lập.
- Thiết lập gồm **hai bước** và thứ tự bắt buộc: cấu hình handler trong CRI **trên node**
  trước, rồi mới tạo object RuntimeClass trỏ tới `handler` đó. Object trong API không tự tạo
  ra cấu hình trên node.
- RuntimeClass là tài nguyên **cấp cluster**, không thuộc namespace nào — nối với bài
  [19](19-namespaces-vi.md).
- Pod dùng nó qua `spec.runtimeClassName`. Nếu RuntimeClass không tồn tại hoặc CRI không chạy
  được handler, **Pod vào phase `Failed`**.
- Không chỉ định `runtimeClassName` thì dùng handler mặc định — đúng như hành vi hiện tại của
  cluster lab.
- Quyền ghi RuntimeClass nên giới hạn cho quản trị viên cluster.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Cấu hình handler trong `config.toml` của containerd và `crio.conf` | thao tác trên node, làm khi cần runtime thứ hai | giai đoạn 8 |
| Toàn bộ mục *Lập lịch* — `nodeSelector`, `tolerations` | chưa học lập lịch | giai đoạn 7 |
| *Overhead của Pod* | cần hiểu requests/limits trước | giai đoạn 3 và 7, bài [144](144-pod-overhead-vi.md) |
| Tham chiếu tới phase `Failed` của Pod | chưa học vòng đời Pod | giai đoạn 3, bài [47](47-pod-lifecycle-vi.md) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.20 [stable]`

Trang này mô tả tài nguyên RuntimeClass và cơ chế lựa chọn runtime.

RuntimeClass là một tính năng dùng để lựa chọn cấu hình container runtime. Cấu hình
container runtime này được dùng để chạy các container của một Pod.

## Động lực (Motivation)

Bạn có thể đặt RuntimeClass khác nhau cho các Pod khác nhau để cân bằng giữa hiệu
năng và bảo mật. Ví dụ, nếu một phần workload của bạn đòi hỏi mức đảm bảo an toàn
thông tin cao, bạn có thể chọn lập lịch (schedule) các Pod đó sao cho chúng chạy
trong một container runtime sử dụng ảo hóa phần cứng (hardware virtualization). Khi
đó bạn được hưởng lợi từ sự cô lập (isolation) bổ sung của runtime thay thế, đổi lại
là một chút chi phí phụ trợ (overhead) tăng thêm.

Bạn cũng có thể dùng RuntimeClass để chạy các Pod khác nhau với cùng một container
runtime nhưng với các thiết lập khác nhau.

## Thiết lập (Setup)

1. Cấu hình phần hiện thực CRI trên các node (tùy thuộc vào runtime)
2. Tạo các tài nguyên RuntimeClass tương ứng

### 1. Cấu hình phần hiện thực CRI trên các node (Configure the CRI implementation on nodes)

Các cấu hình khả dụng thông qua RuntimeClass phụ thuộc vào phần hiện thực Container
Runtime Interface (CRI). Xem tài liệu tương ứng ([bên dưới](#cri-configuration)) cho
phần hiện thực CRI của bạn để biết cách cấu hình.

> **Ghi chú:** Theo mặc định, RuntimeClass giả định cấu hình node là đồng nhất trên
> toàn cluster (nghĩa là tất cả các node được cấu hình giống nhau về container
> runtime). Để hỗ trợ các cấu hình node không đồng nhất, xem mục
> [Lập lịch](#scheduling) bên dưới.

Các cấu hình có một tên `handler` tương ứng, được RuntimeClass tham chiếu tới.
Handler phải là một [tên label DNS (DNS label name)](./17-names-vi.md#dns-label-names)
hợp lệ.

### 2. Tạo các tài nguyên RuntimeClass tương ứng (Create the corresponding RuntimeClass resources)

Mỗi cấu hình được thiết lập ở bước 1 cần có một tên `handler` gắn liền, dùng để định
danh cấu hình đó. Với mỗi handler, hãy tạo một object RuntimeClass tương ứng.

Tài nguyên RuntimeClass hiện chỉ có 2 trường quan trọng: tên của RuntimeClass
(`metadata.name`) và handler (`handler`). Định nghĩa object trông như sau:

```yaml
# RuntimeClass được định nghĩa trong API group node.k8s.io
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  # Tên mà RuntimeClass sẽ được tham chiếu tới.
  # RuntimeClass là tài nguyên không thuộc namespace nào (non-namespaced).
  name: myclass 
# Tên của cấu hình CRI tương ứng
handler: myconfiguration 
```

Tên của một object RuntimeClass phải là một
[tên DNS subdomain (DNS subdomain name)](https://kubernetes.io/docs/concepts/overview/working-with-objects/names#dns-subdomain-names)
hợp lệ.

> **Ghi chú:** Khuyến nghị rằng các thao tác ghi lên RuntimeClass
> (create/update/patch/delete) chỉ nên được giới hạn cho quản trị viên cluster. Đây
> thường là thiết lập mặc định. Xem
> [Tổng quan về ủy quyền (Authorization Overview)](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
> để biết thêm chi tiết.

## Cách sử dụng (Usage)

Khi các RuntimeClass đã được cấu hình cho cluster, bạn có thể chỉ định
`runtimeClassName` trong spec của Pod để sử dụng nó. Ví dụ:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  runtimeClassName: myclass
  # ...
```

Điều này sẽ chỉ thị cho kubelet dùng RuntimeClass có tên được chỉ định để chạy pod
này. Nếu RuntimeClass với tên đó không tồn tại, hoặc CRI không thể chạy handler
tương ứng, pod sẽ đi vào
[phase](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-phase)
kết thúc `Failed`. Hãy tìm
[event](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)
tương ứng để xem thông báo lỗi.

Nếu không chỉ định `runtimeClassName`, RuntimeHandler mặc định sẽ được sử dụng,
tương đương với hành vi khi tính năng RuntimeClass bị tắt.

### Cấu hình CRI (CRI Configuration) {#cri-configuration}

Để biết thêm chi tiết về việc thiết lập các CRI runtime, xem
[Cài đặt CRI (CRI installation)](./00-container-runtimes-vi.md).

#### containerd

Các runtime handler được cấu hình thông qua cấu hình của containerd tại
`/etc/containerd/config.toml`. Các handler hợp lệ được cấu hình trong phần runtimes:

```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.${HANDLER_NAME}]
```

Xem [tài liệu cấu hình](https://github.com/containerd/containerd/blob/main/docs/cri/config.md)
của containerd để biết thêm chi tiết:

#### CRI-O

Các runtime handler được cấu hình thông qua cấu hình của CRI-O tại
`/etc/crio/crio.conf`. Các handler hợp lệ được cấu hình trong bảng
[crio.runtime](https://github.com/cri-o/cri-o/blob/master/docs/crio.conf.5.md#crioruntime-table):

```
[crio.runtime.runtimes.${HANDLER_NAME}]
  runtime_path = "${PATH_TO_BINARY}"
```

Xem [tài liệu cấu hình](https://github.com/cri-o/cri-o/blob/master/docs/crio.conf.5.md)
của CRI-O để biết thêm chi tiết.

## Lập lịch (Scheduling) {#scheduling}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.16 [beta]`

Bằng cách chỉ định trường `scheduling` cho một RuntimeClass, bạn có thể đặt các
ràng buộc để đảm bảo các Pod chạy với RuntimeClass này được lập lịch tới những node
hỗ trợ nó. Nếu `scheduling` không được đặt, RuntimeClass này được giả định là được
hỗ trợ bởi tất cả các node.

Để đảm bảo các pod được đặt lên những node hỗ trợ một RuntimeClass cụ thể, tập hợp
node đó nên có một nhãn (label) chung, sau đó nhãn này được chọn bởi trường
`runtimeclass.scheduling.nodeSelector`. nodeSelector của RuntimeClass được hợp nhất
(merge) với nodeSelector của pod trong bước admission, thực chất là lấy phần giao
(intersection) của tập các node được mỗi bên chọn. Nếu có xung đột, pod sẽ bị từ
chối.

Nếu các node được hỗ trợ bị đánh taint để ngăn các pod dùng RuntimeClass khác chạy
trên node đó, bạn có thể thêm `tolerations` vào RuntimeClass. Giống như với
`nodeSelector`, các toleration được hợp nhất với các toleration của pod trong bước
admission, thực chất là lấy phần hợp (union) của tập các node được mỗi bên dung nạp
(tolerate).

Để tìm hiểu thêm về việc cấu hình node selector và tolerations, xem
[Gán Pod vào Node (Assigning Pods to Nodes)](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/).

### Overhead của Pod (Pod Overhead)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.24 [stable]`

Bạn có thể chỉ định các tài nguyên _overhead_ (chi phí phụ trợ) gắn liền với việc
chạy một Pod. Việc khai báo overhead cho phép cluster (bao gồm cả scheduler) tính
đến nó khi đưa ra các quyết định về Pod và tài nguyên.

Overhead của Pod được định nghĩa trong RuntimeClass thông qua trường `overhead`.
Thông qua trường này, bạn có thể chỉ định overhead của việc chạy các pod sử dụng
RuntimeClass này và đảm bảo các overhead đó được tính đến trong Kubernetes.

## Tiếp theo (What's next)

- [Thiết kế RuntimeClass (RuntimeClass Design)](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/585-runtime-class/README.md)
- [Thiết kế lập lịch cho RuntimeClass (RuntimeClass Scheduling Design)](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/585-runtime-class/README.md#runtimeclass-scheduling)
- [Tài liệu tham khảo API RuntimeClass (RuntimeClass API reference)](https://kubernetes.io/docs/reference/kubernetes-api/node/runtime-class-v1/)
- Đọc về khái niệm [Overhead của Pod (Pod Overhead)](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-overhead/)
- [Thiết kế tính năng PodOverhead (PodOverhead Feature Design)](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/688-pod-overhead)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

1. Bạn `kubectl apply` một RuntimeClass tên `gvisor` với `handler: runsc`, nhưng chưa đụng gì
   tới các node. Pod dùng `runtimeClassName: gvisor` sẽ ra sao?
2. RuntimeClass là tài nguyên thuộc namespace hay cấp cluster? Kiểm bằng lệnh nào?
3. Pod của bạn không khai báo `runtimeClassName`. Nó chạy bằng gì?
4. Vì sao bài khuyến nghị chỉ quản trị viên cluster mới được ghi RuntimeClass?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Pod vào **phase `Failed`**, vì CRI trên node không có handler `runsc` để chạy. Object trong
   API chỉ là cái tên trỏ tới một cấu hình; **cấu hình thật phải tồn tại trên node trước** — đó
   là lý do bài đánh số hai bước theo đúng thứ tự đó. Xem event của Pod để thấy thông báo lỗi.
2. **Cấp cluster** — bài ghi rõ RuntimeClass là tài nguyên không thuộc namespace nào. Kiểm bằng
   `kubectl api-resources --namespaced=false | grep runtimeclass`.
3. Bằng **RuntimeHandler mặc định**, tương đương với hành vi khi tính năng RuntimeClass bị tắt.
   Đây chính là điều đang xảy ra với mọi Pod trong cluster lab của bạn.
4. Vì RuntimeClass quyết định **workload chạy dưới cơ chế cô lập nào**. Ai tạo được
   RuntimeClass thì có thể trỏ tới một handler yếu hơn về bảo mật, hoặc gắn `scheduling` và
   `overhead` ảnh hưởng tới việc Pod được đặt ở đâu và tính tài nguyên ra sao.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
