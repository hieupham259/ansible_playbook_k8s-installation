# Review kiến trúc EKS + ALB và phương án lab 5–6 triệu VND/tháng

Ngày review: **2026-10-13**. Tài liệu nguồn: [kien-truc-eks-alb-100-1000-ccu.md](kien-truc-eks-alb-100-1000-ccu.md).

Người dùng đã xác nhận đây là **lab, được tắt ngoài giờ**. Review này kiểm tra tài liệu và giá công khai, chưa kiểm tra tài khoản AWS, Terraform state, manifest đang triển khai hoặc chạy load test. Các cấu hình dưới đây là đề xuất, chưa áp dụng lên hạ tầng.

## 1. Kết luận và phạm vi giữ nguyên kiến trúc

Kiến trúc tổng thể khả thi: Route 53 → CloudFront/WAF → VPC origin → ALB internal → EKS; S3 riêng tư qua OAC; RDS PostgreSQL Multi-AZ; Valkey primary/replica; SQS cho tác vụ bất đồng bộ. Tuy nhiên, tài liệu **chưa đủ để cam kết 100–1000 CCU, p99 < 300 ms, không bán quá tồn và deploy không downtime**. Có lỗi cấu hình ALB, thiếu ngân sách tài nguyên rollout và thiếu bảo đảm nhất quán trong luồng đặt hàng.

**Ngân sách 5–6 triệu có thể đạt với lab theo kết cấu S**, giữ 2 AZ, 1 NAT, ALB internal, EKS managed node group với 2 node khi mở lab, RDS Multi-AZ và 2 node Valkey. Phương án đề xuất:

- Giữ 2 × `m7g.large`, không cần chuyển node EKS sang CPU burstable.
- Thu RDS về `db.t4g.small` Multi-AZ; Valkey về 2 × `cache.t4g.micro` ở hai AZ.
- Mục tiêu ban đầu **10–20 CCU**, chỉ thử lên 50 CCU sau khi đo.
- Giới hạn **24–32 giờ tính phí compute/tháng**, đã gồm thời gian khởi động và kết thúc buổi học.
- Dự toán khoảng **5,30–5,36 triệu trước thuế**, hoặc **5,83–5,90 triệu khi cộng 10% dự phòng**, với tháng quy ước 730 giờ. Tháng 31 ngày: khoảng **5,38–5,45 triệu trước thuế**, **5,92–5,99 triệu với dự phòng**.

**Nếu phải giữ kết cấu M**, gồm 3 AZ, 3 NAT và nhóm node hệ thống/app riêng, mục tiêu này không khả thi: ngay cả giả định tất cả EC2/RDS đã nghỉ, thu Valkey xuống micro và chuyển edge sang trả theo dùng, một phần phí cố định đã khoảng **7,04 triệu/tháng**, chưa gồm storage và các service còn lại. Chuyển M sang S sẽ thay đổi kết cấu, nên không được xem là tối ưu hợp lệ theo ràng buộc người dùng.

Phương án S giữ đầy đủ service/addon thuộc S trong tài liệu. Nếu yêu cầu giữ cả Karpenter, KEDA, Prometheus, Grafana và Argo CD của M, phải giữ chúng trong phạm vi và tính lại tài nguyên; không được âm thầm bỏ để đạt ngân sách S.

## 2. Những phần đúng

