# Mẫu Operator (Operator pattern)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/extend-kubernetes/operator/>

Operator là các phần mở rộng phần mềm cho Kubernetes, sử dụng
[custom resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
để quản lý các ứng dụng và những thành phần của chúng. Operator tuân theo
các nguyên tắc của Kubernetes, đặc biệt là [vòng lặp điều khiển (control loop)](https://kubernetes.io/docs/concepts/architecture/controller).

## Động lực (Motivation)

_Mẫu operator_ hướng tới việc nắm bắt mục tiêu cốt lõi của một người vận hành (operator) đang
quản lý một dịch vụ hoặc một tập hợp các dịch vụ. Những người vận hành phụ trách các ứng dụng và
dịch vụ cụ thể có hiểu biết sâu về cách hệ thống nên hoạt động, cách triển khai nó, và cách phản
ứng khi có sự cố.

Những người chạy workload trên Kubernetes thường thích dùng tự động hóa để xử lý các tác vụ lặp
đi lặp lại. Mẫu operator nắm bắt cách bạn có thể viết mã để tự động hóa một tác vụ vượt ra ngoài
những gì bản thân Kubernetes cung cấp.

## Operator trong Kubernetes (Operators in Kubernetes)

Kubernetes được thiết kế cho tự động hóa. Ngay khi cài đặt xong, bạn đã có sẵn rất nhiều cơ chế
tự động hóa tích hợp từ phần lõi của Kubernetes. Bạn có thể dùng Kubernetes để tự động hóa việc
triển khai và chạy workload, *và* bạn cũng có thể tự động hóa cách Kubernetes làm điều đó.

Khái niệm mẫu operator (operator pattern) của Kubernetes cho phép bạn mở rộng hành vi của cluster
mà không cần sửa mã nguồn của chính Kubernetes, bằng cách liên kết các controller với một hoặc
nhiều custom resource. Operator là các client của Kubernetes API, đóng vai trò controller cho một
[Custom Resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/).

## Một ví dụ về operator (An example operator) {#example}

Một số việc bạn có thể dùng operator để tự động hóa bao gồm:

* triển khai một ứng dụng theo yêu cầu
* tạo và khôi phục bản sao lưu (backup) trạng thái của ứng dụng đó
* xử lý việc nâng cấp mã ứng dụng cùng với các thay đổi liên quan như
  schema cơ sở dữ liệu hoặc các thiết lập cấu hình bổ sung
* công bố một Service cho các ứng dụng không hỗ trợ Kubernetes API để chúng có thể
  khám phá (discover) được các Service đó
* mô phỏng sự cố ở toàn bộ hoặc một phần cluster để kiểm thử khả năng phục hồi (resilience) của nó
* chọn ra một leader cho một ứng dụng phân tán mà không cần quy trình bầu chọn nội bộ giữa các
  thành viên

Một operator trông chi tiết hơn thì như thế nào? Đây là một ví dụ:

1. Một custom resource tên là SampleDB, mà bạn có thể cấu hình vào cluster.
2. Một Deployment đảm bảo có một Pod đang chạy chứa phần controller của operator.
3. Một container image chứa mã của operator.
4. Mã controller truy vấn control plane để tìm ra những resource SampleDB nào đang được cấu hình.
5. Phần lõi của operator là mã chỉ dẫn cho API server cách làm cho thực tế khớp với các resource
   đã được cấu hình.
   * Nếu bạn thêm một SampleDB mới, operator thiết lập các PersistentVolumeClaim để cung cấp
     lưu trữ bền vững cho cơ sở dữ liệu, một StatefulSet để chạy SampleDB và một Job để xử lý
     cấu hình ban đầu.
   * Nếu bạn xóa nó, operator tạo một snapshot, sau đó đảm bảo StatefulSet và các Volume cũng
     bị xóa theo.
6. Operator cũng quản lý việc sao lưu cơ sở dữ liệu định kỳ. Với mỗi resource SampleDB, operator
   xác định thời điểm cần tạo một Pod có thể kết nối tới cơ sở dữ liệu và thực hiện sao lưu.
   Những Pod này sẽ dựa vào một ConfigMap và/hoặc một Secret chứa thông tin kết nối và thông tin
   xác thực (credential) của cơ sở dữ liệu.
7. Vì operator hướng tới việc cung cấp cơ chế tự động hóa vững chắc cho resource mà nó quản lý,
   sẽ còn có thêm mã hỗ trợ khác. Với ví dụ này, mã sẽ kiểm tra xem cơ sở dữ liệu có đang chạy
   phiên bản cũ hay không, và nếu có thì tạo các đối tượng Job để nâng cấp nó giúp bạn.

## Triển khai operator (Deploying operators)

Cách phổ biến nhất để triển khai một operator là thêm Custom Resource Definition và Controller
tương ứng của nó vào cluster của bạn. Controller thường chạy bên ngoài control plane,
giống như cách bạn chạy bất kỳ ứng dụng container hóa nào. Ví dụ, bạn có thể chạy controller
trong cluster của mình dưới dạng một Deployment.

## Sử dụng một operator (Using an operator) {#using-operators}

Khi đã triển khai xong một operator, bạn sử dụng nó bằng cách thêm, sửa đổi hoặc xóa loại
resource mà operator đó dùng. Theo ví dụ ở trên, bạn sẽ thiết lập một Deployment cho chính
operator, rồi:

```shell
kubectl get SampleDB                   # tìm các cơ sở dữ liệu đã được cấu hình

kubectl edit SampleDB/example-database # thay đổi thủ công một số thiết lập
```

…và thế là xong! Operator sẽ lo việc áp dụng các thay đổi cũng như giữ cho dịch vụ hiện có
luôn ở trạng thái tốt.

## Tự viết operator của riêng bạn (Writing your own operator) {#writing-operator}

Nếu trong hệ sinh thái không có operator nào hiện thực hành vi bạn muốn, bạn có thể tự viết mã
cho nó.

Bạn cũng có thể hiện thực một operator (tức là một Controller) bằng bất kỳ ngôn ngữ/runtime nào
có thể hoạt động như một [client cho Kubernetes API](https://kubernetes.io/docs/reference/using-api/client-libraries/).

Dưới đây là một vài thư viện và công cụ bạn có thể dùng để tự viết operator cloud native của mình.

> **Ghi chú:** Mục này liên kết đến các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án này, vốn được liệt kê theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc [hướng dẫn về nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content) trước khi gửi thay đổi.

* [Charmed Operator Framework](https://juju.is/)
* [Java Operator SDK](https://github.com/operator-framework/java-operator-sdk)
* [Kopf](https://github.com/nolar/kopf) (Kubernetes Operator Pythonic Framework)
* [kube-rs](https://kube.rs/) (Rust)
* [kubebuilder](https://book.kubebuilder.io/)
* [KubeOps](https://dotnet.github.io/dotnet-operator-sdk/) (.NET operator SDK)
* [Mast](https://docs.ansi.services/mast/user_guide/operator/)
* [Metacontroller](https://metacontroller.github.io/metacontroller/intro.html) kết hợp với các WebHook
  do bạn tự hiện thực
* [Operator Framework](https://operatorframework.io)
* [shell-operator](https://github.com/flant/shell-operator)

## Tiếp theo (What's next)

* Đọc [Operator White Paper](https://github.com/cncf/tag-app-delivery/blob/163962c4b1cd70d085107fc579e3e04c2e14d59c/operator-wg/whitepaper/Operator-WhitePaper_v1-0.md)
  của CNCF.
* Tìm hiểu thêm về [Custom Resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
* Tìm các operator có sẵn phù hợp với trường hợp sử dụng của bạn trên [OperatorHub.io](https://operatorhub.io/)
* [Công bố](https://operatorhub.io/) operator của bạn để người khác sử dụng
* Đọc [bài viết gốc của CoreOS](https://web.archive.org/web/20170129131616/https://coreos.com/blog/introducing-operators.html)
  đã giới thiệu mẫu operator (đây là bản lưu trữ của bài viết gốc).
* Đọc [bài viết](https://cloud.google.com/blog/products/containers-kubernetes/best-practices-for-building-kubernetes-operators-and-stateful-apps)
  từ Google Cloud về các thực hành tốt nhất khi xây dựng operator
