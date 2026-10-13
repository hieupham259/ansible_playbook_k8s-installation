# Review kiến trúc EKS + ALB: lab giữ nguyên kết cấu M và giới hạn ngân sách

Ngày review: **2026-10-13**. Tài liệu nguồn: [kien-truc-eks-alb-100-1000-ccu.md](kien-truc-eks-alb-100-1000-ccu.md).

Người dùng đã xác nhận đây là **lab, được tắt ngoài giờ**, và yêu cầu **giữ nguyên kết cấu mức M**: 3 AZ, 3 NAT, nhóm node hệ thống/app riêng và đầy đủ các service/addon của M. Review này kiểm tra tài liệu và giá công khai, chưa kiểm tra tài khoản AWS, Terraform state, manifest đang triển khai hoặc chạy load test. Các cấu hình dưới đây là đề xuất, chưa áp dụng lên hạ tầng.

## 1. Kết luận và phạm vi giữ nguyên kiến trúc

Kiến trúc tổng thể khả thi: Route 53 → CloudFront/WAF → VPC origin → ALB internal → EKS; S3 riêng tư qua OAC; RDS PostgreSQL Multi-AZ; Valkey primary/replica; SQS cho tác vụ bất đồng bộ. Tuy nhiên, tài liệu **chưa đủ để cam kết 100–1000 CCU, p99 < 300 ms, không bán quá tồn và deploy không downtime**. Có lỗi cấu hình ALB, thiếu ngân sách tài nguyên rollout và thiếu bảo đảm nhất quán trong luồng đặt hàng.

**Mục tiêu 5–6 triệu VND/tháng không đạt trong phạm vi giữ nguyên M.** Ngay cả giả định tất cả EC2/RDS compute đã nghỉ, thu Valkey xuống micro và chuyển edge sang trả theo dùng, một phần phí cố định đã khoảng **7,04 triệu/tháng**, chưa gồm storage và các service còn lại. Giảm CCU không loại bỏ phí theo giờ của EKS/NAT/ALB/cache.

**Phương án tối ưu áp dụng là M thu nhỏ cho lab:** giữ 3 node hệ thống + 3 node app khi mở lab, 3 AZ/3 NAT, ALB internal, RDS Multi-AZ, cache primary/replica và Karpenter/KEDA/Prometheus/Grafana/Argo CD. Giảm DB/cache/dataset, đổi gói thanh toán edge và giảm giờ compute; không hợp nhất nhóm node, giảm AZ/NAT hay bỏ addon.

- Mục tiêu khởi đầu **10–20 CCU**, chưa được load test xác nhận; giữ 3 API Pod.
- `db.t4g.small` Multi-AZ; 2 × `cache.t4g.micro`; 6 × `m7g.large` chỉ chạy trong giờ lab.
- **24–32 giờ compute/tháng**, gồm khởi động/kết thúc; ngoài giờ capacity hệ thống/app về 0 và RDS dừng, còn EKS/3 NAT/ALB/cache/storage vẫn tính phí.
- Lab dựng mới, dataset nhỏ: khoảng **8,36–8,51 triệu trước thuế**, **9,20–9,36 triệu với 10% dự phòng**, tại 730 giờ/tháng. Tháng 31 ngày: khoảng **8,49–8,64 triệu trước thuế**, **9,34–9,50 triệu với dự phòng**.

Phương án S đã được loại khỏi phần đề xuất sau khi người dùng chọn M. Không dùng kết quả 5–6 triệu của S để quảng bá cho M. Ngân sách phù hợp để lập kế hoạch M lab nhỏ là khoảng **9,5 triệu/tháng có dự phòng**, cần cộng thêm nếu giữ dataset/storage lớn của M gốc.

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

