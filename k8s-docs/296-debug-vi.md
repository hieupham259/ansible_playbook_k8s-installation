# Giám sát, ghi log và gỡ lỗi (Monitoring, Logging, and Debugging)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/>
>
> Thiết lập giám sát (monitoring) và ghi log (logging) để xử lý sự cố một cluster, hoặc gỡ lỗi
> (debug) một ứng dụng chạy trong container.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 11 — Observability](00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability)
→ dòng **Thực hành**, bài 1/7 · Kiểm chứng ở
[Lab 11a — Observability](labs/LAB-11A-OBSERVABILITY.md) phần B1.3, và phần B11.5 khi lab vẽ
lại ranh giới trước khi sang giai đoạn 24.

Đây là **trang mục lục** mở đầu cả nhánh `/docs/tasks/debug/`, không phải bài dạy thao tác.
Nửa sau trang là hướng dẫn xin trợ giúp từ cộng đồng, không phải kiến thức kỹ thuật. Đọc nó để
biết mình đang đứng ở nhánh nào và nhánh nào để dành lại.

**Phải hiểu ở lần đọc này:**

- Bốn nhánh mà trang chia ra và **tiêu chí chia**: *Gỡ lỗi ứng dụng của bạn* dành cho người
  triển khai code và thắc mắc vì sao code không chạy; *Gỡ lỗi cluster của bạn* dành cho người
  vận hành đang xử lý sự cố với **chính cluster**; *Ghi log trong Kubernetes* và *Giám sát
  trong Kubernetes* dành cho quản trị viên muốn thiết lập hai trụ cột đó. Ở giai đoạn 11 bạn đi
  hai nhánh cuối.
- Việc trang yêu cầu làm **trước** khi đi tìm nguyên nhân: kiểm tra danh sách vấn đề đã biết
  (known issues) của đúng bản phát hành mà cluster đang chạy.
- Mục *Câu hỏi*: tài liệu kubernetes.io được chia theo loại câu hỏi — Concepts giải thích kiến
  trúc, Setup hướng dẫn dựng, [Tasks](367-tasks-index-vi.md) chỉ cách làm một việc, Tutorials là
  kịch bản đầu-cuối, Reference là đặc tả API và CLI. Biết loại câu hỏi nào tra ở đâu.
- Mục *Stack Exchange, Stack Overflow hoặc Server Fault*: ranh giới phân loại — câu hỏi về
  **phát triển phần mềm** cho ứng dụng container hỏi ở Stack Overflow, câu hỏi về **quản lý
  cluster hoặc cấu hình** hỏi ở Server Fault.
