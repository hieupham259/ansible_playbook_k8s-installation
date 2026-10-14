# Kiến trúc EKS + ALB tối giản: 1 control plane + 2 worker `t4g.medium`, chạy 24/7

> **Tài liệu này trả lời:** dựng ứng dụng web trên Amazon EKS với cụm nhỏ nhất mà vẫn giữ các thành phần chính
> (CloudFront, WAF, ALB internal, RDS Multi-AZ, Valkey, SQS), thì phục vụ được bao nhiêu người dùng và mỗi tháng tốn
> bao nhiêu khi chạy 24/7.
>
> **Dẫn xuất từ:** [kien-truc-eks-alb-100-1000-ccu.md](kien-truc-eks-alb-100-1000-ccu.md). Khung kiến trúc giữ
> nguyên. Thay đổi chính: cụm EKS chỉ còn **1 control plane + 2 worker `t4g.medium` on-demand**, và bỏ các addon không
> vừa tài nguyên. Phần nào giống tài liệu gốc thì tóm tắt rồi trỏ về mục tương ứng.
>
> **Ví dụ xuyên suốt:** sàn thương mại điện tử nhỏ có flash sale, giống tài liệu gốc.
>
> **Đối tượng:** kỹ sư backend và DevOps.
>
> **Giá:** AWS Price List ngày 2026-10-13, on-demand, 730 giờ/tháng, chưa thuế, tính cho **us-east-1** và
> **Singapore (`ap-southeast-1`)**. Tỷ giá giả định 26.000 đ/USD.
>
> **Cách viết số:** kiểu Việt, dấu chấm ngăn hàng nghìn, dấu phẩy ngăn phần thập phân. Ví dụ `$0,0336` là 0.0336 USD.
> Khối code giữ cú pháp gốc.

---

## Mục lục

