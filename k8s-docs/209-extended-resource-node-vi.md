# Quảng bá Extended Resource cho một Node (Advertise Extended Resources for a Node)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/extended-resource-node/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 25 — Quản trị tài nguyên theo namespace](00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace),
bài 7/7 · Kiểm chứng bằng chính các bước trong bài trên cluster lab: bài là task tự chứa, có mục
dọn dẹp — quảng bá dongle trên một worker rồi gỡ đi, cluster trở về đúng trạng thái cũ.

Bài này nối tiếp bài [110 — Quản lý tài nguyên cho Pod và Container](110-manage-resources-containers-vi.md):
ngoài `cpu` và `memory`, quản trị viên có thể tự khai thêm loại tài nguyên đếm được ở mức node.

**Phải hiểu ở lần đọc này:**

- Extended resource là gì và tính "mờ đục" (opaque) của nó: Kubernetes không biết dongle là gì và
  không kiểm chứng node thật sự có nó; Kubernetes chỉ biết Node có N đơn vị tài nguyên mang tên
  đó, và N **phải là số nguyên** (không quảng bá được 4.5 dongle).
- Cách quảng bá: gửi HTTP PATCH (định dạng JSON-Patch) vào `/api/v1/nodes/<tên-node>/status`,
  thêm entry vào `/status/capacity/...`; dùng `kubectl proxy` để gửi request tới API server qua
  `localhost:8001`.
- Vì sao path viết là `example.com~1dongle`: `~1` là cách mã hóa ký tự `/` trong path của
  JSON-Patch (path được diễn giải theo JSON-Pointer, RFC 6901).
- Extended resource hành xử như memory và CPU trong lập lịch: node quảng bá capacity, người phát
  triển tạo Pod request một số lượng nhất định — nhưng đơn vị do bạn tự định nghĩa, và ví dụ
  storage cho thấy việc chọn kích thước "chunk" (8 chunk 100Gi hay 800Gi chunk 1 byte) quyết
  định độ mịn mà Pod có thể request.
- Dọn dẹp bằng op `remove` trên đúng path đó, verify bằng `kubectl describe node | grep dongle`
  không còn output.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Link "Assign Extended Resources to a Container" (phía người phát triển) | trang thuộc nhánh configure-pod-container, viết cho dev tạo Pod request tài nguyên này | khi cần viết Pod dùng extended resource, đọc trang gốc ở mục Tiếp theo |
