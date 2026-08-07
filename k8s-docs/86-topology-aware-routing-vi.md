# Định tuyến nhận biết topology (Topology Aware Routing)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/>
>
> *Định tuyến nhận biết topology* (Topology Aware Routing) cung cấp một cơ chế giúp giữ lưu lượng mạng bên trong zone nơi nó xuất phát. Việc ưu tiên lưu lượng cùng zone giữa các Pod trong cluster của bạn có thể có lợi cho độ tin cậy, hiệu năng (độ trễ mạng và thông lượng), hoặc chi phí.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 5](LO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng), bài 7/16 · Kiểm chứng ở
Lab 5a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài này viết cho cluster **đa zone**. Cluster lab chỉ có ba VM trong một mạng phẳng và không
node nào mang label `topology.kubernetes.io/zone`, nên bạn **không bật được** tính năng này ở
đây. Giá trị của bài lúc này nằm ở chỗ khác: nó là ví dụ rõ nhất về việc kube-proxy **lọc** tập
endpoint mà bài [83](83-endpoint-slices-vi.md) đã dựng ra, và về cách Kubernetes tự tắt một tối
ưu khi điều kiện không đủ.

**Phải hiểu ở lần đọc này:**

- Bật bằng annotation `service.kubernetes.io/topology-mode: Auto` trên **từng Service** (trước
  1.27 là annotation `service.kubernetes.io/topology-aware-hints`).
- Cơ chế hai vế: **EndpointSlice controller** điền trường `hints.forZones` cho từng endpoint,
  phân bổ theo tỷ lệ **số CPU core có thể cấp phát** của các node trong zone; **kube-proxy** lọc
  endpoint mà nó định tuyến tới dựa trên các hint đó.
- Điều kiện để tính năng hữu ích: lưu lượng đến **phân bổ đều** giữa các zone, và Service có
  **từ 3 endpoint trở lên mỗi zone** — ít hơn thì xác suất cao controller không phân bổ đều
  được và quay về định tuyến mặc định trên toàn cluster.
- Năm **cơ chế bảo vệ** khiến hint bị bỏ qua và kube-proxy quay về dùng endpoint ở mọi nơi:
  không đủ endpoint, không đạt được phân bổ cân bằng, có node thiếu label zone hoặc thiếu giá
  trị CPU allocatable, có endpoint không mang hint, hoặc zone của chính kube-proxy không xuất
  hiện trong hint nào.
- Ràng buộc phải nhớ: **không dùng được cùng `internalTrafficPolicy: Local` trên cùng một
  Service**; controller còn bỏ qua node chưa sẵn sàng và node có label
  `node-role.kubernetes.io/control-plane` hoặc `node-role.kubernetes.io/master`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Heuristic tùy chỉnh* | mới có những bước đầu tiên, hiện thực còn hạn chế | không cần |
