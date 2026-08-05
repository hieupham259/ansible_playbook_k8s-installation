# Giám sát tình trạng volume (Volume Health Monitoring)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/storage/volume-health-monitoring/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.21 [alpha]`

Tính năng giám sát tình trạng volume của CSI cho phép các CSI Driver
phát hiện các điều kiện bất thường của volume từ hệ thống lưu trữ bên
dưới và báo cáo chúng dưới dạng event trên các PVC hoặc Pod.

## Giám sát tình trạng volume (Volume health monitoring)

*Giám sát tình trạng volume* (volume health monitoring) trong Kubernetes
là một phần trong cách Kubernetes hiện thực Container Storage Interface
(CSI). Tính năng giám sát tình trạng volume được hiện thực trong hai
thành phần: một controller giám sát tình trạng bên ngoài (External
Health Monitor controller), và kubelet.

Nếu một CSI Driver hỗ trợ tính năng Giám sát tình trạng volume từ phía
controller, một event sẽ được báo cáo trên PersistentVolumeClaim (PVC)
liên quan khi một điều kiện bất thường của volume được phát hiện trên
một CSI volume.

Controller External Health Monitor cũng theo dõi các event lỗi node
(node failure). Bạn có thể bật giám sát lỗi node bằng cách đặt cờ
`enable-node-watcher` là true. Khi external health monitor phát hiện
một event lỗi node, controller sẽ báo cáo một Event trên PVC để cho
biết các Pod đang dùng PVC này nằm trên một node bị lỗi.

Nếu một CSI Driver hỗ trợ tính năng Giám sát tình trạng volume từ phía
node, một Event sẽ được báo cáo trên mọi Pod đang dùng PVC khi một điều
kiện bất thường của volume được phát hiện trên một CSI volume. Ngoài
ra, thông tin tình trạng volume được cung cấp dưới dạng các metric
VolumeStats của Kubelet. Một metric mới
kubelet_volume_stats_health_status_abnormal được thêm vào. Metric này
gồm hai nhãn (label): `namespace` và `persistentvolumeclaim`. Giá trị
đếm là 1 hoặc 0. 1 cho biết volume không khỏe mạnh, 0 cho biết volume
khỏe mạnh. Để biết thêm thông tin, hãy xem
[KEP](https://github.com/kubernetes/enhancements/tree/master/keps/sig-storage/1432-volume-health-monitor#kubelet-metrics-changes).

> **Ghi chú:**
> Bạn cần bật [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
> `CSIVolumeHealth` để dùng tính năng này từ phía node.

## Tiếp theo (What's next)

Xem [tài liệu về CSI driver](https://kubernetes-csi.github.io/docs/drivers.html)
để biết những CSI driver nào đã hiện thực tính năng này.
