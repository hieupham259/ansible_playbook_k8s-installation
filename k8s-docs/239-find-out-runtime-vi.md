# Tìm hiểu Container Runtime nào đang được dùng trên Node (Find Out What Container Runtime is Used on a Node)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/find-out-runtime-you-use/

Trang này trình bày các bước để tìm hiểu xem các node trong cluster của bạn đang dùng
[container runtime](https://kubernetes.io/docs/setup/production-environment/container-runtimes/) nào.

Tùy theo cách bạn vận hành cluster, container runtime cho các node có thể
đã được cấu hình sẵn hoặc bạn phải tự cấu hình. Nếu bạn đang dùng dịch vụ
Kubernetes được quản lý (managed), có thể có cách riêng của từng nhà cung cấp để kiểm tra container runtime nào
đang được cấu hình cho các node. Phương pháp mô tả trên trang này hoạt động bất cứ khi nào
việc thực thi `kubectl` được cho phép.

## Trước khi bạn bắt đầu (Before you begin)

Cài đặt và cấu hình `kubectl`. Xem mục [Install Tools](https://kubernetes.io/docs/tasks/tools/#kubectl) để biết chi tiết.

## Tìm container runtime đang được dùng trên một Node (Find out the container runtime used on a Node)

Dùng `kubectl` để lấy và hiển thị thông tin node:

```shell
kubectl get nodes -o wide
```

Kết quả sẽ tương tự như dưới đây. Cột `CONTAINER-RUNTIME` hiển thị
runtime và phiên bản của nó.

Với Docker Engine, kết quả tương tự như sau:

```none
NAME         STATUS   VERSION    CONTAINER-RUNTIME
node-1       Ready    v1.16.15   docker://19.3.1
node-2       Ready    v1.16.15   docker://19.3.1
node-3       Ready    v1.16.15   docker://19.3.1
```

Nếu runtime của bạn hiển thị là Docker Engine, bạn vẫn có thể không bị ảnh hưởng bởi
việc loại bỏ dockershim trong Kubernetes v1.24.
[Kiểm tra runtime endpoint](#which-endpoint) để xem bạn có đang dùng dockershim hay không.
Nếu bạn không dùng dockershim, bạn không bị ảnh hưởng.

Với containerd, kết quả tương tự như sau:

```none
NAME         STATUS   VERSION   CONTAINER-RUNTIME
node-1       Ready    v1.19.6   containerd://1.4.1
node-2       Ready    v1.19.6   containerd://1.4.1
node-3       Ready    v1.19.6   containerd://1.4.1
```

Tìm hiểu thêm thông tin về các container runtime
tại trang [Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/).

## Tìm container runtime endpoint mà bạn đang dùng (Find out what container runtime endpoint you use) {#which-endpoint}

Container runtime giao tiếp với kubelet qua một Unix socket bằng [giao thức CRI](https://kubernetes.io/docs/concepts/architecture/cri/),
vốn dựa trên framework gRPC. Kubelet đóng vai trò client, còn runtime đóng vai trò server.
Trong một số trường hợp, bạn có thể thấy hữu ích khi biết node của mình đang dùng socket nào. Ví dụ,
với việc loại bỏ dockershim trong Kubernetes v1.24 trở về sau, bạn có thể
muốn biết liệu mình có đang dùng Docker Engine với dockershim hay không.

> **Ghi chú:** Nếu bạn hiện đang dùng Docker Engine trên các node của mình với `cri-dockerd`, bạn không
> bị ảnh hưởng bởi việc loại bỏ dockershim.

Bạn có thể kiểm tra socket đang dùng bằng cách kiểm tra cấu hình kubelet trên
các node của mình.

1.  Đọc các lệnh khởi động của tiến trình kubelet:

    ```
    tr \\0 ' ' < /proc/"$(pgrep kubelet)"/cmdline
    ```
    Nếu bạn không có `tr` hoặc `pgrep`, hãy kiểm tra thủ công dòng lệnh của tiến trình
    kubelet.

1.  Trong kết quả, tìm flag `--container-runtime` và flag
    `--container-runtime-endpoint`.

    *   Nếu node của bạn dùng Kubernetes v1.23 trở về trước và các flag này không
        xuất hiện, hoặc flag `--container-runtime` không phải là `remote`,
        thì bạn đang dùng dockershim socket với Docker Engine. Đối số dòng lệnh
        `--container-runtime` không còn khả dụng trong Kubernetes v1.27 trở về sau.
    *   Nếu flag `--container-runtime-endpoint` xuất hiện, hãy kiểm tra tên socket
        để tìm ra runtime bạn đang dùng. Ví dụ,
        `unix:///run/containerd/containerd.sock` là endpoint của containerd.

Nếu bạn muốn thay đổi Container Runtime trên một Node từ Docker Engine sang containerd,
bạn có thể tìm thêm thông tin tại [di chuyển từ Docker Engine sang containerd](https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/change-runtime-containerd/),
hoặc, nếu bạn muốn tiếp tục dùng Docker Engine trong Kubernetes v1.24 trở về sau, hãy chuyển sang một
adapter tương thích CRI như [`cri-dockerd`](https://github.com/Mirantis/cri-dockerd).
