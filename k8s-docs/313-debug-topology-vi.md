# Khắc phục sự cố Topology Management (Troubleshooting Topology Management)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/debug-cluster/topology/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 14 — Khả năng mở rộng](00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng),
bài 1/2 của dòng **Thực hành** · Kiểm chứng ở
[Lab 14 — CRD và Operator](labs/LAB-14-CRD-VA-OPERATOR.md) phần B10.4, nơi lab đọc
`topologyManagerPolicy` và `memoryManagerPolicy` **hiệu lực** của cả ba kubelet qua `configz`,
cộng số NUMA node thật của máy, để chứng minh nhánh debug của bài không thể kích hoạt trên
cluster này.

Bài là bài "khi hỏng thì tra" của hai bài [235 — Memory Manager](235-memory-manager-vi.md) và
[259 — Topology Manager](259-topology-manager-vi.md) ở
[giai đoạn 25](00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace). Nền tảng
lý thuyết nằm ở bài [74 — Các trình quản lý tài nguyên](74-resource-managers-vi.md).

Cluster lab chạy trên VM thường chỉ có một NUMA node và kubelet để policy mặc định, nên bạn
sẽ không tự nhiên gặp lỗi trong bài; hãy đọc để biết **tra ở đâu** khi vận hành máy chủ vật
lý nhiều socket.

**Phải hiểu ở lần đọc này:**

- Bốn nguồn thông tin khi debug topology: trạng thái Pod (Pod status), log hệ thống, file
  trạng thái của kubelet (`/var/lib/kubelet/memory_manager_state`), và device plugin
  resource API.
- Lỗi `TopologyAffinityError` xuất hiện trong status của Pod khi node không đủ tài nguyên
  thỏa mãn request, hoặc request bị chính sách của Topology Manager từ chối; xem chi tiết
  bằng `kubectl describe pod` hoặc `kubectl events`.
- Luồng quyết định: CPU Manager và Memory Manager sinh các hint, Topology Manager gộp chúng
  thành một best hint duy nhất rồi đối chiếu với policy hiện hành để chấp nhận (admit) hoặc
  từ chối Pod — tất cả đều để lại dấu vết trong log.
- Cách đọc file trạng thái Memory Manager: `numaAffinity` cho biết Pod bị ghim (pin) vào
  những NUMA node nào thông qua cấu hình `cgroups`; muốn biết bộ nhớ trống của một group thì
  cộng giá trị `free` của từng NUMA node thuộc group; `systemReserved` phản ánh flag
  `--reserved-memory` mà admin đã đặt.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Kiểm tra device plugin resource API (`PodResourceLister`, message `ContainerMemory`) | cần viết client gRPC, vượt khỏi thao tác admin thường ngày | bài [184 — Device plugins](184-device-plugins-vi.md), tra cứu khi thật sự cần |
| Ý nghĩa từng field còn lại trong JSON state (`checksum`, `entries`, `machineState`) | chỉ cần khi vận hành Memory Manager policy `Static` trên máy thật | bài [235 — Memory Manager](235-memory-manager-vi.md) khi bật policy này |

---

Kubernetes giữ nhiều khía cạnh về cách pod thực thi trên node ở dạng trừu tượng hóa đối với
người dùng. Đây là chủ đích thiết kế. Tuy nhiên, một số workload đòi hỏi những đảm bảo mạnh
hơn về độ trễ (latency) và/hoặc hiệu năng để hoạt động ở mức chấp nhận được. `kubelet` cung
cấp các phương thức cho phép áp dụng những chính sách sắp đặt workload phức tạp hơn, trong
khi vẫn giữ cho lớp trừu tượng không chứa các chỉ thị sắp đặt tường minh.

Bạn có thể quản lý topology bên trong các node. Điều này nghĩa là giúp kubelet cấu hình hệ
điều hành của máy chủ sao cho Pod và container được đặt vào đúng phía của các ranh giới bên
trong, chẳng hạn các _NUMA domain_. (NUMA là viết tắt của _non-uniform memory access_ — truy
cập bộ nhớ không đồng nhất — chỉ ý tưởng rằng các CPU có thể gần một số vùng bộ nhớ cụ thể
hơn về mặt topology, do cách bố trí vật lý của các linh kiện phần cứng và cách chúng được
kết nối với nhau).

## Các nguồn thông tin để khắc phục sự cố (Sources of troubleshooting information)

Trong ngữ cảnh quản lý topology, bạn có thể dùng các phương tiện sau để tìm hiểu lý do một
pod không triển khai được hoặc bị từ chối tại một node:

- _Trạng thái Pod (Pod status)_ — cho biết các lỗi topology affinity
- _Log hệ thống (system logs)_ — chứa thông tin quý giá cho việc debug; ví dụ, về các hint
  đã được sinh ra