| Gạch đầu dòng cuối của *Các ràng buộc* — tương tác với autoscaling | cần HPA và metrics-server, chưa có | giai đoạn 11 |
| Trường `trafficDistribution` nhắc ở *Tiếp theo* | là API mới thay dần annotation, không đổi cơ chế đang học | không cần |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.23 [beta]`

> **Ghi chú:**
> Trước Kubernetes 1.27, tính năng này được gọi là *Topology Aware Hints*.

*Định tuyến nhận biết topology* (Topology Aware Routing) điều chỉnh hành vi định tuyến để ưu tiên giữ lưu lượng trong zone mà nó xuất phát. Trong một số trường hợp, điều này có thể giúp giảm chi phí hoặc cải thiện hiệu năng mạng.

## Động lực (Motivation)

Các cluster Kubernetes ngày càng được triển khai nhiều trong môi trường đa zone (multi-zone).
*Định tuyến nhận biết topology* cung cấp một cơ chế giúp giữ lưu lượng bên trong zone nơi nó xuất phát. Khi tính toán các endpoint cho một Service, EndpointSlice controller xem xét topology (region và zone) của từng endpoint và điền vào trường hints để phân bổ endpoint đó cho một zone. Các thành phần của cluster như kube-proxy sau đó có thể sử dụng các hint này, và dùng chúng để tác động đến cách lưu lượng được định tuyến (ưu tiên các endpoint gần hơn về mặt topology).

## Bật Topology Aware Routing (Enabling Topology Aware Routing)

> **Ghi chú:**
> Trước Kubernetes 1.27, hành vi này được điều khiển bằng annotation `service.kubernetes.io/topology-aware-hints`.

Bạn có thể bật Topology Aware Routing cho một Service bằng cách đặt annotation `service.kubernetes.io/topology-mode` thành `Auto`. Khi có đủ số endpoint khả dụng trong mỗi zone, các Topology Hint sẽ được điền vào EndpointSlice để phân bổ từng endpoint riêng lẻ cho các zone cụ thể, nhờ đó lưu lượng được định tuyến gần hơn với nơi nó xuất phát.

## Khi nào tính năng hoạt động tốt nhất (When it works best)

Tính năng này hoạt động tốt nhất khi:

### 1. Lưu lượng đến được phân bổ đều (Incoming traffic is evenly distributed)

Nếu một tỷ lệ lớn lưu lượng xuất phát từ một zone duy nhất, lưu lượng đó có thể làm quá tải tập con các endpoint đã được phân bổ cho zone đó. Tính năng này không được khuyến nghị khi lưu lượng đến được dự kiến sẽ xuất phát từ một zone duy nhất.

### 2. Service có từ 3 endpoint trở lên trên mỗi zone (The Service has 3 or more endpoints per zone) {#three-or-more-endpoints-per-zone}

Trong một cluster có ba zone, điều này nghĩa là cần 9 endpoint trở lên. Nếu có ít hơn 3 endpoint trên mỗi zone, xác suất cao (≈50%) là EndpointSlice controller sẽ không thể phân bổ các endpoint một cách đồng đều và thay vào đó sẽ quay về cách tiếp cận định tuyến mặc định trên toàn cluster.

## Cách hoạt động (How It Works)

Heuristic "Auto" cố gắng phân bổ một số lượng endpoint theo tỷ lệ cho mỗi zone. Lưu ý rằng heuristic này hoạt động tốt nhất với các Service có số lượng endpoint đáng kể.

### EndpointSlice controller {#implementation-control-plane}

EndpointSlice controller chịu trách nhiệm đặt các hint trên EndpointSlice khi heuristic này được bật. Controller phân bổ một lượng endpoint theo tỷ lệ cho mỗi zone. Tỷ lệ này dựa trên số CPU core [có thể cấp phát](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#node-allocatable) (allocatable) của các node chạy trong zone đó. Ví dụ, nếu một zone có 2 CPU core và một zone khác chỉ có 1 CPU core, controller sẽ phân bổ số endpoint cho zone có 2 CPU core nhiều gấp đôi.

Ví dụ sau đây cho thấy một EndpointSlice trông như thế nào khi các hint đã được điền:

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: example-hints
  labels:
    kubernetes.io/service-name: example-svc
addressType: IPv4
ports:
  - name: http
    protocol: TCP
    port: 80
endpoints:
  - addresses:
      - "10.1.2.3"
    conditions:
      ready: true
    hostname: pod-1
    zone: zone-a
    hints:
      forZones:
        - name: "zone-a"
```

### kube-proxy {#implementation-kube-proxy}

Thành phần kube-proxy lọc các endpoint mà nó định tuyến tới dựa trên các hint do EndpointSlice controller đặt. Trong hầu hết các trường hợp, điều này nghĩa là kube-proxy có thể định tuyến lưu lượng tới các endpoint trong cùng zone. Đôi khi controller phân bổ endpoint từ một zone khác để đảm bảo phân bổ endpoint đồng đều hơn giữa các zone. Điều này dẫn đến việc một phần lưu lượng được định tuyến sang các zone khác.

