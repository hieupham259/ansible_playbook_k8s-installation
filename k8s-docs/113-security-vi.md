# Bảo mật (Security)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/>
>
> Các khái niệm để giữ cho workload cloud-native của bạn được an toàn.

Phần này của tài liệu Kubernetes nhằm giúp bạn học cách chạy
workload một cách an toàn hơn, cũng như tìm hiểu về những khía cạnh thiết yếu của việc giữ cho một
cluster Kubernetes được an toàn.

Kubernetes dựa trên kiến trúc cloud-native, và tiếp thu các khuyến nghị từ
CNCF về những thực hành tốt cho
bảo mật thông tin cloud-native.

Hãy đọc [Bảo mật cloud-native và Kubernetes](https://kubernetes.io/docs/concepts/security/cloud-native-security/)
để có bối cảnh rộng hơn về cách bảo vệ cluster của bạn và các ứng dụng
mà bạn đang chạy trên đó.

## Các cơ chế bảo mật của Kubernetes (Kubernetes security mechanisms) {#security-mechanisms}

Kubernetes bao gồm nhiều API và biện pháp kiểm soát bảo mật (security control), cũng như các cách để
định nghĩa [policy](#policies) — những thứ có thể trở thành một phần trong cách bạn quản lý bảo mật thông tin.

### Bảo vệ control plane (Control plane protection)

Một cơ chế bảo mật then chốt cho bất kỳ cluster Kubernetes nào là
[kiểm soát quyền truy cập tới Kubernetes API](https://kubernetes.io/docs/concepts/security/controlling-access).

Kubernetes kỳ vọng bạn cấu hình và sử dụng TLS để cung cấp
[mã hóa dữ liệu khi truyền (data encryption in transit)](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/)
bên trong control plane, và giữa control plane với các client của nó.
Bạn cũng có thể bật [mã hóa dữ liệu khi lưu trữ (encryption at rest)](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
cho dữ liệu được lưu bên trong control plane của Kubernetes; điều này tách biệt với việc dùng
mã hóa khi lưu trữ cho dữ liệu workload của riêng bạn — việc đó cũng có thể là một ý tưởng tốt.

### Secrets

API [Secret](https://kubernetes.io/docs/concepts/configuration/secret/) cung cấp sự bảo vệ cơ bản cho
các giá trị cấu hình đòi hỏi tính bí mật (confidentiality).

### Bảo vệ workload (Workload protection)

Hãy thực thi [các tiêu chuẩn bảo mật Pod (Pod security standards)](https://kubernetes.io/docs/concepts/security/pod-security-standards/) để
bảo đảm các Pod và những container của chúng được cô lập một cách phù hợp. Bạn cũng có thể dùng
[RuntimeClass](https://kubernetes.io/docs/concepts/containers/runtime-class) để định nghĩa cơ chế cô lập tùy chỉnh
nếu bạn cần.

[Network policy](https://kubernetes.io/docs/concepts/services-networking/network-policies/) cho phép bạn kiểm soát
lưu lượng mạng giữa các Pod, hoặc giữa các Pod và mạng bên ngoài cluster của bạn.

Bạn có thể triển khai các biện pháp kiểm soát bảo mật từ hệ sinh thái rộng hơn để hiện thực các biện pháp
kiểm soát mang tính ngăn ngừa (preventative) hoặc phát hiện (detective) xung quanh các Pod, các container của chúng, và các image chạy trong đó.

### Kiểm soát admission (Admission control) {#admission-control}

[Admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
là các plugin chặn (intercept) các request tới Kubernetes API và có thể kiểm tra hợp lệ (validate) hoặc biến đổi (mutate)
các request đó dựa trên những trường cụ thể trong request. Việc thiết kế cẩn trọng
các controller này giúp tránh những gián đoạn ngoài ý muốn khi các API của Kubernetes
thay đổi qua các lần cập nhật phiên bản. Về các cân nhắc thiết kế, hãy xem
[Các thực hành tốt cho Admission Webhook](https://kubernetes.io/docs/concepts/cluster-administration/admission-webhooks-good-practices/).

### Kiểm toán (Auditing)

[Ghi log kiểm toán (audit logging)](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/) của Kubernetes cung cấp một
tập hợp bản ghi theo trình tự thời gian, liên quan đến bảo mật, ghi lại chuỗi hành động
trong một cluster. Cluster kiểm toán các hoạt động được tạo ra bởi người dùng, bởi các ứng dụng
sử dụng Kubernetes API, và bởi chính control plane.

## Bảo mật của nhà cung cấp cloud (Cloud provider security)

> **Ghi chú:** Mục này liệt kê các nhà cung cấp (vendor) bên ngoài Kubernetes. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những sản phẩm hoặc dự án bên thứ ba đó. Để thêm một nhà cung cấp, sản phẩm hoặc dự án vào danh sách này, hãy đọc [hướng dẫn nội dung](https://kubernetes.io/docs/contribute/style/content-guide/) trước khi gửi thay đổi.

Nếu bạn đang chạy một cluster Kubernetes trên phần cứng của riêng mình hoặc trên một nhà cung cấp cloud khác,
hãy tham khảo tài liệu của họ về các thực hành bảo mật tốt nhất.
Dưới đây là các liên kết tới tài liệu bảo mật của một số nhà cung cấp cloud phổ biến:

*Bảo mật của nhà cung cấp cloud (Cloud provider security)*

| Nhà cung cấp IaaS | Liên kết |
| -------------------- | ------------ |
| Alibaba Cloud | https://www.alibabacloud.com/trust-center |
| Amazon Web Services | https://aws.amazon.com/security |
| Google Cloud Platform | https://cloud.google.com/security |
| Huawei Cloud | https://www.huaweicloud.com/intl/en-us/securecenter/overallsafety |
| IBM Cloud | https://www.ibm.com/cloud/security |
| Microsoft Azure | https://docs.microsoft.com/en-us/azure/security/azure-security |
| Oracle Cloud Infrastructure | https://www.oracle.com/security |
| Tencent Cloud | https://www.tencentcloud.com/solutions/data-security-and-information-protection |
| VMware vSphere | https://www.vmware.com/solutions/security/hardening-guides |

## Các policy (Policies)

Bạn có thể định nghĩa các policy bảo mật bằng những cơ chế thuần Kubernetes (Kubernetes-native),
chẳng hạn như [NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
(kiểm soát khai báo (declarative) đối với việc lọc gói tin mạng) hoặc
[ValidatingAdmissionPolicy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/) (các ràng buộc khai báo về những thay đổi
mà một người nào đó có thể thực hiện thông qua Kubernetes API).

Tuy nhiên, bạn cũng có thể dựa vào các hiện thực policy từ hệ sinh thái
rộng hơn xung quanh Kubernetes. Kubernetes cung cấp các cơ chế mở rộng (extension mechanism)
để cho phép những dự án trong hệ sinh thái đó hiện thực các biện pháp kiểm soát policy của riêng họ
đối với việc rà soát mã nguồn, phê duyệt container image, kiểm soát truy cập API,
mạng, và nhiều thứ khác.

Để biết thêm thông tin về các cơ chế policy và Kubernetes,
hãy đọc [Policies](https://kubernetes.io/docs/concepts/policy/).

## Tiếp theo (What's next)

Tìm hiểu về các chủ đề bảo mật Kubernetes liên quan:

* [Bảo vệ cluster của bạn](https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/)
* [Các lỗ hổng đã biết](https://kubernetes.io/docs/reference/issues-security/official-cve-feed/)
  trong Kubernetes (và các liên kết tới thông tin thêm)
* [Mã hóa dữ liệu khi truyền](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/) cho control plane
* [Mã hóa dữ liệu khi lưu trữ](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
* [Kiểm soát quyền truy cập tới Kubernetes API](https://kubernetes.io/docs/concepts/security/controlling-access)
* [Network policy](https://kubernetes.io/docs/concepts/services-networking/network-policies/) cho các Pod
* [Secret trong Kubernetes](https://kubernetes.io/docs/concepts/configuration/secret/)
* [Các tiêu chuẩn bảo mật Pod](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
* [RuntimeClass](https://kubernetes.io/docs/concepts/containers/runtime-class)

Tìm hiểu bối cảnh:

* [Bảo mật cloud-native và Kubernetes](https://kubernetes.io/docs/concepts/security/cloud-native-security/)

Lấy chứng chỉ:

* Chứng chỉ [Certified Kubernetes Security Specialist](https://training.linuxfoundation.org/certification/certified-kubernetes-security-specialist/)
  và khóa đào tạo chính thức.

Đọc thêm trong phần này:
