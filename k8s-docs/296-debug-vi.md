# Giám sát, ghi log và gỡ lỗi (Monitoring, Logging, and Debugging)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/>
>
> Thiết lập giám sát (monitoring) và ghi log (logging) để xử lý sự cố một cluster, hoặc gỡ lỗi
> (debug) một ứng dụng chạy trong container.

Đôi khi mọi thứ gặp trục trặc. Hướng dẫn này giúp bạn thu thập thông tin liên quan và giải quyết
các vấn đề. Nó gồm bốn phần:

* [Gỡ lỗi ứng dụng của bạn](./297-debug-application-vi.md) - Hữu ích cho người dùng đang triển
  khai code lên Kubernetes và thắc mắc vì sao nó không hoạt động.
* [Gỡ lỗi cluster của bạn](https://kubernetes.io/docs/tasks/debug/debug-cluster/) - Hữu ích cho
  quản trị viên cluster và những người vận hành đang xử lý sự cố với chính cluster Kubernetes.
* [Ghi log trong Kubernetes](https://kubernetes.io/docs/tasks/debug/logging/) - Hữu ích cho
  quản trị viên cluster muốn thiết lập và quản lý việc ghi log trong Kubernetes.
* [Giám sát trong Kubernetes](https://kubernetes.io/docs/tasks/debug/monitoring/) - Hữu ích cho
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
dẫn thực tế để bắt đầu. [Tasks](https://kubernetes.io/docs/tasks/) chỉ ra cách hoàn thành các
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