1. [Mục tiêu và giả định](#1-mục-tiêu-và-giả-định)
2. [Sơ đồ và thành phần](#2-sơ-đồ-và-thành-phần)
3. [Đi theo một request](#3-đi-theo-một-request)
4. [Tài nguyên node và sức chứa](#4-tài-nguyên-node-và-sức-chứa)
5. [ALB và AWS Load Balancer Controller](#5-alb-và-aws-load-balancer-controller)
6. [Mạng](#6-mạng)
7. [Compute và scaling](#7-compute-và-scaling)
8. [Deploy không downtime](#8-deploy-không-downtime)
9. [Tầng dữ liệu](#9-tầng-dữ-liệu)
10. [Bảo mật, quan sát, CI/CD](#10-bảo-mật-quan-sát-cicd)
11. [Chi phí chạy 24/7](#11-chi-phí-chạy-247)
12. [Khi nào phải nâng cấu hình](#12-khi-nào-phải-nâng-cấu-hình)
13. [Nguồn](#13-nguồn)

---

## 1. Mục tiêu và giả định

### 1.1. Giả định tải

Giữ nguyên giả định của tài liệu gốc (mục 1.1). Khi có số đo thật thì thay vào và tính lại:

- Mỗi người gọi API **0,2 request/giây** lúc thường, **1 request/giây** lúc tranh mua flash sale.
- Mỗi người tải ảnh, JS 0,3 request/giây; tổng dữ liệu khoảng 10 KB/giây.
- Mỗi API request ALB xử lý 16 KB (1 KB gửi lên, 15 KB trả về).
- Một Pod `api` xin 0,5 vCPU và 1 GiB RAM, giữ p99 dưới 300 ms tới khoảng **100 RPS**, khi đó CPU khoảng 70%. Đây là
  giả định, phải đo lại bằng load test.

### 1.2. Sức chứa

| | Cả 2 node chạy | Mất 1 node |
| --- | ---: | ---: |
| RPS API bền vững | khoảng 200 | khoảng 100 |
| CCU lúc thường | khoảng 1.000 | khoảng 500 |
| **CCU lúc tranh mua** | khoảng 200 | **khoảng 100** |

Mức bảo đảm (khi mất 1 node) **tương đương mức S = 100 CCU** của tài liệu gốc. Cách tính ở mục 4. DB và cache loại nhỏ
(mục 9) có thể nghẽn trước node, nên con số thật phải lấy từ load test.

### 1.3. Hai kịch bản traffic để tính tiền

- **A. Dev/lab:** trung bình 5 CCU cả tháng, đỉnh khoảng 20 CCU. Mỗi tháng 2,63 triệu API request, 6,57 triệu request
  qua CloudFront, 131,4 GB dữ liệu ra.
- **B. Chạy đủ 100 CCU:** traffic như mức S của tài liệu gốc, gồm cả 10 đợt flash sale. Mỗi tháng 21,69 triệu API
  request, 43,43 triệu request qua CloudFront, 832,5 GB dữ liệu ra.

### 1.4. Yêu cầu

- p99 latency của API dưới 300 ms: 99% request API phản hồi trong 300 ms. Đo ở client và ở `TargetResponseTime` của
  ALB, mỗi lần đo đủ vài nghìn request. Với đặt hàng, chỉ tiêu này áp cho lúc API trả `202`; thời gian xử lý xong đơn
  qua SQS đo riêng. Nếu chọn us-east-1 mà người dùng ở Việt Nam thì yêu cầu này gần như không đạt (mục 11.6).
- Deploy không gây downtime.
- Không bán quá số hàng tồn.
- Chịu được mất 1 node. Hai node nằm ở hai AZ khác nhau, nên mất trọn 1 AZ cũng chỉ mất 1 node; khi đó sức chứa giảm
  còn khoảng một nửa.

---

## 2. Sơ đồ và thành phần

```text
 Người dùng (web / app)
        │
        ▼
 Route 53  (shop.example.com → CloudFront)
        │
        ▼
 CloudFront + AWS WAF ─────────────────────────► S3 riêng tư (ảnh, JS, CSS), đọc qua OAC
        │  /api/* không cache, đi qua VPC origin
        ▼
┌─────────────────────── VPC 10.0.0.0/16, 3 AZ ─────────────────────────┐
│                                                                       │
│  Public subnet    NAT Gateway (mỗi AZ 1 cái)                          │
│                                                                       │
│  Private subnet   ALB internal (Terraform tạo)                        │
│                     │  target-type ip: gửi thẳng tới IP của Pod       │
│                     ▼                                                 │
│                   ┌─────── EKS cluster (1 control plane) ───────┐     │
│                   │ 1 node group: 2 × t4g.medium, 2 AZ          │     │
│                   │ Addon: Load Balancer Controller, CoreDNS,   │     │
│                   │        metrics-server, External Secrets     │     │
│                   │ App:   api × 2, worker × 1 (đọc SQS)        │     │
│                   └─────────────────────────────────────────────┘     │
│                                                                       │
│  Data subnet      RDS PostgreSQL Multi-AZ · ElastiCache Valkey        │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
  Dịch vụ cấp region: SQS · ECR · Secrets Manager · S3 (qua gateway endpoint)
```

| Thành phần | Cấu hình đề xuất | So với lab cũ (6 × `m7g.large`) |
| --- | --- | --- |
| Region | us-east-1 hoặc Singapore (so sánh ở mục 11.6) | Singapore |
| AZ | 3; node chỉ ở 2 AZ | 3 |
| NAT Gateway | 3, mỗi AZ một cái | Giữ nguyên |
| Edge | CloudFront trả theo dùng + WAF; VPC origin tới ALB internal | Giữ nguyên |
| EKS control plane | 1, do AWS quản lý | Giữ nguyên |
| Worker | **1 managed node group, 2 × `t4g.medium` on-demand**, chạy chung addon và app | 3 node hệ thống + 3 node app `m7g.large` |
| Addon | VPC CNI, kube-proxy, CoreDNS, Pod Identity Agent, Load Balancer Controller, metrics-server, External Secrets, Fluent Bit | Bỏ Karpenter, KEDA, Prometheus, Grafana, Argo CD, EBS CSI driver |
| `api` | 2 Pod cố định | 3–4 Pod |
| `worker` | 1 Pod | Tối đa 3 |
| PostgreSQL | RDS `db.t4g.small` Multi-AZ, gp3 20 GB | Giữ nguyên |
| Valkey | 2 × `cache.t4g.micro` (primary + replica khác AZ), Multi-AZ | Giữ nguyên |
| Quan sát | CloudWatch: metric có sẵn, log qua Fluent Bit, khoảng 20 alarm | Bỏ Prometheus và Grafana |
| Triển khai | `helm upgrade` từ máy có đường vào API EKS | Bỏ Argo CD |
| Giờ chạy | 24/7 | |

EKS control plane do AWS chạy và quản lý; bạn không chọn số máy và không nhìn thấy chúng. Theo tài liệu AWS, mỗi cluster
có ít nhất 2 API server và 3 node etcd trải trên 3 AZ, giá cố định $0,10/giờ.

---

## 3. Đi theo một request

Giống tài liệu gốc (mục 3):

1. **Route 53** trả địa chỉ CloudFront cho `shop.example.com`.
2. **CloudFront + WAF:** TLS của trình duyệt kết thúc ở edge gần người dùng. WAF giới hạn tốc độ theo IP và áp managed
   rules. Ảnh, JS, CSS lấy từ S3 riêng tư qua OAC. Request `/api/*` không cache, đi qua **VPC origin** tới ALB
   internal, không qua internet.
3. **ALB** với `target-type: ip` gửi thẳng tới IP của Pod `api`, không vòng qua NodePort.
4. **Pod `api`** đọc Valkey trước, trượt cache mới đọc PostgreSQL. Lệnh đặt hàng đi qua SQS (mục 9).

Điều kiện của VPC origin:

- VPC phải gắn Internet Gateway (chỉ để đánh dấu VPC nhận traffic từ internet, không dùng để định tuyến vào ALB).
- Subnet private còn ít nhất một IPv4 trống để CloudFront tạo ENI.
- Security group của ALB cho inbound từ managed prefix list của CloudFront, hoặc từ security group
  `CloudFront-VPCOrigins-Service-SG`.
- Outbound NACL cho phép cổng tạm 1024–65535 trên đường trả về.
- Không hỗ trợ gRPC.
- Ở us-east-1, VPC origin không hỗ trợ AZ `use1-az3`; chọn AZ theo AZ ID, vì tên `us-east-1a`... ứng với AZ ID khác
  nhau giữa các tài khoản.

---

## 4. Tài nguyên node và sức chứa

### 4.1. Một node `t4g.medium` còn bao nhiêu cho Pod

`t4g.medium` có 2 vCPU, 4 GiB RAM, tối đa 17 Pod (VPC CNI không bật prefix delegation).

| | CPU | RAM | Số Pod |
| --- | ---: | ---: | ---: |
| Máy | 2 vCPU | 4 GiB, hệ điều hành thấy khoảng 3,7 GiB | 17 |
| kubelet và hệ điều hành giữ lại | khoảng 0,07 vCPU | 255 MiB + 11 MiB × 17 = 442 MiB, cộng 100 MiB ngưỡng đuổi Pod | |
| Allocatable | khoảng 1,93 vCPU | khoảng 3,2 GiB | 17 |
| DaemonSet: VPC CNI, kube-proxy, Pod Identity Agent, Fluent Bit | khoảng 0,3 vCPU | khoảng 0,4 GiB | 4 |
| **Còn cho workload** | **khoảng 1,6 vCPU** | **khoảng 2,8 GiB** | **13** |

Cả cluster còn khoảng **3,2 vCPU, 5,6 GiB, 26 Pod**. Số RAM và DaemonSet là ước lượng; xem số thật bằng
`kubectl describe node`.

### 4.2. Xếp workload

| Workload | Số Pod | CPU | RAM |
| --- | ---: | ---: | ---: |
| Addon: Load Balancer Controller × 2, CoreDNS × 2, metrics-server, External Secrets (ước lượng) | 6–7 | 0,6 | 1,0 |
| `api` | 2 | 1,0 | 2,0 |
| `worker` (RAM giả định 0,5 GiB) | 1 | 0,25 | 0,5 |
| Pod `api` tăng thêm khi rollout | 1 | 0,5 | 1,0 |
| **Tổng lúc rollout** | | **2,35** | **4,5** |
| Sức chứa của cluster | | 3,2 | 5,6 |

- **RAM là giới hạn chính.** Thêm Pod `api` thứ 3, cộng Pod tăng thêm khi rollout, đưa RAM lên khoảng 5,5/5,6 GiB,
  không còn biên an toàn. Vì vậy giữ `api` ở 2 Pod.
- **Khi mất 1 node:** còn khoảng 1,6 vCPU và 2,8 GiB. Mỗi addon 1 bản (khoảng 0,4 vCPU, 0,6 GiB), 1 `api` và 1 `worker`
  vẫn vừa (tổng khoảng 1,15 vCPU, 2,1 GiB). Pod `api` thứ hai nằm chờ cho tới khi managed node group dựng node thay
  thế, thường mất vài phút.

### 4.3. Sức chứa theo CPU

`t4g.medium` là máy **burstable**: mức CPU nền là 20% mỗi vCPU, tức khoảng **0,4 vCPU chạy liên tục** cho cả máy. Theo
giả định, 100 RPS dùng khoảng 0,35 vCPU thật (70% của 0,5 vCPU), nên mỗi node chịu bền vững khoảng **100 RPS**.

- Cả 2 node: khoảng 200 RPS, tức khoảng 1.000 CCU lúc thường hoặc 200 CCU lúc tranh mua.
- Mất 1 node: khoảng 100 RPS, tức khoảng 500 CCU lúc thường hoặc **100 CCU lúc tranh mua**.

Chạy vượt mức nền trong thời gian ngắn thì máy tiêu điểm CPU tích luỹ. T4g mặc định ở chế độ unlimited: chạy vượt mức
nền kéo dài thì không bị bóp CPU nhưng bị tính thêm phí theo vCPU-giờ vượt. Theo dõi `CPUCreditBalance` của node; nếu
số này giảm dần qua các ngày thì đổi loại máy (mục 12).

---

## 5. ALB và AWS Load Balancer Controller

Giống tài liệu gốc (mục 5):

- **Terraform tạo ALB internal, listener và target group**, đặt `lifecycle { prevent_destroy = true }`. Controller chỉ
  gắn Pod vào target group qua `TargetGroupBinding`, nên xoá nhầm manifest Kubernetes không làm mất ALB.
- Nhãn `elbv2.k8s.aws/pod-readiness-gate-inject: enabled` trên namespace bật readiness gate. Webhook chỉ chèn gate lúc
  Pod được tạo và khi `TargetGroupBinding` đã tồn tại.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: shop
  labels:
    elbv2.k8s.aws/pod-readiness-gate-inject: enabled
---
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: api
  namespace: shop
spec:
  serviceRef:
    name: api
    port: 8080
  targetGroupARN: arn:aws:elasticloadbalancing:<region>:111122223333:targetgroup/shop-api/0123456789abcdef
  targetType: ip
  networking:
    ingress:
    - from:
      - securityGroup:
          groupID: sg-0123456789abcdef0
      ports:
      - protocol: TCP
        port: 8080
```

Cấu hình target group:

| Thuộc tính | Giá trị | Vì sao |
| --- | --- | --- |
| Kiểu target | `ip` | Gửi thẳng tới Pod, cần cho readiness gate |
| Health check | `/healthz`, mỗi 10 giây; healthy threshold 2 | Target mới đăng ký chỉ cần 1 lần đạt; threshold 2 áp khi phục hồi từ unhealthy |
| `deregistration_delay` | 30 giây | Khớp với `preStop` và grace period ở mục 8 |
| Thuật toán chia tải | `least_outstanding_requests` | Gửi request cho Pod đang ít việc nhất |
| `slow_start` | Không bật | ALB không cho bật cùng `least_outstanding_requests` |

---

## 6. Mạng

VPC `10.0.0.0/16`, mỗi AZ ba subnet:

| Subnet | Cỡ mỗi AZ | Chứa |
| --- | --- | --- |
| Public | /24 | NAT Gateway |
| Private | /19 | ALB internal, node EKS, mọi Pod |
| Data | /24 | RDS, ElastiCache |

- Managed node group chỉ dùng subnet private của **2 AZ**, nên mỗi AZ có một node.
- **3 NAT, mỗi AZ một cái.** Pod cần đường ra internet để gọi cổng thanh toán, kéo image từ registry ngoài AWS và gọi
  API của AWS. AZ thứ ba không có node nên NAT ở đó không phục vụ Pod nào; bỏ nó tiết kiệm $36,50/tháng (us-east-1)
  hoặc $46,72/tháng (Singapore), đổi lại lệch kết cấu "mỗi AZ một NAT" (mục 11.5).
- **Gateway endpoint cho S3:** miễn phí, luôn bật. ECR lưu layer của image trên S3, nên phần lớn dung lượng image không
  đi qua NAT.
- **Không dùng interface endpoint:** mỗi endpoint trải nhiều AZ tốn vài chục USD/tháng, vượt xa số tiền NAT tiết kiệm
  được ở quy mô này.

---

## 7. Compute và scaling

- **Một managed node group:** `t4g.medium` on-demand, `minSize: 2`, `desiredSize: 2`, `maxSize: 2`. Không dùng Spot để
  node không bị thu hồi giữa chừng. Managed node group tự thay node hỏng để giữ đủ 2 node, nhưng không tự thêm node khi
  Pod nằm chờ; muốn thêm node thì tăng `desiredSize` và `maxSize` bằng tay.
- **`api`: 2 Pod cố định, không dùng HPA**, vì RAM không đủ cho Pod thứ 3 cùng Pod tăng thêm khi rollout (mục 4.2).
  Muốn bật HPA thì đo RAM thật rồi giảm request (ví dụ còn 512 MiB), hoặc thêm node.
- **`worker`: 1 Pod**, tăng bằng tay khi hàng đợi SQS dồn (theo dõi `ApproximateAgeOfOldestMessage`).
- **Không dùng Karpenter và KEDA:** cụm chỉ có một node group cố định 2 node, nên không có node nào để Karpenter cấp, và
  KEDA tốn thêm tài nguyên mà cụm không còn.
- metrics-server vẫn giữ để dùng `kubectl top` và để bật HPA sau này.

---

## 8. Deploy không downtime

Cơ chế giống tài liệu gốc (mục 8): readiness gate giữ Pod mới chưa Ready cho tới khi ALB xác nhận healthy; `preStop`
chờ 20 giây để ALB kịp ngừng gửi request mới; `deregistration_delay` 30 giây; grace period 60 giây. Các timeline là ước
lượng; xác nhận bằng cách rollout khi đang chạy tải và đếm `HTTPCode_ELB_5XX_Count`, `HTTPCode_Target_5XX_Count`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: shop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: api
    spec:
      terminationGracePeriodSeconds: 60
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: api
      containers:
      - name: api
        image: 111122223333.dkr.ecr.<region>.amazonaws.com/shop-api:1.4.2
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8080
          periodSeconds: 5
        lifecycle:
          preStop:
            sleep:
              seconds: 20
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api
  namespace: shop
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: api
```

- **`maxSurge: 1`, `maxUnavailable: 0`:** mỗi lần tạo thêm 1 Pod mới, chờ nó Ready rồi mới xoá 1 Pod cũ. Pod tăng thêm
  này đã được tính trong mục 4.2.
- **Trải theo node:** 2 node nằm ở 2 AZ, nên trải đều theo node cũng là trải đều theo AZ.
- **PDB `maxUnavailable: 1`:** khi nâng cấp node group, mỗi lần chỉ rút 1 Pod `api`. PDB không điều khiển rollout của
  Deployment và không ngăn được node hỏng.
- `preStop.sleep` cần Kubernetes 1.30 trở lên.

---

## 9. Tầng dữ liệu

### 9.1. PostgreSQL trên RDS

- **`db.t4g.small` Multi-AZ** (2 vCPU burstable, 2 GiB RAM), gp3 20 GB, backup tự động 7 ngày, khôi phục về thời điểm
  bất kỳ (PITR). Standby ở AZ khác, tự chuyển khi máy chính hỏng trong khoảng 1–2 phút.
- Số kết nối: 2 `api` × pool 10 + 1 `worker` × 5 = 25 kết nối, dư nhiều so với giới hạn mặc định của `db.t4g.small`.
- Theo dõi `CPUCreditBalance`. Nếu số này giảm dần qua các ngày, đổi sang `db.t4g.medium` (mục 12).

### 9.2. Valkey trên ElastiCache

- **2 × `cache.t4g.micro`**, mỗi node 0,5 GiB, primary và replica ở hai AZ, bật Multi-AZ.
- Dùng cho cache sản phẩm, session, bộ đếm rate-limit và cổng chặn nhanh của hàng flash sale. Chỉ đặt dữ liệu nhỏ; theo
  dõi `Evictions`.
- Valkey sao chép sang replica bất đồng bộ, khi failover có thể mất vài lệnh ghi vừa xong, nên **không phải nơi quyết
  định tồn kho**.

### 9.3. Đặt hàng flash sale

Giống tài liệu gốc (mục 9.3):

1. Trước flash sale, nạp số tồn từ PostgreSQL vào Valkey.
2. API chạy một Lua script kiểm tra và trừ trong cùng một lệnh; hết hàng thì trả "hết hàng" ngay.
3. API gửi message vào SQS kèm idempotency key rồi trả `202`. `202` chỉ có nghĩa đã nhận yêu cầu.
4. `worker` làm trong một transaction PostgreSQL: `INSERT` đơn với `idempotency_key` `UNIQUE` (trùng thì bỏ qua), rồi
   `UPDATE ... WHERE stock > 0`; không trúng dòng nào thì đơn "hết hàng". Commit xong mới gọi cổng thanh toán, kèm
   idempotency key của cổng thanh toán.

PostgreSQL quyết định tồn kho cuối cùng nên không bán quá tồn. Cần job đối soát đặt lại số tồn trong Valkey theo
PostgreSQL, và quy định thời hạn giữ chỗ, hoàn tồn khi thanh toán lỗi.

---

## 10. Bảo mật, quan sát, CI/CD

| Lớp | Cấu hình |
| --- | --- |
| Bảo mật | WAF (rate-limit, managed rules, 6 rule); ALB internal chỉ nhận traffic qua VPC origin; Pod Identity riêng cho từng service; Secrets Manager + External Secrets; security group riêng cho RDS và Valkey; NetworkPolicy (VPC CNI hỗ trợ sẵn) |
| Truy cập API EKS | Endpoint private chỉ vào được từ mạng đã nối với VPC (VPN, Direct Connect, máy trong VPC). Nếu chưa có, cách không tốn tiền là bật thêm public endpoint và chỉ cho phép dải IP của máy quản trị hoặc runner tự host; đánh đổi là API server lộ ra internet, dù đã giới hạn IP và vẫn phải qua xác thực IAM |
| CI/CD | GitHub Actions build image `arm64`, đẩy lên ECR. Deploy bằng `helm upgrade` từ runner tự host hoặc máy quản trị có đường vào API; runner GitHub-hosted không vào được endpoint private, và dải IP của nó quá rộng để giới hạn. Nếu đặt runner trên một EC2 riêng thì cộng thêm tiền máy đó. Terraform cho hạ tầng |
| Quan sát | Metric có sẵn của ALB, EC2, RDS, ElastiCache, SQS trên CloudWatch (miễn phí); log app qua Fluent Bit vào CloudWatch Logs, khoảng 300 byte mỗi API request; khoảng 20 alarm. Không có Prometheus vì không đủ tài nguyên; xem tài nguyên Pod bằng `kubectl top`, hoặc bật Container Insights (tính thêm phí) |
| Sao lưu | RDS backup tự động 7 ngày, PITR |
| Kiểm chứng | Load test trước khi mở bán; rollout trong lúc chạy tải để đếm 5xx; tắt 1 node để xác nhận sức chứa còn lại; thử giao trùng message, lỗi gửi SQS và failover Valkey |

Các alarm nên có:

- ALB: `HTTPCode_Target_5XX_Count`, `HTTPCode_ELB_5XX_Count`, `TargetResponseTime` p99 vượt 300 ms, `HealthyHostCount`
  dưới 2.
- Node EC2: `CPUCreditBalance` giảm dần.
- RDS: `CPUUtilization`, `CPUCreditBalance`, `FreeStorageSpace`, `FreeableMemory`, `DatabaseConnections`.
- ElastiCache: `EngineCPUUtilization`, `DatabaseMemoryUsagePercentage`, `Evictions`.
- SQS: `ApproximateAgeOfOldestMessage`.

---

## 11. Chi phí chạy 24/7

### 11.1. Đơn giá

| Hạng mục | us-east-1 | Singapore |
| --- | --- | --- |
| EKS control plane | $0,10/giờ | $0,10/giờ |
| EC2 `t4g.medium` | $0,0336/giờ | $0,0424/giờ |
| EBS gp3 | $0,08/GB-tháng | $0,096/GB-tháng |
| ALB | $0,0225/giờ + $0,008/LCU-giờ | $0,0252/giờ + $0,008/LCU-giờ |
| NAT Gateway | $0,045/giờ + $0,045/GB | $0,059/giờ + $0,059/GB |
| IP public (IPv4) | $0,005/giờ | $0,005/giờ |
| RDS PostgreSQL `db.t4g.small` Multi-AZ | $0,065/giờ | $0,102/giờ |
| Ổ RDS gp3 Multi-AZ | $0,23/GB-tháng | $0,276/GB-tháng |
| ElastiCache Valkey `cache.t4g.micro` | $0,0128/giờ | $0,0192/giờ |
| CloudWatch: nạp log / lưu log / alarm | $0,50/GB; $0,03/GB-tháng; $0,10/alarm | $0,70/GB; $0,03/GB-tháng; $0,10/alarm |
| WAF | $5/web ACL + $1/rule mỗi tháng + $0,60/triệu request | Như us-east-1 |
| CloudFront trả theo dùng | Tính theo nơi người xem, không theo region của origin. Người xem ở châu Á: $0,120/GB, $0,012/10.000 request HTTPS; miễn phí mỗi tháng 1 TB và 10 triệu request | Như us-east-1 |
| Secrets Manager / SQS | $0,40/secret-tháng / $0,40/triệu request, 1 triệu đầu miễn phí | Như us-east-1 |
| S3 Standard / ECR / Route 53 / KMS | $0,023/GB / $0,10/GB / $0,50/zone / $1/key | $0,025/GB / $0,10/GB / $0,50/zone / $1/key |

### 11.2. Kịch bản A: dev/lab, chạy 24/7

| Hạng mục | Cách tính | us-east-1 | Singapore |
| --- | --- | ---: | ---: |
| EKS control plane | $0,10 × 730 | 73,00 | 73,00 |
| 2 worker `t4g.medium` | 2 × đơn giá × 730 | 49,06 | 61,90 |
| Ổ gốc của node | 2 × 20 GB × gp3 | 3,20 | 3,84 |
| ALB | đơn giá giờ × 730 + 42 GB xử lý × $0,008 | 16,76 | 18,73 |
| 3 NAT + 3 IP + 10 GB | 3 × (NAT + $0,005) × 730 + 10 GB × đơn giá GB | 109,95 | 140,75 |
| RDS `db.t4g.small` Multi-AZ + 20 GB | đơn giá giờ × 730 + 20 × gp3 Multi-AZ | 52,05 | 79,98 |
| Valkey 2 × `cache.t4g.micro` | 2 × đơn giá × 730 | 18,69 | 28,03 |
| CloudFront | 6,57 triệu request, 131 GB: trong mức miễn phí; dự trù | 2,00 | 2,00 |
| WAF | $5 + 6 rule × $1 + 6,57 triệu × $0,60 | 14,94 | 14,94 |
| CloudWatch | 20 alarm $2; log khoảng 0,8 GB và lưu trữ, dự trù $1 | 3,00 | 3,00 |
| Lặt vặt | 3 secret 1,20; 2 KMS key 2,00; ECR 2 GB 0,20; Route 53 0,50; S3 10 GB + request 0,53 / 0,55; SQS 0,50; giữa AZ 0,20; khác 1,00 | 6,13 | 6,15 |
| **Tổng** (cộng số chưa làm tròn) | | **$348,78** | **$432,33** |
| **Quy ra tiền Việt** | | **9,07 triệu** | **11,24 triệu** |
| **Cộng 10% dự phòng** | | **9,98 triệu** | **12,36 triệu** |

### 11.3. Kịch bản B: chạy đủ 100 CCU

Các dòng thay đổi so với kịch bản A; các dòng khác giữ nguyên:

| Hạng mục | Cách tính | us-east-1 | Singapore |
| --- | --- | ---: | ---: |
| ALB | đơn giá giờ × 730 + 347 GB × $0,008 | 19,20 | 21,17 |
| NAT | dữ liệu qua NAT 30 GB | 110,85 | 141,93 |
| CloudFront | 832,5 GB nằm trong 1 TB miễn phí; (43,43 − 10) triệu request × $1,20 | 40,11 | 40,11 |
| WAF | $11 + 43,43 triệu × $0,60 | 37,06 | 37,06 |
| CloudWatch | 21,69 triệu request × 300 byte = 6,51 GB log; lưu 3 tháng; 20 alarm | 5,84 | 7,14 |
| Lặt vặt | thêm dữ liệu giữa AZ 50 GB × $0,02 | 6,93 | 6,95 |
| **Tổng** | | **$415,98** | **$501,11** |
| **Quy ra tiền Việt** | | **10,82 triệu** | **13,03 triệu** |
| **Cộng 10% dự phòng** | | **11,90 triệu** | **14,33 triệu** |

### 11.4. So với cấu hình cũ

Cấu hình cũ là lab giữ kết cấu M: 3 node hệ thống + 3 node app `m7g.large` và ổ 50 GB cho Prometheus. Cả hai chạy
24/7 với traffic kịch bản A; các thành phần khác giống nhau.

| | Cũ: 6 × `m7g.large` | Mới: 2 × `t4g.medium` | Chênh lệch |
| --- | ---: | ---: | ---: |
| us-east-1 | $667,53 ≈ 17,36 triệu | $348,78 ≈ 9,07 triệu | −8,29 triệu (−47,8%) |
| Singapore | $829,67 ≈ 21,57 triệu | $432,33 ≈ 11,24 triệu | −10,33 triệu (−47,9%) |

Toàn bộ phần chênh nằm ở worker và ổ đĩa của worker: ở us-east-1, worker giảm từ $357,41 xuống $49,06 và EBS giảm từ
$13,60 xuống $3,20.

### 11.5. Tiền đi đâu

Kịch bản A, us-east-1:

| Hạng mục | $/tháng | Tỉ lệ |
| --- | ---: | ---: |
| 3 NAT + IP | 109,95 | 31,5% |
| EKS control plane | 73,00 | 20,9% |
| RDS | 52,05 | 14,9% |
| 2 worker | 49,06 | 14,1% |
| Valkey | 18,69 | 5,4% |
| ALB | 16,76 | 4,8% |
| WAF | 14,94 | 4,3% |
| Còn lại | 14,33 | 4,1% |

Muốn giảm thêm mà không đụng vào cụm EKS, khoản đáng kể nhất là **NAT ở AZ không có node**: bỏ nó còn 2 NAT, tiết kiệm
$36,50 (us-east-1) hoặc $46,72 (Singapore) mỗi tháng. Kịch bản A khi đó còn khoảng **8,12 triệu** (us-east-1) hoặc
**10,03 triệu** (Singapore), chưa gồm dự phòng.

### 11.6. Chọn region

| | us-east-1 | Singapore |
| --- | --- | --- |
| Kịch bản A, chưa dự phòng | 9,07 triệu | 11,24 triệu |
| Độ trễ API với người dùng ở Việt Nam | Mỗi request `/api/*` phải đi từ edge châu Á sang Virginia và quay về, ước lượng 200–250 ms mỗi vòng (cần đo). **p99 < 300 ms gần như không đạt** | Thấp; đạt được p99 < 300 ms nếu app đáp ứng |
| VPC origin | Không hỗ trợ AZ `use1-az3` | Không giới hạn AZ |

- us-east-1 rẻ hơn khoảng 2,2 triệu/tháng. Phù hợp khi người dùng chủ yếu ở Mỹ, hoặc khi chỉ dùng để dev và thử chức
  năng.
- Người dùng ở Việt Nam và cần p99 < 300 ms thì chọn Singapore.
- Giữ phiên bản Kubernetes trong thời gian hỗ trợ chuẩn (14 tháng). Quá hạn thì control plane lên $0,60/giờ, thêm
  $365/tháng.
- Khoản 10% dự phòng không gồm thuế. Nếu hoá đơn AWS chịu VAT hoặc doanh nghiệp phải nộp thuế nhà thầu nước ngoài thì
  cộng thêm theo xác nhận của kế toán. Mỗi 1.000 đ/USD chênh tỷ giá làm tổng kịch bản A đổi khoảng 0,35 triệu
  (us-east-1) hoặc 0,43 triệu (Singapore).

---

## 12. Khi nào phải nâng cấu hình

| Tín hiệu | Việc làm | Chi phí thêm/tháng (us-east-1 / Singapore) |
| --- | --- | ---: |
| `CPUCreditBalance` của node giảm dần, hoặc hoá đơn có phí CPU vượt mức | Đổi 2 node sang `c7g.large` (2 vCPU, 4 GiB, không burst) | +$56,79 / +$59,71 |
| Cần CPU không burst và thêm RAM | Đổi 2 node sang `m7g.large` (2 vCPU, 8 GiB) | +$70,08 / +$87,02 |
| Pod nằm chờ vì thiếu RAM, cần bật HPA hoặc chạy thêm Prometheus | Thêm node thứ 3 `t4g.medium` | +$26,13 / +$32,87 |
| `CPUCreditBalance` của RDS giảm dần | Đổi RDS sang `db.t4g.medium` Multi-AZ | +$46,72 / +$73,73 |
| Valkey bị đẩy key (`Evictions` tăng) | Đổi sang 2 × `cache.t4g.small` | +$18,69 / +$28,03 |
| Cần quá 100 CCU lúc tranh mua | Chuyển sang thiết kế mức S hoặc M trong [tài liệu gốc](kien-truc-eks-alb-100-1000-ccu.md) | Xem mục 11 của tài liệu gốc |

---

## 13. Nguồn

- AWS Price List API, lấy ngày 2026-10-13:
  `https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/<dịch-vụ>/current/<region>/index.json` với `<region>` là
  `us-east-1` và `ap-southeast-1`, các dịch vụ AmazonEC2, AmazonRDS, AmazonElastiCache, AWSELB, AmazonVPC,
  AmazonCloudWatch, AWSSecretsManager, AmazonECR.
- [Tài liệu gốc: kiến trúc EKS + ALB cho 100–1000 CCU](kien-truc-eks-alb-100-1000-ccu.md)
- [Review kiến trúc và ngân sách lab](review-kien-truc-eks-alb-va-ngan-sach-lab.md)
- [CloudFront VPC origins](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-vpc-origins.html)
- [ALB: target group attributes](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-target-group-attributes.html)
- [ALB: health checks](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html)
- [AWS Load Balancer Controller: TargetGroupBinding](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/targetgroupbinding/targetgroupbinding/)
- [AWS Load Balancer Controller: Pod readiness gate](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/deploy/pod_readiness_gate/)
- [Amazon EKS: cluster endpoint access](https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html)
- [Amazon EKS: Kubernetes version lifecycle](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html)
- [ElastiCache: Multi-AZ](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/AutoFailover.html)
- [EC2: burstable performance instances (unlimited mode)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/burstable-performance-instances-unlimited-mode.html)
- [Kubernetes: Pod topology spread constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [Kubernetes: Disruptions và PodDisruptionBudget](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
