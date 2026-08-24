# Gán Pod cho Node (Assigning Pods to Nodes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 7 → nhóm [7a](00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction), bài 3/13 ·
Kiểm chứng ở Lab 7a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

**Bài dài nhất nhóm 7a.** Nó gộp bốn cơ chế độc lập vào một trang — `nodeSelector`, affinity,
`nodeName`, và một mục dẫn sang topology spread — cộng thêm nhiều trường ở mức beta. Ngoài ra
gần như mọi ví dụ đều dùng label zone (`topology.kubernetes.io/zone`, `antarctica-east1`,
"Zone V"): cluster lab của bạn **không có node nào mang label zone**, nên khi đọc hãy tự thay
zone bằng `kubernetes.io/hostname` — quy tắc y hệt, chỉ đổi miền topology từ zone sang node.

**Phải hiểu ở lần đọc này:**

- `nodeSelector` khớp theo **label của node** và Pod chỉ lên node có **đủ tất cả** các label
  bạn liệt kê. Đây là dạng ràng buộc đơn giản nhất, và là dạng được khuyến nghị khi đủ dùng.
- Node affinity có hai loại: `requiredDuringSchedulingIgnoredDuringExecution` (cứng — không
  thỏa thì không lập lịch) và `preferredDuringSchedulingIgnoredDuringExecution` (mềm — không
  tìm được node phù hợp thì **vẫn lập lịch**, `weight` 1–100 cộng vào điểm ở bước chấm điểm).
  `IgnoredDuringExecution` nghĩa là label node đổi sau khi Pod đã được lập lịch thì **Pod vẫn
  chạy tiếp**.
- Quy tắc kết hợp trong ghi chú ở mục *Node affinity*: nhiều `nodeSelectorTerms` được **OR**;
  nhiều biểu thức trong cùng một `matchExpressions` được **AND**; đặt cả `nodeSelector` lẫn
  `nodeAffinity` thì **cả hai** phải thỏa. Toán tử dùng được: `In`, `NotIn`, `Exists`,
  `DoesNotExist` (và `Gt`, `Lt` chỉ cho node affinity).
- **Inter-pod affinity/anti-affinity** ràng buộc theo label của **các Pod khác** đang chạy
  trong một miền topology do `topologyKey` chỉ ra, không phải label node. Ví dụ redis-cache +
  web-server ở mục *Các trường hợp sử dụng thực tế hơn* là mẫu cần nhớ: anti-affinity `required`
  với `topologyKey: kubernetes.io/hostname` cho mỗi node đúng một bản sao.
- `nodeName` **bỏ qua scheduler hoàn toàn** và ghi đè `nodeSelector` lẫn affinity; nếu node
  không đủ tài nguyên thì Pod thất bại với lý do `OutOfmemory`/`OutOfcpu` chứ không được sắp
  xếp lại. Bài xếp nó vào diện dành cho scheduler tùy biến, kèm cảnh báo rõ ràng.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Cô lập/hạn chế node* — `NodeRestriction`, Node authorizer, tiền tố `node-restriction.kubernetes.io/` | là thao tác hardening, cần authn/authz | giai đoạn 9, bài [119](119-controlling-access-vi.md) |
| *Node affinity theo từng scheduling profile* (`addedAffinity`) | cần biết scheduling profile trước | bài [147](147-scheduling-framework-vi.md) |
| *Bộ chọn namespace*, `matchLabelKeys`, `mismatchLabelKeys` | tinh chỉnh cho rollout nhiều revision và multi-tenant | giai đoạn 9, bài [122](122-multi-tenancy-vi.md) |
| *Hành vi lập lịch* mục 3 — các trường `preferred` của Pod hiện có bị bỏ qua | là chi tiết biên của thuật toán | Lab 7a |
| `nominatedNodeName` | là hệ quả của preemption | bài [141](141-pod-priority-preemption-vi.md) |
| *Ràng buộc phân bố Pod theo topology* (chỉ là mục dẫn) | có bài riêng ngay sau | bài [140](140-topology-spread-constraints-vi.md) |
| *Label topology của Pod* qua Downward API | cluster lab không có label zone/region | không cần |
| Hai dòng `Gt`/`Lt` trong bảng *Toán tử* | chỉ dùng với label node có giá trị số | không cần |

---

Bạn có thể ràng buộc một Pod sao cho nó bị _giới hạn_ chỉ chạy trên (các) node cụ thể,
hoặc _ưu tiên_ chạy trên các node cụ thể.
Có nhiều cách để làm điều này và các cách tiếp cận được khuyến nghị đều dùng
[bộ chọn label (label selector)](./18-labels-vi.md) để thuận tiện cho việc chọn lựa.
Thông thường, bạn không cần đặt bất kỳ ràng buộc nào như vậy; bộ lập lịch (scheduler)
sẽ tự động sắp đặt một cách hợp lý
(ví dụ: phân bố các Pod của bạn ra nhiều node để không đặt Pod lên một node không đủ tài nguyên trống).
Tuy nhiên, có một số tình huống bạn muốn kiểm soát Pod được triển khai lên node nào,
ví dụ để đảm bảo một Pod nằm trên node có gắn ổ SSD,
hoặc để đặt các Pod của hai service khác nhau giao tiếp với nhau nhiều vào cùng một availability zone.

Bạn có thể dùng bất kỳ phương pháp nào sau đây để chọn nơi Kubernetes lập lịch
các Pod cụ thể:

- Trường [nodeSelector](#nodeselector) đối chiếu với [label của node](#built-in-node-labels)
- [Affinity và anti-affinity](#affinity-and-anti-affinity)
- Trường [nodeName](#nodename)
- [Ràng buộc phân bố Pod theo topology](#pod-topology-spread-constraints)

## Label của node (Node labels) {#built-in-node-labels}

Giống như nhiều đối tượng Kubernetes khác, node cũng có
[label](./18-labels-vi.md). Bạn có thể
[gắn label thủ công](267-assign-pods-nodes-vi.md#add-a-label-to-a-node).
Kubernetes cũng tự điền một [tập label chuẩn](https://kubernetes.io/docs/reference/node/node-labels/)
trên mọi node trong một cluster.

> **Ghi chú:**
> Giá trị của các label này tùy thuộc từng nhà cung cấp cloud và không được đảm bảo là đáng tin cậy.
> Ví dụ, giá trị của `kubernetes.io/hostname` có thể trùng với tên node trong một số môi trường
> và mang giá trị khác trong các môi trường khác.

### Cô lập/hạn chế node (Node isolation/restriction) {#node-isolation-restriction}

Việc thêm label cho node cho phép bạn nhắm các Pod được lập lịch lên các node
hoặc nhóm node cụ thể. Bạn có thể dùng chức năng này để đảm bảo các Pod nhất định
chỉ chạy trên những node có các thuộc tính cô lập, bảo mật hoặc tuân thủ
quy định nhất định.

Nếu bạn dùng label để cô lập node, hãy chọn các khóa (key) label mà kubelet
không thể sửa đổi. Điều này ngăn một node bị xâm nhập tự đặt các label đó lên
chính nó, khiến bộ lập lịch đưa workload lên node bị xâm nhập.

[Plugin admission `NodeRestriction`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction)
ngăn kubelet đặt hoặc sửa đổi các label có tiền tố
`node-restriction.kubernetes.io/`.

Để tận dụng tiền tố label đó cho việc cô lập node:

1. Đảm bảo bạn đang dùng [Node authorizer](https://kubernetes.io/docs/reference/access-authn-authz/node/) và đã _bật_ plugin admission `NodeRestriction`.
2. Thêm các label có tiền tố `node-restriction.kubernetes.io/` cho các node của bạn, và dùng các label đó trong [node selector](#nodeselector).
   Ví dụ: `example.com.node-restriction.kubernetes.io/fips=true` hoặc `example.com.node-restriction.kubernetes.io/pci-dss=true`.

## nodeSelector

`nodeSelector` là dạng ràng buộc chọn node đơn giản nhất được khuyến nghị.
Bạn có thể thêm trường `nodeSelector` vào đặc tả (specification) Pod của mình và chỉ định
các [label của node](#built-in-node-labels) mà bạn muốn node đích phải có.
Kubernetes chỉ lập lịch Pod lên các node có đầy đủ từng label mà bạn
chỉ định.

Xem [Gán Pod cho Node](267-assign-pods-nodes-vi.md) để biết thêm
thông tin.

## Affinity và anti-affinity (Affinity and anti-affinity) {#affinity-and-anti-affinity}

`nodeSelector` là cách đơn giản nhất để ràng buộc Pod vào các node có label
cụ thể. Affinity và anti-affinity mở rộng các loại ràng buộc mà bạn có thể
định nghĩa. Một số lợi ích của affinity và anti-affinity gồm:

- Ngôn ngữ affinity/anti-affinity có khả năng biểu đạt phong phú hơn. `nodeSelector` chỉ
  chọn các node có tất cả các label được chỉ định. Affinity/anti-affinity cho bạn
  nhiều quyền kiểm soát hơn đối với logic chọn lựa.
- Bạn có thể chỉ ra rằng một quy tắc là *mềm* (soft) hay *ưu tiên* (preferred), nhờ đó bộ lập lịch
  vẫn lập lịch Pod ngay cả khi không tìm được node phù hợp.
- Bạn có thể ràng buộc một Pod bằng label của các Pod khác đang chạy trên node (hoặc miền topology khác),
  thay vì chỉ dùng label của node, điều này cho phép bạn định nghĩa quy tắc về việc
  những Pod nào có thể được đặt cùng nhau (co-located) trên một node.

Tính năng affinity gồm hai loại affinity:

- *Node affinity* hoạt động giống trường `nodeSelector` nhưng có khả năng biểu đạt phong phú hơn và
  cho phép bạn chỉ định các quy tắc mềm.
- *Inter-pod affinity/anti-affinity* cho phép bạn ràng buộc Pod dựa trên label
  của các Pod khác.

### Node affinity

Node affinity về mặt khái niệm tương tự `nodeSelector`, cho phép bạn ràng buộc những node nào
Pod của bạn có thể được lập lịch lên, dựa trên label của node. Có hai loại node
affinity:

- `requiredDuringSchedulingIgnoredDuringExecution`: Bộ lập lịch không thể
  lập lịch Pod trừ khi quy tắc được thỏa mãn. Loại này hoạt động giống `nodeSelector`,
  nhưng với cú pháp biểu đạt phong phú hơn.
- `preferredDuringSchedulingIgnoredDuringExecution`: Bộ lập lịch cố gắng
  tìm một node thỏa mãn quy tắc. Nếu không có node nào phù hợp, bộ lập lịch
  vẫn lập lịch Pod.

> **Ghi chú:**
> Trong các loại nêu trên, `IgnoredDuringExecution` nghĩa là nếu label của node
> thay đổi sau khi Kubernetes đã lập lịch Pod, Pod vẫn tiếp tục chạy.

Bạn có thể chỉ định node affinity bằng trường `.spec.affinity.nodeAffinity` trong
spec của Pod.

Ví dụ, xét spec Pod sau:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - antarctica-east1
            - antarctica-west1
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: registry.k8s.io/pause:3.8
```

Trong ví dụ này, các quy tắc sau được áp dụng:

- Node *bắt buộc* phải có một label với khóa `topology.kubernetes.io/zone` và
  giá trị của label đó *phải* là `antarctica-east1` hoặc `antarctica-west1`.
- Node *nên* (ưu tiên) có một label với khóa `another-node-label-key` và
  giá trị `another-node-label-value`.

Bạn có thể dùng trường `operator` để chỉ định toán tử logic mà Kubernetes sử dụng khi
diễn giải các quy tắc. Bạn có thể dùng `In`, `NotIn`, `Exists`, `DoesNotExist`,
`Gt` và `Lt`.

Đọc mục [Toán tử](#operators)
để tìm hiểu thêm về cách chúng hoạt động.

`NotIn` và `DoesNotExist` cho phép bạn định nghĩa hành vi node anti-affinity.
Ngoài ra, bạn có thể dùng [taint của node](139-taint-and-toleration-vi.md)
để đẩy Pod ra khỏi các node cụ thể.

> **Ghi chú:**
> Nếu bạn chỉ định cả `nodeSelector` lẫn `nodeAffinity`, *cả hai* phải được thỏa mãn
> thì Pod mới được lập lịch lên node.
>
> Nếu bạn chỉ định nhiều điều khoản (term) trong `nodeSelectorTerms` gắn với các loại
> `nodeAffinity`, thì Pod có thể được lập lịch lên node nếu một trong các điều khoản
> được thỏa mãn (các term được OR với nhau).
>
> Nếu bạn chỉ định nhiều biểu thức trong cùng một trường `matchExpressions` thuộc một
> term trong `nodeSelectorTerms`, thì Pod chỉ có thể được lập lịch lên node
> khi tất cả các biểu thức đều được thỏa mãn (các biểu thức được AND với nhau).

Xem [Gán Pod cho Node bằng Node Affinity](266-assign-pods-nodes-node-affinity-vi.md)
để biết thêm thông tin.

#### Trọng số node affinity (Node affinity weight)

Bạn có thể chỉ định `weight` (trọng số) từ 1 đến 100 cho mỗi thể hiện của loại affinity
`preferredDuringSchedulingIgnoredDuringExecution`. Khi bộ lập lịch
tìm được các node đáp ứng mọi yêu cầu lập lịch khác của Pod, bộ lập lịch
duyệt qua từng quy tắc preferred mà node thỏa mãn và cộng
giá trị `weight` của biểu thức đó vào một tổng.

Tổng cuối cùng được cộng vào điểm số của các hàm ưu tiên khác cho node đó.
Các node có tổng điểm cao nhất được ưu tiên khi bộ lập lịch ra
quyết định lập lịch cho Pod.

Ví dụ, xét spec Pod sau:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-affinity-preferred-weight
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: label-1
            operator: In
            values:
            - key-1
      - weight: 50
        preference:
          matchExpressions:
          - key: label-2
            operator: In
            values:
            - key-2
  containers:
  - name: with-node-affinity
    image: registry.k8s.io/pause:3.8
```

Nếu có hai node khả dĩ đều khớp quy tắc
`preferredDuringSchedulingIgnoredDuringExecution`, một node có
label `label-1:key-1` và node kia có label `label-2:key-2`, bộ lập lịch
xem xét `weight` của từng node và cộng trọng số đó vào các điểm số khác của
node đó, rồi lập lịch Pod lên node có điểm cuối cùng cao nhất.

> **Ghi chú:**
> Nếu bạn muốn Kubernetes lập lịch thành công các Pod trong ví dụ này, bạn
> phải có sẵn các node mang label `kubernetes.io/os=linux`.

#### Node affinity theo từng scheduling profile (Node affinity per scheduling profile)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.20 [beta]`

Khi cấu hình nhiều [scheduling profile](https://kubernetes.io/docs/reference/scheduling/config/#multiple-profiles), bạn có thể gắn
một profile với một node affinity, điều này hữu ích khi một profile chỉ áp dụng cho một tập node cụ thể.
Để làm vậy, thêm `addedAffinity` vào trường `args` của [plugin `NodeAffinity`](https://kubernetes.io/docs/reference/scheduling/config/#scheduling-plugins)
trong [cấu hình bộ lập lịch](https://kubernetes.io/docs/reference/scheduling/config/). Ví dụ:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration

profiles:
  - schedulerName: default-scheduler
  - schedulerName: foo-scheduler
    pluginConfig:
      - name: NodeAffinity
        args:
          addedAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: scheduler-profile
                  operator: In
                  values:
                  - foo
```

`addedAffinity` được áp dụng cho tất cả các Pod đặt `.spec.schedulerName` là `foo-scheduler`, bên cạnh
NodeAffinity được chỉ định trong PodSpec.
Nghĩa là, để khớp với Pod, node cần thỏa mãn cả `addedAffinity` lẫn
`.spec.NodeAffinity` của Pod.

Vì `addedAffinity` không hiển thị với người dùng cuối, hành vi của nó có thể
gây bất ngờ cho họ. Hãy dùng các label node có mối liên hệ rõ ràng với
tên profile của bộ lập lịch.

> **Ghi chú:**
> Controller DaemonSet, thành phần [tạo Pod cho các DaemonSet](66-daemonset-vi.md#how-daemon-pods-are-scheduled),
> không hỗ trợ scheduling profile. Khi controller DaemonSet tạo
> Pod, bộ lập lịch Kubernetes mặc định sẽ sắp đặt các Pod đó và tôn trọng mọi
> quy tắc `nodeAffinity` trong controller DaemonSet.

### Inter-pod affinity và anti-affinity (Inter-pod affinity and anti-affinity) {#inter-pod-affinity-and-anti-affinity}

Inter-pod affinity và anti-affinity cho phép bạn ràng buộc những node nào
Pod của bạn có thể được lập lịch lên dựa trên label của các Pod đã chạy trên
node đó, thay vì label của node.

#### Các loại inter-pod affinity và anti-affinity (Types of Inter-pod Affinity and Anti-affinity)

Inter-pod affinity và anti-affinity có dạng "Pod này nên (hoặc, trong
trường hợp anti-affinity, không nên) chạy trong một X nếu X đó
đã chạy một hoặc nhiều Pod thỏa mãn quy tắc Y", trong đó X là một miền topology
(topology domain) như node, rack, zone hoặc region của nhà cung cấp cloud, hoặc tương tự, và Y là
quy tắc mà Kubernetes cố gắng thỏa mãn.

Bạn biểu đạt các quy tắc (Y) này dưới dạng [bộ chọn label (label selector)](18-labels-vi.md#label-selectors)
kèm theo một danh sách namespace tùy chọn. Pod là các đối tượng thuộc namespace trong
Kubernetes, nên label của Pod cũng ngầm định có namespace. Bất kỳ bộ chọn label nào
cho label của Pod nên chỉ định các namespace mà Kubernetes cần tìm những
label đó.

Bạn biểu đạt miền topology (X) bằng `topologyKey`, là khóa của
label node mà hệ thống dùng để biểu thị miền đó. Để xem ví dụ, tham khảo
[Well-Known Labels, Annotations and Taints](https://kubernetes.io/docs/reference/labels-annotations-taints/).

> **Ghi chú:**
> Inter-pod affinity và anti-affinity đòi hỏi lượng xử lý đáng kể,
> có thể làm chậm việc lập lịch một cách rõ rệt trong các cluster lớn. Chúng tôi
> không khuyến nghị dùng chúng trong các cluster lớn hơn vài trăm node.

> **Ghi chú:**
> Pod anti-affinity yêu cầu các node được gắn label một cách nhất quán, nói cách khác,
> mọi node trong cluster phải có label phù hợp khớp với `topologyKey`.
> Nếu một số hoặc tất cả node thiếu label `topologyKey` được chỉ định, điều đó có thể dẫn
> tới hành vi ngoài ý muốn.

Tương tự [node affinity](#node-affinity), có hai loại Pod affinity và
anti-affinity như sau:

- `requiredDuringSchedulingIgnoredDuringExecution`
- `preferredDuringSchedulingIgnoredDuringExecution`

Ví dụ, bạn có thể dùng affinity
`requiredDuringSchedulingIgnoredDuringExecution` để yêu cầu bộ lập lịch
đặt các Pod của hai service vào cùng một zone của nhà cung cấp cloud vì chúng
giao tiếp với nhau nhiều. Tương tự, bạn có thể dùng anti-affinity
`preferredDuringSchedulingIgnoredDuringExecution` để phân bố các Pod
của một service ra nhiều zone của nhà cung cấp cloud.

Để dùng inter-pod affinity, dùng trường `affinity.podAffinity` trong spec của Pod.
Với inter-pod anti-affinity, dùng trường `affinity.podAntiAffinity` trong spec của
Pod.

#### Hành vi lập lịch (Scheduling Behavior)

Khi lập lịch một Pod mới, bộ lập lịch Kubernetes đánh giá các quy tắc affinity/anti-affinity của Pod trong ngữ cảnh trạng thái hiện tại của cluster:

1. Ràng buộc cứng (lọc node):
   - `podAffinity.requiredDuringSchedulingIgnoredDuringExecution` và `podAntiAffinity.requiredDuringSchedulingIgnoredDuringExecution`:
     - Bộ lập lịch đảm bảo Pod mới được gán vào các node thỏa mãn các quy tắc affinity và anti-affinity bắt buộc này, dựa trên các Pod hiện có.

2. Ràng buộc mềm (chấm điểm):
   - `podAffinity.preferredDuringSchedulingIgnoredDuringExecution` và `podAntiAffinity.preferredDuringSchedulingIgnoredDuringExecution`:
     - Bộ lập lịch chấm điểm các node dựa trên mức độ đáp ứng các quy tắc affinity và anti-affinity ưu tiên này, nhằm tối ưu vị trí đặt Pod.

3. Các trường bị bỏ qua:
   - `podAffinity.preferredDuringSchedulingIgnoredDuringExecution` của các Pod hiện có:
     - Các quy tắc affinity ưu tiên này không được xem xét trong quyết định lập lịch cho Pod mới.
   - `podAntiAffinity.preferredDuringSchedulingIgnoredDuringExecution` của các Pod hiện có:
     - Tương tự, các quy tắc anti-affinity ưu tiên của các Pod hiện có cũng bị bỏ qua khi lập lịch.

#### Lập lịch một nhóm Pod có inter-pod affinity với chính chúng (Scheduling a Group of Pods with Inter-pod Affinity to Themselves)

Nếu Pod hiện đang được lập lịch là Pod đầu tiên trong một loạt Pod có affinity với chính chúng,
nó được phép lập lịch nếu vượt qua mọi kiểm tra affinity khác. Điều này được xác định bằng cách
xác minh rằng không có Pod nào khác trong cluster khớp với namespace và selector của Pod này,
rằng Pod khớp với chính các điều khoản của nó, và node được chọn khớp với mọi topology được yêu cầu.
Điều này đảm bảo sẽ không xảy ra deadlock ngay cả khi tất cả các Pod đều chỉ định inter-pod
affinity.

#### Ví dụ về Pod affinity (Pod Affinity Example) {#an-example-of-a-pod-that-uses-pod-affinity}

Xét spec Pod sau:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: topology.kubernetes.io/zone
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: topology.kubernetes.io/zone
  containers:
  - name: with-pod-affinity
    image: registry.k8s.io/pause:3.8
```

Ví dụ này định nghĩa một quy tắc Pod affinity và một quy tắc Pod anti-affinity. Quy tắc
Pod affinity dùng loại "cứng"
`requiredDuringSchedulingIgnoredDuringExecution`, trong khi quy tắc anti-affinity
dùng loại "mềm" `preferredDuringSchedulingIgnoredDuringExecution`.

Quy tắc affinity chỉ định rằng bộ lập lịch chỉ được phép đặt Pod ví dụ
lên một node nếu node đó thuộc một [zone](140-topology-spread-constraints-vi.md) cụ thể
nơi các Pod khác đã được gắn label `security=S1`.
Chẳng hạn, nếu chúng ta có một cluster với một zone được chỉ định, gọi là "Zone V",
gồm các node mang label `topology.kubernetes.io/zone=V`, bộ lập lịch có thể
gán Pod vào bất kỳ node nào trong Zone V, miễn là có ít nhất một Pod trong
Zone V đã mang label `security=S1`. Ngược lại, nếu không có Pod nào mang label `security=S1`
trong Zone V, bộ lập lịch sẽ không gán Pod ví dụ vào bất kỳ node nào trong zone đó.

Quy tắc anti-affinity chỉ định rằng bộ lập lịch nên cố tránh lập lịch Pod
lên một node nếu node đó thuộc một [zone](140-topology-spread-constraints-vi.md) cụ thể
nơi các Pod khác đã được gắn label `security=S2`.
Chẳng hạn, nếu chúng ta có một cluster với một zone được chỉ định, gọi là "Zone R",
gồm các node mang label `topology.kubernetes.io/zone=R`, bộ lập lịch nên tránh
gán Pod vào bất kỳ node nào trong Zone R, miễn là có ít nhất một Pod trong
Zone R đã mang label `security=S2`. Ngược lại, quy tắc anti-affinity không ảnh hưởng
đến việc lập lịch vào Zone R nếu không có Pod nào mang label `security=S2`.

Để làm quen hơn với các ví dụ về Pod affinity và anti-affinity,
tham khảo [đề xuất thiết kế](https://git.k8s.io/design-proposals-archive/scheduling/podaffinity.md).

Bạn có thể dùng các giá trị `In`, `NotIn`, `Exists` và `DoesNotExist` trong
trường `operator` cho Pod affinity và anti-affinity.

Đọc mục [Toán tử](#operators)
để tìm hiểu thêm về cách chúng hoạt động.

Về nguyên tắc, `topologyKey` có thể là bất kỳ khóa label hợp lệ nào, với các
ngoại lệ sau vì lý do hiệu năng và bảo mật:

- Với Pod affinity và anti-affinity, trường `topologyKey` rỗng không được phép trong cả
  `requiredDuringSchedulingIgnoredDuringExecution`
  lẫn `preferredDuringSchedulingIgnoredDuringExecution`.
- Với các quy tắc Pod anti-affinity loại `requiredDuringSchedulingIgnoredDuringExecution`,
  admission controller `LimitPodHardAntiAffinityTopology` giới hạn
  `topologyKey` chỉ được là `kubernetes.io/hostname`. Bạn có thể sửa đổi hoặc vô hiệu hóa
  admission controller này nếu muốn cho phép các topology tùy biến.

Ngoài `labelSelector` và `topologyKey`, bạn có thể tùy chọn chỉ định một danh sách
namespace mà `labelSelector` sẽ được đối chiếu, bằng
trường `namespaces` ở cùng cấp với `labelSelector` và `topologyKey`.
Nếu bị bỏ trống hoặc rỗng, `namespaces` mặc định là namespace của Pod nơi
định nghĩa affinity/anti-affinity xuất hiện.

#### Bộ chọn namespace (Namespace Selector)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.24 [stable]`

Bạn cũng có thể chọn các namespace khớp bằng `namespaceSelector`, là một truy vấn label trên tập các namespace.
Điều khoản affinity được áp dụng cho các namespace được chọn bởi cả `namespaceSelector` lẫn trường `namespaces`.
Lưu ý rằng một `namespaceSelector` rỗng ({}) khớp với tất cả các namespace, trong khi danh sách `namespaces` null hoặc rỗng và
`namespaceSelector` null sẽ khớp với namespace của Pod nơi quy tắc được định nghĩa.

#### matchLabelKeys

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.33 [stable]`

> **Ghi chú:**
> Trường `matchLabelKeys` là trường ở cấp beta và được bật mặc định trong
> Kubernetes v1.36.
> Khi muốn tắt nó, bạn phải tắt tường minh thông qua
> [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) `MatchLabelKeysInPodAffinity`.

Kubernetes có trường tùy chọn `matchLabelKeys` cho Pod affinity
hoặc anti-affinity. Trường này chỉ định các khóa của những label cần khớp với label của Pod đang đến (incoming Pod),
khi thỏa mãn Pod (anti)affinity.

Các khóa được dùng để tra giá trị từ label của Pod; các label khóa-giá trị đó được kết hợp
(bằng `AND`) với các điều kiện khớp được định nghĩa qua trường `labelSelector`. Việc lọc
kết hợp này chọn ra tập các Pod hiện có sẽ được đưa vào tính toán Pod (anti)affinity.

> **Thận trọng:**
> Không khuyến nghị dùng `matchLabelKeys` với các label có thể bị cập nhật trực tiếp trên pod.
> Ngay cả khi bạn sửa **trực tiếp** label của pod được chỉ định tại `matchLabelKeys` (tức là không thông qua deployment),
> kube-apiserver cũng không phản ánh việc cập nhật label đó vào `labelSelector` đã được hợp nhất.

Một trường hợp sử dụng phổ biến là dùng `matchLabelKeys` với `pod-template-hash` (được đặt trên các Pod
được quản lý như một phần của Deployment, với giá trị là duy nhất cho mỗi bản sửa đổi (revision)).
Dùng `pod-template-hash` trong `matchLabelKeys` cho phép bạn nhắm tới các Pod thuộc
cùng revision với Pod đang đến, nhờ đó một lần nâng cấp cuốn chiếu (rolling upgrade) sẽ không phá vỡ affinity.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: application-server
...
spec:
  template:
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - database
            topologyKey: topology.kubernetes.io/zone
            # Chỉ các Pod thuộc một lần rollout nhất định mới được xét đến khi tính toán pod affinity.
            # Nếu bạn cập nhật Deployment, các Pod thay thế sẽ tuân theo quy tắc affinity của riêng chúng
            # (nếu có được định nghĩa trong template Pod mới)
            matchLabelKeys:
            - pod-template-hash
```

#### mismatchLabelKeys

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.33 [stable]`

> **Ghi chú:**
> Trường `mismatchLabelKeys` là trường ở cấp beta và được bật mặc định trong
> Kubernetes v1.36.
> Khi muốn tắt nó, bạn phải tắt tường minh thông qua
> [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) `MatchLabelKeysInPodAffinity`.

Kubernetes có trường tùy chọn `mismatchLabelKeys` cho Pod affinity
hoặc anti-affinity. Trường này chỉ định các khóa của những label không được khớp với label của Pod đang đến,
khi thỏa mãn Pod (anti)affinity.

> **Thận trọng:**
> Không khuyến nghị dùng `mismatchLabelKeys` với các label có thể bị cập nhật trực tiếp trên pod.
> Ngay cả khi bạn sửa **trực tiếp** label của pod được chỉ định tại `mismatchLabelKeys` (tức là không thông qua deployment),
> kube-apiserver cũng không phản ánh việc cập nhật label đó vào `labelSelector` đã được hợp nhất.

Một ví dụ sử dụng là đảm bảo các Pod đi vào miền topology (node, zone, v.v.) nơi chỉ có các Pod của cùng một tenant hoặc team được lập lịch.
Nói cách khác, bạn muốn tránh chạy Pod của hai tenant khác nhau trên cùng một miền topology cùng lúc.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    # Giả định rằng mọi Pod liên quan đều được đặt label "tenant"
    tenant: tenant-a
...
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      # đảm bảo các Pod gắn với tenant này được đặt lên đúng node pool
      - matchLabelKeys:
          - tenant
        labelSelector: {}
        topologyKey: node-pool
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      # đảm bảo các Pod gắn với tenant này không thể lập lịch lên các node đang dùng cho tenant khác
      - mismatchLabelKeys:
        - tenant # dù giá trị label "tenant" của Pod này là gì, ngăn
                 # việc lập lịch lên các node thuộc bất kỳ pool nào đang chạy
                 # Pod của một tenant khác.
        labelSelector:
          # Phải có labelSelector chỉ chọn các Pod có label tenant,
          # nếu không Pod này sẽ có anti-affinity với cả các Pod từ daemonset chẳng hạn,
          # vốn không được kỳ vọng có label tenant.
          matchExpressions:
          - key: tenant
            operator: Exists
        topologyKey: node-pool
```

#### Các trường hợp sử dụng thực tế hơn (More practical use-cases)

Inter-pod affinity và anti-affinity thậm chí còn hữu ích hơn khi được dùng cùng các tập hợp
cấp cao hơn như ReplicaSet, StatefulSet, Deployment, v.v. Các
quy tắc này cho phép bạn cấu hình để một tập workload được
đặt cùng nhau trong cùng một topology đã định nghĩa; ví dụ, ưu tiên đặt hai Pod
liên quan lên cùng một node.

Ví dụ: hãy hình dung một cluster ba node. Bạn dùng cluster này để chạy một ứng dụng web
và một bộ nhớ đệm trong bộ nhớ (in-memory cache) như Redis. Trong ví dụ này, cũng giả định độ trễ giữa
ứng dụng web và bộ nhớ đệm nên thấp ở mức khả thi. Bạn có thể dùng inter-pod
affinity và anti-affinity để đặt các web server cùng chỗ với cache nhiều nhất có thể.

Trong ví dụ Deployment sau cho Redis cache, các bản sao (replica) được gắn label `app=store`. Quy tắc
`podAntiAffinity` yêu cầu bộ lập lịch tránh đặt nhiều bản sao
có label `app=store` trên cùng một node. Điều này tạo mỗi cache trên một
node riêng biệt.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  selector:
    matchLabels:
      app: store
  replicas: 3
  template:
    metadata:
      labels:
        app: store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: redis-server
        image: redis:3.2-alpine
```

Ví dụ Deployment sau cho các web server tạo các bản sao với label `app=web-store`.
Quy tắc Pod affinity yêu cầu bộ lập lịch đặt mỗi bản sao lên một node có Pod
mang label `app=store`. Quy tắc Pod anti-affinity yêu cầu bộ lập lịch không bao giờ đặt
nhiều server `app=web-store` trên cùng một node.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  selector:
    matchLabels:
      app: web-store
  replicas: 3
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-store
            topologyKey: "kubernetes.io/hostname"
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web-app
        image: nginx:1.16-alpine
```

Việc tạo hai Deployment trên dẫn đến bố cục cluster như sau,
trong đó mỗi web server được đặt cùng chỗ với một cache, trên ba node riêng biệt.

|    node-1     |    node-2     |    node-3     |
| :-----------: | :-----------: | :-----------: |
| *webserver-1* | *webserver-2* | *webserver-3* |
|   *cache-1*   |   *cache-2*   |   *cache-3*   |

Hiệu quả tổng thể là mỗi thể hiện (instance) cache nhiều khả năng chỉ được truy cập bởi một client duy nhất
đang chạy trên cùng node. Cách tiếp cận này nhằm giảm thiểu cả độ lệch (skew — tải mất cân bằng) lẫn độ trễ.

Bạn có thể có những lý do khác để dùng Pod anti-affinity.
Xem [hướng dẫn ZooKeeper](https://kubernetes.io/docs/tutorials/stateful-application/zookeeper/#tolerating-node-failure)
để có ví dụ về một StatefulSet được cấu hình anti-affinity nhằm đạt tính sẵn sàng cao
(high availability), dùng cùng kỹ thuật với ví dụ này.

## nodeName

`nodeName` là cách chọn node trực tiếp hơn so với affinity hay
`nodeSelector`. `nodeName` là một trường trong spec của Pod. Nếu trường `nodeName`
không rỗng, bộ lập lịch sẽ bỏ qua Pod này và kubelet trên node có tên đó
sẽ cố đặt Pod lên node đó. Việc dùng `nodeName` sẽ ghi đè lên việc dùng
`nodeSelector` hoặc các quy tắc affinity và anti-affinity.

Một số hạn chế khi dùng `nodeName` để chọn node:

- Nếu node có tên đó không tồn tại, Pod sẽ không chạy, và trong
  một số trường hợp có thể tự động bị xóa.
- Nếu node có tên đó không có đủ tài nguyên để chứa
  Pod, Pod sẽ thất bại và lý do (reason) của nó sẽ chỉ ra nguyên nhân,
  ví dụ OutOfmemory hoặc OutOfcpu.
- Tên node trong các môi trường cloud không phải lúc nào cũng dự đoán được hay ổn định.

> **Cảnh báo:**
> `nodeName` được thiết kế cho các bộ lập lịch tùy biến hoặc các trường hợp nâng cao
> cần bỏ qua mọi bộ lập lịch đã cấu hình. Việc bỏ qua bộ lập lịch có thể dẫn đến
> các Pod thất bại nếu các Node được gán bị quá tải (oversubscribed). Bạn có thể dùng [node affinity](#node-affinity)
> hoặc [trường `nodeSelector`](#nodeselector) để gán một Pod vào một Node cụ thể mà không bỏ qua bộ lập lịch.

Dưới đây là ví dụ spec Pod dùng trường `nodeName`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  nodeName: kube-01
```

Pod trên sẽ chỉ chạy trên node `kube-01`.

## nominatedNodeName

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [beta]`

`nominatedNodeName` có thể được các thành phần bên ngoài dùng để đề cử (nominate) node cho một pod đang chờ (pending).
Việc đề cử này mang tính nỗ lực tối đa (best effort): nó có thể bị bỏ qua nếu bộ lập lịch xác định pod không thể lên node được đề cử.

Ngoài ra, trường này có thể bị bộ lập lịch ghi (đè):
- Nếu bộ lập lịch tìm được một node để đề cử thông qua preemption.
- Nếu bộ lập lịch quyết định nơi pod sẽ đến, và chuyển pod sang chu trình binding.
  - Lưu ý rằng, trong trường hợp này, `nominatedNodeName` chỉ được đặt khi pod phải đi qua các điểm mở rộng (extension point) `WaitOnPermit` hoặc `PreBind`.

Dưới đây là ví dụ status của Pod dùng trường `nominatedNodeName`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
...
status:
  nominatedNodeName: kube-01
```

## Ràng buộc phân bố Pod theo topology (Pod topology spread constraints) {#pod-topology-spread-constraints}

Bạn có thể dùng _ràng buộc phân bố theo topology_ (topology spread constraints) để kiểm soát cách các Pod
được phân bố trên cluster của bạn giữa các miền chịu lỗi (failure-domain) như region, zone, node, hoặc giữa bất kỳ
miền topology nào khác do bạn định nghĩa. Bạn có thể làm vậy để cải thiện hiệu năng, tính sẵn sàng kỳ vọng, hoặc
mức sử dụng tổng thể.

Đọc [Ràng buộc phân bố Pod theo topology](140-topology-spread-constraints-vi.md)
để tìm hiểu thêm về cách chúng hoạt động.

## Label topology của Pod (Pod topology labels)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [beta]`

Pod thừa hưởng các label topology (`topology.kubernetes.io/zone` và `topology.kubernetes.io/region`) từ Node mà chúng được gán, nếu các label đó tồn tại. Các label này sau đó có thể được sử dụng thông qua Downward API để cung cấp cho workload khả năng nhận biết topology của node.

Dưới đây là ví dụ một Pod dùng downward API để lấy zone và region của nó:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-topology-labels
spec:
  containers:
    - name: app
      image: alpine
      command: ["sh", "-c", "env"]
      env:
        - name: MY_ZONE
          valueFrom:
            fieldRef:
              fieldPath: metadata.labels['topology.kubernetes.io/zone']
        - name: MY_REGION
          valueFrom:
            fieldRef:
              fieldPath: metadata.labels['topology.kubernetes.io/region']
```

## Toán tử (Operators) {#operators}

Dưới đây là tất cả các toán tử logic mà bạn có thể dùng trong trường `operator` cho `nodeAffinity` và `podAffinity` đã đề cập ở trên.

|    Toán tử    |    Hành vi     |
| :------------: | :-------------: |
| `In` | Giá trị của label có mặt trong tập chuỗi được cung cấp |
|   `NotIn`   | Giá trị của label không nằm trong tập chuỗi được cung cấp |
| `Exists` | Tồn tại một label với khóa này trên đối tượng |
| `DoesNotExist` | Không tồn tại label nào với khóa này trên đối tượng |

Các toán tử sau chỉ có thể dùng với `nodeAffinity`.

|    Toán tử    |    Hành vi    |
| :------------: | :-------------: |
| `Gt` | Giá trị của trường sẽ được phân tích thành số nguyên, và số nguyên thu được khi phân tích giá trị của label có tên do selector này chỉ định phải lớn hơn số nguyên đó |
| `Lt` | Giá trị của trường sẽ được phân tích thành số nguyên, và số nguyên thu được khi phân tích giá trị của label có tên do selector này chỉ định phải nhỏ hơn số nguyên đó |

> **Ghi chú:**
> `Gt` và `Lt` sẽ không hoạt động với các giá trị không phải số nguyên. Nếu giá trị đưa vào
> không phân tích được thành số nguyên, Pod sẽ không được lập lịch. Ngoài ra, `Gt` và `Lt`
> không khả dụng cho `podAffinity`.

## Tiếp theo (What's next)

- Đọc thêm về [taint và toleration](139-taint-and-toleration-vi.md).
- Đọc tài liệu thiết kế cho [node affinity](https://git.k8s.io/design-proposals-archive/scheduling/nodeaffinity.md)
  và cho [inter-pod affinity/anti-affinity](https://git.k8s.io/design-proposals-archive/scheduling/podaffinity.md).
- Tìm hiểu cách [topology manager](259-topology-manager-vi.md) tham gia vào
  các quyết định phân bổ tài nguyên ở cấp node.
- Tìm hiểu cách dùng [nodeSelector](267-assign-pods-nodes-vi.md).
- Tìm hiểu cách dùng [affinity và anti-affinity](266-assign-pods-nodes-node-affinity-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 7:

1. `nodeSelector` và node affinity `requiredDuringSchedulingIgnoredDuringExecution` khác nhau
   ở điểm nào? Nếu một Pod đặt **cả hai**, khi nào Pod mới được lập lịch?
2. Một Pod đang chạy nhờ node affinity `required` khớp label `disktype=ssd` của node. Bạn gỡ
   label đó khỏi node. Pod có bị đuổi đi không?
3. Trong một `nodeAffinity`, hai phần tử của `nodeSelectorTerms` quan hệ với nhau thế nào, còn
   hai biểu thức trong cùng một `matchExpressions` thì thế nào?
4. Cluster lab có `k8s-worker1` và `k8s-worker2` nhận Pod thường. Bạn tạo một Deployment 3
   replica với `podAntiAffinity` loại `requiredDuringSchedulingIgnoredDuringExecution`,
   `topologyKey: kubernetes.io/hostname`, `labelSelector` khớp chính label của các replica đó.
   Bao nhiêu Pod chạy được, và replica còn lại ở đâu?
5. Bạn muốn một Pod chạy đúng trên `k8s-worker1`. Đặt `nodeName: k8s-worker1` và đặt
   `nodeSelector` khớp label của riêng node đó khác nhau ra sao? Vì sao bài cảnh báo về cách
   thứ nhất?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Về **hiệu lực** thì giống nhau — cả hai đều là ràng buộc cứng dựa trên label node. Khác ở
   **khả năng biểu đạt**: `nodeSelector` chỉ chọn node có tất cả các label được chỉ định, còn
   node affinity có cú pháp phong phú hơn với `operator` (`In`, `NotIn`, `Exists`,
   `DoesNotExist`, `Gt`, `Lt`) và cho phép cả quy tắc mềm. Nếu đặt cả hai thì **cả hai phải
   được thỏa mãn** thì Pod mới lên node.
2. **Không.** `IgnoredDuringExecution` nghĩa là **nếu label của node thay đổi sau khi
   Kubernetes đã lập lịch Pod, Pod vẫn tiếp tục chạy**. Affinity chỉ được đánh giá tại thời
   điểm lập lịch. Muốn có hành vi đẩy Pod đang chạy ra khỏi node thì phải dùng cơ chế khác —
   taint effect `NoExecute` ở bài [139](139-taint-and-toleration-vi.md).
3. Các **term** trong `nodeSelectorTerms` được **OR**: thỏa một term là đủ. Các **biểu thức**
   trong cùng một `matchExpressions` được **AND**: phải thỏa tất cả. Đây đúng là chỗ dễ nhầm
   ngược, vì cả hai cùng nằm trong một khối YAML lồng nhau.
4. **Hai Pod chạy, Pod thứ ba `Pending`.** Anti-affinity `required` với
   `topologyKey: kubernetes.io/hostname` yêu cầu bộ lập lịch **tránh đặt nhiều bản sao khớp
   selector lên cùng một node** — đúng mẫu Deployment `redis-cache` trong bài, "tạo mỗi cache
   trên một node riêng biệt". Cluster lab chỉ có hai node nhận Pod thường, nên chỉ có hai miền
   topology `hostname` dùng được. Vì đây là loại **cứng**, Pod thứ ba không được lập lịch chứ
   không dồn lên node đã có Pod. Nếu đổi sang `preferred`, nó sẽ vẫn được lập lịch.
5. `nodeSelector` **vẫn đi qua bộ lập lịch**: scheduler lọc, chấm điểm rồi bind. `nodeName`
   **bỏ qua bộ lập lịch** — scheduler bỏ qua Pod, kubelet trên node có tên đó cố đặt Pod lên
   node. Hệ quả: nếu node không tồn tại, Pod không chạy và trong một số trường hợp bị tự động
   xóa; nếu node không đủ tài nguyên, Pod **thất bại** với lý do `OutOfmemory` hoặc `OutOfcpu`
   thay vì được sắp đặt lại chỗ khác. Bài nói thẳng: `nodeName` dành cho bộ lập lịch tùy biến,
   còn để gán Pod vào một Node cụ thể thì dùng node affinity hoặc `nodeSelector`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
