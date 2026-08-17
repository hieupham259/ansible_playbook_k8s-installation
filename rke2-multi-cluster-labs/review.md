# Review lại các lab RKE2 multi-cluster — chỉ các vấn đề còn mở

> Ngày review lại: **17/08/2026**. Phạm vi: đọc lại toàn bộ
> [LAB-M1](LAB-M1-BA-CUM-RKE2-RANCHER-GITLAB.md) và
> [LAB-M2](LAB-M2-CAPSTONE-PRODUCTION-HA.md), đối chiếu những điểm phụ thuộc phiên bản với
> tài liệu/release metadata chính thức. Theo quy ước của repository, mục nào đã được sửa trong
> file LAB thì xóa khỏi file này; `review.md` chỉ giữ vấn đề còn tồn đọng.

## Kết luận hiện tại

**Chưa thể coi hai file là runbook mà người học làm từ trên xuống dưới sẽ PASS 100%.** M1 đã
tiến gần một lab end-to-end, nhưng vẫn còn lỗi chặn ở PostgreSQL/GitLab, Traefik Ingress status
và chuỗi Git/ArgoCD. M2 hiện vẫn là một blueprint/capstone specification: nhiều role, values,
manifest và quy trình restore/upgrade chỉ là snippet hoặc yêu cầu người học tự hoàn thiện.

Tiêu chí “100%” dùng trong review này là:

1. Bắt đầu từ hạ tầng trống đúng prerequisite đã công bố.
2. Không cần tự suy ra nội dung từ các câu “làm y hệt”, “lặp mẫu”, “qua role” hoặc “theo docs”.
3. Mọi file cần tạo đều có nội dung đầy đủ, đường dẫn, nơi chạy và lệnh apply/commit rõ ràng.
4. Mọi version tương thích với nhau và được pin; gate kiểm đúng failure mode, có output PASS
   xác định được.
5. Quy trình đã được pilot thật ít nhất một lượt trên đúng loại hạ tầng đích.

| File | Trạng thái hiện tại | Có thể cam kết chạy nguyên văn 100%? |
| --- | --- | --- |
| M1 | Lab chi tiết nhưng còn lỗi chặn và bước ngầm | **Chưa** |
| M2 | Blueprint tốt, chưa phải executable runbook | **Chưa** |

## A. Các vấn đề còn mở trong M1

### M1-01 — BLOCKER: GitLab 19.2.2 không tương thích PostgreSQL 16

