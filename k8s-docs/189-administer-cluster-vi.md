# Quản trị một Cluster (Administer a Cluster)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/>
>
> Tìm hiểu các tác vụ thường gặp để quản trị một cluster.

Trang gốc là trang mục lục của phần *Tasks → Administer a Cluster*: ngoài phần mô tả ở trên,
nội dung trang là danh sách các trang con, được liệt kê dưới đây theo đúng thứ tự hiển thị
trên trang web.

## Danh sách các trang trong mục này (Pages in this section)

- [Quản trị với kubeadm (Administration with kubeadm)](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/)
- [Cấp phát dư dung lượng node cho một cluster (Overprovision Node Capacity For A Cluster)](https://kubernetes.io/docs/tasks/administer-cluster/node-overprovisioning/)
- [Di trú khỏi dockershim (Migrating from dockershim)](https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/)
- [Tạo certificate thủ công (Generate Certificates Manually)](191-certificates-manual-vi.md) — đã có bản dịch trong repo này
- [Quản lý tài nguyên bộ nhớ, CPU và API (Manage Memory, CPU, and API Resources)](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/)
- [Cài đặt một Network Policy Provider (Install a Network Policy Provider)](https://kubernetes.io/docs/tasks/administer-cluster/network-policy-provider/)
- [Truy cập cluster bằng Kubernetes API (Access Clusters Using the Kubernetes API)](190-access-cluster-api-vi.md) — đã có bản dịch trong repo này
- [Bật hoặc tắt các Feature Gate (Enable Or Disable Feature Gates)](https://kubernetes.io/docs/tasks/administer-cluster/configure-feature-gates/)
- [Quảng bá extended resource cho một node (Advertise Extended Resources for a Node)](https://kubernetes.io/docs/tasks/administer-cluster/extended-resource-node/)
- [Tự động co giãn DNS Service trong một cluster (Autoscale the DNS Service in a Cluster)](https://kubernetes.io/docs/tasks/administer-cluster/dns-horizontal-autoscaling/)
- [Đổi access mode của một PersistentVolume sang ReadWriteOncePod (Change the Access Mode of a PersistentVolume to ReadWriteOncePod)](https://kubernetes.io/docs/tasks/administer-cluster/change-pv-access-mode-readwriteoncepod/)
- [Đổi StorageClass mặc định (Change the default StorageClass)](https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/)
- [Chuyển từ polling sang cập nhật trạng thái container dựa trên sự kiện CRI (Switching from Polling to CRI Event-based Updates to Container Status)](https://kubernetes.io/docs/tasks/administer-cluster/switch-to-evented-pleg/)
- [Đổi reclaim policy của một PersistentVolume (Change the Reclaim Policy of a PersistentVolume)](https://kubernetes.io/docs/tasks/administer-cluster/change-pv-reclaim-policy/)
- [Quản trị Cloud Controller Manager (Cloud Controller Manager Administration)](https://kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/)
- [Cấu hình một kubelet image credential provider (Configure a kubelet image credential provider)](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-credential-provider/)
- [Cấu hình quota cho các API object (Configure Quotas for API Objects)](https://kubernetes.io/docs/tasks/administer-cluster/quota-api-object/)
- [Kiểm soát các chính sách quản lý CPU trên node (Control CPU Management Policies on the Node)](https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/)
- [Kiểm soát các chính sách quản lý bộ nhớ trên một node (Control Memory Management Policies on a Node)](https://kubernetes.io/docs/tasks/administer-cluster/memory-manager/)
- [Kiểm soát các chính sách quản lý topology trên một node (Control Topology Management Policies on a node)](https://kubernetes.io/docs/tasks/administer-cluster/topology-manager/)
- [Tùy chỉnh DNS Service (Customizing DNS Service)](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/)
- [Gỡ lỗi phân giải DNS (Debugging DNS Resolution)](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)
- [Khai báo Network Policy (Declare Network Policy)](https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/)
- [Phát triển Cloud Controller Manager (Developing Cloud Controller Manager)](https://kubernetes.io/docs/tasks/administer-cluster/developing-cloud-controller-manager/)
- [Bật hoặc tắt một Kubernetes API (Enable Or Disable A Kubernetes API)](https://kubernetes.io/docs/tasks/administer-cluster/enable-disable-api/)
- [Mã hóa dữ liệu bí mật khi lưu trữ (Encrypting Confidential Data at Rest)](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
- [Giải mã dữ liệu bí mật đã được mã hóa khi lưu trữ (Decrypt Confidential Data that is Already Encrypted at Rest)](https://kubernetes.io/docs/tasks/administer-cluster/decrypt-data/)
- [Bảo đảm lập lịch cho các Pod add-on quan trọng (Guaranteed Scheduling For Critical Add-On Pods)](https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/)
- [Hướng dẫn sử dụng IP Masquerade Agent (IP Masquerade Agent User Guide)](https://kubernetes.io/docs/tasks/administer-cluster/ip-masq-agent/)
- [Giới hạn mức tiêu thụ lưu trữ (Limit Storage Consumption)](https://kubernetes.io/docs/tasks/administer-cluster/limit-storage-consumption/)
- [Di trú control plane dạng nhân bản sang dùng Cloud Controller Manager (Migrate Replicated Control Plane To Use Cloud Controller Manager)](https://kubernetes.io/docs/tasks/administer-cluster/controller-manager-leader-migration/)
- [Vận hành các cluster etcd cho Kubernetes (Operating etcd clusters for Kubernetes)](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)
- [Dành riêng tài nguyên tính toán cho các daemon hệ thống (Reserve Compute Resources for System Daemons)](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/)
- [Chạy các thành phần node của Kubernetes bằng người dùng không phải root (Running Kubernetes Node Components as a Non-root User)](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-in-userns/)
- [Drain một node an toàn (Safely Drain a Node)](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)
- [Bảo mật một cluster (Securing a Cluster)](https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/)
- [Tăng cường bảo mật Dynamic Resource Allocation trong cluster của bạn (Harden Dynamic Resource Allocation in Your Cluster)](https://kubernetes.io/docs/tasks/administer-cluster/hardening-dra/)
- [Đặt tham số kubelet qua file cấu hình (Set Kubelet Parameters Via A Configuration File)](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/)
- [Chia sẻ một cluster bằng namespace (Share a Cluster with Namespaces)](https://kubernetes.io/docs/tasks/administer-cluster/namespaces/)
- [Nâng cấp một cluster (Upgrade A Cluster)](https://kubernetes.io/docs/tasks/administer-cluster/cluster-upgrade/)
- [Dùng xóa theo tầng trong một cluster (Use Cascading Deletion in a Cluster)](https://kubernetes.io/docs/tasks/administer-cluster/use-cascading-deletion/)
- [Dùng một KMS provider để mã hóa dữ liệu (Using a KMS provider for data encryption)](https://kubernetes.io/docs/tasks/administer-cluster/kms-provider/)
- [Dùng CoreDNS cho khám phá dịch vụ (Using CoreDNS for Service Discovery)](https://kubernetes.io/docs/tasks/administer-cluster/coredns/)
- [Dùng NodeLocal DNSCache trong các cluster Kubernetes (Using NodeLocal DNSCache in Kubernetes Clusters)](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/)
- [Dùng sysctl trong một cluster Kubernetes (Using sysctls in a Kubernetes Cluster)](https://kubernetes.io/docs/tasks/administer-cluster/sysctl-cluster/)
- [Xác minh các artifact Kubernetes đã ký (Verify Signed Kubernetes Artifacts)](https://kubernetes.io/docs/tasks/administer-cluster/verify-signed-artifacts/)
