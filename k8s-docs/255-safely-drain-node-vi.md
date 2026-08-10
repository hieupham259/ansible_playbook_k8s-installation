# Drain một node an toàn (Safely Drain a Node)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/>

Trang này hướng dẫn cách drain (rút toàn bộ Pod khỏi) một node một cách an toàn, tùy chọn có
tôn trọng PodDisruptionBudget mà bạn đã định nghĩa.

## Trước khi bạn bắt đầu (Before you begin)

Tác vụ này giả định rằng bạn đã đáp ứng các điều kiện tiên quyết sau:

1. Bạn không yêu cầu ứng dụng của mình phải có tính sẵn sàng cao (highly available) trong lúc
   drain node, hoặc
1. Bạn đã đọc về khái niệm [PodDisruptionBudget](53-disruptions-vi.md), và đã
   [cấu hình PodDisruptionBudget](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
   cho những ứng dụng cần đến nó.

## (Tùy chọn) Cấu hình disruption budget ((Optional) Configure a disruption budget) {#configure-poddisruptionbudget}

Để đảm bảo workload của bạn vẫn khả dụng trong lúc bảo trì, bạn có thể cấu hình một
[PodDisruptionBudget](53-disruptions-vi.md).

Nếu tính khả dụng là quan trọng đối với bất kỳ ứng dụng nào đang chạy hoặc có thể chạy trên
(các) node mà bạn sắp drain, hãy
[cấu hình PodDisruptionBudget](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
trước, rồi mới tiếp tục làm theo hướng dẫn này.

Bạn nên đặt `AlwaysAllow` cho
[chính sách evict Pod không khỏe mạnh (Unhealthy Pod Eviction Policy)](https://kubernetes.io/docs/tasks/run-application/configure-pdb/#unhealthy-pod-eviction-policy)
trong các PodDisruptionBudget của mình, để hỗ trợ evict những ứng dụng đang hoạt động sai
trong lúc drain node. Hành vi mặc định là chờ các Pod của ứng dụng trở nên
[khỏe mạnh (healthy)](https://kubernetes.io/docs/tasks/run-application/configure-pdb/#healthiness-of-a-pod)
rồi mới cho phép tiến hành drain.

## Dùng `kubectl drain` để đưa một node ra khỏi hoạt động (Use `kubectl drain` to remove a node from service)

Bạn có thể dùng `kubectl drain` để evict một cách an toàn toàn bộ Pod khỏi một node trước khi
bạn thực hiện bảo trì trên node đó (ví dụ: nâng cấp kernel, bảo trì phần cứng, v.v.). Việc
evict an toàn cho phép các container của Pod
[kết thúc một cách nhẹ nhàng (gracefully terminate)](47-pod-lifecycle-vi.md#pod-termination)
và sẽ tôn trọng các PodDisruptionBudget mà bạn đã chỉ định.

> **Ghi chú:** Theo mặc định, `kubectl drain` bỏ qua một số Pod hệ thống trên node vốn không
> thể bị kill; xem tài liệu
> [kubectl drain](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#drain)
> để biết thêm chi tiết.

Khi `kubectl drain` trả về thành công, điều đó cho biết toàn bộ các Pod (trừ những Pod được
loại trừ như mô tả ở đoạn trước) đã được evict một cách an toàn (tôn trọng khoảng thời gian
kết thúc nhẹ nhàng mong muốn, và tôn trọng PodDisruptionBudget mà bạn đã định nghĩa). Khi đó,
việc đưa node xuống bằng cách tắt nguồn máy vật lý — hoặc nếu chạy trên nền tảng cloud, xóa
máy ảo của node — là an toàn.

> **Ghi chú:** Nếu có bất kỳ Pod mới nào toleration (dung nạp) taint
> `node.kubernetes.io/unschedulable`, thì những Pod đó có thể được lập lịch (schedule) lên node
> mà bạn đã drain. Hãy tránh toleration taint này cho bất kỳ thứ gì ngoài DaemonSet.
>
> Nếu bạn hoặc một người dùng API khác trực tiếp đặt trường
> [`nodeName`](138-assign-pod-node-vi.md#nodename) cho một Pod (bỏ qua scheduler), thì Pod đó
> bị ràng buộc vào node đã chỉ định và sẽ chạy ở đó, ngay cả khi bạn đã drain node này và đánh
> dấu nó là unschedulable (không thể lập lịch).

Trước tiên, xác định tên của node mà bạn muốn drain. Bạn có thể liệt kê tất cả các node trong
cluster bằng

```shell
kubectl get nodes
```

Tiếp theo, yêu cầu Kubernetes drain node đó:

```shell
kubectl drain --ignore-daemonsets <node name>
```

Nếu có các Pod do một DaemonSet quản lý, bạn sẽ cần chỉ định `--ignore-daemonsets` với
`kubectl` để drain node thành công. Bản thân lệnh con `kubectl drain` không thực sự rút được
các Pod DaemonSet khỏi node: DaemonSet controller (một phần của control plane) lập tức thay
thế các Pod bị thiếu bằng các Pod tương đương mới. DaemonSet controller cũng tạo ra các Pod
bỏ qua taint unschedulable, điều này cho phép các Pod mới khởi chạy lên chính node mà bạn
đang drain.

Khi lệnh trả về (mà không báo lỗi), bạn có thể tắt nguồn node (hoặc tương đương, nếu ở trên
nền tảng cloud, xóa máy ảo đứng sau node đó). Nếu bạn để node ở lại trong cluster trong suốt
quá trình bảo trì, bạn cần chạy

```shell
kubectl uncordon <node name>
```

sau đó, để báo cho Kubernetes rằng nó có thể tiếp tục lập lịch các Pod mới lên node này.

## Drain nhiều node song song (Draining multiple nodes in parallel)

Lệnh `kubectl drain` chỉ nên được đưa ra cho một node tại một thời điểm. Tuy nhiên, bạn có thể
chạy nhiều lệnh `kubectl drain` cho các node khác nhau một cách song song, trong các terminal
khác nhau hoặc chạy nền (background). Nhiều lệnh drain chạy đồng thời vẫn sẽ tôn trọng
PodDisruptionBudget mà bạn chỉ định.

Ví dụ, nếu bạn có một StatefulSet với ba replica và đã đặt một PodDisruptionBudget cho tập đó
với `minAvailable: 2`, `kubectl drain` chỉ evict một Pod khỏi StatefulSet khi cả ba Pod
replica đều
[khỏe mạnh (healthy)](https://kubernetes.io/docs/tasks/run-application/configure-pdb/#healthiness-of-a-pod);
nếu khi đó bạn đưa ra nhiều lệnh drain song song, Kubernetes vẫn tôn trọng PodDisruptionBudget
và đảm bảo rằng chỉ có 1 Pod (tính bằng `replicas - minAvailable`) là không khả dụng tại bất
kỳ thời điểm nào. Bất kỳ lệnh drain nào có thể khiến số lượng replica
[khỏe mạnh](https://kubernetes.io/docs/tasks/run-application/configure-pdb/#healthiness-of-a-pod)
tụt xuống dưới ngân sách (budget) đã chỉ định đều bị chặn lại.

## API Eviction (The Eviction API) {#eviction-api}

Nếu bạn không muốn dùng
[kubectl drain](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#drain)
(chẳng hạn để tránh gọi một lệnh bên ngoài, hoặc để kiểm soát chi tiết hơn quá trình evict
Pod), bạn cũng có thể kích hoạt eviction theo cách lập trình bằng eviction API.

Để biết thêm thông tin, xem [Eviction do API khởi phát (API-initiated eviction)](143-api-eviction-vi.md).

## Tiếp theo (What's next)

* Làm theo các bước để bảo vệ ứng dụng của bạn bằng cách
  [cấu hình một Pod Disruption Budget](https://kubernetes.io/docs/tasks/run-application/configure-pdb/).
