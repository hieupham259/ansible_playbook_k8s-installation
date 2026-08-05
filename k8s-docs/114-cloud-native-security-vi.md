# Bảo mật Cloud Native và Kubernetes (Cloud Native Security and Kubernetes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/cloud-native-security/>
>
> Các khái niệm để giữ an toàn cho workload cloud native của bạn.

Kubernetes được xây dựng trên kiến trúc cloud native và tham khảo các khuyến nghị từ
CNCF về những thực hành tốt cho bảo mật thông tin cloud native.

Hãy đọc tiếp để có cái nhìn tổng quan về cách Kubernetes được thiết kế nhằm giúp bạn
triển khai một nền tảng cloud native an toàn.

## Bảo mật thông tin cloud native (Cloud native information security)

[Sách trắng (white paper)](https://github.com/cncf/tag-security/blob/main/community/resources/security-whitepaper/v2/CNCF_cloud-native-security-whitepaper-May2022-v2.pdf)
của CNCF về bảo mật cloud native định nghĩa các biện pháp kiểm soát bảo mật (security
control) và các thực hành phù hợp với từng _giai đoạn vòng đời_ (lifecycle phase) khác nhau.

## Giai đoạn vòng đời _Phát triển_ (Develop lifecycle phase) {#lifecycle-phase-develop}

- Đảm bảo tính toàn vẹn (integrity) của môi trường phát triển.
- Thiết kế ứng dụng tuân theo các thực hành tốt về bảo mật thông tin,
  phù hợp với bối cảnh của bạn.
- Xem xét bảo mật cho người dùng cuối như một phần của việc thiết kế giải pháp.

Để đạt được điều này, bạn có thể:

1. Áp dụng một kiến trúc, chẳng hạn [zero trust](https://glossary.cncf.io/zero-trust-architecture/),
   giúp giảm thiểu bề mặt tấn công (attack surface), kể cả đối với các mối đe dọa từ nội bộ.
1. Định nghĩa một quy trình review code có xem xét đến các vấn đề bảo mật.
1. Xây dựng một _mô hình mối đe dọa_ (threat model) cho hệ thống hoặc ứng dụng của bạn,
   trong đó xác định các ranh giới tin cậy (trust boundary). Dùng mô hình đó để nhận diện
   rủi ro và quyết định cách xử lý chúng.
1. Tích hợp tự động hóa bảo mật nâng cao, chẳng hạn _fuzzing_ và
   [security chaos engineering](https://glossary.cncf.io/security-chaos-engineering/),
   khi việc đó là hợp lý.

## Giai đoạn vòng đời _Phân phối_ (Distribute lifecycle phase) {#lifecycle-phase-distribute}

- Đảm bảo an toàn cho chuỗi cung ứng (supply chain) của các container image mà bạn thực thi.
- Đảm bảo an toàn cho chuỗi cung ứng của cluster và các thành phần khác thực thi
  ứng dụng của bạn. Ví dụ, điều này có thể bao gồm một cơ sở dữ liệu bên ngoài mà
  ứng dụng cloud native của bạn dùng để lưu trữ lâu dài (persistence).

Để đạt được điều này, bạn có thể:

1. Quét (scan) container image và các artifact khác để tìm những lỗ hổng đã biết.
1. Đảm bảo việc phân phối phần mềm sử dụng mã hóa trên đường truyền (encryption in
   transit), với một chuỗi tin cậy (chain of trust) cho nguồn gốc phần mềm.
1. Áp dụng và tuân theo các quy trình cập nhật dependency khi có bản cập nhật,
   đặc biệt là để phản ứng với các thông báo bảo mật.
1. Sử dụng các cơ chế kiểm chứng như chứng chỉ số (digital certificate) để đảm bảo
   chuỗi cung ứng.
1. Đăng ký (subscribe) các feed và những cơ chế khác để được cảnh báo về các rủi ro
   bảo mật.
1. Hạn chế quyền truy cập vào artifact. Đặt container image trong một
   [private registry](https://kubernetes.io/docs/concepts/containers/images/#using-a-private-registry)
   chỉ cho phép các client được ủy quyền pull image.

## Giai đoạn vòng đời _Triển khai_ (Deploy lifecycle phase) {#lifecycle-phase-deploy}

Đảm bảo các hạn chế phù hợp về những gì có thể được triển khai, ai có thể triển khai,
và nơi có thể triển khai.
Bạn có thể thực thi các biện pháp từ giai đoạn _phân phối_, chẳng hạn xác minh
danh tính mật mã (cryptographic identity) của các artifact container image.

Bạn có thể triển khai các ứng dụng và thành phần cluster khác nhau vào các
namespace khác nhau. Container và namespace đều cung cấp cơ chế cô lập
(isolation) có liên quan đến bảo mật thông tin.

Khi bạn triển khai Kubernetes, bạn cũng đồng thời đặt nền móng cho môi trường
runtime của các ứng dụng: một cluster Kubernetes (hoặc nhiều cluster).
Hạ tầng đó phải cung cấp các đảm bảo bảo mật mà những tầng cao hơn mong đợi.

## Giai đoạn vòng đời _Runtime_ (Runtime lifecycle phase) {#lifecycle-phase-runtime}

Giai đoạn Runtime bao gồm ba lĩnh vực trọng yếu: [truy cập](#protection-runtime-access),
[tính toán](#protection-runtime-compute) và [lưu trữ](#protection-runtime-storage).

### Bảo vệ runtime: truy cập (Runtime protection: access) {#protection-runtime-access}

API của Kubernetes là thứ khiến cluster của bạn hoạt động. Bảo vệ API này là chìa khóa
để bảo mật cluster một cách hiệu quả.

Các trang khác trong tài liệu Kubernetes có chi tiết hơn về cách thiết lập từng khía cạnh
cụ thể của kiểm soát truy cập. [Danh sách kiểm tra bảo mật (security checklist)](https://kubernetes.io/docs/concepts/security/security-checklist/)
đưa ra các bước kiểm tra cơ bản được đề xuất cho cluster của bạn.

Ngoài ra, bảo mật cluster của bạn có nghĩa là triển khai hiệu quả
[xác thực (authentication)](https://kubernetes.io/docs/concepts/security/controlling-access/#authentication) và
[phân quyền (authorization)](https://kubernetes.io/docs/concepts/security/controlling-access/#authorization)
cho việc truy cập API. Hãy dùng [ServiceAccounts](https://kubernetes.io/docs/concepts/security/service-accounts/)
để cung cấp và quản lý danh tính bảo mật cho các workload và các thành phần của cluster.

Kubernetes dùng TLS để bảo vệ lưu lượng API; hãy đảm bảo triển khai cluster sử dụng
TLS (bao gồm cả lưu lượng giữa các node và control plane) và bảo vệ các khóa mã hóa.
Nếu bạn dùng chính API của Kubernetes cho
[CertificateSigningRequests](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#certificate-signing-requests),
hãy đặc biệt chú ý hạn chế việc lạm dụng ở đó.

### Bảo vệ runtime: tính toán (Runtime protection: compute) {#protection-runtime-compute}

Container cung cấp hai thứ: sự cô lập giữa các ứng dụng và một cơ chế để gom các
ứng dụng đã được cô lập đó chạy trên cùng một máy chủ vật lý. Hai khía cạnh
này — cô lập và gom nhóm — có nghĩa là bảo mật runtime đòi hỏi phải nhận diện
các đánh đổi (trade-off) và tìm ra điểm cân bằng phù hợp.

Kubernetes dựa vào một container runtime
để thiết lập và chạy các container. Dự án Kubernetes không khuyến nghị một
container runtime cụ thể nào, và bạn nên đảm bảo rằng (các) runtime bạn chọn
đáp ứng được nhu cầu bảo mật thông tin của mình.

Để bảo vệ phần tính toán tại runtime, bạn có thể:

1. Thực thi [Chuẩn bảo mật Pod (Pod Security Standards)](./115-pod-security-standards-vi.md)
   cho các ứng dụng, nhằm giúp đảm bảo chúng chỉ chạy với những đặc quyền cần thiết.
1. Chạy một hệ điều hành chuyên biệt trên các node, được thiết kế riêng
   cho việc chạy các workload container hóa. Hệ điều hành này thường dựa trên một
   hệ điều hành chỉ đọc (_immutable image_) chỉ cung cấp các dịch vụ
   thiết yếu cho việc chạy container.

   Các hệ điều hành chuyên dành cho container giúp cô lập các thành phần hệ thống và
   thu hẹp bề mặt tấn công trong trường hợp xảy ra container escape (thoát khỏi container).
1. Định nghĩa [ResourceQuotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/) để
   phân bổ công bằng các tài nguyên dùng chung, và dùng
   các cơ chế như [LimitRanges](https://kubernetes.io/docs/concepts/policy/limit-range/)
   để đảm bảo các Pod khai báo yêu cầu tài nguyên của chúng.
1. Phân chia workload trên các node khác nhau để tăng mức độ cô lập.
   Dùng các cơ chế [cô lập node (node isolation)](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-isolation-restriction),
   từ chính Kubernetes hoặc từ hệ sinh thái, để đảm bảo rằng
   các Pod có bối cảnh tin cậy (trust context) khác nhau chạy trên các nhóm node tách biệt.
1. Dùng một container runtime
   cung cấp các hạn chế về bảo mật.
1. Trên các node Linux, dùng một module bảo mật của Linux như [AppArmor](https://kubernetes.io/docs/tutorials/security/apparmor/)
   hoặc [seccomp](https://kubernetes.io/docs/tutorials/security/seccomp/).

### Bảo vệ runtime: lưu trữ (Runtime protection: storage) {#protection-runtime-storage}

Để bảo vệ hệ thống lưu trữ cho cluster và các ứng dụng chạy trong đó, bạn có thể:

1. Tích hợp cluster với một storage plugin bên ngoài cung cấp mã hóa dữ liệu
   khi lưu trữ (encryption at rest) cho các volume.
1. Bật [mã hóa khi lưu trữ (encryption at rest)](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
   cho các đối tượng API.
1. Bảo vệ độ bền của dữ liệu (durability) bằng các bản sao lưu (backup), và kiểm tra
   rằng bạn có thể khôi phục chúng bất cứ khi nào cần.
1. Xác thực các kết nối giữa các node của cluster và bất kỳ hệ thống lưu trữ mạng nào
   mà chúng phụ thuộc vào.
1. Triển khai mã hóa dữ liệu bên trong chính ứng dụng của bạn.

Đối với các khóa mã hóa, việc sinh khóa bên trong phần cứng chuyên dụng mang lại
sự bảo vệ tốt nhất trước các rủi ro lộ khóa. Một _mô-đun bảo mật phần cứng_
(hardware security module) cho phép bạn thực hiện các phép toán mật mã mà không
để khóa bảo mật bị sao chép đi nơi khác.

### Mạng và bảo mật (Networking and security)

Bạn cũng nên xem xét các biện pháp bảo mật mạng, chẳng hạn
[NetworkPolicy](./84-network-policies-vi.md) hoặc một
[service mesh](https://glossary.cncf.io/service-mesh/).
Một số network plugin cho Kubernetes cung cấp mã hóa cho mạng của cluster
bằng các công nghệ như lớp phủ (overlay) mạng riêng ảo (VPN).
Theo thiết kế, Kubernetes cho phép bạn dùng networking plugin của riêng mình cho
cluster. Nếu bạn dùng Kubernetes được quản lý (managed Kubernetes), nhà cung cấp
có thể đã chọn sẵn một network plugin cho bạn.

Network plugin bạn chọn và cách bạn tích hợp nó có thể ảnh hưởng mạnh
đến độ an toàn của thông tin trên đường truyền.

### Khả năng quan sát và bảo mật runtime (Observability and runtime security)

Kubernetes cho phép bạn mở rộng cluster bằng các công cụ bổ sung. Bạn có thể thiết lập
các giải pháp bên thứ ba để giúp giám sát hoặc khắc phục sự cố cho các ứng dụng và các
cluster mà chúng đang chạy. Bạn cũng có sẵn một số tính năng quan sát (observability)
cơ bản được tích hợp trong chính Kubernetes. Code của bạn chạy trong container có thể
sinh log, xuất bản metric, hoặc cung cấp dữ liệu quan sát khác; tại thời điểm triển khai,
bạn cần đảm bảo cluster cung cấp mức bảo vệ phù hợp cho những dữ liệu đó.

Nếu bạn thiết lập một dashboard hiển thị metric hay thứ gì đó tương tự, hãy rà soát chuỗi
các thành phần đưa dữ liệu vào dashboard đó, cũng như chính dashboard. Hãy đảm bảo
toàn bộ chuỗi này được thiết kế với đủ khả năng chống chịu (resilience) và bảo vệ tính
toàn vẹn để bạn có thể tin cậy nó ngay cả trong một sự cố khi cluster của bạn có thể bị
suy giảm.

Khi phù hợp, hãy triển khai các biện pháp bảo mật ở tầng bên dưới Kubernetes,
chẳng hạn khởi động được đo lường bằng mật mã (cryptographically measured boot) hoặc
phân phối thời gian có xác thực (giúp đảm bảo độ tin cậy của log và các bản ghi audit).

Đối với môi trường yêu cầu độ đảm bảo cao, hãy triển khai các cơ chế bảo vệ bằng mật mã
để đảm bảo log vừa chống giả mạo (tamper-proof) vừa được giữ bí mật.

## Tiếp theo (What's next)

### Bảo mật cloud native (Cloud native security) {#further-reading-cloud-native}

* [Sách trắng](https://github.com/cncf/tag-security/blob/main/community/resources/security-whitepaper/v2/CNCF_cloud-native-security-whitepaper-May2022-v2.pdf)
  của CNCF về bảo mật cloud native.
* [Sách trắng](https://github.com/cncf/tag-security/blob/f80844baaea22a358f5b20dca52cd6f72a32b066/supply-chain-security/supply-chain-security-paper/CNCF_SSCP_v1.pdf)
  của CNCF về các thực hành tốt để bảo vệ chuỗi cung ứng phần mềm.
* [Fixing the Kubernetes clusterf\*\*k: Understanding security from the kernel up](https://archive.fosdem.org/2020/schedule/event/kubernetes/) (FOSDEM 2020)
* [Kubernetes Security Best Practices](https://www.youtube.com/watch?v=wqsUfvRyYpw) (Kubernetes Forum Seoul 2019)
* [Towards Measured Boot Out of the Box](https://www.youtube.com/watch?v=EzSkU3Oecuw) (Linux Security Summit 2016)

### Kubernetes và bảo mật thông tin (Kubernetes and information security) {#further-reading-k8s}

* [Bảo mật Kubernetes](https://kubernetes.io/docs/concepts/security/)
* [Bảo vệ cluster của bạn](https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/)
* [Mã hóa dữ liệu trên đường truyền](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/) cho control plane
* [Mã hóa dữ liệu khi lưu trữ](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
* [Secret trong Kubernetes](https://kubernetes.io/docs/concepts/configuration/secret/)
* [Kiểm soát truy cập vào API của Kubernetes](https://kubernetes.io/docs/concepts/security/controlling-access)
* [Network policy](./84-network-policies-vi.md) cho các Pod
* [Chuẩn bảo mật Pod (Pod security standards)](./115-pod-security-standards-vi.md)
* [RuntimeClasses](./43-runtime-class-vi.md)
