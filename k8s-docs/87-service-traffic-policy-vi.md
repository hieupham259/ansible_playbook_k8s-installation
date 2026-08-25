# Chính sách lưu lượng nội bộ của Service (Service Internal Traffic Policy)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/services-networking/service-traffic-policy/>
>
> Nếu hai Pod trong cluster của bạn muốn giao tiếp với nhau, và cả hai Pod thực tế đang chạy trên cùng một node, hãy dùng *Chính sách lưu lượng nội bộ của Service* (Service Internal Traffic Policy) để giữ lưu lượng mạng bên trong node đó. Việc tránh một vòng đi-về qua mạng của cluster có thể có lợi cho độ tin cậy, hiệu năng (độ trễ mạng và thông lượng), hoặc chi phí.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 5](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng), bài 8/16 · Kiểm chứng ở
[Lab 5a](labs/LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md).

Bài rất ngắn và là cặp đôi của bài trước: bài [86](86-topology-aware-routing-vi.md) giữ lưu
lượng trong **zone**, bài này giữ lưu lượng trong **node**. Khác biệt quan trọng là bài này
**kiểm chứng được ngay trên cluster lab hai worker**, còn bài kia thì không.

**Phải hiểu ở lần đọc này:**

- Đặt `.spec.internalTrafficPolicy: Local` báo kube-proxy **chỉ dùng các endpoint cục bộ trên
  node** cho lưu lượng nội bộ cluster. Giá trị mặc định là `Cluster`, khi đó Kubernetes xem xét
  **tất cả** endpoint.
- "Nội bộ" ở đây có nghĩa hẹp và chính xác: **lưu lượng xuất phát từ các Pod trong cluster hiện
  tại**.
- Hệ quả nguy hiểm phải nhớ: với các Pod trên node **không có endpoint nào** của Service, Service
  hành xử **như thể nó không có endpoint nào**, ngay cả khi Service thực sự có endpoint trên các
  node khác.
- Cơ chế không phải một proxy mới: vẫn là **kube-proxy lọc tập endpoint** theo giá trị của
  trường này.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Tiếp theo* — `externalTrafficPolicy` và việc bảo toàn IP nguồn của client | thuộc chủ đề expose ra ngoài cluster, trang đó nằm ngoài bộ bài | không cần |
| Việc dùng chung với định tuyến nhận biết topology | hai tính năng loại trừ nhau trên cùng một Service; ràng buộc đã nêu ở bài trước | bài [86](86-topology-aware-routing-vi.md) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.26 [stable]`

*Chính sách lưu lượng nội bộ của Service* (Service Internal Traffic Policy) cho phép áp dụng các giới hạn lưu lượng nội bộ để chỉ định tuyến lưu lượng nội bộ tới các endpoint nằm trong node mà lưu lượng xuất phát. Lưu lượng "nội bộ" ở đây là lưu lượng xuất phát từ các Pod trong cluster hiện tại. Điều này có thể giúp giảm chi phí và cải thiện hiệu năng.

## Sử dụng Service Internal Traffic Policy (Using Service Internal Traffic Policy)

Bạn có thể bật chính sách chỉ-lưu-lượng-nội-bộ cho một Service bằng cách đặt `.spec.internalTrafficPolicy` của nó thành `Local`. Điều này báo cho kube-proxy chỉ sử dụng các endpoint cục bộ trên node cho lưu lượng nội bộ của cluster.

> **Ghi chú:**
> Với các pod trên những node không có endpoint nào cho một Service nhất định, Service hành xử như thể nó không có endpoint nào (đối với các Pod trên node này), ngay cả khi Service thực sự có endpoint trên các node khác.

Ví dụ sau đây cho thấy một Service trông như thế nào khi bạn đặt `.spec.internalTrafficPolicy` thành `Local`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  internalTrafficPolicy: Local
```

## Cách hoạt động (How it works)

Kube-proxy lọc các endpoint mà nó định tuyến tới dựa trên thiết lập `spec.internalTrafficPolicy`. Khi thiết lập này là `Local`, chỉ các endpoint cục bộ trên node được xem xét. Khi thiết lập này là `Cluster` (giá trị mặc định), hoặc không được đặt, Kubernetes xem xét tất cả các endpoint.

## Tiếp theo (What's next)

* Đọc về [Định tuyến nhận biết topology (Topology Aware Routing)](86-topology-aware-routing-vi.md)
* Đọc về [Chính sách lưu lượng bên ngoài của Service (Service External Traffic Policy)](364-create-external-load-balancer-vi.md#preserving-the-client-source-ip)
* Làm theo hướng dẫn [Kết nối ứng dụng với Service (Connecting Applications with Services)](https://kubernetes.io/docs/tutorials/services/connect-applications-service/)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 5:

1. Cluster lab có hai worker. Một Deployment 2 replica nằm hết trên `lab-k8s-worker1`, và Service
   của nó đặt `internalTrafficPolicy: Local`. Một Pod client trên `lab-k8s-worker2` gọi Service đó
   thì kết quả ra sao?
2. `internalTrafficPolicy: Local` có làm thay đổi lưu lượng đi vào từ bên ngoài cluster qua
   NodePort không?
3. Thành phần nào thực sự lọc endpoint, và giá trị mặc định của `.spec.internalTrafficPolicy`
   là gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Gọi không được.** Với các pod trên những node **không có endpoint nào** cho Service đó,
   Service hành xử **như thể nó không có endpoint nào**, ngay cả khi Service thực sự có endpoint
   trên node khác. Đây chính là cái giá của việc giữ lưu lượng trong node: bạn tự chịu trách
   nhiệm bảo đảm mỗi node có ít nhất một endpoint.
2. **Không.** Trường này chỉ chi phối lưu lượng **nội bộ**, mà bài định nghĩa hẹp là "lưu lượng
   xuất phát từ các Pod trong cluster hiện tại". Lưu lượng đi vào từ bên ngoài thuộc về
   `externalTrafficPolicy` — thứ mà bài này chỉ nhắc tên ở mục *Tiếp theo*, không trình bày.
3. **kube-proxy** lọc các endpoint mà nó định tuyến tới dựa trên thiết lập
   `spec.internalTrafficPolicy`. Mặc định là **`Cluster`** (hoặc không đặt gì), khi đó Kubernetes
   xem xét tất cả các endpoint.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