| Nội dung trong file nguồn | Đánh giá |
| --- | --- |
| Mục 1.1, dòng 47–68: đổi hành vi người dùng thành RPS, request/tháng và GB/tháng | Các phép tính đúng với giả định đã nêu. Tỉ lệ CCU trung bình/đỉnh 25% là giả định, không phải thông số mặc định của ứng dụng. |
| Mục 3, dòng 158–167: S3 OAC và ALB internal làm CloudFront VPC origin | Hợp lệ ở Singapore; cần thêm điều kiện mạng/SG tại mục 3 của review. [AWS VPC origins](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-vpc-origins.html). |
| Mục 5.1, dòng 230–259: Terraform quản lý ALB, controller dùng TargetGroupBinding | Hợp lý để tách vòng đời ALB và cluster. Terraform cần bảo vệ vòng đời tài nguyên phù hợp; việc đổi target group trong blue/green vẫn cần kiểm thử. [TargetGroupBinding](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/targetgroupbinding/targetgroupbinding/). |
| Mục 5.1, dòng 270–271: readiness gate được chèn khi tạo Pod | Đúng; namespace label, Service phù hợp và TargetGroupBinding phải có trước Pod. [Pod readiness gate](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/deploy/pod_readiness_gate/). |
| Mục 6.3, dòng 315–321: S3 gateway endpoint và tính hoà vốn endpoint ECR | Hướng tối ưu hợp lý. Image layer ECR được lưu ở S3; có gateway endpoint thì không nên mặc định tính toàn bộ dung lượng layer vào NAT. [ECR VPC endpoints](https://docs.aws.amazon.com/AmazonECR/latest/userguide/vpc-endpoints.html). |
| Mục 9.1, dòng 494–495: RDS Multi-AZ standby không phục vụ đọc | Đúng đối với Multi-AZ DB instance một standby, khác Multi-AZ DB cluster có reader. [RDS Multi-AZ DB instance](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZSingleStandby.html). |
| Mục 11.2–11.3: cộng tổng chi phí | Cộng lại được **$586,94** và **$2.277,43**; làm tròn $587 và $2.277 đúng. Đây là kiểm tra số học, chưa chứng minh mọi giả định sử dụng đều đủ. |
| Mục 11.4: CloudFront flat-rate | Bảng giá/hạn mức và Business/Premium hỗ trợ VPC origin phù hợp tài liệu AWS hiện hành. Chọn Premium cho 328,5 triệu request theo giả định gốc hợp lý. [CloudFront plans](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/flat-rate-pricing-plan.html). |

## 3. Các điểm cần sửa trước khi dùng như thiết kế triển khai

### 3.1. P1 — `least_outstanding_requests` không đi cùng `slow_start`

**Nguồn:** mục 5.2, dòng **282–283**.

Bảng đang khuyến nghị cả hai thuộc tính cho cùng target group. AWS không hỗ trợ slow start khi dùng least outstanding requests. Chọn một trong hai cấu hình: giữ `least_outstanding_requests` và tắt slow start; hoặc dùng round robin nếu cần slow start. Đây là sửa thuộc tính ALB, không thay đổi kết cấu hạ tầng. [AWS target group attributes](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-target-group-attributes.html).

### 3.2. P1 — Valkey → SQS chưa bảo đảm không bán quá tồn

**Nguồn:** mục 9.2–9.3, dòng **508–535**; yêu cầu không bán quá tồn tại dòng **94**.

Luồng hiện tại có ba điểm hở:

1. Valkey thường dùng replication bất đồng bộ. Failover có thể mất lần giảm tồn vừa được primary xác nhận, rồi cho phép giữ lại số hàng đã bán. Tài liệu không cấu hình durability nên không thể giả định zero data loss. [ElastiCache failover](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/AutoFailover.html).
2. Giảm tồn và gửi SQS là hai thao tác độc lập. API chết hoặc `SendMessage` lỗi giữa hai thao tác có thể làm mất reservation; timeout không rõ kết quả có thể dẫn tới retry sai.
3. SQS Standard có thể giao message nhiều lần. Chỉ gắn idempotency key trong payload chưa thực thi chống trùng. [SQS at-least-once delivery](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html).

Đề xuất với lab giảm tải: vẫn giữ Valkey cho cache/rate-limit/reservation tạm, nhưng **PostgreSQL là nơi quyết định tồn kho và trạng thái đơn bền vững**. Worker phải thực hiện transaction giảm tồn có điều kiện, ghi đơn với UNIQUE idempotency key và chống trùng khi gọi thanh toán. API `202` chỉ có nghĩa đã nhận xử lý, chưa xác nhận chắc chắn giữ được hàng. Nếu muốn xác nhận reservation trước `202`, cần lưu reservation và outbox trong một transaction PostgreSQL rồi chuyển outbox vào SQS bằng worker hiện có. Đây là thay đổi logic ứng dụng trong các service hiện hữu.

Lua có thể gộp kiểm tra tồn và giảm tồn trong Valkey thành một thao tác, nhưng **không tạo transaction chung với SQS/PostgreSQL và không tự giải quyết mất dữ liệu do failover**. Cần định nghĩa reservation hết hạn, hoàn tồn, timeout thanh toán và đối soát.

### 3.3. P1 — CPU đủ ở trạng thái ổn định, chưa đủ cho rollout ở mức tối đa

**Nguồn:** mục 4, dòng **186–214**; mục 7.1, dòng **332–333**; mục 8, dòng **390, 420–421**.

Theo chính các số requests của tài liệu:

- S khi HPA lên 4 API Pod: `4 × 0,5 + 0,25 + 0,6 + 2 × 0,3 = 3,45 vCPU`. Rollout cần thêm 1 Pod: **3,95 vCPU**, vượt tổng 3,8 vCPU allocatable của 2 node.
- M tại flash sale: 15 API + 4 worker dùng 8,5 vCPU. `maxSurge: 25%` cho thêm **4** API Pod, đưa nhu cầu lên **10,5 vCPU**, vượt 9,6 vCPU dành cho workload trên 6 node app.

`maxUnavailable: 0` có thể làm rollout đứng chờ capacity. Cần tính headroom cho surge, Pod đang terminating, RAM, IP và ràng buộc placement. Với S lab, giữ 2 API Pod trong cấu hình ngân sách và tránh rollout đồng thời với load test sát trần; nếu scale node thứ ba phải cộng giờ máy/EBS. [Kubernetes Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

### 3.4. P2 — Managed node group không tự scale chỉ vì đặt `maxSize: 3`

**Nguồn:** mục 7.1, dòng **332** và mục 4.1, dòng **201**.

Managed node group thay node lỗi để duy trì desired size; việc tăng node theo Pod Pending cần cơ chế điều khiển như Cluster Autoscaler hoặc thao tác đổi desired size. File S chưa nêu cơ chế này. Theo ràng buộc không thêm service, lab có thể đổi desired size có chủ đích qua AWS API khi cần, thay vì ngầm giả định HPA tự tăng node. [AWS Cluster Autoscaler](https://docs.aws.amazon.com/eks/latest/best-practices/cas.html).

### 3.5. P2 — Trải AZ chưa đồng nghĩa trải node và chưa bảo đảm chống mọi loại gián đoạn

**Nguồn:** mục 4.1, dòng **192–201**; YAML dòng **428–434**; PDB và giải thích dòng **453–484**.

`topologySpreadConstraints` theo zone không buộc Pod khác hostname nếu có nhiều node trong cùng AZ. Managed node group cũng không phải cam kết cứng luôn có một node ở mỗi AZ trong mọi tình huống capacity. Cần kiểm tra placement thật và thêm ràng buộc theo hostname nếu yêu cầu chống mất một node.

PDB điều tiết eviction chủ động; không ngăn node/AZ hỏng và không trực tiếp điều khiển rollout của Deployment. PDB `maxUnavailable: 1` cũng không tự đảm bảo luôn có Pod thay thế healthy trước mọi lần drain. [Kubernetes disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/), [PDB](https://kubernetes.io/docs/tasks/run-application/configure-pdb/).

### 3.6. P2 — Timeline readiness/draining là ước lượng, không phải bảo đảm zero downtime

**Nguồn:** mục 5.2, dòng **280–281**; mục 8, dòng **394–478**.

Target vừa đăng ký ban đầu chỉ cần một health check thành công; healthy threshold áp dụng khi phục hồi từ unhealthy. Vì vậy câu “Pod mới phải qua 2 lần” không chính xác trong mọi trường hợp. [AWS giải thích initial health check](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/load-balancer-healthcheck.html).

`deregistration_delay` bắt đầu khi target bị deregister, không bắt đầu cùng lúc với preStop. 20 + 10 = 30 giây không phải quan hệ đồng hồ bắt buộc. Giữ app phục vụ trong preStop, xử lý SIGTERM đúng, đặt grace period có biên an toàn và đo 5xx khi rollout dưới tải. Probe chỉ kiểm tra bản thân Pod có thể phù hợp để tránh cascading failure, nhưng phải có quan sát riêng cho lỗi DB/dependency.

### 3.7. P2 — Thiếu đường quản trị/CI đến private EKS API và điều kiện VPC origin

**Nguồn:** mục 3, dòng **162–166**; mục 10, dòng **544–546**.

GitHub-hosted runner thông thường không có tuyến tới private EKS API để `helm upgrade`. Tài liệu cần chỉ rõ runner hoặc đường kết nối **đã có** trong VPC/connected network. Không tự thêm bastion/VPN/runner AWS rồi để ngoài dự toán. Với lab, AWS API điều khiển node group/RDS có thể gọi từ máy quản trị; `kubectl` vẫn cần đường vào private endpoint. [EKS endpoint access](https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html).

VPC origin cần Internet Gateway gắn vào VPC dù không định tuyến trực tiếp vào ALB, địa chỉ IPv4 subnet khả dụng và SG cho CloudFront vào ALB. SG nội bộ ALB → Pod chưa đủ cho cả luồng. VPC origins hiện không hỗ trợ gRPC; cần xác nhận app dùng HTTP phù hợp. [AWS prerequisites](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-vpc-origins.html).

### 3.8. P2 — Chi phí traffic và flash sale dùng hai cách hiểu trung bình khác nhau

**Nguồn:** mục 1.1, dòng **62–68**; mục 11.2, dòng **594–607**; mục 11.3, dòng **614–630**.

Bảng edge tính request từ RPS trung bình, nhưng ALB cộng thêm 30 giờ flash sale trên mức trung bình đó. Nếu trung bình đã bao gồm flash sale thì ALB đang cộng lại; nếu chưa bao gồm thì CloudFront/WAF thiếu request flash sale. Cần định nghĩa baseline và cộng phần tăng thêm thống nhất cho API, ALB, CloudFront, WAF và byte tải ra.

Ví dụ coi 5 RPS của S là baseline và 30 giờ đều có 100 RPS: số API tăng thêm `(100 − 5) × 30 × 3600 = 10,26 triệu`. Edge request tăng tương ứng khi các API vẫn đi qua CloudFront. Số liệu 30 giờ còn chứa thời gian scale trước nên cần tách giờ có tải khỏi giờ giữ capacity.

ALB tính LCU theo **giá trị lớn nhất của bốn chiều**, không mặc định chỉ processed bytes. Reuse connection có lợi nhưng phải kiểm tra cả connection và rule evaluations. [ELB pricing](https://aws.amazon.com/elasticloadbalancing/pricing/).

### 3.9. P2 — SQS, log và một số thành phần chưa được định lượng đủ

**Nguồn:** dòng **214, 545–547, 609, 623, 630**.

- SQS tính API requests, không chỉ số đơn: send, receive, delete, empty receive và các khối payload đều ảnh hưởng. Cần long polling/batching và đo tỷ lệ đơn trên API traffic. Không có cơ sở chắc chắn S luôn dưới 1 triệu request.
- 5 GB log/13,14 triệu API request tương đương khoảng 380 byte/request; M 30 GB/131,4 triệu tương đương 228 byte/request. Nếu ghi JSON dài hoặc cả body, con số dễ vượt. File chưa tách log control plane, WAF/ALB access log, query logs, custom metrics và phí đọc/query.
- M sử dụng Argo CD nhưng chưa có tài nguyên requests cho nó trong phép tính node; Prometheus persistent EBS cần EBS CSI driver, placement theo AZ và ngân sách snapshot. EBS của node flash sale cũng chưa cộng phần giờ lưu trữ.
- Flat-rate bao gồm DNS khi attach hosted zone và có S3 storage credits. Nếu đáp ứng điều kiện, bảng M đang tính dư $0,50 Route 53 và có thể dư $5 S3. Đây là điều chỉnh nhỏ. [CloudFront plan coverage](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/flat-rate-pricing-plan.html).

### 3.10. P2 — Các tuyên bố hiệu năng cần chuyển thành giả thuyết cần đo

**Nguồn:** dòng **44–45, 183–184, 521–535**.

100 RPS/Pod với requests 0,5 vCPU, 1 ms lock PostgreSQL và 0,1 ms Valkey không phải sức chịu được AWS bảo đảm. p99 phụ thuộc query/index, cache hit, TLS, network, pool, payload và tỉ lệ thao tác ghi. Latency dưới 300 ms cho API nhận `202` cũng khác thời gian đơn được xử lý/hoàn tất thanh toán. Đo hai chỉ tiêu riêng.

## 4. Cấu hình lab giữ kết cấu S

| Thành phần | Đề xuất | Điều kiện |
| --- | --- | --- |
| Region/VPC/subnet | Giữ Singapore, 2 AZ, public/private/data subnet | Giữ route, SG, S3 gateway endpoint; không đổi sang public node hoặc NAT instance. |
| EKS | 1 cluster trong standard support | Control plane tồn tại cả tháng; không bật Auto Mode/Fargate hoặc tier tính phí khác trong mô hình này. |
| Node group | Giữ managed node group; 2 × `m7g.large` khi lab mở | Sau buổi lab giảm desired/min xuống 0, giữ node group; khi mở khôi phục 2 node và kiểm tra phân bố AZ. |
| API/worker | Giữ 2 API Pod, mỗi Pod requests 500m/1 GiB; giữ worker | Không scale khỏi cấu hình này trong ngân sách chuẩn. Đo mức dùng worker/addon và RAM; không hạ requests chỉ để ép Pod vào node. |
| Addon S | Giữ VPC CNI, kube-proxy, CoreDNS, metrics-server, Load Balancer Controller, Pod Identity agent, External Secrets, Fluent Bit | Deployment/Service/Secret giữ trong cluster khi node về 0. |
| ALB | Giữ internal, TargetGroupBinding, target `ip`, listener/target group | ALB tồn tại 24/7; ngoài giờ lab có thể trả lỗi vì không có target, không cam kết app liên tục hoạt động. |
| NAT Gateway | Giữ 1 NAT và EIP 24/7 | NAT không có thao tác stop. Không giả định tắt EC2 sẽ tắt NAT. |
| RDS PostgreSQL | `db.t4g.small` Multi-AZ, gp3 **20 GB cho lab dựng mới** | Dừng ngoài giờ; primary/standby vẫn thuộc cấu hình Multi-AZ. Giới hạn pool: API khoảng 5 kết nối/Pod, worker 2; chừa kết nối quản trị. |
| ElastiCache Valkey | 2 × `cache.t4g.micro`, 1 primary + 1 replica khác AZ | Bật Multi-AZ/automatic failover; chạy 24/7. Mỗi node 0,5 GiB, replica không tăng dung lượng cache hữu dụng. |
| CloudFront/WAF | Giữ pay-as-you-go, VPC origin; 1 ACL và 6 rule tính phí như file gốc | Không dùng Pro vì thiếu VPC origin; Business $200 riêng edge đã gần hết ngân sách lab. Tránh rule trả phí bổ sung nếu chưa lập ngân sách. |
| CloudWatch | Giữ Fluent Bit, khoảng 10 alarm; log gọn, retention 7–14 ngày | Dự toán khoảng $2; không ghi body/token, không bật debug toàn thời gian. |
| S3/ECR/Secrets/KMS/SQS/Route 53 | Giữ tất cả | Lab mới: S3 10 GB, ECR 2 GB, 3 secret độc lập cần thiết, 1 key, 1 hosted zone; lifecycle image/log phù hợp. Không gộp secret khác quyền truy cập chỉ để tiết kiệm. |

**Lý do giữ `m7g.large`:** hai node chỉ chạy 32 giờ tốn khoảng **$6,53**. Đổi sang `t4g.medium` tiết kiệm khoảng **$3,81/tháng**, gần 99.000 VND, nhưng RAM giảm một nửa và có giới hạn CPU credit. Node được terminate khi scale về 0 không giữ credit để lần sau dùng lại; T4g không có launch credit. Lợi ích nhỏ so với rủi ro làm load test khó diễn giải. [EC2 CPU credits](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/burstable-credits-baseline-concepts.html).

RDS nhỏ dùng CPU burstable: theo dõi credit balance/surplus và bộ nhớ; phần CPU vượt baseline có thể tính thêm phí. Không dùng cấu hình này để mặc định xác nhận tải ghi cao liên tục.

**Nếu lab đã dựng:** không được giảm trực tiếp allocated storage RDS từ 50 GB xuống 20 GB. Phải giữ 50 GB và cộng **$8,28/tháng**, hoặc lập kế hoạch migration riêng. Review này không thực hiện migration. Storage RDS tăng được nhưng không thu nhỏ trực tiếp. [RDS storage](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PIOPS.StorageTypes.html).

## 5. Bảng chi phí có thể tính lại

### 5.1. Các giả định

- Tháng chuẩn 730 giờ; USD/VND **26.000 là giả định kế hoạch**, không phải tỷ giá thị trường được xác nhận.
- `H = 32` là số giờ mỗi node và RDS ở trạng thái phát sinh phí compute. Hai node tương đương **64 instance-hours**. Không hiểu H là tổng giờ của hai node.
- Ví dụ 8 buổi/tháng: mỗi buổi 3 giờ sử dụng và 1 giờ cho khởi động/kết thúc; nếu RDS chậm phải tính thời gian thực tế, giảm giờ sử dụng hoặc chấp nhận tăng tiền.
- Đỉnh mục tiêu 20 CCU; trung bình **trong giờ tính tải** là 5 CCU. Ngoài giờ không có tải nghiệp vụ; lượng scan internet/WAF phải theo dõi riêng.
- Giữ giả định 0,2 API request/s, 0,5 tổng request/s, 10 KB/s mỗi người. Mô hình không cộng flash sale liên tục hoặc benchmark lớn ngoài lịch.
- 32 giờ → 115.200 API request, 288.000 CloudFront request, khoảng 5,76 GB trả về người dùng. Dùng đơn vị KB/GB thập phân nhất quán với phép tính file gốc.
- Root EBS: 2 × 20 GB, **tính bảo thủ đủ cả tháng** dù node group scale về 0 thường xoá root volume. RDS storage và cache tính đủ tháng.

### 5.2. Dự toán lab mới, 730 giờ/tháng

| Hạng mục | Cách tính | USD/tháng |
| --- | --- | ---: |
| EKS control plane | 0,10 × 730 | 73,000 |
| 2 node `m7g.large` | 2 × 0,102 × 32 | 6,528 |
| Root EBS gp3 | 2 × 20 × 0,096, bảo thủ 24/7 | 3,840 |
| ALB | 0,0252 × 730 + dự phòng LCU 0,10 | 18,496 |
| 1 NAT + IPv4 + 10 GB NAT | (0,059 + 0,005) × 730 + 10 × 0,059 | 47,310 |
| RDS `db.t4g.small` Multi-AZ | 0,102 × 32; giá này đã gồm deployment Multi-AZ, không nhân đôi thêm | 3,264 |
| RDS gp3 Multi-AZ 20 GB | 20 × 0,276 | 5,520 |
| Valkey 2 × `cache.t4g.micro` | 2 × 0,0192 × 730 | 28,032 |
| CloudFront | Khoản dự phòng, không bắt buộc dựa vào free tier | 2,000 |
| WAF | 5 + 6 × 1 + 0,288 triệu × 0,60 | 11,173 |
| CloudWatch logs/retention/alarms | Hạn mức dự toán | 2,000 |
| Secrets Manager | 3 × 0,40 | 1,200 |
| KMS | 1 key | 1,000 |
| ECR | 2 GB × 0,10 | 0,200 |
| Route 53 | 1 hosted zone; alias đến CloudFront | 0,500 |
| S3 storage | 10 GB × 0,025 | 0,250 |
| S3 requests | Dự phòng | 0,300 |
| SQS | Dự phòng request, không coi message là request duy nhất | 0,500 |
| Traffic giữa AZ | Dự phòng khoảng 10 GB tính hai chiều | 0,200 |
| Backup dư/API khác | Dự phòng nhỏ; cần đo | 1,000 |
| **Tổng chưa thuế/phí** | Tính từ số chưa làm tròn từng dòng | **206,313** |
| **Quy đổi VND** | × 26.000 | **5.364.133 VND** |
| **Cộng 10% dự phòng** | Không phải xác nhận thuế suất thực tế | **5.900.546 VND** |

CloudFront pay-as-you-go có free tier 1 TB và 10 triệu request nhưng được chia sẻ theo phạm vi billing; không nên mặc định lab được hưởng toàn bộ. Với lượng tải này, tiền byte/request kể cả chưa miễn phí vẫn thấp hơn khoản $2 dự phòng. [CloudFront billing FAQ](https://aws.amazon.com/cloudfront/faqs/).

Khoản LCU $0,10 giả định processed bytes là chiều lớn nhất, không nhiều kết nối mới/rule evaluations. Đây là allowance theo tải nhỏ, không phải trần tính phí. Tải bất thường, paid managed rules, CPU credit, log/query và backup có thể vượt dự toán.

### 5.3. Giờ chạy là biến quan trọng hơn CCU

Giữ các giả định trên, tại 730 giờ/tháng:

```text
USD(H) = 196,348 + 0,3114 × H

0,3114 = 2 × 0,102       # hai node m7g.large
         + 0,102        # RDS t4g.small Multi-AZ
         + 0,0054       # request WAF với trung bình 5 CCU

VND(H) = USD(H) × 26.000
```

Phần $196,348 gồm phí cố định và các allowance đã nêu, nên không phải hoá đơn tối thiểu tuyệt đối ở tải bằng 0. LCU đang được dự toán trong allowance riêng, không nhân lại với H.

| Giờ compute H/tháng | Trước thuế/phí | Cộng 10% dự phòng |
| ---: | ---: | ---: |
| 24 | 5,30 triệu | 5,83 triệu |
| **32** | **5,36 triệu** | **5,90 triệu** |
| 40 | 5,43 triệu | 5,97 triệu |
| 80 | 5,75 triệu | 6,33 triệu |
| 120 | 6,08 triệu | 6,68 triệu |
| 160 | 6,40 triệu | 7,04 triệu |
| 730 | 11,02 triệu | 12,12 triệu |

Nếu tháng có T giờ, thêm `(T − 730) × 0,2276 USD` cho EKS, NAT/IPv4, ALB và Valkey. Tháng 31 ngày có 744 giờ, thêm khoảng **$3,19**, tức 82.846 VND trước dự phòng. Khi H = 32, tổng với dự phòng là **5,99 triệu**; H = 40 lên **6,06 triệu**. Vì vậy đề xuất **24–32 giờ**, không lấy 40 giờ làm lịch chắc chắn dưới 6 triệu cho mọi tháng.

Ở 730 giờ, trần 6 triệu trước thuế tương đương khoảng 110 giờ compute; nếu chừa 10% thì khoảng 43 giờ. Tháng 744 giờ và chừa 10% chỉ còn khoảng 33 giờ. Đổi tỷ giá hoặc tăng các allowance phải tính lại, không dùng các ngưỡng này như cam kết cứng.

Nếu giữ nguyên cấu hình dung lượng/phụ trợ S gốc: RDS 50 GB, root 30 GB/node, 10 secrets, ECR 5 GB, S3 50 GB, cộng khoảng **$14,30/tháng** vào bảng. H = 32 khi đó khoảng **5,74 triệu trước thuế**, **6,31 triệu với dự phòng**. Không nên quảng bá 5,90 triệu nếu chưa thu được các khoản này. Chỉ riêng RDS vẫn 50 GB thì cộng $8,28: khoảng 5,58 triệu trước thuế ở H = 32.

### 5.4. Vì sao giữ M không đạt mục tiêu

Lấy giả định rất thuận lợi: EC2 và RDS compute không chạy, Valkey đã thu về 2 micro, CloudFront trả theo dùng, không tính request:

```text
EKS                  = 73,000
3 NAT + 3 IPv4       = 3 × 46,72 = 140,160
ALB hourly          = 18,396
2 Valkey micro      = 28,032
WAF ACL + 6 rules   = 11,000
--------------------------------
Chỉ các khoản này   = 270,588 USD ≈ 7,035 triệu VND
```

Chưa có storage RDS/EBS/Prometheus, ECR/S3, Secrets/KMS, logs, request, thuế hoặc một giờ node hệ thống/app. Giảm CCU hay giảm giờ node không xoá được các khoản này. Giữ Premium $1.000 thì còn vượt xa hơn. Đây là giới hạn ngân sách dưới ràng buộc giữ kết cấu M.

## 6. Tắt/mở lab mà vẫn giữ các service

Không dùng “stop EC2” trực tiếp cho managed node group: Auto Scaling có thể thay node để duy trì desired size. Cách phù hợp là đổi min/desired của node group về 0, giữ max hợp lệ; số node quay lại bằng 2 khi mở. Đây là giảm capacity, không xoá EKS/node group hoặc service. [NodegroupScalingConfig](https://docs.aws.amazon.com/eks/latest/APIReference/API_NodegroupScalingConfig.html).

Quy trình cần xây dựng trên máy/runner quản trị đã có, nằm ngoài các node sẽ bị tắt:

1. Trước giờ lab, start RDS và tăng node group về 2 node. Chờ RDS available, node Ready, addon hoạt động, Pod có readiness gate và target ALB healthy; kiểm tra một node/Pod mỗi AZ. Không chỉ sleep một khoảng cố định rồi coi đã sẵn sàng.
2. Khi kết thúc, ngừng phát sinh đơn mới; cho worker xử lý/đóng tác vụ an toàn, lưu dữ liệu cần giữ. Đưa compute node group về 0 và chờ hoàn tất; dừng RDS. Kiểm tra trạng thái thực tế để không bỏ quên máy.
3. Giữ ALB, NAT/EIP, CloudFront/WAF, S3/ECR, RDS storage, Valkey, Secrets/KMS, Route 53 và các cấu hình Kubernetes.
4. Theo dõi giờ chạy thực tế; nếu RDS vẫn starting/stopping hoặc node còn tồn tại, lấy số đo billing làm căn cứ. RDS start có thể mất từ phút tới giờ.

RDS PostgreSQL **Multi-AZ DB instance** có thể stop/start, vẫn tính storage/backup khi dừng và **tự bật sau 7 ngày dừng liên tục**. Lịch cuối tuần không vướng 7 ngày nhưng kỳ nghỉ dài phải kiểm tra việc tự bật. Không áp quy tắc này máy móc cho Multi-AZ DB cluster. [AWS stop DB instance](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_StopInstance.html).

ElastiCache provisioned không có cơ chế stop/start như RDS; ngân sách tính đủ tháng. Muốn ngừng phí node phải xoá tài nguyên, không thuộc đề xuất giữ nguyên service. [ElastiCache cluster deletion](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/Clusters.Delete.html).

PVC/EBS cần giữ sau khi node terminate phải có vòng đời độc lập với root disk. Root disk, emptyDir và dữ liệu container không phải nơi lưu dữ liệu lab bền vững. [EC2 lifecycle](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html).

Không mua Savings Plan/Reserved Instance cho compute chỉ dùng vài chục giờ/tháng: cam kết vẫn phát sinh dù máy nghỉ. Valkey chạy 24/7 có thể xem xét reserved sau khi xác nhận kích thước và thời gian dùng, nhưng phương án hiện tại **không cần cam kết dài hạn** để đạt ngân sách.

## 7. CCU đề xuất và cách xác nhận tính khả thi

Mức **10–20 CCU** là giới hạn thử ban đầu do người lập kế hoạch đặt ra, không phải benchmark đã đạt. Theo giả định file gốc:

| CCU đỉnh | API thông thường | API lúc tranh mua |
| ---: | ---: | ---: |
| 10 | 2 RPS | 10 RPS |
| 20 | 4 RPS | 20 RPS |
| 50, chỉ thử sau | 10 RPS | 50 RPS |

Vẫn giữ requests 500m/1 GiB và hai API Pod; việc hạ CCU cho phép giảm DB/cache và giới hạn chi phí biến đổi. Không tự kết luận giảm 5 lần CCU thì giảm 5 lần máy/tiền. EKS + NAT/IPv4 + ALB hourly + Valkey micro riêng đã khoảng **$166,16**, tương đương 4,32 triệu/tháng tại 730 giờ.

Điều kiện trước khi chấp nhận 20 CCU:

- Trong giờ lab, chạy tải 20 CCU đúng hành vi đọc/ghi; thử đỉnh ngắn 30 CCU. Nếu thử flash sale thì dùng tần suất API 1/s/người thay vì 0,2/s/người. Tổng thời gian benchmark phải nằm trong H.
- Đo p99 API ở vị trí client xác định, lỗi 5xx/timeout và độ trễ hoàn tất đơn riêng. Chưa đo thì không cam kết p99 < 300 ms.
- Đo CPU requests/actual, memory, DB connections, RDS CPU credits, cache memory/evictions và tuổi message SQS. Giữ cache dataset nhỏ; không đặt cả dữ liệu lab vào 0,5 GiB.
- Rollout khi có tải, kiểm tra graceful shutdown/readiness gate và capacity surge. Diễn tập mất một node trong buổi riêng; phân biệt với mất AZ vì S chỉ có một NAT.
- Retry đơn, giao SQS trùng, lỗi gửi message và failover cache phải không tạo đơn/thanh toán trùng hoặc âm tồn trong DB.
- Đối chiếu chi phí thật trong tuần đầu rồi cập nhật allowance. Theo dõi cả traffic ngoài giờ; WAF pay-as-you-go vẫn tính request bị chặn, nên lab nghỉ không đồng nghĩa traffic edge bằng 0.

Nếu không đạt: giảm CCU về 10 hoặc giảm độ phức tạp/dung lượng dữ liệu lab, sửa query/index/pool và luồng đơn trước. Chỉ tăng cấu hình sau khi tính lại ngân sách. Không xoá service, chuyển RDS Single-AZ, bỏ cache replica, đổi NAT hoặc đổi EKS thành ECS để làm đẹp bảng giá.

## 8. Bằng chứng giá và giới hạn kiểm chứng

Các bảng giá AWS công khai được đọc bằng HTTP GET **vào RAM**, lọc SKU rồi tính bằng Python sẵn có. Không tải/cài phần mềm, không mua cam kết và không truy cập hay thay đổi tài khoản AWS.

| Service | Catalog công khai / ngày publication | SKU hoặc rate code đã đối chiếu |
| --- | --- | --- |
| EC2 | [Singapore JSON](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonEC2/current/ap-southeast-1/index.json), 2026-10-09 | `m7g.large`: `JG4ZKG7AYPEUPDCQ.JRTCKXETXF.6YS6EN2CT7`, $0,102/h; NAT hours `98TA3WPUP3A4AH4Y`, $0,059/h; NAT bytes `E9U985ZGMWKXKT47`, $0,059/GB; EBS gp3 `X8SY8CJFS8WV7VQN`, $0,096/GB-tháng. |
| RDS | [Singapore JSON](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonRDS/current/ap-southeast-1/index.json), 2026-10-06 | PostgreSQL `db.t4g.small` Multi-AZ `UKJCZAE8NMNQEK2C`, $0,102/h; `db.t4g.medium` Multi-AZ `3G8THFFBJ944M2HP`, $0,203/h; gp3 Multi-AZ `BGF4DVRTWCBBXHUW`, $0,276/GB-tháng. |
| ElastiCache | [Singapore JSON](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonElastiCache/current/ap-southeast-1/index.json), 2026-09-14 | Valkey micro `BVGCCR29DKNKBPZ8`, $0,0192/h; small `RB5G8C34SDZKRM3M`, $0,0384/h; medium `VTJEJB7V4X2NRW68`, $0,076/h. |
| ALB | [Singapore JSON](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AWSELB/current/ap-southeast-1/index.json), 2026-09-11 | Hourly `UXD3JNG6UAM79CW6`, $0,0252/h; LCU `45KY3J5YX2QM6KS7`, $0,008/LCU-h. |
| VPC public IPv4 | [Singapore JSON](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonVPC/current/ap-southeast-1/index.json), 2026-09-17 | `C8D8VMQRDHMB9KRU`, $0,005/h. |
| WAF | [Pricing](https://aws.amazon.com/waf/pricing/) và [catalog](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/awswaf/current/ap-southeast-1/index.json), 2026-09-14 | Basic $5/ACL-tháng, $1/rule-tháng, $0,60/triệu request. WAF gắn CloudFront dùng scope global; đơn giá basic dùng trong bảng không khác mức này, paid features chưa tính. |
| CloudWatch | [Singapore JSON](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonCloudWatch/current/ap-southeast-1/index.json), 2026-10-07 | Log standard ingestion $0,70/GB; storage $0,03/GB-tháng; standard alarm $0,10/metric-tháng. Bảng lab dùng allowance $2. |
| Secrets/SQS | [Secrets JSON](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AWSSecretsManager/current/ap-southeast-1/index.json), [SQS JSON](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AWSQueueService/current/ap-southeast-1/index.json), 2026-09-11 | $0,40/secret-tháng; Secrets API $0,05/10.000; SQS Standard $0,40/triệu request ở tier đầu, trước free tier. |
| EKS/CloudFront | [EKS pricing](https://aws.amazon.com/eks/pricing/), [CloudFront plans](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/flat-rate-pricing-plan.html) | EKS standard $0,10/h, extended $0,60/h; kiểm tra điều kiện flat-rate và free tier bằng tài liệu hiện hành. |

Đơn giá nhỏ S3/ECR/Route 53/KMS và cross-AZ giữ theo tài liệu gốc để lập ngân sách; chưa kiểm chứng toàn bộ SKU, request tier hoặc từng đường truyền. Phí backup/API và một số dòng là allowance, không phải báo giá AWS. Mức Savings Plan/RI trong file gốc chưa được xác nhận lại theo offer/term cụ thể; không dùng vào phương án lab.

Đã kiểm tra số học tổng S/M, từng dòng lab, công thức H và tháng 28–31 ngày. Chưa xác nhận phiên bản PostgreSQL hỗ trợ instance class trong tài khoản, cache version/giới hạn resize, lượng dataset, quyền IAM, private connectivity hay hiệu năng app. Các điều kiện này phải kiểm tra trước triển khai.
