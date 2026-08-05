# Chính sách lưu lượng nội bộ của Service (Service Internal Traffic Policy)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/services-networking/service-traffic-policy/>
>
> Nếu hai Pod trong cluster của bạn muốn giao tiếp với nhau, và cả hai Pod thực tế đang chạy trên cùng một node, hãy dùng *Chính sách lưu lượng nội bộ của Service* (Service Internal Traffic Policy) để giữ lưu lượng mạng bên trong node đó. Việc tránh một vòng đi-về qua mạng của cluster có thể có lợi cho độ tin cậy, hiệu năng (độ trễ mạng và thông lượng), hoặc chi phí.

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
* Đọc về [Chính sách lưu lượng bên ngoài của Service (Service External Traffic Policy)](https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/#preserving-the-client-source-ip)
* Làm theo hướng dẫn [Kết nối ứng dụng với Service (Connecting Applications with Services)](https://kubernetes.io/docs/tutorials/services/connect-applications-service/)
