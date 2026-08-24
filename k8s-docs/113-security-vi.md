# Bảo mật (Security)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/>
>
> Các khái niệm để giữ cho workload cloud-native của bạn được an toàn.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), bài 1/18 · Kiểm chứng ở Lab 9a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài này là **trang mục lục của cả giai đoạn 9**, không phải bài dạy cơ chế. Nó gọi tên từng
nhóm cơ chế bảo mật rồi trỏ sang trang chi tiết. Đọc để có bản đồ và biết mỗi cơ chế nằm ở
đâu; đừng cố hiểu sâu mục nào, vì 17 bài sau sẽ mở từng mục ra.

**Phải hiểu ở lần đọc này:**

- Bản đồ năm nhóm trong mục *Các cơ chế bảo mật của Kubernetes*: bảo vệ control plane, Secret,
  bảo vệ workload, kiểm soát admission, kiểm toán. Nhóm **then chốt** là kiểm soát quyền truy
  cập tới Kubernetes API — chính là bài [119](119-controlling-access-vi.md).
- Hai thứ bài tách bạch rõ: **mã hóa khi truyền** (TLS trong control plane và giữa control
  plane với client) và **mã hóa khi lưu trữ** dữ liệu bên trong control plane. Bài còn tách
  tiếp mã hóa khi lưu trữ của control plane với mã hóa dữ liệu workload của bạn.
- Secret API chỉ cung cấp **sự bảo vệ cơ bản** cho giá trị cấu hình cần giữ bí mật — đó là
  giới hạn mà bài tự nói ra, không phải một cơ chế mã hóa.
- Admission controller là plugin **chặn request tới API** và có thể **kiểm tra hợp lệ hoặc
  biến đổi** request dựa trên những trường cụ thể trong request.
- Audit logging cho một tập bản ghi theo trình tự thời gian, và nó kiểm toán **ba nguồn**:
  người dùng, ứng dụng dùng Kubernetes API, và chính control plane.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Bảng *Bảo mật của nhà cung cấp cloud* | cluster lab chạy trên VM tự dựng, không có IaaS | không cần |
| Chi tiết mã hóa khi lưu trữ và cấu hình audit | là thao tác cấu hình API server | CP7 audit/encryption |
| RuntimeClass để định nghĩa cơ chế cô lập tùy chỉnh | chỉ được nhắc tên ở đây | bài [43](43-runtime-class-vi.md), đã đọc ở giai đoạn 2 |
| ValidatingAdmissionPolicy và các hiện thực policy của hệ sinh thái | cần biết các điểm mở rộng API trước | giai đoạn 14 |
| Mục *Tiếp theo* — CVE feed, chứng chỉ CKS | tài nguyên tham khảo, không phải nội dung học | không cần |

---

Phần này của tài liệu Kubernetes nhằm giúp bạn học cách chạy
workload một cách an toàn hơn, cũng như tìm hiểu về những khía cạnh thiết yếu của việc giữ cho một
cluster Kubernetes được an toàn.

Kubernetes dựa trên kiến trúc cloud-native, và tiếp thu các khuyến nghị từ
CNCF về những thực hành tốt cho
bảo mật thông tin cloud-native.

Hãy đọc [Bảo mật cloud-native và Kubernetes](114-cloud-native-security-vi.md)
để có bối cảnh rộng hơn về cách bảo vệ cluster của bạn và các ứng dụng
mà bạn đang chạy trên đó.

## Các cơ chế bảo mật của Kubernetes (Kubernetes security mechanisms) {#security-mechanisms}