- Mục *Bug và yêu cầu tính năng*: phải tìm trong các issue đã có trước khi mở issue mới, và khi
  báo bug phải kèm bộ dữ kiện tối thiểu — `kubectl version`, nhà cung cấp cloud / bản phân phối
  OS / cấu hình mạng / phiên bản container runtime, và các bước tái hiện.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Nhánh *Gỡ lỗi cluster của bạn* — bài [305](305-debug-cluster-vi.md) | là quy trình chẩn đoán có hệ thống, cần `crictl` và ephemeral container chưa học ở giai đoạn 11 | [giai đoạn 24](00-ALO-TRINH-ADMIN.md#giai-đoạn-24--xử-lý-sự-cố), bài [305](305-debug-cluster-vi.md) |
| Phần chẩn đoán Pod, Service và StatefulSet của nhánh *Gỡ lỗi ứng dụng* | giai đoạn 11 chỉ dùng `logs` và `exec`; phần còn lại là quy trình chẩn đoán riêng | [giai đoạn 24](00-ALO-TRINH-ADMIN.md#giai-đoạn-24--xử-lý-sự-cố), bài [299](299-debug-pods-vi.md), [301](301-debug-service-vi.md), [302](302-debug-statefulset-vi.md) |
| Bảng kênh Slack theo quốc gia / ngôn ngữ, mục *Diễn đàn* | danh bạ cộng đồng, không phải cơ chế của Kubernetes | không thuộc giai đoạn nào — tra khi thật sự cần hỏi cộng đồng |

---

Đôi khi mọi thứ gặp trục trặc. Hướng dẫn này giúp bạn thu thập thông tin liên quan và giải quyết
các vấn đề. Nó gồm bốn phần:

* [Gỡ lỗi ứng dụng của bạn](./297-debug-application-vi.md) - Hữu ích cho người dùng đang triển
  khai code lên Kubernetes và thắc mắc vì sao nó không hoạt động.
* [Gỡ lỗi cluster của bạn](305-debug-cluster-vi.md) - Hữu ích cho
  quản trị viên cluster và những người vận hành đang xử lý sự cố với chính cluster Kubernetes.
* [Ghi log trong Kubernetes](316-debug-logging-vi.md) - Hữu ích cho
  quản trị viên cluster muốn thiết lập và quản lý việc ghi log trong Kubernetes.
* [Giám sát trong Kubernetes](317-debug-monitoring-vi.md) - Hữu ích cho
  quản trị viên cluster muốn bật tính năng giám sát trong một cluster Kubernetes.

Bạn cũng nên kiểm tra các vấn đề đã biết (known issues) của
[bản phát hành](https://github.com/kubernetes/kubernetes/releases) mà bạn đang sử dụng.

## Nhận trợ giúp (Getting help)

Nếu vấn đề của bạn không được giải đáp bởi bất kỳ hướng dẫn nào ở trên, có nhiều cách khác nhau
để bạn nhận trợ giúp từ cộng đồng Kubernetes.

### Câu hỏi (Questions)

Tài liệu trên trang này đã được cấu trúc để cung cấp câu trả lời cho rất nhiều loại câu hỏi.
[Concepts](https://kubernetes.io/docs/concepts/) giải thích kiến trúc Kubernetes và cách từng
thành phần hoạt động, trong khi [Setup](https://kubernetes.io/docs/setup/) cung cấp các hướng
dẫn thực tế để bắt đầu. [Tasks](367-tasks-index-vi.md) chỉ ra cách hoàn thành các
tác vụ thường dùng, và [Tutorials](https://kubernetes.io/docs/tutorials/) là những bài hướng
dẫn toàn diện hơn về các kịch bản thực tế, đặc thù theo ngành, hoặc phát triển từ đầu đến cuối.
Phần [Reference](https://kubernetes.io/docs/reference/) cung cấp tài liệu chi tiết về
[Kubernetes API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/)
và các giao diện dòng lệnh (CLI), chẳng hạn như
[`kubectl`](https://kubernetes.io/docs/reference/kubectl/).

## Cứu! Câu hỏi của tôi chưa được đề cập! Tôi cần trợ giúp ngay! (Help! My question isn't covered! I need help now!)

### Stack Exchange, Stack Overflow hoặc Server Fault {#stack-exchange}

Nếu bạn có câu hỏi liên quan đến *phát triển phần mềm* cho ứng dụng chạy trong container của
mình, bạn có thể hỏi trên [Stack Overflow](https://stackoverflow.com/questions/tagged/kubernetes).

Nếu bạn có câu hỏi Kubernetes liên quan đến *quản lý cluster* hoặc *cấu hình*, bạn có thể hỏi
trên [Server Fault](https://serverfault.com/questions/tagged/kubernetes).

Ngoài ra còn có một số trang chuyên biệt hơn trong mạng lưới Stack Exchange có thể là nơi phù
hợp để đặt câu hỏi Kubernetes trong các lĩnh vực như
[DevOps](https://devops.stackexchange.com/questions/tagged/kubernetes),
[Software Engineering](https://softwareengineering.stackexchange.com/questions/tagged/kubernetes),
hoặc [InfoSec](https://security.stackexchange.com/questions/tagged/kubernetes).

Ai đó trong cộng đồng có thể đã hỏi một câu tương tự hoặc có thể giúp giải quyết vấn đề của bạn.

Đội ngũ Kubernetes cũng sẽ theo dõi
[các bài đăng gắn tag Kubernetes](https://stackoverflow.com/questions/tagged/kubernetes).
Nếu không có câu hỏi sẵn có nào giúp được bạn, **hãy đảm bảo rằng câu hỏi của bạn
[đúng chủ đề trên Stack Overflow](https://stackoverflow.com/help/on-topic),
[Server Fault](https://serverfault.com/help/on-topic), hoặc trang thuộc mạng lưới Stack Exchange
mà bạn đang hỏi**, và đọc kỹ hướng dẫn về
[cách đặt một câu hỏi mới](https://stackoverflow.com/help/how-to-ask)
trước khi đặt câu hỏi mới!

### Slack

Nhiều người trong cộng đồng Kubernetes tụ họp trên Kubernetes Slack tại kênh `#kubernetes-users`.
Slack yêu cầu đăng ký; bạn có thể [yêu cầu một lời mời](https://slack.kubernetes.io),
và việc đăng ký mở cho tất cả mọi người. Cứ thoải mái tham gia và đặt bất kỳ câu hỏi nào.
Sau khi đăng ký, hãy truy cập [tổ chức Kubernetes trên Slack](https://kubernetes.slack.com)
qua trình duyệt web hoặc qua ứng dụng riêng của Slack.

Sau khi đã đăng ký, hãy duyệt qua danh sách kênh ngày càng nhiều về các chủ đề khác nhau mà bạn
quan tâm. Ví dụ, những người mới với Kubernetes có thể muốn tham gia kênh
[`#kubernetes-novice`](https://kubernetes.slack.com/messages/kubernetes-novice). Một ví dụ
khác, các nhà phát triển nên tham gia kênh
[`#kubernetes-contributors`](https://kubernetes.slack.com/messages/kubernetes-contributors).

Ngoài ra còn có nhiều kênh theo quốc gia / ngôn ngữ địa phương. Cứ thoải mái tham gia các kênh
này để được hỗ trợ và nhận thông tin bằng ngôn ngữ của bạn:

*Các kênh Slack theo quốc gia / ngôn ngữ*

Quốc gia | Kênh
:---------|:------------
Trung Quốc | [`#cn-users`](https://kubernetes.slack.com/messages/cn-users), [`#cn-events`](https://kubernetes.slack.com/messages/cn-events)
Phần Lan | [`#fi-users`](https://kubernetes.slack.com/messages/fi-users)
Pháp | [`#fr-users`](https://kubernetes.slack.com/messages/fr-users), [`#fr-events`](https://kubernetes.slack.com/messages/fr-events)
Đức | [`#de-users`](https://kubernetes.slack.com/messages/de-users), [`#de-events`](https://kubernetes.slack.com/messages/de-events)
Ấn Độ | [`#in-users`](https://kubernetes.slack.com/messages/in-users), [`#in-events`](https://kubernetes.slack.com/messages/in-events)
Ý | [`#it-users`](https://kubernetes.slack.com/messages/it-users), [`#it-events`](https://kubernetes.slack.com/messages/it-events)
Nhật Bản | [`#jp-users`](https://kubernetes.slack.com/messages/jp-users), [`#jp-events`](https://kubernetes.slack.com/messages/jp-events)
Hàn Quốc | [`#kr-users`](https://kubernetes.slack.com/messages/kr-users)
Hà Lan | [`#nl-users`](https://kubernetes.slack.com/messages/nl-users)
Na Uy | [`#norw-users`](https://kubernetes.slack.com/messages/norw-users)
Ba Lan | [`#pl-users`](https://kubernetes.slack.com/messages/pl-users)
Nga | [`#ru-users`](https://kubernetes.slack.com/messages/ru-users)
Tây Ban Nha | [`#es-users`](https://kubernetes.slack.com/messages/es-users)
Thụy Điển | [`#se-users`](https://kubernetes.slack.com/messages/se-users)
Thổ Nhĩ Kỳ | [`#tr-users`](https://kubernetes.slack.com/messages/tr-users), [`#tr-events`](https://kubernetes.slack.com/messages/tr-events)

### Diễn đàn (Forum)

Bạn được chào đón tham gia diễn đàn Kubernetes chính thức:
[discuss.kubernetes.io](https://discuss.kubernetes.io).

### Bug và yêu cầu tính năng (Bugs and feature requests)

Nếu bạn thấy điều gì đó có vẻ là một bug, hoặc bạn muốn đưa ra một yêu cầu tính năng, hãy dùng
[hệ thống theo dõi issue trên GitHub](https://github.com/kubernetes/kubernetes/issues).

Trước khi tạo một issue, hãy tìm kiếm trong các issue hiện có để xem vấn đề của bạn đã được
đề cập chưa.

Nếu báo cáo một bug, hãy kèm theo thông tin chi tiết về cách tái hiện vấn đề, chẳng hạn như:

* Phiên bản Kubernetes: `kubectl version`
* Nhà cung cấp cloud, bản phân phối hệ điều hành, cấu hình mạng, và phiên bản container runtime
* Các bước để tái hiện vấn đề

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 11:

1. Trang chia bốn nhánh. Nhánh nào bạn đi ngay ở giai đoạn 11, nhánh nào để lại, và để lại tới
   giai đoạn nào?
2. **Câu bẫy.** Bạn là quản trị viên cluster. Vậy mọi sự cố bạn gặp đều tra ở nhánh *Gỡ lỗi
   cluster của bạn*, đúng không? Trang chia nhánh theo **chức danh người hỏi** hay theo **thứ
   đang hỏng**?
3. Trên cluster lab, `lab-k8s-worker2` có một hành vi lạ mà bạn chưa lý giải được. Trước khi đào
   sâu đi tìm nguyên nhân, trang bảo bạn kiểm tra thứ gì, và kiểm tra ở đâu?
4. Bạn có hai câu hỏi: (a) "vì sao code trong container của tôi không kết nối được database",
   (b) "nên đặt `--authorization-mode` nào cho kube-apiserver". Trang chỉ mỗi câu tới diễn đàn
   nào, và tiêu chí phân loại là gì?
5. Bạn định mở một issue trên GitHub báo một bug. Trang yêu cầu làm gì **trước** khi mở, và bắt
   buộc kèm những dữ kiện nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Đi hai nhánh **Ghi log trong Kubernetes** và **Giám sát trong Kubernetes** — đó là hai nhánh
   trang mô tả là dành cho quản trị viên muốn *thiết lập và quản lý* việc ghi log, và *bật tính
   năng giám sát* cho cluster; chúng ứng đúng với nhóm bài [158](158-logging-vi.md)–[161](161-system-traces-vi.md)
   của giai đoạn 11. Hai nhánh **Gỡ lỗi ứng dụng của bạn** và **Gỡ lỗi cluster của bạn** để lại
   tới [giai đoạn 24](00-ALO-TRINH-ADMIN.md#giai-đoạn-24--xử-lý-sự-cố), vì chúng là quy trình
   chẩn đoán chứ không phải việc dựng nguồn dữ liệu.
2. **Không.** Trang chia nhánh theo **thứ đang hỏng**, không theo chức danh: nhánh *Gỡ lỗi ứng
   dụng của bạn* được mô tả là cho "người dùng đang triển khai code lên Kubernetes và thắc mắc
   vì sao nó không hoạt động", còn nhánh *Gỡ lỗi cluster của bạn* là cho người "xử lý sự cố với
   **chính cluster Kubernetes**". Bạn là admin nhưng khi Pod bạn vừa triển khai không chạy thì
   vẫn là nhánh thứ nhất. Đây là chỗ dễ sai: chức danh không quyết định nhánh.
3. Kiểm tra **các vấn đề đã biết (known issues) của chính bản phát hành Kubernetes mà cluster
   đang chạy** — trang trỏ thẳng tới trang releases của kho `kubernetes/kubernetes`. Cluster lab
   chạy đúng một phiên bản đã khóa ở [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md), nên danh sách
   cần đọc là danh sách của đúng phiên bản đó. Lý do: nếu đó là bug đã biết thì mọi công sức
   chẩn đoán tiếp theo là lãng phí.
4. Câu (a) là câu hỏi về **phát triển phần mềm** cho ứng dụng chạy trong container → **Stack
   Overflow**. Câu (b) là câu hỏi về **quản lý cluster hoặc cấu hình** → **Server Fault**. Tiêu
   chí là *loại vấn đề*, không phải mức độ khó. Trang còn nêu vài trang chuyên biệt khác trong
   mạng lưới Stack Exchange (DevOps, Software Engineering, InfoSec) cho các lĩnh vực hẹp hơn.
5. Trước khi mở: **tìm trong các issue hiện có** xem vấn đề đã được đề cập chưa. Khi báo bug
   phải kèm: **phiên bản Kubernetes** (`kubectl version`); **nhà cung cấp cloud, bản phân phối
   hệ điều hành, cấu hình mạng và phiên bản container runtime**; và **các bước để tái hiện** vấn
   đề. Thiếu bộ này thì báo cáo không tái hiện được.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
