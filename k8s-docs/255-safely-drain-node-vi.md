# Drain một node an toàn (Safely Drain a Node)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 16 — Vòng đời node](00-ALO-TRINH-ADMIN.md#giai-đoạn-16--vòng-đời-node), bài 1/4 · Kiểm chứng trên cluster
lab: chạy trọn `cordon → drain → uncordon` trên `lab-k8s-worker2` (node duy nhất được phép gây lỗi), và
ở **Lab 12 — Vận hành vòng đời node** khi bạn tới lab đó.

Đây là bài đầu của giai đoạn 16 và là thao tác bảo trì node dùng nhiều nhất trong thực tế. Nó dựa
trực tiếp lên hai bài đã đọc ở mạch chính: [53](53-disruptions-vi.md) (PodDisruptionBudget)
và [143](143-api-eviction-vi.md) (eviction do API khởi phát).

**Phải hiểu ở lần đọc này:**

- `drain` **không phải** là xóa Pod: nó *evict* — tôn trọng `terminationGracePeriodSeconds` để
  container kết thúc êm, và tôn trọng PodDisruptionBudget. Đây là điểm phân biệt với việc tắt
  thẳng máy.
- Vì sao `--ignore-daemonsets` gần như luôn cần: `kubectl drain` **không** rút được Pod của
  DaemonSet — DaemonSet controller thay thế ngay lập tức, và Pod nó tạo ra bỏ qua taint
  `node.kubernetes.io/unschedulable` nên vẫn lên đúng node bạn đang drain.
- Ý nghĩa của việc lệnh **trả về thành công**: mọi Pod (trừ nhóm được loại trừ) đã evict an
  toàn, từ đó mới an toàn tắt nguồn máy. Nếu node vẫn ở lại cluster sau bảo trì thì phải
  `kubectl uncordon` — drain đánh dấu node unschedulable và trạng thái đó không tự mất đi.