Kubernetes bao gồm nhiều API và biện pháp kiểm soát bảo mật (security control), cũng như các cách để
định nghĩa [policy](#policies) — những thứ có thể trở thành một phần trong cách bạn quản lý bảo mật thông tin.

### Bảo vệ control plane (Control plane protection)

Một cơ chế bảo mật then chốt cho bất kỳ cluster Kubernetes nào là
[kiểm soát quyền truy cập tới Kubernetes API](119-controlling-access-vi.md).

Kubernetes kỳ vọng bạn cấu hình và sử dụng TLS để cung cấp
[mã hóa dữ liệu khi truyền (data encryption in transit)](399-managing-tls-in-a-cluster-vi.md)
bên trong control plane, và giữa control plane với các client của nó.
Bạn cũng có thể bật [mã hóa dữ liệu khi lưu trữ (encryption at rest)](208-encrypt-data-vi.md)
cho dữ liệu được lưu bên trong control plane của Kubernetes; điều này tách biệt với việc dùng
mã hóa khi lưu trữ cho dữ liệu workload của riêng bạn — việc đó cũng có thể là một ý tưởng tốt.

### Secrets

API [Secret](109-secret-vi.md) cung cấp sự bảo vệ cơ bản cho
các giá trị cấu hình đòi hỏi tính bí mật (confidentiality).

### Bảo vệ workload (Workload protection)

Hãy thực thi [các tiêu chuẩn bảo mật Pod (Pod security standards)](115-pod-security-standards-vi.md) để
bảo đảm các Pod và những container của chúng được cô lập một cách phù hợp. Bạn cũng có thể dùng
[RuntimeClass](43-runtime-class-vi.md) để định nghĩa cơ chế cô lập tùy chỉnh
nếu bạn cần.

[Network policy](84-network-policies-vi.md) cho phép bạn kiểm soát
lưu lượng mạng giữa các Pod, hoặc giữa các Pod và mạng bên ngoài cluster của bạn.

Bạn có thể triển khai các biện pháp kiểm soát bảo mật từ hệ sinh thái rộng hơn để hiện thực các biện pháp
kiểm soát mang tính ngăn ngừa (preventative) hoặc phát hiện (detective) xung quanh các Pod, các container của chúng, và các image chạy trong đó.

### Kiểm soát admission (Admission control) {#admission-control}

[Admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
là các plugin chặn (intercept) các request tới Kubernetes API và có thể kiểm tra hợp lệ (validate) hoặc biến đổi (mutate)
các request đó dựa trên những trường cụ thể trong request. Việc thiết kế cẩn trọng
các controller này giúp tránh những gián đoạn ngoài ý muốn khi các API của Kubernetes
thay đổi qua các lần cập nhật phiên bản. Về các cân nhắc thiết kế, hãy xem
[Các thực hành tốt cho Admission Webhook](173-admission-webhooks-vi.md).

### Kiểm toán (Auditing)

[Ghi log kiểm toán (audit logging)](306-audit-vi.md) của Kubernetes cung cấp một
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

## Các policy (Policies) {#policies}

Bạn có thể định nghĩa các policy bảo mật bằng những cơ chế thuần Kubernetes (Kubernetes-native),
chẳng hạn như [NetworkPolicy](84-network-policies-vi.md)
(kiểm soát khai báo (declarative) đối với việc lọc gói tin mạng) hoặc
[ValidatingAdmissionPolicy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/) (các ràng buộc khai báo về những thay đổi
mà một người nào đó có thể thực hiện thông qua Kubernetes API).

Tuy nhiên, bạn cũng có thể dựa vào các hiện thực policy từ hệ sinh thái
rộng hơn xung quanh Kubernetes. Kubernetes cung cấp các cơ chế mở rộng (extension mechanism)
để cho phép những dự án trong hệ sinh thái đó hiện thực các biện pháp kiểm soát policy của riêng họ
đối với việc rà soát mã nguồn, phê duyệt container image, kiểm soát truy cập API,
mạng, và nhiều thứ khác.

Để biết thêm thông tin về các cơ chế policy và Kubernetes,
hãy đọc [Policies](132-policies-vi.md).

## Tiếp theo (What's next)

Tìm hiểu về các chủ đề bảo mật Kubernetes liên quan:

* [Bảo vệ cluster của bạn](256-securing-a-cluster-vi.md)
* [Các lỗ hổng đã biết](https://kubernetes.io/docs/reference/issues-security/official-cve-feed/)
  trong Kubernetes (và các liên kết tới thông tin thêm)
* [Mã hóa dữ liệu khi truyền](399-managing-tls-in-a-cluster-vi.md) cho control plane
* [Mã hóa dữ liệu khi lưu trữ](208-encrypt-data-vi.md)
* [Kiểm soát quyền truy cập tới Kubernetes API](119-controlling-access-vi.md)
* [Network policy](84-network-policies-vi.md) cho các Pod
* [Secret trong Kubernetes](109-secret-vi.md)
* [Các tiêu chuẩn bảo mật Pod](115-pod-security-standards-vi.md)
* [RuntimeClass](43-runtime-class-vi.md)

Tìm hiểu bối cảnh:

* [Bảo mật cloud-native và Kubernetes](114-cloud-native-security-vi.md)

Lấy chứng chỉ:

* Chứng chỉ [Certified Kubernetes Security Specialist](https://training.linuxfoundation.org/certification/certified-kubernetes-security-specialist/)
  và khóa đào tạo chính thức.

Đọc thêm trong phần này:

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 9:

1. Bài nói cluster nên dùng TLS cho dữ liệu khi truyền và **có thể** bật mã hóa khi lưu trữ.
   Hai thứ đó bảo vệ dữ liệu ở trạng thái nào, và bật cái này có thay được cái kia không?
2. Đồng nghiệp nói: "cứ để mật khẩu trong Secret thay vì ConfigMap là đã mã hóa rồi." Theo
   đúng câu chữ của bài, Secret API cung cấp mức bảo vệ nào, và còn thiếu bước gì?
3. Admission controller làm được gì với một request mà các cơ chế kiểm soát khác được liệt kê
   trong bài không làm được?
4. Trên cluster lab, bạn dùng kubeconfig copy từ `/etc/kubernetes/admin.conf` chạy
   `kubectl get pods`. Theo bài, audit log kiểm toán hoạt động từ ba nguồn nào, và hành động
   vừa rồi của bạn thuộc nguồn nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không thay được cho nhau.** Mã hóa khi truyền bảo vệ dữ liệu **đang đi trên đường**
   — bên trong control plane và giữa control plane với các client của nó; mã hóa khi lưu trữ
   bảo vệ dữ liệu **đang nằm trong control plane**. Bài còn nói rõ mã hóa khi lưu trữ cho dữ
   liệu của control plane **tách biệt** với mã hóa khi lưu trữ cho dữ liệu workload của bạn —
   nghĩa là ba thứ khác nhau, không cái nào bao cái nào.
2. **Chưa đủ.** Bài chỉ nói Secret API cung cấp **sự bảo vệ cơ bản** cho các giá trị cấu hình
   đòi hỏi tính bí mật. Muốn dữ liệu đó được mã hóa trong control plane thì phải **bật mã hóa
   khi lưu trữ** — một việc riêng mà bài liệt kê ở mục *Bảo vệ control plane*.
3. Admission controller **chặn request tới Kubernetes API** rồi **kiểm tra hợp lệ hoặc biến
   đổi** request đó **dựa trên những trường cụ thể trong request**. Tức là nó làm việc với
   **nội dung** của thứ đang được tạo hoặc sửa, chứ không chỉ với danh tính người gửi.
4. Ba nguồn: **người dùng**, **các ứng dụng sử dụng Kubernetes API**, và **chính control
   plane**. Lệnh `kubectl get pods` chạy bằng credential của bạn thuộc nguồn **người dùng**.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
