# Lưu trữ trên Windows (Windows Storage)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/storage/windows-storage/>

Trang này cung cấp cái nhìn tổng quan về lưu trữ dành riêng cho hệ điều hành Windows.

## Lưu trữ bền vững (Persistent storage) {#storage}

Windows có driver hệ thống file phân lớp (layered filesystem driver) để mount các lớp
(layer) của container và tạo một hệ thống file sao chép dựa trên NTFS. Mọi đường dẫn
file trong container chỉ được phân giải trong ngữ cảnh của chính container đó.

* Với Docker, volume mount chỉ có thể trỏ tới một thư mục trong container, không thể
  trỏ tới một file riêng lẻ. Hạn chế này không áp dụng cho containerd.
* Volume mount không thể chiếu (project) file hoặc thư mục ngược trở lại hệ thống file
  của host.
* Hệ thống file chỉ đọc (read-only) không được hỗ trợ vì quyền ghi luôn được yêu cầu
  đối với registry của Windows và cơ sở dữ liệu SAM. Tuy nhiên, volume chỉ đọc vẫn
  được hỗ trợ.
* User-mask và quyền (permission) trên volume không khả dụng. Vì SAM không được chia sẻ
  giữa host và container, không tồn tại ánh xạ nào giữa hai bên. Mọi quyền đều được
  phân giải trong ngữ cảnh của container.

Do đó, các chức năng lưu trữ sau không được hỗ trợ trên node Windows:

* Mount subpath của volume: chỉ có thể mount toàn bộ volume vào một container Windows
* Mount volume theo subpath cho Secret
* Chiếu mount ngược về host (host mount projection)
* Hệ thống file gốc (root filesystem) chỉ đọc (các volume được ánh xạ vẫn hỗ trợ `readOnly`)
* Ánh xạ thiết bị block (block device mapping)
* Dùng bộ nhớ (memory) làm phương tiện lưu trữ (ví dụ, `emptyDir.medium` đặt thành `Memory`)
* Các tính năng hệ thống file như uid/gid; quyền hệ thống file Linux theo từng người dùng
* Thiết lập [quyền cho secret bằng DefaultMode](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/#set-posix-permissions-for-secret-keys) (do phụ thuộc vào UID/GID)
* Hỗ trợ lưu trữ/volume dựa trên NFS
* Mở rộng volume đã mount (resizefs)

Các volume của Kubernetes cho phép triển khai trên Kubernetes những ứng dụng phức tạp
có yêu cầu lưu dữ liệu bền vững và chia sẻ volume giữa các Pod. Việc quản lý các
persistent volume gắn với một backend hoặc giao thức lưu trữ cụ thể bao gồm các thao
tác như: cấp phát/thu hồi cấp phát/thay đổi kích thước (provisioning/de-provisioning/resizing)
volume, gắn (attach) một volume vào / tháo (detach) một volume khỏi một node Kubernetes,
và mount/unmount một volume vào/khỏi từng container trong một Pod cần lưu dữ liệu bền vững.

Các thành phần quản lý volume được phát hành dưới dạng
[plugin](./91-volumes-vi.md#volume-types) volume của Kubernetes.
Các nhóm plugin volume Kubernetes lớn sau đây được hỗ trợ trên Windows:

* [`FlexVolume plugins`](./91-volumes-vi.md#flexvolume)
  * Lưu ý rằng FlexVolume đã bị loại bỏ dần (deprecated) kể từ phiên bản 1.23
* [`CSI Plugins`](https://kubernetes.io/docs/concepts/storage/volumes/#csi)

##### Các plugin volume in-tree (In-tree volume plugins)

Các plugin in-tree sau hỗ trợ lưu trữ bền vững trên node Windows:

* [`azureFile`](https://kubernetes.io/docs/concepts/storage/volumes/#azurefile)
* [`vsphereVolume`](https://kubernetes.io/docs/concepts/storage/volumes/#vspherevolume)
