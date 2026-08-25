# Mẫu Operator (Operator pattern)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/extend-kubernetes/operator/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 14](00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng), bài 5/7 ·
Kiểm chứng ở [Lab 14](labs/LAB-14-CRD-VA-OPERATOR.md).

Giai đoạn này lộ trình ghi rõ là **dành cho platform administrator / người phát triển operator**;
đây là bài đích của cả giai đoạn.

Bài này **nối thẳng vào bài [25 — Controller](25-controllers-vi.md)** của giai đoạn 1: operator
tuân theo đúng vòng lặp điều khiển bạn đã học ở đó, chỉ khác là resource nó điều khiển do bạn
định nghĩa (bài [179](179-custom-resources-vi.md)) chứ không phải Deployment hay Job dựng sẵn.
Đọc bài này như phần kết của chuỗi "custom resource + custom controller".

**Phải hiểu ở lần đọc này:**

- Operator là **client của Kubernetes API đóng vai trò controller cho một custom resource**. Nó
  cho phép mở rộng hành vi cluster **mà không cần sửa mã nguồn của chính Kubernetes**.
- Bảy bước trong mục *Một ví dụ về operator*: custom resource `SampleDB`, một Deployment chạy
  Pod chứa controller, image chứa mã, mã truy vấn control plane tìm các resource `SampleDB` đang
  được cấu hình, rồi phần lõi làm cho **thực tế khớp với cấu hình** — tạo PersistentVolumeClaim,
  StatefulSet và Job khi thêm; **tạo snapshot trước** rồi mới xóa StatefulSet và Volume khi xóa.
- Operator ôm cả việc **định kỳ**: sao lưu theo lịch, và kiểm tra phiên bản cũ để tự tạo Job
  nâng cấp. Đó là phần "tri thức của người vận hành" được mã hóa lại.
- Nơi chạy: **controller thường chạy bên ngoài control plane**, như một ứng dụng container hóa
  bình thường — ví dụ một Deployment trong cluster của bạn.
- Cách dùng: không có CLI riêng. Bạn **thêm, sửa hoặc xóa chính custom resource đó** bằng
  `kubectl get SampleDB` và `kubectl edit SampleDB/example-database`; operator lo phần còn lại.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Tự viết operator của riêng bạn* và danh sách framework (kubebuilder, Kopf, kube-rs, Operator Framework…) | là công việc lập trình, chọn khi thực sự viết operator | Lab 14 |
| Danh sách việc có thể tự động hóa ở đầu mục *Một ví dụ về operator* | là cảm hứng, không phải cơ chế | Lab 14 |
| Operator White Paper của CNCF, OperatorHub.io, bài viết gốc của CoreOS | tài liệu tham khảo mở rộng | không cần |

---

Operator là các phần mở rộng phần mềm cho Kubernetes, sử dụng
[custom resource](179-custom-resources-vi.md)
để quản lý các ứng dụng và những thành phần của chúng. Operator tuân theo
các nguyên tắc của Kubernetes, đặc biệt là [vòng lặp điều khiển (control loop)](25-controllers-vi.md).

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
[Custom Resource](179-custom-resources-vi.md).

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
* Tìm hiểu thêm về [Custom Resource](179-custom-resources-vi.md)
* Tìm các operator có sẵn phù hợp với trường hợp sử dụng của bạn trên [OperatorHub.io](https://operatorhub.io/)
* [Công bố](https://operatorhub.io/) operator của bạn để người khác sử dụng
* Đọc [bài viết gốc của CoreOS](https://web.archive.org/web/20170129131616/https://coreos.com/blog/introducing-operators.html)
  đã giới thiệu mẫu operator (đây là bản lưu trữ của bài viết gốc).
* Đọc [bài viết](https://cloud.google.com/blog/products/containers-kubernetes/best-practices-for-building-kubernetes-operators-and-stateful-apps)
  từ Google Cloud về các thực hành tốt nhất khi xây dựng operator

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 14:

1. Bài [25](25-controllers-vi.md) mô tả controller là vòng lặp đưa trạng thái hiện tại về trạng
   thái mong muốn. Operator thêm gì vào đó mà Deployment controller dựng sẵn không có?
2. Câu bẫy: bạn đã cài CRD `SampleDB`, `kubectl get SampleDB` chạy được, `kubectl edit` sửa được
   và object lưu lại đúng — nhưng trong cluster không có gì xảy ra. Thiếu thứ gì, và vì sao
   `kubectl` vẫn hoạt động bình thường dù thiếu nó?
3. Trên cluster lab ba VM của bạn, operator sẽ chạy ở đâu — cạnh control plane trên `lab-k8s-master`
   dưới dạng thành phần đặc biệt, hay như một workload thường trên worker? Bài nói gì?
4. Trong ví dụ `SampleDB`, khi bạn xóa một resource `SampleDB` thì operator làm những gì, và
   theo thứ tự nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Operator thêm **custom resource của riêng bạn** và **tri thức nghiệp vụ về một ứng dụng cụ
   thể**. Bài định nghĩa: mẫu operator "cho phép bạn mở rộng hành vi của cluster mà không cần
   sửa mã nguồn của chính Kubernetes, bằng cách **liên kết các controller với một hoặc nhiều
   custom resource**". Deployment controller cũng là vòng lặp điều khiển, nhưng nó chỉ biết một
   kind dựng sẵn và không biết gì về "nâng cấp schema cơ sở dữ liệu" hay "tạo snapshot trước khi
   xóa". Điểm giống nhau: **cả hai đều chỉ là client của Kubernetes API**.
2. Thiếu **controller** — tức thiếu chính phần "operator". Custom resource chỉ là nơi lưu trạng
   thái mong muốn; `kubectl` vẫn hoạt động vì API server phục vụ và lưu trữ CRD y như mọi
   resource khác, hoàn toàn không cần biết có ai đang lắng nghe hay không. Bài nói cách triển
   khai phổ biến nhất là "thêm **Custom Resource Definition và Controller tương ứng** của nó vào
   cluster" — hai vế, không phải một. Đây đúng là điều mà checkpoint của giai đoạn 14 trong
   [lộ trình](00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng) bắt bạn giải thích được.
3. **Như một workload thường.** Bài viết: "Controller thường chạy **bên ngoài control plane**,
   giống như cách bạn chạy bất kỳ ứng dụng container hóa nào. Ví dụ, bạn có thể chạy controller
   trong cluster của mình dưới dạng một **Deployment**." Trong ví dụ, chính bước 2 là "một
   Deployment đảm bảo có một Pod đang chạy chứa phần controller của operator". Operator không
   phải thành phần control plane và không cần đặc quyền của control plane node.
4. **Tạo một snapshot trước**, sau đó mới đảm bảo StatefulSet và các Volume bị xóa theo. Thứ tự
   này chính là "tri thức của người vận hành" được mã hóa: một người vận hành cơ sở dữ liệu có
   kinh nghiệm sẽ không xóa volume trước khi có bản sao lưu. Chiều ngược lại, khi bạn **thêm**
   một `SampleDB`, operator thiết lập PersistentVolumeClaim để cấp lưu trữ bền vững, một
   StatefulSet để chạy SampleDB, và một Job để xử lý cấu hình ban đầu.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
