# Cấu hình DNS cho một cluster (Configure DNS for a Cluster)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/access-application-cluster/configure-dns-cluster/>
>
> Trang này giới thiệu addon DNS cho cluster của Kubernetes và nơi tìm hướng dẫn
> cấu hình chi tiết.

Kubernetes cung cấp một addon DNS cho cluster, mà hầu hết các môi trường được hỗ trợ đều
bật mặc định. Từ Kubernetes phiên bản 1.11 trở đi, CoreDNS là lựa chọn được khuyến nghị
và được cài đặt mặc định cùng với kubeadm.

Để biết thêm thông tin về cách cấu hình CoreDNS cho một cluster Kubernetes, xem
[Tùy chỉnh DNS Service](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/)
(đã có [bản dịch tiếng Việt](204-dns-custom-nameservers-vi.md)).
