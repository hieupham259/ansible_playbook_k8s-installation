# Tính năng do Node khai báo (Node Declared Features)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/node-declared-features/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [beta]`

Các node trong Kubernetes sử dụng _tính năng được khai báo_ (declared features) để báo cáo
tính khả dụng của những tính năng cụ thể còn mới hoặc đang được kiểm soát bằng feature gate.
Các thành phần của control plane tận dụng thông tin này để đưa ra quyết định tốt hơn.
kube-scheduler, thông qua plugin `NodeDeclaredFeatures`, đảm bảo các Pod chỉ được đặt lên
những node hỗ trợ tường minh các tính năng mà Pod yêu cầu. Ngoài ra, admission controller
`NodeDeclaredFeatureValidator` kiểm tra tính hợp lệ của các cập nhật Pod dựa trên các tính
năng mà node đã khai báo.

Cơ chế này giúp quản lý sự chênh lệch phiên bản (version skew) và cải thiện độ ổn định của
cluster, đặc biệt trong quá trình nâng cấp cluster hoặc trong các môi trường lẫn lộn nhiều
phiên bản (mixed-version), nơi không phải tất cả các node đều bật cùng một tập tính năng.
Cơ chế này dành cho các nhà phát triển tính năng của Kubernetes khi giới thiệu các tính năng
mới ở cấp node và hoạt động ngầm ở phía sau; các nhà phát triển ứng dụng triển khai Pod
không cần tương tác trực tiếp với framework này.

## Cách hoạt động (How it Works)

1.  **Kubelet báo cáo tính năng (Kubelet Feature Reporting):** Khi khởi động, kubelet trên
    mỗi node phát hiện những tính năng Kubernetes được quản lý nào hiện đang được bật và
    báo cáo chúng trong trường `.status.declaredFeatures` của Node. Chỉ những tính năng
    đang trong quá trình phát triển tích cực mới được đưa vào trường này.
2.  **Scheduler lọc node (Scheduler Filtering):** kube-scheduler mặc định sử dụng plugin
    `NodeDeclaredFeatures`. Plugin này:
    * Ở giai đoạn `PreFilter`, kiểm tra `PodSpec` để suy ra tập các tính năng cấp node
      mà Pod yêu cầu.
    * Ở giai đoạn `Filter`, kiểm tra xem các tính năng được liệt kê trong
      `.status.declaredFeatures` của node có thỏa mãn các yêu cầu đã suy ra cho Pod hay
      không. Pod sẽ không được lập lịch (schedule) lên những node thiếu các tính năng cần thiết.
    Các scheduler tùy chỉnh cũng có thể tận dụng trường
    `.status.declaredFeatures` để áp đặt các ràng buộc tương tự.
3.  **Kiểm soát admission (Admission Control):** Admission controller
    `nodedeclaredfeaturevalidator` có thể từ chối những Pod yêu cầu các tính năng chưa được
    khai báo bởi node mà chúng được gán vào (bound), giúp ngăn ngừa sự cố khi cập nhật Pod.

## Bật tính năng do node khai báo (Enabling node declared features)

Để sử dụng Node Declared Features, [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#NodeDeclaredFeatures)
`NodeDeclaredFeatures` phải được bật trên các thành phần `kube-apiserver`,
`kube-scheduler` và `kubelet`.

## Tiếp theo (What's next)

* Đọc KEP để biết thêm chi tiết:
    [KEP-5328: Node Declared Features](https://github.com/kubernetes/enhancements/blob/6d3210f7dd5d547c8f7f6a33af6a09eb45193cd7/keps/sig-node/5328-node-declared-features/README.md)
* Đọc về [admission controller `NodeDeclaredFeatureValidator`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#nodedeclaredfeaturevalidator).
