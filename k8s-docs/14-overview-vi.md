# Tổng quan (Overview)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/overview/>
>
> Kubernetes là một nền tảng mã nguồn mở, có tính di động (portable) và khả năng mở rộng (extensible), dùng để quản lý
> các workload và service chạy trong container, hỗ trợ cả cấu hình khai báo (declarative configuration) lẫn tự động hóa.
> Kubernetes có một hệ sinh thái lớn và đang phát triển nhanh chóng. Các dịch vụ, sự hỗ trợ và công cụ dành cho
> Kubernetes được cung cấp rộng rãi.

Trang này là phần tổng quan về Kubernetes.

Cái tên Kubernetes bắt nguồn từ tiếng Hy Lạp, có nghĩa là người cầm lái (helmsman) hoặc hoa tiêu (pilot).
Tên viết tắt K8s có được từ việc đếm tám chữ cái nằm giữa chữ "K" và chữ "s". Google đã mở mã nguồn
dự án Kubernetes vào năm 2014. Kubernetes kết hợp
[hơn 15 năm kinh nghiệm của Google](https://kubernetes.io/blog/2015/04/borg-predecessor-to-kubernetes/) trong việc
vận hành workload production ở quy mô lớn với những ý tưởng và thực tiễn tốt nhất từ cộng đồng.

## Vì sao bạn cần Kubernetes và Kubernetes có thể làm gì (Why you need Kubernetes and what it can do) {#why-you-need-kubernetes-and-what-can-it-do}

Container là một cách tốt để đóng gói và chạy các ứng dụng của bạn. Trong môi trường production,
bạn cần quản lý các container đang chạy ứng dụng và bảo đảm không có thời gian ngừng hoạt động (downtime).
Ví dụ, nếu một container bị dừng, một container khác cần được khởi động thay thế.
Sẽ dễ dàng hơn biết bao nếu hành vi này được một hệ thống tự động xử lý?

Đó chính là lúc Kubernetes xuất hiện để giải cứu! Kubernetes cung cấp cho bạn một framework
để chạy các hệ thống phân tán một cách bền bỉ (resilient). Nó lo việc mở rộng (scaling) và
chuyển đổi dự phòng (failover) cho ứng dụng của bạn, cung cấp các mẫu triển khai (deployment pattern),
và nhiều hơn nữa. Ví dụ: Kubernetes có thể dễ dàng quản lý một đợt triển khai canary (canary deployment)
cho hệ thống của bạn.

Kubernetes cung cấp cho bạn:

* **Khám phá dịch vụ và cân bằng tải (Service discovery and load balancing)**
  Kubernetes có thể expose một container thông qua tên DNS hoặc địa chỉ IP riêng của container đó.
  Nếu lưu lượng đến một container cao, Kubernetes có thể cân bằng tải và phân phối
  lưu lượng mạng để việc triển khai được ổn định.
* **Điều phối lưu trữ (Storage orchestration)**
  Kubernetes cho phép bạn tự động mount một hệ thống lưu trữ tùy chọn, chẳng hạn như
  lưu trữ cục bộ, các nhà cung cấp cloud công cộng, và nhiều loại khác.
* **Tự động rollout và rollback (Automated rollouts and rollbacks)**
  Bạn có thể mô tả trạng thái mong muốn (desired state) cho các container đã triển khai bằng Kubernetes,
  và Kubernetes có thể thay đổi trạng thái thực tế về trạng thái mong muốn theo một tốc độ được kiểm soát.
  Ví dụ, bạn có thể tự động hóa để Kubernetes tạo các container mới cho đợt triển khai của bạn,
  xóa các container hiện có và chuyển toàn bộ tài nguyên của chúng sang container mới.
* **Tự động sắp xếp tối ưu (Automatic bin packing)**
  Bạn cung cấp cho Kubernetes một cluster gồm các node mà nó có thể dùng để chạy các tác vụ đóng gói trong container.
  Bạn cho Kubernetes biết mỗi container cần bao nhiêu CPU và bộ nhớ (RAM). Kubernetes có thể
  sắp xếp các container lên các node của bạn sao cho tận dụng tài nguyên một cách tốt nhất.
* **Tự phục hồi (Self-healing)**
  Kubernetes khởi động lại các container bị lỗi, thay thế container, dừng (kill) các container
  không phản hồi health check do bạn định nghĩa, và không quảng bá chúng tới client
  cho đến khi chúng sẵn sàng phục vụ.
* **Quản lý secret và cấu hình (Secret and configuration management)**
  Kubernetes cho phép bạn lưu trữ và quản lý thông tin nhạy cảm, chẳng hạn như mật khẩu, OAuth token
  và khóa SSH. Bạn có thể triển khai và cập nhật secret cũng như cấu hình ứng dụng mà không cần
  build lại container image, và không để lộ secret trong cấu hình hệ thống (stack configuration) của bạn.
* **Thực thi tác vụ theo lô (Batch execution)**
  Ngoài các service, Kubernetes còn có thể quản lý các workload dạng batch và CI, thay thế các container bị lỗi nếu muốn.
* **Mở rộng theo chiều ngang (Horizontal scaling)**
  Mở rộng hoặc thu hẹp ứng dụng của bạn bằng một lệnh đơn giản, qua giao diện người dùng (UI), hoặc tự động dựa trên mức sử dụng CPU.
* **Dual-stack IPv4/IPv6**
  Cấp phát địa chỉ IPv4 và IPv6 cho các Pod và Service.
* **Được thiết kế cho khả năng mở rộng (Designed for extensibility)**
  Thêm tính năng vào cluster Kubernetes của bạn mà không cần thay đổi mã nguồn upstream.

## Kubernetes không phải là gì (What Kubernetes is not)

Kubernetes không phải là một hệ thống PaaS (Platform as a Service — nền tảng như một dịch vụ) truyền thống,
bao trọn tất cả. Vì Kubernetes hoạt động ở cấp độ container thay vì cấp độ phần cứng,
nó cung cấp một số tính năng áp dụng chung thường thấy ở các sản phẩm PaaS, chẳng hạn như
triển khai, mở rộng, cân bằng tải, và cho phép người dùng tích hợp các giải pháp logging,
giám sát (monitoring) và cảnh báo (alerting) của riêng họ. Tuy nhiên, Kubernetes không phải là một khối
nguyên khối (monolithic), và các giải pháp mặc định này là tùy chọn và có thể lắp ghép thay thế (pluggable).
Kubernetes cung cấp các khối xây dựng (building block) để dựng nên các nền tảng cho nhà phát triển,
nhưng vẫn bảo toàn quyền lựa chọn và sự linh hoạt của người dùng ở những chỗ quan trọng.

Kubernetes:

* Không giới hạn các loại ứng dụng được hỗ trợ. Kubernetes hướng đến việc hỗ trợ vô cùng
  đa dạng các loại workload, bao gồm workload stateless (không trạng thái), stateful (có trạng thái)
  và xử lý dữ liệu. Nếu một ứng dụng có thể chạy trong container, nó hẳn sẽ chạy tốt trên Kubernetes.
* Không triển khai mã nguồn và không build ứng dụng của bạn. Các quy trình Tích hợp,
  Chuyển giao và Triển khai liên tục (CI/CD) được quyết định bởi văn hóa và sở thích của
  từng tổ chức cũng như các yêu cầu kỹ thuật.
* Không cung cấp các dịch vụ ở tầng ứng dụng, chẳng hạn như middleware (ví dụ: message bus),
  các framework xử lý dữ liệu (ví dụ: Spark), cơ sở dữ liệu (ví dụ: MySQL), cache, hay
  các hệ thống lưu trữ cluster (ví dụ: Ceph) dưới dạng dịch vụ tích hợp sẵn. Những thành phần như vậy
  có thể chạy trên Kubernetes, và/hoặc có thể được các ứng dụng chạy trên Kubernetes truy cập
  thông qua các cơ chế di động, chẳng hạn như [Open Service Broker](https://openservicebrokerapi.org/).
* Không áp đặt giải pháp logging, giám sát hay cảnh báo. Nó cung cấp một số tích hợp
  mang tính minh chứng khái niệm (proof of concept), và các cơ chế để thu thập và xuất (export) các chỉ số (metrics).
* Không cung cấp cũng không bắt buộc một ngôn ngữ/hệ thống cấu hình nào (ví dụ: Jsonnet). Nó cung cấp
  một API khai báo mà mọi hình thức đặc tả khai báo tùy ý đều có thể nhắm tới.
* Không cung cấp cũng không áp dụng bất kỳ hệ thống toàn diện nào về cấu hình máy, bảo trì,
  quản lý hay tự phục hồi máy.
* Ngoài ra, Kubernetes không chỉ đơn thuần là một hệ thống điều phối (orchestration). Thực tế, nó loại bỏ
  nhu cầu điều phối. Định nghĩa kỹ thuật của điều phối là thực thi một quy trình (workflow) được định nghĩa sẵn:
  đầu tiên làm A, rồi đến B, rồi đến C. Ngược lại, Kubernetes bao gồm một tập các tiến trình điều khiển
  (control process) độc lập, có thể kết hợp với nhau, liên tục đưa trạng thái hiện tại về trạng thái
  mong muốn đã được cung cấp. Việc bạn đi từ A đến C bằng cách nào không quan trọng. Điều khiển tập trung
  cũng không cần thiết. Kết quả là một hệ thống dễ sử dụng hơn và mạnh mẽ hơn, vững chắc hơn,
  bền bỉ hơn và dễ mở rộng hơn.

## Bối cảnh lịch sử của Kubernetes (Historical context for Kubernetes) {#going-back-in-time}

Hãy cùng quay ngược thời gian để xem vì sao Kubernetes lại hữu ích đến vậy.

![Sự tiến hóa của các mô hình triển khai](https://kubernetes.io/images/docs/Container_Evolution.svg)

**Kỷ nguyên triển khai truyền thống (Traditional deployment era):**

Thời kỳ đầu, các tổ chức chạy ứng dụng trên các máy chủ vật lý. Không có cách nào để định nghĩa
ranh giới tài nguyên cho các ứng dụng trên một máy chủ vật lý, và điều này gây ra các vấn đề về
cấp phát tài nguyên. Ví dụ, nếu nhiều ứng dụng cùng chạy trên một máy chủ vật lý, có thể
xảy ra trường hợp một ứng dụng chiếm phần lớn tài nguyên, và hệ quả là
các ứng dụng khác hoạt động kém đi. Một giải pháp cho vấn đề này là chạy mỗi ứng dụng
trên một máy chủ vật lý riêng. Nhưng cách này không mở rộng được vì tài nguyên bị sử dụng dưới mức,
và việc duy trì nhiều máy chủ vật lý rất tốn kém đối với các tổ chức.

**Kỷ nguyên triển khai ảo hóa (Virtualized deployment era):**

Để giải quyết vấn đề đó, công nghệ ảo hóa (virtualization) đã được giới thiệu. Nó cho phép bạn
chạy nhiều máy ảo (Virtual Machine — VM) trên CPU của một máy chủ vật lý duy nhất. Ảo hóa
cho phép các ứng dụng được cách ly giữa các VM và cung cấp một mức độ bảo mật, vì
thông tin của một ứng dụng không thể bị một ứng dụng khác truy cập tự do.

Ảo hóa giúp sử dụng tài nguyên trong một máy chủ vật lý hiệu quả hơn và cho
khả năng mở rộng tốt hơn vì một ứng dụng có thể được thêm hoặc cập nhật dễ dàng, giúp giảm
chi phí phần cứng, và nhiều lợi ích khác nữa. Với ảo hóa, bạn có thể trình bày một tập tài nguyên
vật lý dưới dạng một cluster gồm các máy ảo dùng xong có thể bỏ đi (disposable).

Mỗi VM là một máy đầy đủ chạy tất cả các thành phần, bao gồm cả hệ điều hành
riêng của nó, bên trên phần cứng được ảo hóa.

**Kỷ nguyên triển khai container (Container deployment era):**

Container tương tự như VM, nhưng chúng có các đặc tính cách ly được nới lỏng
để chia sẻ Hệ điều hành (OS) giữa các ứng dụng.
Do đó, container được coi là nhẹ. Tương tự một VM, một container
có hệ thống tệp (filesystem) riêng, phần chia sẻ CPU, bộ nhớ, không gian tiến trình (process space), và nhiều thứ khác.
Vì được tách rời khỏi hạ tầng bên dưới, container có tính di động giữa các cloud
và các bản phân phối hệ điều hành.

Container đã trở nên phổ biến vì chúng mang lại thêm nhiều lợi ích, chẳng hạn như:

* Tạo và triển khai ứng dụng linh hoạt: việc tạo container image dễ dàng và
  hiệu quả hơn so với việc dùng VM image.
* Phát triển, tích hợp và triển khai liên tục: cung cấp việc build và triển khai
  container image một cách tin cậy và thường xuyên, với khả năng rollback nhanh chóng,
  hiệu quả (nhờ tính bất biến của image).
* Tách biệt mối quan tâm giữa Dev và Ops: tạo container image của ứng dụng vào
  thời điểm build/release thay vì thời điểm triển khai, qua đó tách rời
  ứng dụng khỏi hạ tầng.
* Khả năng quan sát (Observability): không chỉ hiển thị thông tin và chỉ số ở cấp hệ điều hành,
  mà còn cả tình trạng ứng dụng và các tín hiệu khác.
* Tính nhất quán môi trường xuyên suốt phát triển, kiểm thử và production: chạy
  trên laptop giống hệt như chạy trên cloud.
* Tính di động giữa các cloud và bản phân phối hệ điều hành: chạy trên Ubuntu, RHEL, CoreOS, on-premises,
  trên các cloud công cộng lớn, và bất cứ đâu khác.
* Quản lý lấy ứng dụng làm trung tâm: nâng mức trừu tượng từ việc chạy một
  hệ điều hành trên phần cứng ảo lên việc chạy một ứng dụng trên một hệ điều hành bằng các tài nguyên logic.
* Các micro-service gắn kết lỏng lẻo, phân tán, co giãn (elastic), tự do: ứng dụng được
  chia thành các phần nhỏ hơn, độc lập, và có thể được triển khai, quản lý một cách linh hoạt —
  không phải một khối nguyên khối chạy trên một cỗ máy lớn chuyên dụng duy nhất.
* Cách ly tài nguyên: hiệu năng ứng dụng có thể dự đoán được.
* Tận dụng tài nguyên: hiệu suất và mật độ cao.

## Tiếp theo (What's next)

* Xem qua [Các thành phần của Kubernetes](./15-components-vi.md)
* Xem qua [Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/)
* Xem qua [kubectl](https://kubernetes.io/docs/concepts/overview/kubectl/): công cụ dòng lệnh (CLI) chính của Kubernetes
* Xem qua [Kiến trúc cluster](https://kubernetes.io/docs/concepts/architecture/)
* Sẵn sàng để [Bắt đầu](https://kubernetes.io/docs/setup/)?