- Hai đường vòng khiến Pod vẫn lên được node đã drain: Pod **toleration** taint
  `node.kubernetes.io/unschedulable` (chỉ nên để DaemonSet làm), và Pod đặt thẳng
  [`nodeName`](138-assign-pod-node-vi.md#nodename) — bỏ qua scheduler nên ràng buộc vào node
  bất kể unschedulable.
- Drain nhiều node **song song** là hợp lệ và PDB vẫn được tôn trọng: lệnh drain nào làm số
  replica khỏe mạnh tụt dưới budget sẽ bị chặn lại. Nên đặt `AlwaysAllow` cho Unhealthy Pod
  Eviction Policy, nếu không drain sẽ **đứng chờ** Pod lỗi trở nên khỏe mạnh.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Cách viết và tinh chỉnh một PodDisruptionBudget cụ thể | ở đây chỉ cần biết PDB **chặn** drain, chưa cần tự cấu hình | bài [339](339-configure-pdb-vi.md) ở khối Thực hành 3c — [giai đoạn 3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn) |
| Gọi Eviction API bằng code thay cho `kubectl drain` | chỉ cần khi tự động hóa bảo trì | bài [143](143-api-eviction-vi.md) đã đọc ở giai đoạn 7a |

---

Trang này hướng dẫn cách drain (rút toàn bộ Pod khỏi) một node một cách an toàn, tùy chọn có
tôn trọng PodDisruptionBudget mà bạn đã định nghĩa.

## Trước khi bạn bắt đầu (Before you begin)

Tác vụ này giả định rằng bạn đã đáp ứng các điều kiện tiên quyết sau:

1. Bạn không yêu cầu ứng dụng của mình phải có tính sẵn sàng cao (highly available) trong lúc
   drain node, hoặc
1. Bạn đã đọc về khái niệm [PodDisruptionBudget](53-disruptions-vi.md), và đã
   [cấu hình PodDisruptionBudget](339-configure-pdb-vi.md)
   cho những ứng dụng cần đến nó.

## (Tùy chọn) Cấu hình disruption budget ((Optional) Configure a disruption budget) {#configure-poddisruptionbudget}

Để đảm bảo workload của bạn vẫn khả dụng trong lúc bảo trì, bạn có thể cấu hình một
[PodDisruptionBudget](53-disruptions-vi.md).

Nếu tính khả dụng là quan trọng đối với bất kỳ ứng dụng nào đang chạy hoặc có thể chạy trên
(các) node mà bạn sắp drain, hãy
[cấu hình PodDisruptionBudget](339-configure-pdb-vi.md)
trước, rồi mới tiếp tục làm theo hướng dẫn này.

Bạn nên đặt `AlwaysAllow` cho
[chính sách evict Pod không khỏe mạnh (Unhealthy Pod Eviction Policy)](339-configure-pdb-vi.md#unhealthy-pod-eviction-policy)
trong các PodDisruptionBudget của mình, để hỗ trợ evict những ứng dụng đang hoạt động sai
trong lúc drain node. Hành vi mặc định là chờ các Pod của ứng dụng trở nên
[khỏe mạnh (healthy)](339-configure-pdb-vi.md#healthiness-of-a-pod)
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
[khỏe mạnh (healthy)](339-configure-pdb-vi.md#healthiness-of-a-pod);
nếu khi đó bạn đưa ra nhiều lệnh drain song song, Kubernetes vẫn tôn trọng PodDisruptionBudget
và đảm bảo rằng chỉ có 1 Pod (tính bằng `replicas - minAvailable`) là không khả dụng tại bất
kỳ thời điểm nào. Bất kỳ lệnh drain nào có thể khiến số lượng replica
[khỏe mạnh](339-configure-pdb-vi.md#healthiness-of-a-pod)
tụt xuống dưới ngân sách (budget) đã chỉ định đều bị chặn lại.

## API Eviction (The Eviction API) {#eviction-api}

Nếu bạn không muốn dùng
[kubectl drain](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#drain)
(chẳng hạn để tránh gọi một lệnh bên ngoài, hoặc để kiểm soát chi tiết hơn quá trình evict
Pod), bạn cũng có thể kích hoạt eviction theo cách lập trình bằng eviction API.

Để biết thêm thông tin, xem [Eviction do API khởi phát (API-initiated eviction)](143-api-eviction-vi.md).

## Tiếp theo (What's next)

* Làm theo các bước để bảo vệ ứng dụng của bạn bằng cách
  [cấu hình một Pod Disruption Budget](339-configure-pdb-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 16:

1. Trên cluster lab, `lab-k8s-worker2` đang chạy Pod của DaemonSet (CNI, kube-proxy). Bạn gõ
   `kubectl drain lab-k8s-worker2` và lệnh dừng lại báo lỗi. Vì sao phải thêm `--ignore-daemonsets`,
   và sau khi thêm cờ đó thì các Pod DaemonSet **có bị rút khỏi node** không?
2. **Câu bẫy.** `cordon` và `drain` khác nhau ở chỗ nào? Sau khi drain xong, bảo trì xong và bật
   máy trở lại, node có tự nhận Pod mới không?
3. Lệnh `drain` trả về thành công. Điều đó bảo đảm chính xác những gì — và **không** bảo đảm
   điều gì về số Pod còn chạy trên node?
4. Bạn có StatefulSet 3 replica với PodDisruptionBudget `minAvailable: 2`. Bạn mở hai terminal
   và drain hai node song song. Kubernetes cho phép bao nhiêu Pod của StatefulSet đó không khả
   dụng cùng lúc, và điều gì xảy ra với lệnh drain thứ hai?
5. Vì sao bài khuyên đặt `AlwaysAllow` cho Unhealthy Pod Eviction Policy? Hành vi mặc định gây
   ra vấn đề gì khi bạn đang cần drain gấp một node?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Vì **`kubectl drain` không thể rút Pod do DaemonSet quản lý**: DaemonSet controller lập tức
   tạo lại Pod tương đương, nên drain sẽ không bao giờ "sạch" node. Cờ `--ignore-daemonsets`
   bảo drain **bỏ qua** nhóm Pod đó thay vì thất bại. Trả lời phần sau: **không** — các Pod
   DaemonSet **vẫn ở lại và vẫn chạy** trên node. Bỏ qua ở đây nghĩa là không tính chúng, không
   phải là đã rút chúng đi.
2. `cordon` **chỉ** đánh dấu node là unschedulable — Pod đang chạy vẫn nguyên tại chỗ. `drain`
   làm cả hai việc: đánh dấu unschedulable **và** evict Pod hiện có một cách an toàn. Phần sau
   là chỗ dễ sai: **không**, bật máy lên là chưa đủ. Trạng thái unschedulable nằm trên **object
   Node trong cluster**, không nằm trên máy, nên nó sống sót qua việc tắt/bật. Phải chạy
   `kubectl uncordon <node>` thì node mới nhận Pod mới trở lại.
3. Nó bảo đảm rằng mọi Pod **trừ nhóm được loại trừ** đã được evict an toàn — tôn trọng thời
   gian kết thúc êm và tôn trọng PodDisruptionBudget — nên có thể tắt nguồn máy. Nó **không**
   bảo đảm node trống rỗng: Pod DaemonSet vẫn chạy (khi dùng `--ignore-daemonsets`), và một số
   Pod hệ thống vốn được `kubectl drain` bỏ qua theo mặc định.
4. Đúng **1 Pod** — tính bằng `replicas - minAvailable` = 3 − 2. PDB được thực thi ở tầng
   eviction API chứ không phải ở tầng lệnh `kubectl`, nên **nhiều lệnh drain song song vẫn
   tuân thủ cùng một budget**. Lệnh drain thứ hai không làm hỏng budget: yêu cầu evict nào sẽ
   khiến số replica khỏe mạnh tụt xuống dưới 2 sẽ **bị chặn**, và lệnh đó chờ cho tới khi
   evict được.
5. Vì hành vi **mặc định là chờ Pod của ứng dụng trở nên khỏe mạnh rồi mới cho evict**. Khi
   node đang hỏng và chính các Pod trên đó đang lỗi, điều kiện "khỏe mạnh" có thể **không bao
   giờ đạt được**, và lệnh drain treo vô hạn đúng lúc bạn cần rút node gấp. `AlwaysAllow` cho
   phép evict cả Pod đang hoạt động sai, phá được thế kẹt đó.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng, rồi thực hành trọn vòng
`cordon → drain --ignore-daemonsets → uncordon` trên `lab-k8s-worker2` trước khi sang bài kế của
[Giai đoạn 16 — Vòng đời node](00-ALO-TRINH-ADMIN.md#giai-đoạn-16--vòng-đời-node).
