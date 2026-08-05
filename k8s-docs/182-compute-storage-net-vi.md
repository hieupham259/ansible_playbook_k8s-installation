# Các phần mở rộng về Tính toán, Lưu trữ và Mạng (Compute, Storage, and Networking Extensions)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/>

Phần này trình bày về các phần mở rộng cho cluster của bạn mà không đi kèm sẵn trong bản thân
Kubernetes. Bạn có thể dùng những phần mở rộng này để tăng cường khả năng cho các node trong
cluster, hoặc để cung cấp hạ tầng mạng (network fabric) kết nối các Pod với nhau.

* Các storage plugin [CSI](https://kubernetes.io/docs/concepts/storage/volumes/#csi) và [FlexVolume](https://kubernetes.io/docs/concepts/storage/volumes/#flexvolume)

  Các plugin Container Storage Interface (CSI) cung cấp một cách để mở rộng Kubernetes với khả
  năng hỗ trợ các loại volume mới. Các volume này có thể được hậu thuẫn bởi hệ thống lưu trữ
  ngoài bền vững, hoặc cung cấp lưu trữ tạm thời (ephemeral), hoặc chúng có thể cung cấp một
  giao diện chỉ đọc tới thông tin theo mô hình hệ thống tập tin (filesystem).

  Kubernetes cũng bao gồm hỗ trợ cho các plugin [FlexVolume](https://kubernetes.io/docs/concepts/storage/volumes/#flexvolume),
  vốn đã bị deprecated (không khuyến khích dùng) từ Kubernetes v1.23 (thay bằng CSI).

  Các plugin FlexVolume cho phép người dùng mount những loại volume mà Kubernetes không hỗ trợ
  sẵn. Khi bạn chạy một Pod dựa vào lưu trữ FlexVolume, kubelet gọi một plugin dạng binary để
  mount volume đó. Bản đề xuất thiết kế [FlexVolume](https://git.k8s.io/design-proposals-archive/storage/flexvolume-deployment.md)
  đã được lưu trữ có thêm chi tiết về cách tiếp cận này.

  Tài liệu [Kubernetes Volume Plugin FAQ for Storage Vendors](https://github.com/kubernetes/community/blob/main/sig-storage/volume-plugin-faq.md#kubernetes-volume-plugin-faq-for-storage-vendors)
  bao gồm thông tin tổng quát về các storage plugin.

* [Device plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)

  Device plugin cho phép một node khám phá (discover) các tiện ích mới của Node (bổ sung cho các
  resource có sẵn của node như `cpu` và `memory`), và cung cấp những tiện ích cục bộ trên node
  tùy chỉnh này cho các Pod có yêu cầu chúng.

* [Network plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)

  Network plugin cho phép Kubernetes làm việc với các topology và công nghệ mạng khác nhau.
  Cluster Kubernetes của bạn cần một _network plugin_ để có được một mạng Pod hoạt động được
  và để hỗ trợ các khía cạnh khác của mô hình mạng Kubernetes.

  Kubernetes v1.36 tương thích với các network plugin CNI.