## Các cơ chế bảo vệ (Safeguards)

Control plane của Kubernetes và kube-proxy trên mỗi node áp dụng một số quy tắc bảo vệ trước khi sử dụng Topology Aware Hints. Nếu các quy tắc này không được thỏa mãn, kube-proxy sẽ chọn endpoint từ bất kỳ đâu trong cluster của bạn, bất kể zone.

1. **Số lượng endpoint không đủ:** Nếu số endpoint ít hơn số zone trong cluster, controller sẽ không gán bất kỳ hint nào.

2. **Không thể đạt được phân bổ cân bằng:** Trong một số trường hợp, không thể đạt được sự phân bổ endpoint cân bằng giữa các zone. Ví dụ, nếu zone-a lớn gấp đôi zone-b, nhưng chỉ có 2 endpoint, endpoint được phân bổ cho zone-a có thể nhận lưu lượng nhiều gấp đôi zone-b. Controller không gán hint nếu nó không thể đưa giá trị "quá tải kỳ vọng" (expected overload) này xuống dưới ngưỡng chấp nhận được cho mỗi zone. Điều quan trọng là cơ chế này không dựa trên phản hồi thời gian thực. Từng endpoint riêng lẻ vẫn có khả năng bị quá tải.

3. **Một hoặc nhiều Node thiếu thông tin:** Nếu bất kỳ node nào không có label `topology.kubernetes.io/zone` hoặc không báo cáo giá trị CPU có thể cấp phát, control plane sẽ không đặt bất kỳ hint endpoint nhận biết topology nào, và do đó kube-proxy không lọc endpoint theo zone.

4. **Một hoặc nhiều endpoint không có zone hint:** Khi điều này xảy ra, kube-proxy giả định rằng đang có một quá trình chuyển đổi sang hoặc rời khỏi Topology Aware Hints. Việc lọc endpoint cho một Service ở trạng thái này sẽ nguy hiểm, nên kube-proxy quay về sử dụng tất cả các endpoint.

5. **Một zone không được đại diện trong các hint:** Nếu kube-proxy không thể tìm thấy ít nhất một endpoint có hint nhắm tới zone mà nó đang chạy, nó sẽ quay về sử dụng endpoint từ tất cả các zone. Điều này nhiều khả năng xảy ra nhất khi bạn thêm một zone mới vào cluster hiện có.

## Các ràng buộc (Constraints)

* Topology Aware Hints không được sử dụng khi `internalTrafficPolicy` được đặt là `Local` trên một Service. Có thể sử dụng cả hai tính năng trong cùng một cluster trên các Service khác nhau, chỉ là không thể trên cùng một Service.

* Cách tiếp cận này sẽ không hoạt động tốt với các Service có tỷ lệ lớn lưu lượng xuất phát từ một tập con các zone. Thay vào đó, nó giả định rằng lưu lượng đến sẽ xấp xỉ tỷ lệ thuận với dung lượng của các Node trong mỗi zone.

* EndpointSlice controller bỏ qua các node chưa sẵn sàng (unready) khi tính toán tỷ lệ của mỗi zone. Điều này có thể gây ra những hệ quả ngoài ý muốn nếu một phần lớn các node đang ở trạng thái chưa sẵn sàng.

* EndpointSlice controller bỏ qua các node có label `node-role.kubernetes.io/control-plane` hoặc `node-role.kubernetes.io/master`. Điều này có thể gây vấn đề nếu workload cũng đang chạy trên các node đó.

* EndpointSlice controller không xét đến các toleration khi triển khai hoặc tính toán tỷ lệ của mỗi zone. Nếu các Pod đứng sau một Service bị giới hạn trong một tập con các Node của cluster, điều này sẽ không được tính đến.

