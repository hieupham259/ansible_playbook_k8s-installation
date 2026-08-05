# Tầng tổng hợp API của Kubernetes (Kubernetes API Aggregation Layer)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/>
>
> Tầng tổng hợp (aggregation layer) cho phép mở rộng Kubernetes bằng các API bổ sung, vượt ra ngoài những gì các API lõi của Kubernetes cung cấp.

Tầng tổng hợp (aggregation layer) cho phép mở rộng Kubernetes bằng các API bổ sung, vượt ra ngoài những gì các API lõi của Kubernetes cung cấp. Các API bổ sung này có thể là những giải pháp có sẵn như [metrics server](https://github.com/kubernetes-sigs/metrics-server), hoặc là các API do chính bạn phát triển.

Tầng tổng hợp khác với [CustomResourceDefinition](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) — vốn là cách để giúp kube-apiserver nhận biết các loại (kind) đối tượng mới.

## Tầng tổng hợp (Aggregation layer)

Tầng tổng hợp chạy trong cùng tiến trình (in-process) với kube-apiserver. Cho tới khi có một extension resource được đăng ký, tầng tổng hợp sẽ không làm gì cả. Để đăng ký một API, bạn thêm một đối tượng _APIService_ — đối tượng này "chiếm giữ" (claim) một đường dẫn URL trong Kubernetes API. Kể từ thời điểm đó, tầng tổng hợp sẽ chuyển tiếp (proxy) mọi thứ được gửi tới đường dẫn API đó (ví dụ `/apis/myextension.mycompany.io/v1/…`) tới APIService đã đăng ký.

Cách phổ biến nhất để hiện thực một APIService là chạy một *extension API server* trong các Pod chạy bên trong cluster của bạn. Nếu bạn dùng extension API server để quản lý các resource trong cluster, thì extension API server (còn được viết là "extension-apiserver") thường đi kèm với một hoặc nhiều controller. Thư viện apiserver-builder cung cấp bộ khung (skeleton) cho cả extension API server lẫn các controller đi kèm.

### Độ trễ phản hồi (Response latency)

Extension API server nên có kết nối mạng độ trễ thấp tới và từ kube-apiserver. Các request discovery bắt buộc phải hoàn tất một vòng khứ hồi (round-trip) từ kube-apiserver trong vòng năm giây hoặc ít hơn.

Nếu extension API server của bạn không đạt được yêu cầu độ trễ đó, hãy cân nhắc thực hiện các thay đổi để đáp ứng được yêu cầu này.

## Tiếp theo (What's next)

* Để bộ tổng hợp (aggregator) hoạt động trong môi trường của bạn, hãy [cấu hình tầng tổng hợp](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-aggregation-layer/).
* Sau đó, [thiết lập một extension api-server](https://kubernetes.io/docs/tasks/extend-kubernetes/setup-extension-api-server/) để làm việc với tầng tổng hợp.
* Đọc về [APIService](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/api-service-v1/) trong tài liệu tham chiếu API.
* Tìm hiểu về [Khái niệm Declarative Validation](https://kubernetes.io/docs/reference/using-api/declarative-validation/), một cơ chế nội bộ để định nghĩa các luật kiểm tra hợp lệ (validation) mà trong tương lai sẽ hỗ trợ việc kiểm tra hợp lệ cho quá trình phát triển extension API server.

Ngoài ra: tìm hiểu cách mở rộng Kubernetes API bằng [CustomResourceDefinition](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/).
