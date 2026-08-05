# Windows trong Kubernetes (Windows in Kubernetes)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/concepts/windows/
>
> Kubernetes hỗ trợ các node chạy Microsoft Windows.

Kubernetes hỗ trợ các worker node chạy Linux hoặc Microsoft Windows.

> **Ghi chú:** Nội dung trên trang này đề cập đến một sản phẩm hoặc dự án bên thứ ba cung cấp chức năng mà Kubernetes cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những sản phẩm hoặc dự án bên thứ ba đó. Xem [nguyên tắc website của CNCF](https://github.com/cncf/foundation/blob/main/website-guidelines.md) để biết thêm chi tiết.

CNCF và tổ chức mẹ của nó là Linux Foundation có cách tiếp cận trung lập với nhà cung cấp (vendor-neutral) đối với khả năng tương thích. Bạn hoàn toàn có thể thêm [Windows server](https://www.microsoft.com/en-us/windows-server) của mình làm worker node vào một cluster Kubernetes.

Bạn có thể [cài đặt và thiết lập kubectl trên Windows](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/) bất kể bạn sử dụng hệ điều hành nào bên trong cluster.

Nếu bạn đang sử dụng node Windows, bạn có thể đọc:

* [Mạng trên Windows](89-windows-networking-vi.md)
* [Lưu trữ Windows trong Kubernetes](106-windows-storage-vi.md)
* [Quản lý tài nguyên cho node Windows](112-windows-resource-management-vi.md)
* [Cấu hình RunAsUserName cho Pod và container Windows](https://kubernetes.io/docs/tasks/configure-pod-container/configure-runasusername/)
* [Tạo một Windows HostProcess Pod](https://kubernetes.io/docs/tasks/configure-pod-container/create-hostprocess-pod/)
* [Cấu hình Group Managed Service Accounts cho Pod và container Windows](https://kubernetes.io/docs/tasks/configure-pod-container/configure-gmsa/)
* [Bảo mật cho node Windows](131-windows-security-vi.md)
* [Mẹo gỡ lỗi trên Windows](https://kubernetes.io/docs/tasks/debug/debug-cluster/windows/)
* [Hướng dẫn lập lịch Windows container trong Kubernetes](https://kubernetes.io/docs/concepts/windows/user-guide)

hoặc, để có cái nhìn tổng quan, hãy đọc:

* [Windows containers trong Kubernetes](175-windows-intro-vi.md)
* [Hướng dẫn lập lịch Windows container trong Kubernetes](https://kubernetes.io/docs/concepts/windows/user-guide/)