- _File trạng thái của kubelet (kubelet state file)_ — bản dump trạng thái nội bộ của Memory
  Manager (bao gồm _node map_ và các _memory map_)
- Bạn có thể dùng [device plugin resource API](#device-plugin-resource-api) để truy xuất
  thông tin về bộ nhớ đã được dành riêng (reserve) cho các container

## Khắc phục lỗi `TopologyAffinityError` {#TopologyAffinityError}

Lỗi này thường xảy ra trong các tình huống sau:

* node không còn đủ tài nguyên khả dụng để thỏa mãn request của pod
* request của pod bị từ chối do các ràng buộc cụ thể của chính sách Topology Manager

Lỗi xuất hiện trong status của pod:

```shell
kubectl get pods
```

```none
NAME         READY   STATUS                  RESTARTS   AGE
guaranteed   0/1     TopologyAffinityError   0          113s
```

Dùng `kubectl describe pod <id>` hoặc `kubectl events` để lấy thông báo lỗi chi tiết:

```none
Warning  TopologyAffinityError  10m   kubelet, dell8  Resources cannot be allocated with Topology locality
```

## Kiểm tra log hệ thống (Examine system logs)

Tìm trong log hệ thống các dòng liên quan đến pod cụ thể.

Tập hợp các hint do CPU Manager sinh ra sẽ có mặt trong log. Ngoài ra, tập hợp các hint mà
Memory Manager sinh ra cho pod cũng có thể tìm thấy trong log.

Topology Manager gộp (merge) các hint này để tính ra một best hint duy nhất. Best hint đó
cũng sẽ có mặt trong log.

Best hint chỉ ra nơi cần cấp phát toàn bộ tài nguyên. Topology Manager đối chiếu hint này
với policy hiện hành của nó, và dựa trên kết quả phán quyết, nó hoặc chấp nhận (admit) pod
vào node, hoặc từ chối pod.

Ngoài ra, hãy tìm trong log các mục liên quan đến Memory Manager; ví dụ để biết thông tin về
các cập nhật `cgroups` và `cpuset.mems`.

## Các ví dụ (Examples)

### Kiểm tra trạng thái memory manager trên một node (Examine the memory manager state on a node)

Trước tiên, hãy triển khai một pod `Guaranteed` mẫu có spec như sau:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed
spec:
  containers:
  - name: guaranteed
    image: consumer
    imagePullPolicy: Never
    resources:
      limits:
        cpu: "2"
        memory: 150Gi
      requests:
        cpu: "2"
        memory: 150Gi
    command: ["sleep","infinity"]
```

Tiếp theo, đăng nhập vào node nơi pod được triển khai và kiểm tra file trạng thái tại
`/var/lib/kubelet/memory_manager_state`:

```json
{
   "policyName":"Static",
   "machineState":{
      "0":{
         "numberOfAssignments":1,
         "memoryMap":{
            "hugepages-1Gi":{
               "total":0,
               "systemReserved":0,
               "allocatable":0,
               "reserved":0,
               "free":0
            },
            "memory":{
               "total":134987354112,
               "systemReserved":3221225472,
               "allocatable":131766128640,
               "reserved":131766128640,
               "free":0
            }
         },
         "nodes":[
            0,
            1
         ]
      },
      "1":{
         "numberOfAssignments":1,
         "memoryMap":{
            "hugepages-1Gi":{
               "total":0,
               "systemReserved":0,
               "allocatable":0,
               "reserved":0,
               "free":0
            },
            "memory":{
               "total":135286722560,
               "systemReserved":2252341248,
               "allocatable":133034381312,
               "reserved":29295144960,
               "free":103739236352
            }
         },
         "nodes":[
            0,
            1
         ]
      }
   },
   "entries":{
      "fa9bdd38-6df9-4cf9-aa67-8c4814da37a8":{
         "guaranteed":[
            {
               "numaAffinity":[
                  0,
                  1
               ],
               "type":"memory",
               "size":161061273600
            }
         ]
      }
   },
   "checksum":4142013182
}
```

Từ file trạng thái, có thể suy ra rằng pod đã bị ghim (pin) vào cả hai NUMA node, tức là:

```json
"numaAffinity":[
   0,
   1
],
```

Thuật ngữ "ghim" (pinned) nghĩa là mức tiêu thụ bộ nhớ của pod bị ràng buộc (thông qua cấu
hình `cgroups`) vào các NUMA node này.

Điều đó tự động hàm ý rằng Memory Manager đã khởi tạo một group mới bao gồm hai NUMA node
này, tức là các NUMA node có chỉ số `0` và `1`.

Để phân tích lượng tài nguyên bộ nhớ khả dụng trong một group, phải cộng các mục tương ứng
của những NUMA node thuộc group đó.

Ví dụ, tổng lượng bộ nhớ "thông thường" (conventional) còn trống trong group có thể tính
bằng cách cộng lượng bộ nhớ trống tại từng NUMA node trong group, tức là trong phần
`"memory"` của NUMA node `0` (`"free":0`) và NUMA node `1` (`"free":103739236352`). Vậy,
tổng lượng bộ nhớ "thông thường" còn trống trong group này bằng `0 + 103739236352` byte.

Dòng `"systemReserved":3221225472` cho biết admin của node này đã dành riêng `3221225472`
byte (tức `3Gi`) để phục vụ kubelet và các process hệ thống tại NUMA node `0`, thông qua
flag `--reserved-memory`.

## Kiểm tra device plugin resource API (Check the device plugin resource API) {#device-plugin-resource-api}

Kubelet cung cấp một dịch vụ gRPC `PodResourceLister` để cho phép khám phá các tài nguyên và
metadata đi kèm. Bằng cách dùng
[List gRPC endpoint](184-device-plugins-vi.md#grpc-endpoint-list)
của nó, có thể truy xuất thông tin về bộ nhớ đã dành riêng cho từng container — thông tin
này nằm trong message protobuf `ContainerMemory`.

Thông tin này chỉ có thể truy xuất được cho các pod thuộc QoS class Guaranteed.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho một lần đọc tra cứu:

1. Một pod hiện `STATUS` là `TopologyAffinityError`. Hai nguyên nhân điển hình là gì, và bạn
   dùng lệnh nào để xem thông báo lỗi chi tiết?
2. Kể tên các thành phần trong luồng quyết định: ai sinh hint, ai gộp hint thành best hint,
   và quyết định admit/reject pod dựa trên điều gì?
3. **Câu bẫy.** File trạng thái ghi `"numaAffinity":[0, 1]` cho một pod. Có phải điều đó chỉ
   là "gợi ý ưu tiên" và kubelet vẫn có thể cấp bộ nhớ ở nơi khác khi thiếu? Và muốn biết
   group hai NUMA node này còn trống bao nhiêu bộ nhớ "thông thường", bạn tính thế nào từ
   file trạng thái trong bài?
4. Bạn nghi ngờ Memory Manager trên một node (giả sử `lab-k8s-worker2` của cluster lab, nếu nó
   là máy vật lý nhiều NUMA node) đã ghim một pod Guaranteed vào NUMA node nào đó — bạn mở
   file nào trên node, và dòng `"systemReserved":3221225472` trong đó nói lên điều gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Hai nguyên nhân: **node không đủ tài nguyên khả dụng** để thỏa mãn request của pod, hoặc
   **request bị từ chối do ràng buộc của chính sách Topology Manager**. Xem chi tiết bằng
   **`kubectl describe pod <id>`** hoặc **`kubectl events`** — sẽ thấy warning kiểu
   `Resources cannot be allocated with Topology locality`.
2. **CPU Manager và Memory Manager sinh các hint** cho pod; **Topology Manager gộp** chúng
   để tính ra **một best hint duy nhất** (nơi nên cấp phát toàn bộ tài nguyên); rồi Topology
   Manager **đối chiếu best hint với policy hiện hành** của nó — kết quả phán quyết quyết
   định admit hay reject pod. Tất cả các hint và best hint đều tìm được trong log hệ thống.
3. **Không phải gợi ý — pod bị ghim thật sự**: mức tiêu thụ bộ nhớ của pod bị **ràng buộc
   qua cấu hình `cgroups`** vào đúng các NUMA node đó, và Memory Manager đã tạo một group
   gồm NUMA node `0` và `1`. Bộ nhớ trống của group = **cộng `free` của từng NUMA node thuộc
   group**: `0 + 103739236352` byte (node `0` có `"free":0`, node `1` có
   `"free":103739236352`). Trực giác "affinity chỉ là ưu tiên" sai vì đây là ràng buộc cứng
   ở tầng cgroups, không phải trọng số lập lịch.
4. Mở **`/var/lib/kubelet/memory_manager_state`** trên node, tìm mục `entries` với
   `numaAffinity` của pod. Dòng `"systemReserved":3221225472` nghĩa là admin đã **dành riêng
   3221225472 byte (3Gi) cho kubelet và các process hệ thống tại NUMA node đó**, thông qua
   flag **`--reserved-memory`** của kubelet.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng; khi cần vận hành thật, đọc kèm
bài [235 — Memory Manager](235-memory-manager-vi.md) và
[259 — Topology Manager](259-topology-manager-vi.md).