| Link "Extended Resource allocation by DRA" | cơ chế cấp phát qua DRA thuộc giai đoạn 13 | bài [149](149-dynamic-resource-allocation-vi.md#extended-resource) |

---

Trang này hướng dẫn cách chỉ định extended resource cho một Node. Extended resource cho phép
quản trị viên cluster quảng bá (advertise) các tài nguyên mức node mà nếu không có cơ chế này,
Kubernetes sẽ không hề biết đến.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, nhập `kubectl version`.

## Lấy tên các Node của bạn (Get the names of your Nodes)

```shell
kubectl get nodes
```

Chọn một trong các Node của bạn để dùng cho bài thực hành này.

## Quảng bá một extended resource mới trên một Node (Advertise a new extended resource on one of your Nodes)

Để quảng bá một extended resource mới trên một Node, hãy gửi một HTTP PATCH request tới
Kubernetes API server. Ví dụ, giả sử một trong các Node của bạn có bốn dongle được gắn vào.
Dưới đây là ví dụ về một PATCH request quảng bá bốn tài nguyên dongle cho Node của bạn.

```
PATCH /api/v1/nodes/<your-node-name>/status HTTP/1.1
Accept: application/json
Content-Type: application/json-patch+json
Host: lab-k8s-master:8080

[
  {
    "op": "add",
    "path": "/status/capacity/example.com~1dongle",
    "value": "4"
  }
]
```

Lưu ý rằng Kubernetes không cần biết dongle là gì hay dongle dùng để làm gì. PATCH request phía
trên chỉ cho Kubernetes biết rằng Node của bạn có bốn thứ mà bạn gọi là dongle.

Khởi động một proxy để bạn có thể dễ dàng gửi request tới Kubernetes API server:

```shell
kubectl proxy
```

Trong một cửa sổ dòng lệnh khác, gửi HTTP PATCH request. Thay `<your-node-name>` bằng tên Node
của bạn:

```shell
curl --header "Content-Type: application/json-patch+json" \
  --request PATCH \
  --data '[{"op": "add", "path": "/status/capacity/example.com~1dongle", "value": "4"}]' \
  http://localhost:8001/api/v1/nodes/<your-node-name>/status
```

> **Ghi chú:** Trong request phía trên, `~1` là cách mã hóa của ký tự / trong path của patch.
> Giá trị operation path trong JSON-Patch được diễn giải như một JSON-Pointer. Để biết chi tiết,
> xem [IETF RFC 6901](https://tools.ietf.org/html/rfc6901), mục 3.

Kết quả cho thấy Node có capacity là 4 dongle:

```
"capacity": {
  "cpu": "2",
  "memory": "2049008Ki",
  "example.com/dongle": "4",
```

Mô tả (describe) Node của bạn:

```
kubectl describe node <your-node-name>
```

Một lần nữa, kết quả hiển thị tài nguyên dongle:

```yaml
Capacity:
  cpu: 2
  memory: 2049008Ki
  example.com/dongle: 4
```

Giờ đây, người phát triển ứng dụng có thể tạo các Pod request một số lượng dongle nhất định. Xem
[Gán Extended Resource cho một Container](284-extended-resource-vi.md).

## Thảo luận (Discussion)

Extended resource tương tự như tài nguyên memory và CPU. Ví dụ, giống như một Node có một lượng
memory và CPU nhất định để chia sẻ cho tất cả các thành phần chạy trên Node đó, nó cũng có thể
có một số lượng dongle nhất định để chia sẻ cho tất cả các thành phần chạy trên Node. Và giống
như người phát triển ứng dụng có thể tạo các Pod request một lượng memory và CPU nhất định, họ
cũng có thể tạo các Pod request một số lượng dongle nhất định.

Extended resource là "mờ đục" (opaque) đối với Kubernetes; Kubernetes không biết gì về bản chất
của chúng. Kubernetes chỉ biết rằng một Node có một số lượng nhất định các tài nguyên đó.
Extended resource phải được quảng bá theo số lượng nguyên. Ví dụ, một Node có thể quảng bá bốn
dongle, nhưng không thể quảng bá 4.5 dongle.

### Ví dụ về lưu trữ (Storage example)

Giả sử một Node có 800 GiB của một loại lưu trữ đĩa đặc biệt. Bạn có thể tạo một cái tên cho
loại lưu trữ đặc biệt đó, chẳng hạn example.com/special-storage. Sau đó bạn có thể quảng bá nó
theo từng phần (chunk) có kích thước nhất định, chẳng hạn 100 GiB. Trong trường hợp đó, Node của
bạn sẽ quảng bá rằng nó có tám tài nguyên loại example.com/special-storage.

```yaml
Capacity:
 ...
 example.com/special-storage: 8
```

Nếu bạn muốn cho phép request lưu trữ đặc biệt với số lượng tùy ý, bạn có thể quảng bá lưu trữ
đặc biệt theo các chunk kích thước 1 byte. Trong trường hợp đó, bạn sẽ quảng bá 800Gi tài nguyên
loại example.com/special-storage.

```yaml
Capacity:
 ...
 example.com/special-storage:  800Gi
```

Khi đó một Container có thể request số byte lưu trữ đặc biệt bất kỳ, tối đa tới 800Gi.

## Dọn dẹp (Clean up)

Đây là một PATCH request gỡ bỏ phần quảng bá dongle khỏi một Node.

```
PATCH /api/v1/nodes/<your-node-name>/status HTTP/1.1
Accept: application/json
Content-Type: application/json-patch+json
Host: lab-k8s-master:8080

[
  {
    "op": "remove",
    "path": "/status/capacity/example.com~1dongle",
  }
]
```

Khởi động một proxy để bạn có thể dễ dàng gửi request tới Kubernetes API server:

```shell
kubectl proxy
```

Trong một cửa sổ dòng lệnh khác, gửi HTTP PATCH request. Thay `<your-node-name>` bằng tên Node
của bạn:

```shell
curl --header "Content-Type: application/json-patch+json" \
  --request PATCH \
  --data '[{"op": "remove", "path": "/status/capacity/example.com~1dongle"}]' \
  http://localhost:8001/api/v1/nodes/<your-node-name>/status
```

Xác minh rằng phần quảng bá dongle đã bị gỡ bỏ:

```
kubectl describe node <your-node-name> | grep dongle
```

(bạn sẽ không thấy output nào)

## Tiếp theo (What's next)

### Dành cho người phát triển ứng dụng (For application developers)

- [Gán Extended Resource cho một Container](284-extended-resource-vi.md)
- [Cấp phát Extended Resource bằng DRA](149-dynamic-resource-allocation-vi.md#extended-resource)

### Dành cho quản trị viên cluster (For cluster administrators)

- [Cấu hình ràng buộc Memory tối thiểu và tối đa cho một Namespace](231-memory-constraint-namespace-vi.md)
- [Cấu hình ràng buộc CPU tối thiểu và tối đa cho một Namespace](229-cpu-constraint-namespace-vi.md)
- [Cấp phát Extended Resource bằng DRA](149-dynamic-resource-allocation-vi.md#extended-resource)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 25:

1. **Câu bẫy.** Để quảng bá `example.com/dongle: 4` trên một Node, bạn có cần cài driver hay bất
   kỳ phần mềm nào trên node để Kubernetes "nhìn thấy" dongle trước không? Nếu node thực tế
   không có dongle nào mà bạn vẫn PATCH capacity là 4, Kubernetes có phát hiện ra không?
2. Trong path của PATCH request, vì sao phải viết `example.com~1dongle` thay vì
   `example.com/dongle`, và `~1` được diễn giải theo chuẩn nào?
3. Trên một worker của cluster lab, bạn có 800Gi lưu trữ đặc biệt và cân nhắc hai cách quảng bá:
   `example.com/special-storage: 8` (chunk 100Gi) hoặc `example.com/special-storage: 800Gi`
   (chunk 1 byte). Với một Container muốn dùng khoảng 250Gi, hai cách này khác nhau thế nào?
4. Extended resource giống và khác tài nguyên memory/CPU ở những điểm nào theo bài, và một Node
   có thể quảng bá 4.5 dongle không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không cần cài gì, và Kubernetes cũng không phát hiện ra.** Extended resource là opaque:
   Kubernetes không cần biết dongle là gì hay dùng để làm gì; PATCH request chỉ khai báo rằng
   Node có bốn thứ mà *bạn* gọi là dongle. Trực giác thường sai ở chỗ nghĩ rằng Kubernetes xác
   minh phần cứng — thực tế nó chỉ ghi nhận con số vào `status.capacity` và dùng con số đó khi
   lập lịch; tính đúng đắn của con số là trách nhiệm của quản trị viên.
2. Vì tên tài nguyên chứa ký tự `/` (`example.com/dongle`), mà trong JSON-Patch, giá trị
   operation path được diễn giải như một **JSON-Pointer** (IETF RFC 6901, mục 3) — trong đó `/`
   là ký tự phân tách các cấp của path. Do đó `/` trong tên phải được mã hóa thành **`~1`**, cho
   ra `/status/capacity/example.com~1dongle`.
3. Với chunk 100Gi (capacity 8), Container chỉ request được theo **số nguyên chunk** — muốn
   khoảng 250Gi phải xin 3 chunk, tức 300Gi, thừa 50Gi. Với chunk 1 byte (capacity 800Gi),
   Container có thể request **số byte tùy ý** tối đa tới 800Gi, nên xin đúng 250Gi được. Kích
   thước chunk bạn chọn khi quảng bá quyết định độ mịn mà Pod có thể request.
4. **Giống:** Node có một lượng nhất định để chia sẻ cho mọi thành phần chạy trên Node, và người
   phát triển tạo Pod request một số lượng nhất định, y như request memory/CPU. **Khác:**
   Kubernetes hiểu bản chất của memory/CPU, còn extended resource thì opaque — Kubernetes chỉ
   biết số lượng; và extended resource **phải quảng bá theo số nguyên**, nên không thể quảng bá
   4.5 dongle (trong khi memory/CPU có các đơn vị chia nhỏ như milliCPU).

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng. Đây là bài cuối của
[giai đoạn 25](00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace) — hoàn thành loạt bài thực
hành LimitRange và ResourceQuota của checkpoint này trên cluster lab trước khi sang giai đoạn 26.