`maxUnavailable: 0` có thể làm rollout đứng chờ capacity. Cần tính headroom cho surge, Pod đang terminating, RAM, IP và ràng buộc placement. Phương án M lab giảm trần replica để giữ capacity rollout trên 3 node app; nếu vẫn dùng cron 15 replica của M gốc thì phải cấp thêm node và tính lại ngân sách. [Kubernetes Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

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

## 4. Phương án áp dụng: lab giữ nguyên M

### 4.1. Thành phần và cấu hình

| Thành phần | Cấu hình lab M đề xuất | Phần được giữ nguyên |
| --- | --- | --- |
| VPC/AZ/subnet | Singapore; 3 AZ; public/private/data subnet mỗi AZ | Kết cấu mạng, route và security boundary. |
| NAT | **3 NAT + 3 EIP**, mỗi AZ một NAT, chạy cả tháng | Không giảm NAT hoặc đổi sang NAT instance. |
| Edge/ALB | CloudFront pay-as-you-go + WAF; VPC origin đến ALB internal do Terraform tạo | Giữ toàn bộ service và luồng request; chỉ đổi phương thức thanh toán từ Premium. |
| EKS | 1 cluster standard support | Control plane tồn tại cả tháng. |
| Hệ thống | Managed node group **3 × m7g.large** khi mở, một node mỗi AZ theo placement thực tế | Karpenter, LBC, CoreDNS, KEDA, metrics-server, External Secrets, Prometheus, Grafana, Argo CD và các agent/DaemonSet. |
| App | Karpenter cấp **3 × m7g.large**, mỗi AZ một node khi mở | Giữ nhóm app riêng; API 3 replica, KEDA/HPA tối đa 4; worker tối đa 3. Không chạy Karpenter trên node do chính nó cấp. |
| PostgreSQL | `db.t4g.small` **Multi-AZ DB instance**, gp3 20 GB nếu dựng mới | Primary/standby khác AZ; stop/start ngoài giờ, storage còn nguyên. |
| Valkey | 2 × `cache.t4g.micro`, primary + replica khác AZ, Multi-AZ/failover bật | Giữ ElastiCache và cache replication. Dung lượng mỗi node 0,5 GiB, chỉ phù hợp dataset lab nhỏ. |
| Quan sát | CloudWatch khoảng 20 alarm; Prometheus/Grafana vẫn tự host; PVC Prometheus 50 GB | Giữ observability stack; giảm log, cardinality và retention phù hợp lab. |
| Dịch vụ khác | S3, ECR, Secrets Manager, KMS, SQS, Route 53 và GitOps | Giữ tất cả; lab mới dùng S3 10 GB, ECR 2 GB, 3 secret độc lập cần thiết, 2 KMS key như M gốc. |

Giữ `m7g.large` giúp tránh CPU credits và giảm rủi ro thiếu RAM khi chạy Prometheus/Argo CD. 6 node chỉ chạy 32 giờ tốn **$19,58**, nên tiền tiết kiệm từ đổi sang burstable tương đối nhỏ. Requests/limits của Argo CD và mọi addon phải được cộng vào capacity trước khi triển khai; số 3,1 vCPU/8,3 GiB hệ thống trong file gốc chưa bao gồm rõ Argo CD.

Karpenter có thể consolidate khiến số app node thực tế giảm. Nếu mục tiêu là giữ một node app mỗi AZ trong giờ lab, phải định nghĩa placement/capacity tương ứng và kiểm tra sau mỗi lần mở; không chỉ nhìn tổng số Pod. Việc tăng thêm node để rollout hoặc diễn tập AZ phải cộng giờ máy. [AWS Karpenter recommendations](https://docs.aws.amazon.com/eks/latest/best-practices/karpenter.html).

Giữ KEDA/CPU scaler nhưng giảm `minReplicaCount`/`maxReplicaCount` của API về **3/4** và cron `desiredReplicas` về **4** nếu có buổi thử scale theo lịch; không giữ cron 15 Pod lúc 19:30 thứ Sáu từ file gốc. Worker scaler giữ service hoạt động, giới hạn tối đa 3 worker cho lab nhỏ. Đây là giảm capacity/autoscaling theo CCU, không loại bỏ KEDA hoặc đổi nhóm compute. 4 API + 3 worker dùng khoảng 2,75 vCPU requests; surge 1 API đưa lên 3,25 vCPU, dưới 4,8 vCPU workload trên 3 node app theo giả định gốc. Vẫn phải kiểm tra RAM, bin packing, placement và Pod terminating thực tế.

Chuyển Premium sang pay-as-you-go không thay kết cấu CloudFront/WAF/VPC origin. Với tải lab vài trăm nghìn request/tháng, Premium $1.000 không phù hợp. Việc chuyển gói thật còn phụ thuộc điều kiện billing/eligibility và thời điểm áp dụng; phương án này không thực hiện thay đổi subscription.

### 4.2. Bảng giá lab M, H = 32, tháng 730 giờ

H là số giờ tính phí của mỗi node và RDS. Hai nhóm 3 node có tổng **192 EC2 instance-hours** ở H = 32; không được tính thành 32 giờ cho toàn bộ 6 máy. Root EBS tính bảo thủ cả tháng; PVC Prometheus luôn tính cả tháng.

| Hạng mục | Cách tính | USD/tháng |
| --- | --- | ---: |
| EKS | 0,10 × 730 | 73,000 |
| 3 node hệ thống | 3 × 0,102 × 32 | 9,792 |
| 3 node app | 3 × 0,102 × 32 | 9,792 |
| Root EBS | 6 × 20 GB × 0,096, bảo thủ cả tháng | 11,520 |
| Prometheus EBS | 50 GB × 0,096 | 4,800 |
| ALB + LCU allowance | 0,0252 × 730 + 0,10 | 18,496 |
| **3 NAT + IPv4 + 10 GB tổng NAT data** | 3 × (0,059 + 0,005) × 730 + 10 × 0,059 | **140,750** |
| RDS compute | 0,102 × 32, giá Multi-AZ đã gồm deployment | 3,264 |
| RDS gp3 Multi-AZ | 20 × 0,276 | 5,520 |
| Valkey 2 micro | 2 × 0,0192 × 730 | 28,032 |
| CloudFront | Allowance tải lab nhỏ | 2,000 |
| WAF | 5 + 6 + 0,288 triệu × 0,60 | 11,173 |
| CloudWatch | 20 alarm + log/retention allowance | 3,000 |
| Secrets | 3 × 0,40 | 1,200 |
| KMS | 2 key | 2,000 |
| ECR | 2 GB | 0,200 |
| Route 53 | 1 zone | 0,500 |
| S3 storage | 10 GB | 0,250 |
| S3 requests | Allowance | 0,300 |
| SQS requests | Allowance | 0,500 |
| Cross-AZ | Allowance | 0,200 |
| Backup dư/API khác | Allowance | 1,000 |
| **Tổng chưa thuế/phí** | Cộng số chưa làm tròn từng dòng | **327,289** |
| **Quy đổi × 26.000 VND/USD** | Tỷ giá giả định | **8.509.509 VND** |
| **Cộng 10% dự phòng** | Không phải xác nhận thuế suất | **9.360.460 VND** |

Trung bình 5 CCU trong 32 giờ tương đương 288.000 request CloudFront, 115.200 API request và 5,76 GB dữ liệu theo giả định gốc. Khoản CloudFront $2 không cần dựa hoàn toàn vào free tier. Traffic scan ngoài giờ, CPU credits RDS, backup/log lớn và các WAF paid features có thể vượt allowance.

```text
USD_M(H) = 304,268 + 0,7194 × H

0,7194 = 6 × 0,102 + 0,102 + 0,0054
         # 6 node    # RDS   # request WAF

Nếu tháng có T giờ: thêm (T − 730) × 0,3556 USD
```

Các allowance trong phần cố định là khoản kế hoạch, không phải hoá đơn tối thiểu tuyệt đối. Công thức giả định 6 node và RDS có cùng H; nếu khác phải tính riêng `0,102 × tổng EC2 instance-hours + 0,102 × giờ RDS` và cộng request thực tế.

| Giờ compute H/tháng | Trước thuế/phí | Cộng 10% dự phòng |
| ---: | ---: | ---: |
| 0, chưa mở buổi nào, vẫn giữ các allowance | 7,91 triệu | 8,70 triệu |
| **24** | **8,36 triệu** | **9,20 triệu** |
| **32** | **8,51 triệu** | **9,36 triệu** |
| 40 | 8,66 triệu | 9,53 triệu |
| 80 | 9,41 triệu | 10,35 triệu |
| 160 | 10,90 triệu | 11,99 triệu |
| 730 | 21,57 triệu | 23,72 triệu |

Tháng 744 giờ tăng khoảng **$4,98**, tương đương 129.438 VND trước dự phòng. H = 32 khi đó khoảng **8,64 triệu trước thuế**, **9,50 triệu với dự phòng**. H = 0 vẫn vượt mục tiêu, nên không có số giờ H không âm nào giải được ngân sách 6 triệu trong mô hình này.

Nếu phải giữ storage/dataset M gốc: RDS 100 GB, root 30 GB/node, 10 secrets, ECR 10 GB và S3 200 GB, cộng khoảng **$36,19/tháng** so với lab nhỏ. H = 32 khi đó khoảng **9,45 triệu trước thuế**, **10,40 triệu với dự phòng**, tại 730 giờ; Prometheus 50 GB đã tính ở cả hai phương án. Riêng RDS 100 GB không thu nhỏ trực tiếp được: giữ nó cộng **$22,08/tháng**. Dataset không vừa DB/cache nhỏ thì giữ cấu hình lớn hơn và tính lại.

### 4.3. Giới hạn cần chốt với ngân sách M

Đề xuất kế hoạch **khoảng 9,5 triệu/tháng** cho M lab dựng mới, ít dữ liệu và 24–32 giờ compute. Nếu giữ dung lượng M gốc, dùng khoảng **10,5–10,6 triệu/tháng có dự phòng** cho cùng lịch. Đây là dự toán, chưa phải cam kết hóa đơn.

Giảm CCU, giảm thêm giờ máy, chọn Spot hoặc mua reservation cho cache có thể giảm một phần, nhưng không đưa khoản phí cố định xuống 5–6 triệu. Không xoá/dựng lại EKS/NAT/ALB theo buổi để giả định chúng chỉ tính H: các thao tác đó làm thay đổi vòng đời tài nguyên và chưa được người dùng cho phép trong yêu cầu giữ service này.

### 4.4. Chứng minh phí cố định vượt mục tiêu

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

## 5. Tắt/mở lab mà vẫn giữ các service

Với nhóm hệ thống, không dùng “stop EC2” trực tiếp cho managed node group: Auto Scaling có thể thay node để duy trì desired size. Giảm min/desired về 0, giữ max hợp lệ; khi mở khôi phục 3 node. Nhóm app do Karpenter quản lý cần quy trình giảm capacity riêng, không dùng API managed node group cho các node này. [NodegroupScalingConfig](https://docs.aws.amazon.com/eks/latest/APIReference/API_NodegroupScalingConfig.html), [Karpenter disruption](https://karpenter.sh/docs/concepts/disruption/).

Quy trình cần xây dựng trên máy/runner quản trị đã có, nằm ngoài các node sẽ bị tắt:

1. Trước giờ lab, start RDS và tăng managed node group hệ thống về 3 node. Chờ node Ready và Karpenter/addon hoạt động, rồi khôi phục workload để Karpenter cấp 3 node app. Karpenter không thể tự khởi động từ 0 nếu chính controller chưa có node hệ thống để chạy.
2. Khôi phục trạng thái GitOps/autoscaling của giờ lab; chờ RDS available, 3 API Pod có readiness gate và target ALB healthy. Kiểm tra một node hệ thống và một node app mỗi AZ. Không chỉ sleep một khoảng cố định rồi coi sẵn sàng.
3. Khi kết thúc, ngừng phát sinh đơn mới và đóng tác vụ an toàn. Ngăn Argo CD self-heal/KEDA/HPA khôi phục replica trong lúc tắt; hạ workload app có chủ đích, cho Karpenter thu hồi capacity và xác nhận app node về 0 **khi controller còn chạy trên nhóm hệ thống**. PDB, Pod chống disruption hoặc PVC/placement có thể cản thu hồi, nên phải kiểm tra chứ không mặc định tự hoàn tất. [Argo CD automated sync](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/).
4. Sau khi app node về 0 và dữ liệu persistent đã an toàn, giảm managed node group hệ thống về 0 và stop RDS. Giữ EKS, cả hai cấu hình nhóm compute, ALB, 3 NAT/EIP, CloudFront/WAF, S3/ECR, RDS storage, Valkey, Secrets/KMS, Route 53, Prometheus PVC và cấu hình GitOps/KEDA/Kubernetes.
5. Khi mở lại, phục hồi mọi trạng thái GitOps/autoscaling đã tạm ngưng; không bỏ Argo CD/KEDA/Karpenter ra khỏi lab. Theo dõi số giờ từng nhóm thực tế; nếu RDS còn starting/stopping hoặc node còn tồn tại thì tính theo billing. RDS start có thể mất từ phút tới giờ.

RDS PostgreSQL **Multi-AZ DB instance** có thể stop/start, vẫn tính storage/backup khi dừng và **tự bật sau 7 ngày dừng liên tục**. Lịch cuối tuần không vướng 7 ngày nhưng kỳ nghỉ dài phải kiểm tra việc tự bật. Không áp quy tắc này máy móc cho Multi-AZ DB cluster. [AWS stop DB instance](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_StopInstance.html).

ElastiCache provisioned không có cơ chế stop/start như RDS; ngân sách tính đủ tháng. Muốn ngừng phí node phải xoá tài nguyên, không thuộc đề xuất giữ nguyên service. [ElastiCache cluster deletion](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/Clusters.Delete.html).

PVC/EBS cần giữ sau khi node terminate phải có vòng đời độc lập với root disk. Root disk, emptyDir và dữ liệu container không phải nơi lưu dữ liệu lab bền vững. [EC2 lifecycle](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html).

Không mua Savings Plan/Reserved Instance cho compute chỉ dùng vài chục giờ/tháng: cam kết vẫn phát sinh dù máy nghỉ. Valkey chạy 24/7 có thể xem xét reserved sau khi xác nhận kích thước và thời gian dùng; khoản tiết kiệm này không đủ khép khoảng cách giữa M và ngân sách 5–6 triệu. Phương án dự toán không giả định đã mua cam kết dài hạn.

## 6. CCU đề xuất và cách xác nhận tính khả thi

Mức **10–20 CCU** là giới hạn thử ban đầu do người lập kế hoạch đặt ra, không phải benchmark đã đạt. Theo giả định file gốc:

| CCU đỉnh | API thông thường | API lúc tranh mua |
| ---: | ---: | ---: |
| 10 | 2 RPS | 10 RPS |
| 20 | 4 RPS | 20 RPS |
| 50, chỉ thử sau | 10 RPS | 50 RPS |

Vẫn giữ requests 500m/1 GiB và **3 API Pod** trải 3 AZ; việc hạ CCU cho phép giảm DB/cache và giới hạn chi phí biến đổi. Không tự kết luận giảm 5 lần CCU thì giảm 5 lần máy/tiền. EKS + 3 NAT/IPv4 + ALB hourly + Valkey micro riêng đã khoảng **$259,59**, tương đương 6,75 triệu/tháng tại 730 giờ, chưa có WAF/storage/compute.

Điều kiện trước khi chấp nhận 20 CCU:

- Trong giờ lab, chạy tải 20 CCU đúng hành vi đọc/ghi; thử đỉnh ngắn 30 CCU. Nếu thử flash sale thì dùng tần suất API 1/s/người thay vì 0,2/s/người. Tổng thời gian benchmark phải nằm trong H.
- Đo p99 API ở vị trí client xác định, lỗi 5xx/timeout và độ trễ hoàn tất đơn riêng. Chưa đo thì không cam kết p99 < 300 ms.
- Đo CPU requests/actual, memory, DB connections, RDS CPU credits, cache memory/evictions và tuổi message SQS. Giữ cache dataset nhỏ; không đặt cả dữ liệu lab vào 0,5 GiB.
- Rollout khi có tải, kiểm tra graceful shutdown/readiness gate và capacity surge. Diễn tập mất node/AZ trong buổi riêng khi lab mở; M vẫn có NAT từng AZ. RDS/cache failover có gián đoạn, vì vậy không mặc định yêu cầu p99 300 ms được giữ trong cả khoảng failover. Đo cả khả năng phục hồi và độ đúng dữ liệu.
- Retry đơn, giao SQS trùng, lỗi gửi message và failover cache phải không tạo đơn/thanh toán trùng hoặc âm tồn trong DB.
- Đối chiếu chi phí thật trong tuần đầu rồi cập nhật allowance. Theo dõi cả traffic ngoài giờ; WAF pay-as-you-go vẫn tính request bị chặn, nên lab nghỉ không đồng nghĩa traffic edge bằng 0.

Nếu không đạt: giảm CCU về 10 hoặc giảm độ phức tạp/dung lượng dữ liệu lab, sửa query/index/pool và luồng đơn trước. Chỉ tăng cấu hình sau khi tính lại ngân sách. Không xoá service, chuyển RDS Single-AZ, bỏ cache replica, đổi NAT hoặc đổi EKS thành ECS để làm đẹp bảng giá.

## 7. Bằng chứng giá và giới hạn kiểm chứng

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