[Bảng version M1](LAB-M1-BA-CUM-RKE2-RANCHER-GITLAB.md#21-phiên-bản-được-khóa) pin GitLab
chart `10.2.2` / GitLab `19.2.2`, nhưng pin PostgreSQL 16; [§6.2](LAB-M1-BA-CUM-RKE2-RANCHER-GITLAB.md#62-postgresql--redis--minio-trên-mc-db1)
còn coi `psql --version` trả `16.x` là PASS. Bảng requirements chính thức hiện ghi GitLab
19.x/chart 10.x chỉ hỗ trợ PostgreSQL **17.x**. Đây không phải cảnh báo tối ưu hóa mà là lỗi
tương thích của stack đã pin.

Cách đóng: chọn một trong hai hướng rồi đồng bộ toàn bộ bảng version, lệnh cài, đường dẫn
`postgresql.conf`, gate và M2:

- giữ GitLab 19.2.2/chart 10.2.2 và cài PostgreSQL 17 từ nguồn được pin/verify; hoặc
- giữ PostgreSQL 16 và hạ GitLab về một bản 18.x/chart 9.x tương thích.

Nguồn: [GitLab installation requirements](https://docs.gitlab.com/install/requirements/),
[external database for the chart](https://docs.gitlab.com/charts/advanced/external-db/).

### M1-02 — BLOCKER: gate `status.loadBalancer` của Traefik không có nguồn địa chỉ

[§4.1](LAB-M1-BA-CUM-RKE2-RANCHER-GITLAB.md#41-rke2-server-với-traefik-ngay-từ-đầu)
và §5.1 bật `providers.kubernetesIngress.publishedService.enabled`, còn
[gate §7.2](LAB-M1-BA-CUM-RKE2-RANCHER-GITLAB.md#72-argocd-instance-riêng-cho-từng-cụm)
đòi `Ingress.status.loadBalancer` không rỗng. RKE2 hiện để ServiceLB **tắt mặc định**. Với
Service kiểu LoadBalancer nhưng không có LB controller, Traefik chỉ copy một
`service.status.loadBalancer` đang rỗng; `hostPort` không tự điền trường status. Vì vậy gate
ArgoCD Healthy có thể fail đúng như lỗi mà lab tuyên bố đã xử lý.

Cách đóng: dùng một cơ chế bare-metal có kết quả xác định, ví dụ
`providers.kubernetesIngress.reportNodeInternalIPs: true` cho DaemonSet, hoặc cấu hình Service
có `externalIPs`/LB controller thật; sau đó gate cả Service lẫn Ingress status. Ở M2 nên công
bố VIP ingress thay vì một địa chỉ tình cờ của node nếu đó là endpoint người dùng.

Nguồn: [RKE2 ServiceLB mặc định false](https://docs.rke2.io/reference/server_config),
[Traefik Kubernetes Ingress status/reportNodeInternalIPs](https://doc.traefik.io/traefik/master/reference/install-configuration/providers/kubernetes/kubernetes-ingress/).

### M1-03 — BLOCKER: GitLab/ArgoCD end-to-end chưa có chuỗi thao tác hoàn chỉnh

Các đoạn [§6.4](LAB-M1-BA-CUM-RKE2-RANCHER-GITLAB.md#64-gitlab-runner-với-kubernetes-executor)
và [§7.2](LAB-M1-BA-CUM-RKE2-RANCHER-GITLAB.md#72-argocd-instance-riêng-cho-từng-cụm)
mới đưa nội dung `.gitlab-ci.yml`, Dockerfile gợi ý, manifest workload và `Application`, nhưng
chưa có chuỗi lệnh hoàn chỉnh để:

- clone hai repo, tạo đúng cây thư mục, copy source/Dockerfile, commit và push;
- lấy tag pipeline một cách xác định rồi cập nhật repo `deploy-config`;
- apply `Application` vào namespace `argocd` (comment “apply trên mc-app1” không phải lệnh);
- thực hiện manual sync bằng UI **hoặc** CLI với command cụ thể.

Ngoài ra `demo-app` được `kubectl create ns` ở §7.1 rồi lại create ở §7.2, nên lần chạy thứ hai
trả `AlreadyExists`. Cần biến thao tác namespace thành idempotent hoặc chỉ tạo một lần.

Cách đóng: bổ sung transcript từ `git clone` đến `curl app.mc.lab`, không để bất kỳ file hay
lần apply nào chỉ tồn tại dưới dạng đoạn YAML minh họa.

### M1-04 — BLOCKER: bước dựng cụm CICD vẫn là “làm y hệt” và bỏ ngầm prerequisite Helm

[§6.1](LAB-M1-BA-CUM-RKE2-RANCHER-GITLAB.md#61-rke2-server-và-import) yêu cầu “cài RKE2
server y hệt §4.1” với các giá trị thay đổi, thay vì cung cấp `config.yaml`, HelmChartConfig,
lệnh cài, kubeconfig và import hoàn chỉnh. §6.3 sau đó dùng `helm` nhưng không có bước cài Helm
trên `mc-cicd1`; §4.2 chỉ cài trên `mc-admin1`.

Cách đóng: viết nguyên block cho CICD hoặc tạo script/role thật được gọi bằng một command. Với
mục tiêu dành cho người học, tham chiếu một đoạn rồi yêu cầu tự biến đổi token, SAN, IP và
hostname không đạt tiêu chí chạy nguyên văn.

### M1-05 — MAJOR: DNS trên `mc-router` có thể tự làm hỏng resolver của chính router

[§3.4](LAB-M1-BA-CUM-RKE2-RANCHER-GITLAB.md#34-router-nat-dns-và-firewall-giữa-các-dải)
tắt `DNSStubListener` của `systemd-resolved`, nhưng không đổi `/etc/resolv.conf` khỏi stub
`127.0.0.53`. Trên Ubuntu 24.04, symlink mặc định vẫn có thể trỏ vào
`/run/systemd/resolve/stub-resolv.conf`; khi listener đã tắt, resolver của router không còn
phục vụ tại địa chỉ đó. `DNS=127.0.0.1` trong drop-in không tự sửa symlink này.

Cách đóng: công bố và gate một phương án resolver hoàn chỉnh, chẳng hạn để `/etc/resolv.conf`
trỏ đúng resolver đang nghe, rồi kiểm `resolvectl`, `ss -lntup '( sport = :53 )'`, phân giải
record nội bộ và domain Internet sau reboot router.

### M1-06 — MAJOR: đường dẫn CA/cert không nhất quán giữa các node

§4.2 nói copy `mc-lab-ca.crt` tới từng VM rồi cài vào trust store nhưng không quy định file
đích. Các bước sau lại giả định `/tmp/mc-lab-ca.crt` tồn tại trên `mc-cicd1` và các node chạy
ArgoCD; tương tự, §6.3 nói `scp` wildcard cert sang node nhưng command dùng tên file ở working
directory không được xác định. Người học có thể làm đúng bước copy nhưng vẫn fail vì khác
đường dẫn.

Cách đóng: pin đường dẫn tuyệt đối trên từng máy, có gate `test -s`, quyền file private key,
và lệnh xóa bản key tạm sau khi tạo Secret.

### M1-07 — MAJOR: sizing 8 GB cho node GitLab chưa có cấu hình memory-constrained

`mc-cicd1` chỉ có 8 GB cho Ubuntu + RKE2 + toàn bộ GitLab chart + runner. Requirements chính
thức coi 16 GB là baseline và 8 GB chỉ là mức tối thiểu khi cấu hình memory-constrained. Lab
giảm replica nhưng không pin requests/limits cho toàn stack và không có gate OOM/headroom.

Cách đóng: hoặc tăng RAM, hoặc bổ sung values memory-constrained đã pilot; gate phải kiểm
`kubectl top`, requests/limits, restart count, OOMKilled và headroom sau pipeline. Nguồn:
[GitLab requirements — memory](https://docs.gitlab.com/install/requirements/).

### M1-08 — MAJOR: version/reproducibility vẫn chưa khép kín

Ba thành phần vẫn trôi: MinIO server/client, `gitlab-runner` chart và Reflector. Riêng runner
chart tương thích với GitLab chart đã biết có thể pin thay vì lấy “mới nhất”. Ngoài ra tài
liệu GitLab hiện yêu cầu Helm 4; M1 pin Helm 3.21.3 sau thời điểm Helm 3 EOL mà không ghi đây
là cấu hình unsupported hay chứng minh chart 10.2.2 vẫn chạy bằng pilot.

Cách đóng: pilot rồi pin version/digest/checksum cho MinIO, `mc`, runner, Reflector và các
image demo; quyết định Helm 4 hoặc ghi rõ support exception có bằng chứng. Nguồn:
[GitLab chart prerequisites](https://docs.gitlab.com/charts/installation/tools/).

### M1-09 — MAJOR: chiến lược wildcard rotation chỉ tự động trong một cluster

Cert wildcard được cert-manager cấp ở Admin rồi copy tay sang CICD/App. Reflector chỉ nhân
bản secret **bên trong** cụm App. Khi cert-manager renew wildcard ở Admin, secret GitLab và
secret nguồn ở App không tự cập nhật. Vì vậy thiết kế chưa thực hiện được lời hứa “update một
lần, thay đổi tất cả cert” ở quy mô multi-cluster.

Cách đóng: nói rõ đây chỉ là bootstrap thủ công của lab hoặc thêm cơ chế phân phối xuyên cụm
được quản lý (GitOps + secret encryption/external secret), kèm gate rotation giả lập.

### M1-10 — MAJOR: các gate cuối còn thao tác bằng lời và có gate dễ false-negative

Gate storage §9 chỉ nói “xóa pod rồi đọc heartbeat” mà không có lệnh, timeout và cách lấy
pod thay thế. Gate Internet §3.5 bắt đúng `HTTP/2 200` từ một website bên ngoài; redirect,
HTTP/1.1 hoặc thay đổi phía website sẽ làm gate fail dù egress vẫn tốt. Monitoring lại là tùy
chọn và logging không có, nên M1 chưa khớp trọn mô tả “Admin chứa monitoring/logging”.

Cách đóng: biến mọi gate thành command có timeout/expected output ổn định; monitoring/logging
nếu thuộc đích bắt buộc thì không để optional.

## B. Các vấn đề còn mở trong M2

### M2-01 — BLOCKER: M2 chưa chứa repo IaC có thể chạy

[§4](LAB-M2-CAPSTONE-PRODUCTION-HA.md#4-iac-ansible--vault--ci-validation) chỉ cho cây thư
mục và một phần task/template. Các role `common`, `fw`, `lb`, `rke2_agent`, `longhorn_node`,
inventory, playbook con, handlers, keepalived, PostgreSQL/Redis/MinIO và toàn bộ
`deploy/` chưa có nội dung hoàn chỉnh. Trong khi đó §5–§10 liên tục nói “qua role”, “lặp mẫu”,
“làm như M1” hoặc “manifest trong git”. Người học không thể chạy `site.yml` lần đầu, càng
không thể chứng minh lần hai `changed=0`.

Cách đóng: đưa repo `capstone-iac` hoàn chỉnh vào codebase (hoặc generate đầy đủ từ file LAB)
và để lab chỉ gọi những artifact có thật. Nếu việc tự viết role là bài tập, phải đổi tuyên bố
đích: đó là capstone assignment, không phải runbook PASS 100%.

### M2-02 — BLOCKER: số VM và số node App tự mâu thuẫn

Bảng §2 cộng thành **18 VM**: 1 firewall + 2 LB + 3 Admin + 4 CICD + 6 App + 2 DATA. Tuy vậy
PASS §2 và comment inventory ghi **17 VM**. Kiến trúc có 3 App server + 3 App agent = 6 node,
nhưng gate §6 lại đòi cụm App có **5 node**.

Cách đóng: chọn một topology duy nhất, bổ sung bảng IP/hostname/NIC/disk đầy đủ cho từng VM,
sửa tổng tài nguyên, inventory và mọi gate theo topology đó.

### M2-03 — BLOCKER: HAProxy/keepalived chưa đủ để VIP failover

§5 chỉ đưa cấu hình HAProxy cụm Admin rồi yêu cầu “lặp mẫu”; không có cấu hình keepalived,
`virtual_ipaddress`, authentication, unicast/multicast decision, health script, firewall VRRP
hay role `lb`. HAProxy trên BACKUP bind vào sáu VIP chưa sở hữu sẽ không start nếu chưa cấu
hình `net.ipv4.ip_nonlocal_bind=1` (hoặc thiết kế bind khác).

Gate `ip addr | grep -c '10.30.5.'` cũng sai: nó đếm cả IP chính của NIC, nên MASTER không
phải 6 và BACKUP không phải 0. Cần đếm chính xác sáu địa chỉ VIP hoặc kiểm từng VIP.

Cách đóng: cung cấp trọn config cho hai LB và cả ba cụm, sysctl cần thiết, validate
`haproxy -c`, trạng thái service, từng VIP và probe 6443/9345/80/443 trước/sau failover.

### M2-04 — BLOCKER: ma trận firewall chưa phải ruleset và còn thiếu control path

§3 yêu cầu người học tự chuyển từng dòng ma trận thành nftables. Chưa có ruleset thực,
interface mapping, chain `input/forward/output`, NAT, state rule, logging hay cách gỡ rule
tạm của drill. Các luồng sau chưa được xác định đầy đủ:

- Admin ArgoCD/Alertmanager tới VIP GitLab/alert sink;
- máy chạy Ansible tới SSH của 18 VM (và đường lấy code/vault nếu chạy từ CI runner);
- địa chỉ đích thực của dòng “ADMIN → CICD, APP 6443 qua VIP” là EDGE VIP, không phải subnet
  CICD/App như bảng đang biểu diễn.

Cách đóng: commit `/etc/nftables.conf` template hoàn chỉnh, pin interface/CIDR/VIP, thêm test
allow **và** deny cho mọi nhóm luồng, rồi chạy gate sau reboot firewall.

### M2-05 — BLOCKER: Traefik HA và placement Longhorn chưa được cấu hình đúng topology

§5 giả định Traefik chạy hostPort trên mọi node, nhưng config RKE2 §6 không tạo
HelmChartConfig DaemonSet/report status. HAProxy lại gửi ingress tới toàn bộ node; node không
có Traefik sẽ bị health check fail hoặc giảm capacity ngoài dự kiến.

Longhorn được mô tả chỉ dùng ba worker có disk, nhưng mặc định Longhorn tạo default disk
`/var/lib/longhorn` trên **mọi node mới**. Nếu không bật `createDefaultDiskLabeledNodes` và
label đúng ba worker, replica có thể nằm trên OS disk của server, làm sai gate “ba storage
node”.

Cách đóng: đưa HelmChartConfig Traefik và Longhorn values/labels đầy đủ vào IaC; gate liệt kê
Traefik pod trên đúng backend và disk/replica Longhorn chỉ trên `w1..3`. Nguồn:
[Longhorn default disk behavior](https://longhorn.io/docs/1.12.0/nodes-and-volumes/nodes/default-disk-and-node-config/).

### M2-06 — BLOCKER: gate Ansible `--check` không thể chứng minh điều lab yêu cầu

Task cài RKE2 dùng `ansible.builtin.shell` không có `creates/removes`. Ở check mode, module
shell sẽ bị skip thay vì dự báo chính xác install/restart; do đó gate “đổi version rồi chạy
`--check --diff` phải hiện đúng task install/restart” không thể PASS theo implementation hiện
tại. Pipeline cũng load `all_vault.yml` nhưng không chỉ ra vault password/CI variable, và
`pip install ansible ansible-lint` không pin version nên kết quả lint có thể trôi.

Cách đóng: thiết kế check-mode behavior tường minh, thêm Molecule/integration test hoặc một
host pilot cho role nâng cấp; pin Ansible/ansible-lint image/dependencies và cấu hình vault
cho CI mà không lộ secret.

### M2-07 — BLOCKER: bootstrap GitOps và secret management chưa được giải

§4 yêu cầu mọi add-on apply qua ArgoCD từng cụm, nhưng chưa định nghĩa cách bootstrap chính
ArgoCD trước khi ArgoCD tồn tại. Mọi Helm values/manifest phải commit vào git, trong khi
credential MinIO, DB, runner, repo và registry không thể commit plaintext; Ansible Vault chỉ
được mô tả cho biến host/RKE2 và không có cầu nối sang Kubernetes Secret.

Cách đóng: cung cấp bootstrap command/app-of-apps và một chiến lược secret cụ thể như SOPS,
Sealed Secrets hoặc External Secrets, bao gồm key recovery trong DR.

### M2-08 — BLOCKER: Rancher HA chưa bảo đảm trải ba node

Rancher chart mặc định dùng anti-affinity `preferred`, nhưng gate §7 khẳng định ba pod sẽ
trải trên ba node. `preferred` không bảo đảm điều đó. Cần set `antiAffinity=required` hoặc
affinity/topologySpreadConstraints rõ ràng, đồng thời gate PDB và failover từng replica.

Nguồn: [Rancher Helm chart option `antiAffinity`](https://ranchermanager.docs.rancher.com/v2.14/getting-started/installation-and-upgrade/installation-references/helm-chart-options).

### M2-09 — BLOCKER: PSA được gán sai phạm vi; các negative test chưa có manifest

§8 nói gán PSA template `restricted` cho **project**. Rancher PSA configuration template được
gán cho **cluster**; sau đó namespace chịu policy/exemption ở cấp namespace. UI path trong lab
không khớp mô hình chính thức. Nếu muốn chỉ áp `team-a-web`, cách đơn giản là label namespace
`pod-security.kubernetes.io/enforce=restricted` cùng version phù hợp.

Các gate quota/privileged pod/Ingress allow mới là câu mô tả, không có manifest workload,
quota-overrun pod, privileged pod, Service/Ingress hoặc cách lấy kubeconfig `dev-a` và pod IP.

Cách đóng: sửa phạm vi PSA và cung cấp toàn bộ manifest/lệnh cho cả positive lẫn negative
test. Nguồn: [Rancher PSA configuration templates](https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/authentication-permissions-and-global-configuration/psa-config-templates).

### M2-10 — BLOCKER: observability/alerting chưa có artifact chạy được

§9 không pin chart/values monitoring, không có ArgoCD Application, Ingress của alert sink
chỉ được mô tả bằng comment, và `AlertmanagerConfig` không có manifest/label/namespace selector
để Prometheus Operator thực sự chọn. Vì vậy các gate receiver reachable và alert firing chưa
thể được tạo từ nội dung lab.

Cách đóng: commit chart version, values, App, receiver Deployment/Service/Ingress/TLS và
AlertmanagerConfig hoàn chỉnh; gate thêm `kubectl get alertmanagerconfig`, generated config,
alert state Pending/Firing và body webhook.

### M2-11 — BLOCKER: Rancher Backup CR hiện tại không hợp lệ cho MinIO HTTP

Backup YAML §10.2 thiếu `resourceSetName`, trong khi Rancher yêu cầu khi tạo YAML phải là
`rancher-resource-set-full` hoặc `rancher-resource-set-basic`. MinIO của lab chạy HTTP nhưng
CR lại đặt `insecureTLSSkipVerify: false` và `endpointCA`; CA không có ý nghĩa với endpoint
HTTP. Secret credential cũng chỉ được nhắc tới, không có manifest/key `accessKey` và
`secretKey`; chart/operator version chưa pin.

Cách đóng: cài CRD/operator bằng version tương thích Rancher 2.14.3, tạo Secret, sửa CR cho
MinIO HTTP hoặc bật TLS MinIO thật, thêm `resourceSetName`, apply và restore một Backup theo
filename cụ thể. Nguồn: [Rancher Backup examples](https://ranchermanager.docs.rancher.com/reference-guides/backup-restore-configuration/examples),
[backup-restore-operator](https://github.com/rancher/backup-restore-operator).

### M2-12 — BLOCKER: GitLab backup thiếu cấu hình toolbox object storage

M1 chỉ cấu hình Fog/consolidated object storage cho Rails và registry. `backup-utility` của
chart cần cấu hình riêng ở `gitlab.toolbox.backups.objectStorage.config` (thường là secret
kiểu s3cmd) để upload/restore backup. M2 nói bucket đã có nhưng không tạo secret/config này,
nên gate backup lên MinIO không được bảo đảm.

Cách đóng: tạo config/Secret toolbox theo chart 10.2.2, thêm backend/bucket/tmpBucket nhất
quán, pin command backup và restore đúng release. Nguồn:
[GitLab external object storage — Backups](https://docs.gitlab.com/charts/advanced/external-object-storage/),
[GitLab chart backup/restore](https://docs.gitlab.com/charts/backup-restore/).

### M2-13 — BLOCKER: Longhorn/GitLab/PostgreSQL restore vẫn là hướng dẫn khái quát

§10.3 yêu cầu thao tác UI, “tạo PV/PVC”, “theo đúng docs”, `<timestamp>` và “dropdb” nhưng
không cung cấp secret backup target, PV/PVC/pod manifest, exact backup name, chế độ maintenance,
scale-down/up GitLab hay command cleanup. Phần PostgreSQL gọi đây là “cron hằng ngày” nhưng
chỉ có ba command chạy tay, không có cron/systemd timer, retention, checksum hoặc cleanup file
`/tmp`.

Cách đóng: biến ba restore thành runbook command-by-command, dùng database/namespace/volume
cách ly; thêm verify checksum/count và cleanup xác định. Không chạy restore GitLab phá hủy
trên instance đang phục vụ nếu chưa có maintenance/downtime/rollback plan.

### M2-14 — BLOCKER: DR drill để lại node cordoned và restore etcd còn placeholder

§11.2 nói “sau uncordon” nhưng không có `kubectl uncordon mc2-app-w1`; người học chạy nguyên
văn sẽ để node cordoned. §11.4 dùng path `pre-drill-<...>`, không chỉ cách lấy tên snapshot
thật, câu “cả 3 server” không chỉ rõ command chạy ở đâu, và câu xóa DB/join lại không có gate
từng member/etcd health.

Cách đóng: thêm uncordon/wait Healthy; trong drill etcd phải lấy filename từ
`rke2 etcd-snapshot ls`, ghi command theo từng host, backup token, kiểm member list/node Ready
và workload sau restore. Với S3, diễn tập thêm restore từ object S3 chứ không chỉ local path.

### M2-15 — BLOCKER: bài nâng cấp không có version đích chạy được và bỏ sót agent

§12 dùng placeholder `<2.14.x+1>` và `v1.35.<x+1>+rke2r1`; đây không phải version hợp lệ. Vì
lab bắt đầu ở các bản gần mới nhất, patch tiếp theo có thể chưa tồn tại. Command chỉ dùng
`INSTALL_RKE2_TYPE=server`, restart `rke2-server` và gate server; agent cần type/service riêng.

Cách đóng: pin cặp **source → target** đều đã phát hành và tương thích, tốt nhất cài source
ngay từ đầu rồi upgrade tới target; viết vòng nâng cấp server/agent riêng, drain/uncordon khi
cần, health gate sau mỗi node và rollback drill với snapshot cụ thể.

### M2-16 — MAJOR: logging và performance/capacity vẫn không thuộc gate bắt buộc

Logging tập trung tiếp tục là phần mở rộng tùy chọn, nên chưa đạt kiến trúc gốc “Admin chứa
monitor/logging”. M2 có RTO/RPO nhưng không đo ingress throughput/latency, etcd disk latency,
Longhorn rebuild time, saturation và CPU/RAM/disk headroom. Sizing 115 GB vì vậy vẫn là ước
lượng chưa được chứng minh.

Cách đóng: nếu đích là khớp kiến trúc gốc, thêm logging bắt buộc với query log xuyên cụm;
thêm workload/capacity gate có ngưỡng và công cụ/version pin.

## C. Gate cuối cùng áp dụng cho cả hai file: pilot thật

Ngay cả sau khi sửa hết lỗi tĩnh ở trên, chưa được tuyên bố “100%” cho tới khi có pilot. Một
runbook phụ thuộc 18 VM, nhiều chart và UI không thể được chứng minh chỉ bằng review văn bản.

Điều kiện xóa mục này khỏi `review.md`:

1. Chạy M1 từ snapshot host trống đến `m1-complete`, không dùng kiến thức ngầm ngoài các
   prerequisite được link.
2. Chạy M2 trên topology đã chốt, từ tạo inventory đến toàn bộ gate §13.
3. Lưu transcript: command, version/digest, output gate, thời gian, mọi deviation và cách sửa.
4. Reset về điểm đầu rồi để một người học khác chạy lại bằng đúng tài liệu đã sửa.
5. Chỉ khi lượt thứ hai không cần “mentor tự điền phần thiếu” mới coi hai LAB là runbook PASS
   100%.

## Thứ tự sửa khuyến nghị

1. Sửa M1-01 → M1-04 để M1 có chuỗi kỹ thuật có thể boot và deploy thật.
2. Đóng M1-05 → M1-10, pilot M1 và pin nốt version.
3. Chuyển M2 từ snippet thành repo IaC thật (M2-01), đồng thời chốt topology/network/LB
   (M2-02 → M2-06).
4. Hoàn thiện GitOps/security/observability/backup/DR/upgrade (M2-07 → M2-16).
5. Chạy hai pilot độc lập; sau mỗi lần sửa LAB, xóa đúng mục đã đóng khỏi file này.
