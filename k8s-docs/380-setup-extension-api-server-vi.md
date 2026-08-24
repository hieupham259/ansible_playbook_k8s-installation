# Thiết lập một Extension API Server (Set up an Extension API Server)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/extend-kubernetes/setup-extension-api-server/>
>
> Thiết lập một extension API server để làm việc với aggregation layer cho phép mở rộng
> Kubernetes apiserver bằng các API bổ sung không thuộc nhóm API lõi của Kubernetes.

Thiết lập một extension API server để làm việc với aggregation layer cho phép mở rộng Kubernetes
apiserver bằng các API bổ sung không thuộc nhóm API lõi của Kubernetes.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

* Bạn phải [cấu hình aggregation layer](374-configure-aggregation-layer-vi.md) và bật các flag
  tương ứng của apiserver.

## Thiết lập một extension api-server để làm việc với aggregation layer (Set up an extension api-server to work with the aggregation layer)

Các bước sau mô tả cách thiết lập một extension-apiserver *ở mức tổng quát*. Các bước này áp
dụng được dù bạn dùng file cấu hình YAML hay dùng API trực tiếp; chỗ nào hai cách khác nhau thì
tài liệu sẽ cố gắng chỉ rõ. Để xem một ví dụ cụ thể về cách hiện thực chúng bằng file cấu hình
YAML, bạn có thể tham khảo
[sample-apiserver](https://github.com/kubernetes/sample-apiserver/blob/master/README.md) trong
repo của Kubernetes.

Ngoài ra, bạn có thể dùng một giải pháp bên thứ ba có sẵn, chẳng hạn
[apiserver-builder](https://github.com/kubernetes-sigs/apiserver-builder-alpha/blob/master/README.md);
công cụ này sẽ sinh ra bộ khung (skeleton) và tự động hóa toàn bộ các bước dưới đây cho bạn.

1. Đảm bảo API APIService đang được bật (kiểm tra `--runtime-config`). Mặc định nó đã bật, trừ
   khi có ai đó cố ý tắt nó trong cluster của bạn.
2. Bạn có thể cần tạo một luật RBAC cho phép mình thêm các đối tượng APIService, hoặc nhờ quản
   trị viên cluster tạo giúp. (Vì các phần mở rộng API ảnh hưởng tới toàn bộ cluster, không nên
   thử nghiệm/phát triển/gỡ lỗi một API extension trên một cluster đang chạy thật.)
3. Tạo namespace Kubernetes mà bạn muốn chạy extension api-service bên trong đó.
4. Tạo (hoặc lấy) một CA certificate dùng để ký server certificate mà extension api-server sẽ
   dùng cho HTTPS.
5. Tạo một cặp certificate/key phía server để api-server dùng cho HTTPS. Certificate này phải
   được ký bởi CA ở trên. Nó cũng phải có CN là tên DNS trong Kubernetes (Kube DNS name). Tên
   này được suy ra từ Service Kubernetes và có dạng `<service name>.<service name namespace>.svc`
6. Tạo một Secret Kubernetes chứa cặp certificate/key phía server, đặt trong namespace của bạn.
7. Tạo một Deployment Kubernetes cho extension api-server và đảm bảo bạn nạp Secret đó vào dưới
   dạng một volume. Deployment phải tham chiếu tới một image hoạt động được của extension
   api-server. Deployment cũng phải nằm trong namespace của bạn.
8. Đảm bảo extension-apiserver của bạn nạp các certificate đó từ volume vừa nói, và chúng thực
   sự được dùng trong quá trình bắt tay (handshake) HTTPS.
9. Tạo một ServiceAccount Kubernetes trong namespace của bạn.
10. Tạo một ClusterRole Kubernetes cho những thao tác mà bạn muốn cho phép thực hiện trên các
    resource của mình.
11. Tạo một ClusterRoleBinding Kubernetes từ ServiceAccount trong namespace của bạn tới
    ClusterRole mà bạn vừa tạo.
12. Tạo một ClusterRoleBinding Kubernetes từ ServiceAccount trong namespace của bạn tới
    ClusterRole `system:auth-delegator`, để ủy quyền (delegate) các quyết định xác thực và phân
    quyền cho API server lõi của Kubernetes.
13. Tạo một RoleBinding Kubernetes từ ServiceAccount trong namespace của bạn tới Role
    `extension-apiserver-authentication-reader`. Việc này cho phép extension api-server của bạn
    truy cập ConfigMap `extension-apiserver-authentication`.
14. Tạo một apiservice Kubernetes. CA certificate ở trên phải được mã hóa base64, loại bỏ các ký
    tự xuống dòng, rồi dùng làm `spec.caBundle` trong apiservice. Đối tượng này không thuộc
    namespace nào. Nếu bạn dùng [kube-aggregator API](https://github.com/kubernetes/kube-aggregator/),
    chỉ cần truyền vào CA bundle ở dạng mã hóa PEM, vì phần mã hóa base64 đã được thực hiện sẵn
    cho bạn.
15. Dùng kubectl để lấy resource của bạn. Khi chạy, kubectl sẽ trả về "No resources found.".
    Thông báo này cho biết mọi thứ đã hoạt động, nhưng hiện tại bạn chưa tạo đối tượng nào thuộc
    loại resource đó.

## Tiếp theo (What's next)

* Đi qua các bước để [cấu hình aggregation layer của API](374-configure-aggregation-layer-vi.md)
  và bật các flag tương ứng của apiserver.
* Để có cái nhìn tổng quan ở mức cao, xem
  [Mở rộng Kubernetes API bằng aggregation layer](180-apiserver-aggregation-vi.md).
* Tìm hiểu cách [mở rộng Kubernetes API bằng CustomResourceDefinition](378-custom-resource-definitions-vi.md).
