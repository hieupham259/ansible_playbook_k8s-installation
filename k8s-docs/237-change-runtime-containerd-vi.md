# Thay đổi Container Runtime trên Node từ Docker Engine sang containerd (Changing the Container Runtime on a Node from Docker Engine to containerd)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/change-runtime-containerd/

Trang này trình bày các bước cần thiết để chuyển container runtime của bạn từ Docker sang containerd. Nội dung áp dụng cho những người vận hành cluster đang chạy Kubernetes 1.23 trở về trước. Trang này cũng bao gồm một kịch bản ví dụ về việc di chuyển (migrate) từ dockershim sang containerd. Bạn có thể chọn container runtime thay thế khác từ [trang này](https://kubernetes.io/docs/setup/production-environment/container-runtimes/).

## Trước khi bạn bắt đầu (Before you begin)

> **Ghi chú:** Mục này liên kết tới các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án đó. Trang này tuân theo [nguyên tắc nội dung của website CNCF](https://github.com/cncf/foundation/blob/master/website-guidelines.md), liệt kê các mục theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc [hướng dẫn nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content) trước khi gửi thay đổi.

Cài đặt containerd. Để biết thêm thông tin, xem
[tài liệu cài đặt của containerd](https://containerd.io/docs/getting-started/)
và với các điều kiện tiên quyết cụ thể, hãy làm theo
[hướng dẫn containerd](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd).

## Drain node (Drain the node)

```shell
kubectl drain <node-to-drain> --ignore-daemonsets
```

Thay `<node-to-drain>` bằng tên của node mà bạn đang drain.

## Dừng Docker daemon (Stop the Docker daemon)

```shell
systemctl stop kubelet
systemctl disable docker.service --now
```

## Cài đặt containerd (Install Containerd)

Làm theo [hướng dẫn](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd)
để biết các bước chi tiết cài đặt containerd.

#### Linux

1. Cài đặt gói `containerd.io` từ các kho (repository) chính thức của Docker.
   Hướng dẫn thiết lập kho Docker cho từng bản phân phối Linux tương ứng và
   cài đặt gói `containerd.io` có tại
   [Getting started with containerd](https://github.com/containerd/containerd/blob/main/docs/getting-started.md).

2. Cấu hình containerd:

   ```shell
   sudo mkdir -p /etc/containerd
   containerd config default | sudo tee /etc/containerd/config.toml
   ```

3. Khởi động lại containerd:

   ```shell
   sudo systemctl restart containerd
   ```

#### Windows (PowerShell)

Mở một phiên PowerShell, đặt `$Version` thành phiên bản mong muốn (ví dụ: `$Version="1.4.3"`), rồi chạy các lệnh sau:

1. Tải containerd:

   ```powershell
   curl.exe -L https://github.com/containerd/containerd/releases/download/v$Version/containerd-$Version-windows-amd64.tar.gz -o containerd-windows-amd64.tar.gz
   tar.exe xvf .\containerd-windows-amd64.tar.gz
   ```

2. Giải nén và cấu hình:

   ```powershell
   Copy-Item -Path ".\bin\" -Destination "$Env:ProgramFiles\containerd" -Recurse -Force
   cd $Env:ProgramFiles\containerd\
   .\containerd.exe config default | Out-File config.toml -Encoding ascii

   # Xem lại cấu hình. Tùy vào thiết lập, bạn có thể muốn điều chỉnh:
   # - sandbox_image (image pause của Kubernetes)
   # - vị trí bin_dir và conf_dir của cni
   Get-Content config.toml

   # (Tùy chọn - nhưng rất nên làm) Loại trừ containerd khỏi quét của Windows Defender
   Add-MpPreference -ExclusionProcess "$Env:ProgramFiles\containerd\containerd.exe"
   ```

3. Khởi động containerd:

   ```powershell
   .\containerd.exe --register-service
   Start-Service containerd
   ```

## Cấu hình kubelet sử dụng containerd làm container runtime (Configure the kubelet to use containerd as its container runtime)

Chỉnh sửa file `/var/lib/kubelet/kubeadm-flags.env` và thêm containerd runtime vào các flag:
`--container-runtime-endpoint=unix:///run/containerd/containerd.sock`.

Người dùng kubeadm cần lưu ý rằng công cụ kubeadm lưu CRI socket của host trong

file `/var/lib/kubelet/instance-config.yaml` trên mỗi node. Bạn có thể tạo file `/var/lib/kubelet/instance-config.yaml` này trên node.

File `/var/lib/kubelet/instance-config.yaml` cho phép thiết lập tham số `containerRuntimeEndpoint`.

Bạn có thể đặt giá trị của tham số này thành đường dẫn của CRI socket mà bạn đã chọn (ví dụ `unix:///run/containerd/containerd.sock`).

## Khởi động lại kubelet (Restart the kubelet)

```shell
systemctl start kubelet
```

## Xác minh node ở trạng thái healthy (Verify that the node is healthy)

Chạy `kubectl get nodes -o wide` và containerd sẽ hiển thị là runtime của node mà chúng ta vừa thay đổi.

## Gỡ bỏ Docker Engine (Remove Docker Engine)

> **Ghi chú:** Mục này liên kết tới các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án đó. Trang này tuân theo [nguyên tắc nội dung của website CNCF](https://github.com/cncf/foundation/blob/master/website-guidelines.md), liệt kê các mục theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc [hướng dẫn nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content) trước khi gửi thay đổi.

Nếu node có vẻ đã healthy, hãy gỡ bỏ Docker.

#### CentOS

```shell
sudo yum remove docker-ce docker-ce-cli
```

#### Debian

```shell
sudo apt-get purge docker-ce docker-ce-cli
```

#### Fedora

```shell
sudo dnf remove docker-ce docker-ce-cli
```

#### Ubuntu

```shell
sudo apt-get purge docker-ce docker-ce-cli
```

Các lệnh trên không xóa image, container, volume hay các file cấu hình tùy chỉnh trên host của bạn.
Để xóa chúng, hãy làm theo hướng dẫn của Docker tại [Uninstall Docker Engine](https://docs.docker.com/engine/install/ubuntu/#uninstall-docker-engine).

> **Thận trọng:** Hướng dẫn gỡ cài đặt Docker Engine của Docker tiềm ẩn rủi ro xóa nhầm containerd. Hãy cẩn thận khi thực thi các lệnh.

## Uncordon node (Uncordon the node)

```shell
kubectl uncordon <node-to-uncordon>
```

Thay `<node-to-uncordon>` bằng tên của node mà bạn đã drain trước đó.
