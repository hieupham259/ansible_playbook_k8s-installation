# Quản trị một Cluster (Administer a Cluster)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/>
>
> Tìm hiểu các tác vụ thường gặp để quản trị một cluster.

Trang gốc là trang mục lục của phần *Tasks → Administer a Cluster*: ngoài phần mô tả ở trên,
nội dung trang là danh sách các trang con, được liệt kê dưới đây theo đúng thứ tự hiển thị
trên trang web.

## Danh sách các trang trong mục này (Pages in this section)

- [Quản trị với kubeadm (Administration with kubeadm)](214-kubeadm-tasks-vi.md)
- [Cấp phát dư dung lượng node cho một cluster (Overprovision Node Capacity For A Cluster)](250-node-overprovisioning-vi.md)
- [Di trú khỏi dockershim (Migrating from dockershim)](236-migrating-from-dockershim-vi.md)
- [Tạo certificate thủ công (Generate Certificates Manually)](191-certificates-manual-vi.md) — đã có bản dịch trong repo này
- [Quản lý tài nguyên bộ nhớ, CPU và API (Manage Memory, CPU, and API Resources)](228-manage-resources-tasks-vi.md)
- [Cài đặt một Network Policy Provider (Install a Network Policy Provider)](243-network-policy-provider-vi.md)
- [Truy cập cluster bằng Kubernetes API (Access Clusters Using the Kubernetes API)](190-access-cluster-api-vi.md) — đã có bản dịch trong repo này
- [Bật hoặc tắt các Feature Gate (Enable Or Disable Feature Gates)](196-configure-feature-gates-vi.md)
- [Quảng bá extended resource cho một node (Advertise Extended Resources for a Node)](209-extended-resource-node-vi.md)
- [Tự động co giãn DNS Service trong một cluster (Autoscale the DNS Service in a Cluster)](206-dns-horizontal-autoscaling-vi.md)
- [Đổi access mode của một PersistentVolume sang ReadWriteOncePod (Change the Access Mode of a PersistentVolume to ReadWriteOncePod)](193-change-pv-access-mode-vi.md)
- [Đổi StorageClass mặc định (Change the default StorageClass)](192-change-default-storage-class-vi.md)
- [Chuyển từ polling sang cập nhật trạng thái container dựa trên sự kiện CRI (Switching from Polling to CRI Event-based Updates to Container Status)](257-switch-to-evented-pleg-vi.md)
- [Đổi reclaim policy của một PersistentVolume (Change the Reclaim Policy of a PersistentVolume)](194-change-pv-reclaim-policy-vi.md)
- [Quản trị Cloud Controller Manager (Cloud Controller Manager Administration)](254-running-cloud-controller-vi.md)
- [Cấu hình một kubelet image credential provider (Configure a kubelet image credential provider)](225-kubelet-credential-provider-vi.md)
- [Cấu hình quota cho các API object (Configure Quotas for API Objects)](252-quota-api-object-vi.md)
- [Kiểm soát các chính sách quản lý CPU trên node (Control CPU Management Policies on the Node)](200-cpu-management-policies-vi.md)
- [Kiểm soát các chính sách quản lý bộ nhớ trên một node (Control Memory Management Policies on a Node)](235-memory-manager-vi.md)
- [Kiểm soát các chính sách quản lý topology trên một node (Control Topology Management Policies on a node)](259-topology-manager-vi.md)
- [Tùy chỉnh DNS Service (Customizing DNS Service)](204-dns-custom-nameservers-vi.md)
- [Gỡ lỗi phân giải DNS (Debugging DNS Resolution)](205-dns-debugging-resolution-vi.md)
- [Khai báo Network Policy (Declare Network Policy)](201-declare-network-policy-vi.md)
- [Phát triển Cloud Controller Manager (Developing Cloud Controller Manager)](203-developing-cloud-controller-manager-vi.md)
- [Bật hoặc tắt một Kubernetes API (Enable Or Disable A Kubernetes API)](207-enable-disable-api-vi.md)
- [Mã hóa dữ liệu bí mật khi lưu trữ (Encrypting Confidential Data at Rest)](208-encrypt-data-vi.md)
- [Giải mã dữ liệu bí mật đã được mã hóa khi lưu trữ (Decrypt Confidential Data that is Already Encrypted at Rest)](202-decrypt-data-vi.md)
- [Bảo đảm lập lịch cho các Pod add-on quan trọng (Guaranteed Scheduling For Critical Add-On Pods)](210-guaranteed-scheduling-critical-addon-pods-vi.md)
- [Hướng dẫn sử dụng IP Masquerade Agent (IP Masquerade Agent User Guide)](212-ip-masq-agent-vi.md)
- [Giới hạn mức tiêu thụ lưu trữ (Limit Storage Consumption)](227-limit-storage-consumption-vi.md)
- [Di trú control plane dạng nhân bản sang dùng Cloud Controller Manager (Migrate Replicated Control Plane To Use Cloud Controller Manager)](198-controller-manager-leader-migration-vi.md)
- [Vận hành các cluster etcd cho Kubernetes (Operating etcd clusters for Kubernetes)](197-configure-upgrade-etcd-vi.md)
- [Dành riêng tài nguyên tính toán cho các daemon hệ thống (Reserve Compute Resources for System Daemons)](253-reserve-compute-resources-vi.md)
- [Chạy các thành phần node của Kubernetes bằng người dùng không phải root (Running Kubernetes Node Components as a Non-root User)](226-kubelet-in-userns-vi.md)
- [Drain một node an toàn (Safely Drain a Node)](255-safely-drain-node-vi.md)
- [Bảo mật một cluster (Securing a Cluster)](256-securing-a-cluster-vi.md)
- [Tăng cường bảo mật Dynamic Resource Allocation trong cluster của bạn (Harden Dynamic Resource Allocation in Your Cluster)](211-hardening-dra-tasks-vi.md)
- [Đặt tham số kubelet qua file cấu hình (Set Kubelet Parameters Via A Configuration File)](224-kubelet-config-file-vi.md)
- [Chia sẻ một cluster bằng namespace (Share a Cluster with Namespaces)](242-namespaces-tasks-vi.md)
- [Nâng cấp một cluster (Upgrade A Cluster)](195-cluster-upgrade-vi.md)
- [Dùng xóa theo tầng trong một cluster (Use Cascading Deletion in a Cluster)](260-use-cascading-deletion-vi.md)
- [Dùng một KMS provider để mã hóa dữ liệu (Using a KMS provider for data encryption)](213-kms-provider-vi.md)
- [Dùng CoreDNS cho khám phá dịch vụ (Using CoreDNS for Service Discovery)](199-coredns-vi.md)
- [Dùng NodeLocal DNSCache trong các cluster Kubernetes (Using NodeLocal DNSCache in Kubernetes Clusters)](251-nodelocaldns-vi.md)
- [Dùng sysctl trong một cluster Kubernetes (Using sysctls in a Kubernetes Cluster)](258-sysctl-cluster-vi.md)
- [Xác minh các artifact Kubernetes đã ký (Verify Signed Kubernetes Artifacts)](261-verify-signed-artifacts-vi.md)
