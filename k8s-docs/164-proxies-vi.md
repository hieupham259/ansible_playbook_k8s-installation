# Các loại proxy trong Kubernetes (Proxies in Kubernetes)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/concepts/cluster-administration/proxies/

Trang này giải thích các loại proxy được sử dụng với Kubernetes.

## Proxy (Proxies)

Có một số loại proxy khác nhau mà bạn có thể gặp khi sử dụng Kubernetes:

1.  [kubectl proxy](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/#directly-accessing-the-rest-api):

    - chạy trên máy tính của người dùng hoặc trong một pod
    - proxy từ một địa chỉ localhost đến Kubernetes apiserver
    - kết nối từ client đến proxy dùng HTTP
    - kết nối từ proxy đến apiserver dùng HTTPS
    - tự định vị apiserver
    - thêm các header xác thực (authentication headers)

2.  [apiserver proxy](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster-services/#discovering-builtin-services):

    - là một bastion được tích hợp sẵn trong apiserver
    - kết nối người dùng ở bên ngoài cluster đến các cluster IP mà nếu không có nó thì có thể không truy cập được
    - chạy trong các tiến trình của apiserver
    - kết nối từ client đến proxy dùng HTTPS (hoặc HTTP nếu apiserver được cấu hình như vậy)
    - kết nối từ proxy đến đích có thể dùng HTTP hoặc HTTPS, do proxy lựa chọn dựa trên thông tin sẵn có
    - có thể được dùng để truy cập một Node, Pod hoặc Service
    - thực hiện cân bằng tải (load balancing) khi được dùng để truy cập một Service

3.  [kube proxy](https://kubernetes.io/docs/concepts/services-networking/service/#ips-and-vips):

    - chạy trên mỗi node
    - proxy các giao thức UDP, TCP và SCTP
    - không hiểu HTTP
    - cung cấp cân bằng tải
    - chỉ được dùng để truy cập các service

4.  Một Proxy/Load-balancer đứng trước (các) apiserver:

    - sự tồn tại và cách hiện thực khác nhau tùy từng cluster (ví dụ nginx)
    - nằm giữa tất cả các client và một hoặc nhiều apiserver
    - đóng vai trò bộ cân bằng tải (load balancer) nếu có nhiều apiserver.

5.  Cloud Load Balancer trên các service bên ngoài:

    - được cung cấp bởi một số nhà cung cấp đám mây (ví dụ AWS ELB, Google Cloud Load Balancer)
    - được tạo tự động khi Kubernetes service có type là `LoadBalancer`
    - thường chỉ hỗ trợ UDP/TCP
    - việc hỗ trợ SCTP tùy thuộc vào cách hiện thực bộ cân bằng tải của nhà cung cấp đám mây
    - cách hiện thực khác nhau tùy theo nhà cung cấp đám mây.

Người dùng Kubernetes thường sẽ không cần quan tâm đến bất cứ điều gì ngoài hai loại đầu tiên. Quản trị viên cluster
thường sẽ đảm bảo rằng các loại còn lại được thiết lập đúng.

## Yêu cầu chuyển hướng (Requesting redirects)

Proxy đã thay thế khả năng chuyển hướng (redirect). Chuyển hướng đã bị loại bỏ (deprecated).
