# Chuyển từ polling sang cập nhật trạng thái container dựa trên sự kiện CRI (Switching from Polling to CRI Event-based Updates to Container Status)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/switch-to-evented-pleg/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.26 [alpha]`

Trang này hướng dẫn cách chuyển các node sang sử dụng cơ chế cập nhật trạng thái container dựa trên
sự kiện (event-based updates). Cách triển khai dựa trên sự kiện giúp giảm mức tiêu thụ tài nguyên
node của kubelet, so với cách tiếp cận cũ dựa vào polling (thăm dò định kỳ).
Bạn có thể biết đến tính năng này với tên gọi _evented Pod lifecycle event generator (PLEG)_ —
bộ sinh sự kiện vòng đời Pod theo cơ chế sự kiện. Đó là tên được dùng nội bộ trong dự án Kubernetes
cho một chi tiết triển khai quan trọng.

Cách tiếp cận dựa trên polling được gọi là _generic PLEG_.

## Trước khi bạn bắt đầu (Before you begin)

* Bạn cần chạy một phiên bản Kubernetes có cung cấp tính năng này.
  Kubernetes v1.27 bao gồm hỗ trợ beta cho cập nhật trạng thái container
  dựa trên sự kiện. Tính năng này ở trạng thái beta nhưng bị _tắt_ theo mặc định
  vì nó cần sự hỗ trợ từ phía container runtime.
* Phiên bản Kubernetes server của bạn phải bằng hoặc mới hơn phiên bản 1.26.
  Để kiểm tra phiên bản, nhập `kubectl version`.
  Nếu bạn đang chạy một phiên bản Kubernetes khác, hãy xem tài liệu của bản phát hành đó.
* Container runtime đang dùng phải hỗ trợ các sự kiện vòng đời container (container lifecycle
  events). Kubelet sẽ tự động chuyển về cơ chế generic PLEG cũ nếu container runtime
  không công bố hỗ trợ các sự kiện vòng đời container, ngay cả khi bạn đã bật
  feature gate này.

## Tại sao nên chuyển sang Evented PLEG? (Why switch to Evented PLEG?)

* _Generic PLEG_ gây ra chi phí (overhead) không hề nhỏ do việc polling trạng thái container diễn
  ra thường xuyên.
* Chi phí này càng trầm trọng hơn do kubelet polling song song trạng thái của các container, từ đó
  hạn chế khả năng mở rộng (scalability) của nó và gây ra các vấn đề về hiệu năng kém cũng như độ
  tin cậy.
* Mục tiêu của _Evented PLEG_ là giảm bớt công việc không cần thiết trong thời gian không có hoạt
  động, bằng cách thay thế việc polling định kỳ.

## Chuyển sang Evented PLEG (Switching to Evented PLEG)

1. Khởi động kubelet với [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
   `EventedPLEG` được bật. Bạn có thể quản lý các feature gate của kubelet bằng cách sửa
   [file cấu hình](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/) của kubelet
   và khởi động lại dịch vụ kubelet. Bạn cần làm việc này trên từng node mà bạn muốn sử dụng
   tính năng này.

2. Đảm bảo node đã được [drain](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)
   trước khi tiếp tục.

3. Khởi động container runtime với tính năng sinh sự kiện container (container event generation)
   được bật.

   #### Containerd

   Phiên bản 1.7+

   #### CRI-O

   Phiên bản 1.26+

   Kiểm tra xem CRI-O đã được cấu hình để phát ra các sự kiện CRI hay chưa bằng cách xác minh
   cấu hình,

   ```shell
   crio config | grep enable_pod_events
   ```

   Nếu đã được bật, output sẽ tương tự như sau:

   ```none
   enable_pod_events = true
   ```

   Để bật nó, khởi động CRI-O daemon với flag `--enable-pod-events=true` hoặc
   dùng một file cấu hình drop-in với các dòng sau:

   ```toml
   [crio.runtime]
   enable_pod_events: true
   ```

   Phiên bản Kubernetes server của bạn phải bằng hoặc mới hơn phiên bản 1.26.
   Để kiểm tra phiên bản, nhập `kubectl version`.

4. Xác minh rằng kubelet đang sử dụng cơ chế giám sát thay đổi trạng thái container dựa trên
   sự kiện. Để kiểm tra, hãy tìm cụm từ `EventedPLEG` trong log của kubelet.

   Output sẽ tương tự như sau:

   ```console
   I0314 11:10:13.909915 1105457 feature_gate.go:249] feature gates: &{map[EventedPLEG:true]}
   ```

   Nếu bạn đã đặt `--v` từ 4 trở lên, bạn có thể thấy thêm các mục cho biết
   kubelet đang dùng cơ chế giám sát trạng thái container dựa trên sự kiện.

   ```console
   I0314 11:12:42.009542 1110177 evented.go:238] "Evented PLEG: Generated pod status from the received event" podUID=3b2c6172-b112-447a-ba96-94e7022912dc
   I0314 11:12:44.623326 1110177 evented.go:238] "Evented PLEG: Generated pod status from the received event" podUID=b3fba5ea-a8c5-4b76-8f43-481e17e8ec40
   I0314 11:12:44.714564 1110177 evented.go:238] "Evented PLEG: Generated pod status from the received event" podUID=b3fba5ea-a8c5-4b76-8f43-481e17e8ec40
   ```

## Tiếp theo (What's next)

* Tìm hiểu thêm về thiết kế này trong Kubernetes Enhancement Proposal (KEP):
  [Kubelet Evented PLEG for Better Performance](https://github.com/kubernetes/enhancements/blob/5b258a990adabc2ffdc9d84581ea6ed696f7ce6c/keps/sig-node/3386-kubelet-evented-pleg/README.md).
