# Chuyển đổi từ dockershim (Migrating from dockershim)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/>

Phần này trình bày những thông tin bạn cần biết khi chuyển đổi (migrate) từ dockershim sang các
container runtime khác.

Kể từ thông báo về việc
[ngừng hỗ trợ dockershim (dockershim deprecation)](https://kubernetes.io/blog/2020/12/08/kubernetes-1-20-release-announcement/#dockershim-deprecation)
trong Kubernetes 1.20, đã có nhiều câu hỏi về việc điều này sẽ ảnh hưởng như thế nào tới các
workload và các bản cài đặt Kubernetes khác nhau. Bài
[Dockershim Removal FAQ](https://kubernetes.io/blog/2022/02/17/dockershim-faq/) sẽ giúp bạn
hiểu rõ hơn về vấn đề này.

Dockershim đã bị loại bỏ khỏi Kubernetes cùng với bản phát hành v1.24.
Nếu bạn đang dùng Docker Engine thông qua dockershim làm container runtime và muốn nâng cấp lên
v1.24, bạn nên chuyển sang một runtime khác hoặc tìm một phương án thay thế để có được sự hỗ trợ
cho Docker Engine. Hãy xem mục
[container runtimes](00-container-runtimes-vi.md)
để biết các lựa chọn của bạn.

Phiên bản Kubernetes còn có dockershim (1.23) đã hết hạn hỗ trợ và v1.24 cũng sẽ
[sớm](https://kubernetes.io/releases/#release-v1-24) hết hạn hỗ trợ. Hãy nhớ
[báo cáo các vấn đề (issue)](https://github.com/kubernetes/kubernetes/issues) mà bạn gặp phải
trong quá trình chuyển đổi, để các vấn đề đó có thể được sửa kịp thời và cluster của bạn sẵn
sàng cho việc loại bỏ dockershim. Sau khi v1.24 hết hạn hỗ trợ, bạn sẽ phải liên hệ nhà cung cấp
Kubernetes của mình để được hỗ trợ, hoặc nâng cấp nhiều phiên bản cùng lúc nếu có các vấn đề
nghiêm trọng ảnh hưởng tới cluster của bạn.

Cluster của bạn có thể có nhiều hơn một loại node, mặc dù đây không phải là cấu hình phổ biến.

Các tác vụ sau sẽ giúp bạn thực hiện việc chuyển đổi:

* [Kiểm tra xem việc loại bỏ Dockershim có ảnh hưởng tới bạn không](238-check-dockershim-removal-vi.md)
* [Chuyển đổi các agent telemetry và bảo mật khỏi dockershim](240-migrating-telemetry-agents-vi.md)

## Tiếp theo (What's next)

* Xem mục [container runtimes](00-container-runtimes-vi.md)
  để hiểu các lựa chọn thay thế của bạn.
* Nếu bạn phát hiện lỗi hoặc vấn đề kỹ thuật khác liên quan tới việc chuyển đổi khỏi dockershim,
  bạn có thể [báo cáo issue](https://github.com/kubernetes/kubernetes/issues/new/choose)
  cho dự án Kubernetes.
