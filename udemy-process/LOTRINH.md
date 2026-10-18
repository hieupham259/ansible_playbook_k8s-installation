# Lộ trình học Udemy

> **Ngày lập:** 18/10/2026. Thời lượng, đánh giá và ngày cập nhật của các khoá lấy từ trang Udemy
> ngày này. Những mục bị ẩn trên trang khoá ("N more sections") chưa kiểm được.

---

## 1. Lộ trình 4 khoá

1. https://www.udemy.com/course/devops-projects-using-kubernetes-a-hands-on-guide/ (GitLab CI, Velero)
2. https://www.udemy.com/course/devops-workshop/ (GitOps, quét bảo mật, đưa bản mới qua từng môi trường)
3. https://www.udemy.com/course/prometheus-grafana-bootcamp/
4. https://www.udemy.com/course/elasticsearch-and-elastic-stack/

Sau bốn khoá và phần áp dụng, làm [chặng nghiệm thu năng lực ở mục 4](#4-chặng-nghiệm-thu-năng-lực).
Hoàn thành video chưa đồng nghĩa đã đạt các năng lực ở mục 3; cần có kết quả thực hành và bằng chứng
nghiệm thu. Lộ trình admin được theo dõi riêng theo checkpoint của nó.

---

## 2. Môi trường thực hành

Bạn cần hai môi trường. Khoá 1 và 2 học trên AWS, làm theo đúng giảng viên và xoá tài nguyên sau mỗi
buổi. Phần áp dụng của cả bốn khoá, cùng khoá 3 và 4, làm trên các VM VMware ở nhà.

Máy host của bạn có 31,8 GB RAM. Riêng cụm Phase 1 đã chiếm 20 GB, nên chỉ còn khoảng 12 GB cho
Windows, VMware và mọi VM khác. RAM chính là giới hạn khi xếp lịch, không phải CPU hay đĩa.

### 2.1. DevOps Projects on AWS

**Khoá dùng:**
- AWS, với ba cách dựng cụm: kops (S3, Route 53), EKS, và kubeadm trên EC2 (Packer, Terraform,
  Ansible).
- GitLab CI, Docker Hub.
- Backup: EBS snapshot, Velero lưu vào S3.

**Bạn áp dụng trên cụm Phase 1/2:**
- **Staging và production:** hai namespace trên cùng cụm.
- **Registry:** Docker Hub, như Phase 2 đang dùng.
- **GitLab:** tạo project mirror từ repo GitHub `three-tier-crud`. Runner phải chạy trong cụm theo
  [runbook-kubectl-in-cluster-cicd.md](../runbook-kubectl-in-cluster-cicd.md), vì runner của
  GitLab.com không gọi được API server sau NAT.
- **Velero:** lưu backup vào MinIO thay cho S3.
  - **Velero không backup được dữ liệu MongoDB của bạn.** Velero không hỗ trợ volume kiểu `hostPath`,
    mà local-path-provisioner mặc định tạo `hostPath`. Có hai cách: tạo PVC mới có annotation
    `volumeType: local` (README ghi là tính năng *experimental*, và phải chuyển dữ liệu sang), hoặc
    giữ `mongodump` của Phase 2 §14 cho dữ liệu và chỉ dùng Velero cho các object.
  - **Bài "clone sang cụm thứ hai":** máy không đủ RAM cho cụm thứ hai. Thay bằng restore sang
    namespace mới với `--namespace-mappings`, hoặc làm trên AWS như khoá.
- **Bỏ qua:** phần EBS snapshot (chỉ có trên AWS) và bài kubeadm dùng Weave.

### 2.2. DevSecOps on EKS

**Khoá dùng:**
- Hạ tầng AWS dựng hoàn toàn bằng Terraform: VPC, EKS, ECR, RDS, IAM, Secrets Manager, state lưu
  trên S3.
- GitHub Actions, SonarCloud, Trivy.
- Trên cụm: AWS Load Balancer Controller, Argo CD, External Secrets.

Cài sẵn trên máy: AWS CLI v2, Terraform, kubectl, Helm, Git, Python, VS Code.

Nội dung khoá gắn chặt với dịch vụ AWS, nên học trên AWS.

**Bạn áp dụng trên cụm Phase 1/2:**
- Cài Argo CD vào cụm.
- GitHub Actions (runner của GitHub) build image → push Docker Hub → commit tag mới vào repo GitOps.
  Argo CD tự kéo thay đổi về, chỉ cần kết nối đi ra Internet.
- **Khi chuyển sang Argo CD, tắt job deploy của GitLab ở bước 1, chỉ giữ phần build.** Nếu không,
  Argo CD sẽ ghi đè thứ GitLab vừa deploy.
- **Dev, QA và prod là ba namespace,** tức là ba bản MongoDB. Worker 6 GB sẽ chật, nên có thể chỉ
  giữ dev + prod.
- **Secret:** giữ Kubernetes Secret như Phase 2; External Secrets là phần riêng của AWS.

### 2.3. Prometheus Bootcamp

**Khoá dùng:** VM Ubuntu trên DigitalOcean, Docker, app mẫu viết bằng Python và Go. Phần Kubernetes
cài Prometheus bằng Helm chart của cộng đồng.

**Bạn dùng:**
- **Phần VM:** một VM Ubuntu 24.04 trên VMware. Dùng lại `ubuntu-2404` của Lab A (4 GB) nếu còn,
  nếu không thì clone VM mới. Không cần DigitalOcean.
- **Phần Docker và Python:** Docker trên máy Windows (đã có bản 28.4.0).
- **Áp dụng:** cài kube-prometheus-stack lên cụm Phase 1/2 và thêm metric vào FastAPI. Đo headroom
  của node trước khi cài.

### 2.4. Elasticsearch 9

**Khoá dùng:** một VM Ubuntu trên máy bàn, có bài chạy *"three nodes on a single vm"*. Khoá cần ổ
đĩa trống ≥ 20 GB; trang khoá không ghi cần bao nhiêu RAM.

**Bạn dùng:**
- **Học bài:** một VM Ubuntu riêng, không cài vào cụm. Bài chạy ba node trên một VM thì nên tắt cụm
  khi học.
- **Áp dụng:** chạy DaemonSet Filebeat trong cụm, gửi log tới VM Elasticsearch. Lúc này cụm (20 GB)
  và VM Elasticsearch phải chạy cùng lúc, nên VM chỉ để được khoảng 6 GB, heap nhỏ. Đây là chỗ chật
  nhất.
- **Sau đó nối sang [runbook log tập trung](../runbook-log-tap-trung-vmware.md):** 5 VM, 21 GB,
  phải tắt cụm. Chính runbook đó cũng ghi điều này trong bảng sự cố.

### 2.5. Tài khoản cần có

AWS, GitLab, GitHub, Docker Hub (đã có), SonarCloud.

Nếu sau này muốn cho Grafana và log của cụm chạy cùng lúc mà không phải tắt bớt thứ gì, nâng RAM
host lên 64 GB là cách gọn nhất.

---

## 3. Người học sau khi hoàn tất

### 3.1. Background

- **Đã làm việc trên hai loại môi trường:**
  - AWS: EKS dựng bằng Terraform, cùng kops và kubeadm trên EC2.
  - Cụm kubeadm tự quản trên VMware.
- **Đã đưa một app 3-tier (React → FastAPI → MongoDB) qua trọn vòng đời:** commit → build → quét
  bảo mật → registry → deploy lên nhiều môi trường → monitoring → gom log → backup và restore.
- **Đã dùng cả hai cách triển khai:** CI đẩy thẳng vào cụm (GitLab CI chạy Helm), và cụm tự kéo cấu
  hình từ Git về (Argo CD).
- **Công cụ đã dùng thực tế:** GitLab CI, GitHub Actions, Helm, Kustomize, Argo CD, Terraform,
  SonarCloud, Trivy, Velero, MinIO, Prometheus, Alertmanager, Grafana, Elasticsearch, Logstash,
  Filebeat, Kibana.
- **Đã diễn tập:** rollback một bản lỗi, restore bằng Velero, cố tình gây lỗi để kiểm tra cảnh báo,
  điều tra một sự cố từ cảnh báo tới dòng log gây ra nó.
- **Có sản phẩm để trình bày:** file pipeline, repo GitOps, dashboard Grafana, alert rule và cấu
  hình gom log, tất cả cho cùng một app.

### 3.2. Trình độ

| Mảng | Làm được |
| --- | --- |
| CI/CD đẩy vào cụm | Pipeline GitLab CI: build image → Docker Hub → Helm/Kustomize deploy lên staging → duyệt lên production → rollback. Runner chạy trong cụm bằng ServiceAccount, không cầm kubeconfig. |
| GitOps và bảo mật pipeline | Pipeline GitHub Actions có SonarCloud và Trivy. Argo CD tự đồng bộ cụm theo Git. Đưa cùng một image qua dev → QA → prod bằng PR. |
| Hạ tầng bằng code | Terraform theo module, state trên S3, plan khi mở PR. Bootstrap EKS: Load Balancer Controller, External Secrets, Argo CD. Dựng cụm trên AWS bằng kops, EKS và kubeadm. |
| Backup và DR | Backup và restore object bằng Velero, restore sang namespace khác; biết rõ giới hạn với volume `hostPath`. |
| Metric và cảnh báo | PromQL, exporter, recording rule, alert rule, Alertmanager, dashboard Grafana. Thêm metric request, lỗi, độ trễ vào FastAPI. Chạy kube-prometheus-stack trên cụm. |
| Log | Mapping, Logstash/Grok, Filebeat, Kibana. Chọn số shard, đặt heap, ILM, snapshot. Gom log Pod bằng Filebeat DaemonSet, rồi áp dụng sang lab OpenSearch. |

**Dấu hiệu đã đạt:**

- Đưa phiên bản mới lên staging, duyệt lên production và quay lại phiên bản trước khi có lỗi.
- Một thay đổi trong Git được Argo CD triển khai và hoàn tác được; pipeline tự cập nhật image tag
  trong Git.
- Nhìn dashboard giải thích được ứng dụng đang chậm, lỗi hay thiếu tài nguyên ở đâu; cố tình gây lỗi
  thì cảnh báo bắn.
- Điều tra được một sự cố bằng cả metric và log, lần theo request ID.

### 3.3. Level

Lộ trình hướng tới năng lực **Middle DevOps trong phạm vi triển khai và vận hành ứng dụng
Kubernetes**: tự đưa một dự án từ commit tới môi trường production mô phỏng, có monitoring, log,
backup và khả năng phục hồi. Mức đạt được phải đánh giá bằng mức tự chủ và bằng chứng ở
[chặng nghiệm thu](#4-chặng-nghiệm-thu-năng-lực), không tự động suy ra từ số khoá hoặc công cụ đã học.
Kết quả lab không thay thế bằng chứng vận hành production phục vụ người dùng thật; năng lực quản
trị cluster được đánh giá riêng bằng các checkpoint đã hoàn thành trong lộ trình admin.

**Các phần cần theo dõi riêng ngoài chặng nghiệm thu ứng dụng:**

- HA control plane, nâng cấp cụm, vòng đời certificate: đánh giá theo checkpoint giai đoạn 8 và
  16–19 của lộ trình admin, không mặc định còn thiếu nếu đã nghiệm thu các phần đó.
- Bảo mật và multi-tenancy (giai đoạn 9); baseline Lab 00 dùng Flannel, Lab 5b chuyển sang Calico
  để thực thi NetworkPolicy.
- Tracing và OpenTelemetry, service mesh.
- Load test để tính capacity ở quy mô production, ngoài tải kiểm chứng SLO của lab; kinh nghiệm
  vận hành production thật có on-call.
- Velero cho dữ liệu nằm trên volume `hostPath`, nếu chưa chuyển sang `local`.

---

## 4. Chặng nghiệm thu năng lực

### 4.1. Mục tiêu, môi trường và bằng chứng

Chặng này kiểm chứng khả năng nhận yêu cầu mới, thay đổi hệ thống, điều tra sự cố, phục hồi và bàn
giao. Dùng dự án React → FastAPI → MongoDB sau phần áp dụng của bốn khoá; không làm lại nguyên mẫu
video. Được tra tài liệu và dùng công cụ hỗ trợ, nhưng phải tự giải thích và kiểm chứng kết quả.

- Chỉ thực hiện trên lab có thể phục hồi; namespace `production` ở đây là môi trường mô phỏng.
- Trước mỗi bài, ghi commit/image digest, cấu hình, trạng thái dữ liệu và cách quay về baseline.
  Kiểm tra đủ tài nguyên để ứng dụng, metrics và log cùng hoạt động trong lần nghiệm thu; chưa đủ
  thì điều chỉnh quy mô lab trước khi bắt đầu và ghi lại giới hạn.
- Lưu bằng chứng trong repo dự án, chẳng hạn `evidence/acceptance/`: ticket, PR/diff, kết quả kiểm
  thử, số đo, truy vấn/dashboard và biên bản sự cố. Loại bỏ credential, token, dữ liệu cá nhân;
  không đưa dump database hoặc snapshot etcd chứa Secret vào Git.
- Mỗi bài kết thúc bằng kết luận `PASS` hoặc `STOP`, kèm đường dẫn bằng chứng. Chưa đủ bằng chứng
  hoặc chưa phục hồi baseline thì `STOP`, không đánh dấu hoàn thành.

### 4.2. Mục tiêu độ tin cậy theo trải nghiệm người dùng

Trước khi thay đổi hệ thống, lập một tài liệu SLO cho luồng người dùng, ví dụ tạo bản ghi → đọc
lại đúng nội dung qua endpoint frontend/Ingress. Pod `Running`, probe xanh hoặc trang chủ trả
HTTP 200 chưa đủ để xác nhận luồng này thành công.

**Chốt cách đo trước khi chạy:**

- Ghi endpoint, kết quả mong đợi, dữ liệu thử và cách dọn dữ liệu. Một thao tác thành công phải có
  cả status code đúng và nội dung/kết quả dữ liệu đúng. Định nghĩa trước cách xử lý request sai
  đầu vào; không loại lỗi khỏi thống kê sau khi đã nhìn thấy kết quả.
- Đo từ máy phát request qua đường truy cập đã chọn, ghi rõ có đi qua Cloudflare hay chỉ Ingress.
  Tính cả timeout/lỗi kết nối mà metrics của backend có thể không ghi nhận. Nếu chỉ đo từ LAN,
  kết luận cũng chỉ áp dụng cho đường LAN đó.
- Ghi cửa sổ đo, tổng số thao tác, tải/concurrency và giới hạn timeout. Giữ cùng cấu hình tải và
  đường truy cập khi so sánh trước/sau; lưu kết quả thô cùng truy vấn dùng để tính.

**Mục tiêu mẫu cho lab — được điều chỉnh có lý do trước khi nghiệm thu:**

| Chỉ số | Cách tính | Mục tiêu mẫu |
| --- | --- | --- |
| Tỷ lệ thao tác thành công | Số thao tác hoàn tất đúng / tổng thao tác hợp lệ đã phát, kể cả timeout và lỗi kết nối | ≥ 99% trong cửa sổ 30 phút, tối thiểu 1.000 thao tác |
| Độ trễ | Từ lúc phát thao tác đến lúc nhận đủ kết quả; tính trên cùng tập thao tác thành công | ≥ 95% thao tác thành công hoàn tất trong 500 ms |

Đánh giá hai mục tiêu cùng nhau để việc nhiều request lỗi không làm số đo độ trễ trông đẹp hơn.
Với mục tiêu thành công 99%, ngân sách lỗi là 1% số thao tác hợp lệ trong cửa sổ đo. Đây là SLO
của bài lab, không phải SLA với khách hàng hay bằng chứng đạt độ tin cậy theo tháng.

**PASS:** baseline đạt cả hai mục tiêu đã chốt, có dashboard/truy vấn, kết quả thô và kết luận cho
từng mục tiêu. Chỉ bắt đầu các bài sau khi baseline `PASS`. Thiếu mẫu đo hoặc công cụ đo bị gián
đoạn thì `STOP`, không coi khoảng không có dữ liệu là hệ thống hoạt động tốt.

Trong bài phá lỗi, vẫn ghi đầy đủ tác động và phần ngân sách lỗi đã dùng; không yêu cầu sự cố cố ý
luôn nằm trong SLO. Sau phục hồi, đo một cửa sổ mới với cùng điều kiện để chứng minh đạt lại mục
tiêu; kết quả mới không xoá vi phạm đã ghi nhận ở cửa sổ sự cố.

### 4.3. Bài 1 — Một yêu cầu mới

Thêm một service hoặc môi trường mà bài mẫu chưa hướng dẫn. Viết ticket trước khi thực hiện:
hành vi cần có, ràng buộc, tiêu chí nghiệm thu và ảnh hưởng tới hệ thống hiện tại.

- Tự xác định phần cần sửa trong pipeline, image, Helm/GitOps, cấu hình, quyền và monitoring.
- Triển khai qua quy trình PR/pipeline hiện có; giải thích các lựa chọn và đánh đổi.
- Kiểm tra chức năng mới, luồng cũ và SLO sau triển khai; kiểm tra rollback của thay đổi.

**PASS:** yêu cầu mới hoạt động, luồng cũ không hỏng, đạt SLO và rollback được. Bằng chứng gồm
ticket, PR, kết quả pipeline, kiểm thử và số đo. Cần sửa tay ngoài quy trình nhưng chưa ghi lại
hoặc chưa tái hiện được thì `STOP`.

### 4.4. Bài 2 — Một thay đổi có rủi ro

Chọn một thay đổi cấu hình hạ tầng hoặc database có khả năng ảnh hưởng dịch vụ, không chỉ đổi tag
image. Trước khi chạy, ghi phạm vi ảnh hưởng, dữ liệu có thể mất, mức gián đoạn, cách phát hiện
thất bại, điều kiện dừng và phương án quay lại. Có backup dùng được cho phần bị ảnh hưởng.

- Với Terraform: đọc plan, xác định resource bị sửa/thay thế/xoá và phụ thuộc liên quan.
- Với database: xác định tương thích giữa phiên bản ứng dụng và cấu trúc dữ liệu trước/sau;
  giải thích rollback image có đủ hay phải có bước dữ liệu riêng.
- Thử ở môi trường phụ trước, đo tác động, rồi diễn tập phương án quay lại trên lab.

**PASS:** dự đoán và ghi nhận được tác động, quay lại trạng thái dùng được, dữ liệu đúng và đạt lại
SLO. Lưu đánh giá rủi ro, diff/plan hoặc migration, kết quả kiểm thử và phục hồi. Chưa có đường
quay lại khả thi hoặc tác động vượt phạm vi đã chốt thì `STOP`.

### 4.5. Bài 3 — Một sự cố chưa biết trước

Nhờ một người khác tạo lỗi trong lab. Thống nhất trước phạm vi được phép thay đổi, thời gian và
cách cứu hộ; người tạo lỗi giữ kín nguyên nhân, lưu đáp án và cách hoàn tác riêng. Không dùng dữ
liệu thật và không gây lỗi ra ngoài phạm vi lab. Nếu chưa có người hỗ trợ thì để bài này chưa
hoàn thành; lỗi do chính mình cài sẵn không chứng minh được điều kiện “chưa biết trước”.

- Bắt đầu từ triệu chứng/cảnh báo; lập timeline, giả thuyết và kiểm tra bằng metrics, log,
  request ID, sự kiện Kubernetes hoặc trạng thái node phù hợp với từng giả thuyết.
- Ưu tiên phục hồi dịch vụ, sau đó xác nhận nguyên nhân bằng bằng chứng và đối chiếu đáp án.
- Ghi ảnh hưởng tới luồng người dùng/SLO, biện pháp khắc phục và hành động ngăn tái diễn.

**PASS:** tìm đúng nguyên nhân, giải thích được vì sao loại các giả thuyết khác, phục hồi đạt lại
SLO và có biên bản sự cố. Chỉ restart cho hết lỗi nhưng chưa chứng minh nguyên nhân thì `STOP`.

### 4.6. Bài 4 — Một lần phục hồi thật trong lab

Chốt trước **RTO** (thời gian phục hồi tối đa) và **RPO** (khoảng dữ liệu mới nhất có thể chấp nhận
mất). Ghi mục tiêu số cụ thể phù hợp với lab, mốc bắt đầu/kết thúc đo và cách xác định dữ liệu mất.

- Tạo dữ liệu thử có mã nhận diện và thời gian, lưu tập dữ liệu kỳ vọng, rồi tạo backup. Lưu bằng
  chứng dữ liệu ngoài hệ thống sẽ bị phá để còn đối chiếu sau phục hồi.
- Gây mất tài nguyên hoặc dữ liệu trong phạm vi đã chốt, rồi phục hồi bằng quy trình backup/restore
  thực sự. Snapshot VM chỉ là cứu hộ nếu bài thất bại, không thay cho quy trình cần nghiệm thu.
- Khôi phục cả tài nguyên ứng dụng và dữ liệu: dùng Velero trong phạm vi nó hỗ trợ; với MongoDB
  trên `hostPath`, dùng backup logic của Phase 2. Restore object không chứng minh đã restore dữ liệu.
- Đo RTO/RPO thực tế; đối chiếu ID, nội dung, số lượng và CRUD end-to-end, sau đó kiểm tra SLO.

**PASS:** RTO/RPO thực tế không vượt mục tiêu, dữ liệu đúng trong giới hạn mất đã chốt và ứng dụng
đạt lại SLO. Lưu backup ID/mốc thời gian, timeline restore và kết quả đối chiếu. Vượt mục tiêu hoặc
dữ liệu sai thì `STOP`; ghi nguyên nhân và diễn tập lại sau khi sửa.

### 4.7. Bài 5 — Một lần bàn giao

Nhờ một người khác dùng repo và tài liệu để dựng lại ở môi trường lab sạch trong phạm vi đã thống
nhất. Cung cấp trước điều kiện tài nguyên, tài khoản/quyền và cách cấp secret an toàn; không yêu
cầu dựng lại toàn bộ hạ tầng ngoài phạm vi bàn giao.

- Người nhận tự triển khai, chạy kiểm thử chức năng, truy cập dashboard/log và kiểm tra cảnh báo.
- Người nhận tự phát hành một thay đổi nhỏ, rollback và thực hiện một thao tác phục hồi đã ghi
  trong tài liệu. Ghi thời gian và mọi chỗ phải hỏi hoặc cần tác giả can thiệp.
- Sửa các bước thiếu trong tài liệu/tự động hoá rồi chạy lại phần bị vướng từ trạng thái đầu vào
  phù hợp; không tính một lần tác giả sửa tay hộ là bàn giao thành công.

**PASS:** người nhận hoàn thành bằng tài liệu, ứng dụng đạt SLO và không cần tác giả thao tác thay.
Lưu phiên bản repo, biên bản người nhận và các điểm đã sửa. Chưa có người nhận thực hành thì `STOP`.

### 4.8. Kết luận nghiệm thu

Chỉ đánh dấu hoàn tất chặng khi baseline/SLO và cả năm bài đều `PASS`. Tổng hợp thành một báo cáo:
phạm vi hệ thống, vai trò cá nhân, bằng chứng từng bài, RTO/RPO thực đo, SLO trước/trong/sau sự cố,
các giới hạn còn lại và kết quả bàn giao. Kết quả dùng để đánh giá năng lực trong phạm vi dự án;
không tự động quy đổi thành chức danh hoặc kinh nghiệm production thật.

Tham khảo chính thức:

- [Google SRE — Implementing SLOs](https://sre.google/workbook/implementing-slos/).
- [Google SRE — Incident Response](https://sre.google/workbook/incident-response/).
