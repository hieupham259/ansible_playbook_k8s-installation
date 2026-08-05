# Bảo mật cho các node Windows (Security For Windows Nodes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/windows-security/>
>
> Trang này mô tả các cân nhắc về bảo mật và các thực hành tốt nhất dành riêng cho hệ điều hành Windows.

Trang này mô tả các cân nhắc về bảo mật và các thực hành tốt nhất dành riêng cho hệ điều hành Windows.

## Bảo vệ dữ liệu Secret trên node (Protection for Secret data on nodes)

Trên Windows, dữ liệu từ Secret được ghi ra dưới dạng văn bản thuần (clear text) trên bộ lưu trữ
cục bộ của node (khác với việc dùng tmpfs / filesystem trong bộ nhớ trên Linux). Với tư cách là
người vận hành cluster, bạn nên thực hiện cả hai biện pháp bổ sung sau:

1. Sử dụng file ACL để bảo vệ vị trí file của các Secret.
1. Áp dụng mã hóa cấp volume bằng
   [BitLocker](https://docs.microsoft.com/windows/security/information-protection/bitlocker/bitlocker-how-to-deploy-on-windows-server).

## Người dùng trong container (Container users)

[RunAsUsername](https://kubernetes.io/docs/tasks/configure-pod-container/configure-runasusername)
có thể được chỉ định cho các Pod hoặc container Windows để thực thi các tiến trình
của container dưới một người dùng cụ thể. Điều này gần tương đương với
[RunAsUser](https://kubernetes.io/docs/concepts/security/pod-security-policy/#users-and-groups).

Các container Windows cung cấp hai tài khoản người dùng mặc định là ContainerUser và ContainerAdministrator.
Sự khác biệt giữa hai tài khoản người dùng này được trình bày trong
[When to use ContainerAdmin and ContainerUser user accounts](https://docs.microsoft.com/virtualization/windowscontainers/manage-containers/container-security#when-to-use-containeradmin-and-containeruser-user-accounts)
thuộc tài liệu _Secure Windows containers_ của Microsoft.

Người dùng cục bộ (local user) có thể được thêm vào container image trong quá trình build container.

> **Ghi chú:**
>
> * Các image dựa trên [Nano Server](https://hub.docker.com/_/microsoft-windows-nanoserver) chạy dưới
>   `ContainerUser` theo mặc định
> * Các image dựa trên [Server Core](https://hub.docker.com/_/microsoft-windows-servercore) chạy dưới
>   `ContainerAdministrator` theo mặc định

Các container Windows cũng có thể chạy dưới danh tính Active Directory bằng cách sử dụng
[Group Managed Service Accounts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-gmsa/)

## Cô lập bảo mật cấp Pod (Pod-level security isolation)

Các cơ chế security context của pod dành riêng cho Linux (như SELinux, AppArmor, Seccomp, hay
POSIX capability tùy chỉnh) không được hỗ trợ trên các node Windows.

Container đặc quyền (privileged container) [không được hỗ trợ](https://kubernetes.io/docs/concepts/windows/intro/#compatibility-v1-pod-spec-containers-securitycontext)
trên Windows.
Thay vào đó, có thể dùng [HostProcess container](https://kubernetes.io/docs/tasks/configure-pod-container/create-hostprocess-pod)
trên Windows để thực hiện nhiều tác vụ mà container đặc quyền thực hiện trên Linux.