* Tính năng này có thể không hoạt động tốt với autoscaling. Ví dụ, nếu nhiều lưu lượng xuất phát từ một zone duy nhất, chỉ các endpoint được phân bổ cho zone đó mới xử lý lưu lượng ấy. Điều này có thể khiến Horizontal Pod Autoscaler hoặc là không nhận biết được sự kiện này, hoặc là các pod mới thêm vào lại khởi động ở một zone khác.

## Heuristic tùy chỉnh (Custom heuristics)

Kubernetes được triển khai theo nhiều cách khác nhau, không có một heuristic phân bổ endpoint cho các zone nào phù hợp với mọi trường hợp sử dụng. Một mục tiêu then chốt của tính năng này là cho phép phát triển các heuristic tùy chỉnh nếu heuristic tích hợp sẵn không phù hợp với trường hợp sử dụng của bạn. Những bước đầu tiên để hỗ trợ heuristic tùy chỉnh đã được đưa vào bản phát hành 1.27. Đây là một hiện thực còn hạn chế, có thể chưa bao quát một số tình huống liên quan và khả dĩ.

## Tiếp theo (What's next)

* Làm theo hướng dẫn [Kết nối ứng dụng với Service (Connecting Applications with Services)](https://kubernetes.io/docs/tutorials/services/connect-applications-service/)
* Tìm hiểu về trường [trafficDistribution](https://kubernetes.io/docs/concepts/services-networking/service/#traffic-distribution), trường này liên quan chặt chẽ tới annotation `service.kubernetes.io/topology-mode` và cung cấp các tùy chọn linh hoạt cho việc định tuyến lưu lượng trong Kubernetes.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 5:

1. Ba node của cluster lab không node nào có label `topology.kubernetes.io/zone`. Bạn vẫn đặt
   annotation `service.kubernetes.io/topology-mode: Auto` lên một Service. Điều gì xảy ra?
2. Một Service đã đặt `internalTrafficPolicy: Local`. Bật thêm topology-mode `Auto` cho chính
   Service đó thì hai hiệu ứng có cộng dồn không?
3. Thành phần nào **đặt** hint, thành phần nào **dùng** hint, và hint được lưu ở đâu?
4. Vì sao bài khuyến nghị có từ 3 endpoint trở lên trên mỗi zone?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không có hint nào được đặt, và kube-proxy không lọc endpoint theo zone.** Đây là cơ chế
   bảo vệ số 3: nếu bất kỳ node nào không có label `topology.kubernetes.io/zone` hoặc không báo
   cáo giá trị CPU có thể cấp phát, control plane sẽ không đặt bất kỳ hint nào. Annotation vẫn
   nằm đó nhưng hành vi định tuyến không đổi.
2. **Không.** Bài nêu ràng buộc rõ: Topology Aware Hints **không được sử dụng khi
   `internalTrafficPolicy` được đặt là `Local`** trên một Service. Hai tính năng dùng chung
   được **trong cùng một cluster trên những Service khác nhau**, nhưng không trên cùng một
   Service.
3. **EndpointSlice controller đặt hint**; nó ghi vào trường `hints.forZones` của từng endpoint
   **trong đối tượng EndpointSlice**. **kube-proxy dùng hint**: nó lọc các endpoint mà nó định
   tuyến tới dựa trên hint đó, nên phần lớn trường hợp traffic ở lại trong cùng zone. Đôi khi
   controller cố ý gán endpoint sang zone khác để cân bằng, và khi đó một phần traffic vẫn đi
   liên zone.
4. Vì heuristic phân bổ theo **tỷ lệ**, và tỷ lệ chỉ có nghĩa khi có đủ endpoint để chia. Với
   ít hơn 3 endpoint mỗi zone, **xác suất cao (≈50%) EndpointSlice controller không phân bổ đều
   được** và sẽ quay về cách tiếp cận định tuyến mặc định trên toàn cluster. Trong một cluster
   ba zone, điều đó nghĩa là cần từ 9 endpoint trở lên.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
