# Lab M1 — Ba cụm RKE2 + Rancher + GitLab + Longhorn trên VM Windows host

> **Vị trí của lab này:** đây là track riêng, nằm ngoài chuỗi snapshot của
> [k8s-docs/labs](../k8s-docs/labs/README.md). Lab dựng lại **toàn bộ hệ thống trong bài viết
> "3 cụm K8s"** (Admin/CICD/App, dải mạng riêng, DB ngoài cluster, Longhorn) ở quy mô homelab,
> với các sửa đổi đã rút từ nhận xét: dùng **Traefik ngay từ đầu** thay vì ingress-nginx,
> blacklist multipathd thay vì disable, `max_connections` có tính toán, PVC tối thiểu 2 replica.
>
> **Điểm bắt đầu:** máy Windows host trống, chưa có VM nào của track này.
> **Điểm kết thúc:** 3 cụm RKE2 hoạt động, Rancher quản lý cả 3, pipeline GitLab build image
> và ArgoCD deploy app end-to-end. Ngày đối chiếu phiên bản: **17/08/2026**.
>
> Lab tiếp theo của track: [Lab M2 — Capstone production HA](LAB-M2-CAPSTONE-PRODUCTION-HA.md).
>
> **Trạng thái kiểm chứng:** file chỉ được gắn nhãn runtime `READY` sau khi maintainer chạy từ
> host trống tới `m1-complete`, restore thử một mốc, sửa mọi deviation, rồi một người học khác
> chạy lại chỉ bằng tài liệu. Trước khi có hai evidence đó, đây là runbook **READY FOR PILOT**:
> gate fail phải dừng theo §2.4; không tự bỏ qua, đổi command hoặc ghi output PASS giả định.

---

## 1. Bức tranh toàn cảnh

Đọc kỹ sơ đồ này trước khi gõ lệnh đầu tiên. Mọi mục từ §3 trở đi chỉ là dựng dần từng khối
của đúng sơ đồ này.

```mermaid
flowchart TB
    subgraph HOST["Máy Windows host — VMware Workstation"]
        subgraph NET_ADMIN["VMnet2 — dải ADMIN 10.20.10.0/24"]
            ADMIN1["mc-admin1 — 10.20.10.11<br/>RKE2 server (cụm Admin)<br/>Traefik · cert-manager · CA lab<br/>Rancher · ArgoCD-admin"]
        end
        subgraph NET_CICD["VMnet3 — dải CICD 10.20.20.0/24"]
            CICD1["mc-cicd1 — 10.20.20.11<br/>RKE2 server (cụm CICD)<br/>Traefik · GitLab · Registry<br/>gitlab-runner · ArgoCD-cicd"]
        end
        subgraph NET_APP["VMnet4 — dải APP 10.20.30.0/24"]
            APP1["mc-app1 — 10.20.30.11<br/>RKE2 server (cụm App)<br/>Traefik · Longhorn · app demo"]
            APP2["mc-app2 — 10.20.30.12<br/>RKE2 agent<br/>Longhorn replica 2"]
        end
        subgraph NET_DATA["VMnet5 — dải DATA 10.20.40.0/24"]
            DB1["mc-db1 — 10.20.40.11<br/>PostgreSQL 17<br/>(DB ngoài cluster)"]
        end
        ROUTER["mc-router — .1 của cả 4 dải<br/>NAT ra Internet · dnsmasq DNS *.mc.lab<br/>nftables: chặn mọi dải → DATA,<br/>chỉ mở CICD → DATA:5432"]
    end
    ROUTER --- NET_ADMIN
    ROUTER --- NET_CICD
    ROUTER --- NET_APP
    ROUTER --- NET_DATA
    ADMIN1 -- "import (cattle-cluster-agent<br/>outbound 443 về rancher.mc.lab)" --> CICD1
    ADMIN1 -- "import" --> APP1
    CICD1 -- "SQL 5432" --> DB1
    CICD1 -- "pipeline build image → registry.mc.lab" --> CICD1
    APP1 -- "ArgoCD-app kéo manifest<br/>từ gitlab.mc.lab" --> CICD1
```

Đối chiếu với thiết kế trong bài viết gốc:

| Tính chất bài viết nêu | Hiện thực trong lab |
| --- | --- |
| Cụm Admin quản lý tập trung | Rancher trên cụm Admin, import 2 cụm còn lại (§4, §5.4, §6.1) |
| Cụm CICD lưu code + triển khai | GitLab + Registry + runner Kubernetes executor (§6) |
| Cụm App phục vụ ứng dụng | Cụm 2 node chạy app demo qua ArgoCD (§5, §7) |
| Các cụm nằm trên dải mạng riêng | 4 VMnet host-only, định tuyến qua `mc-router` (§3) |
| DB trên máy chủ riêng, dải nội bộ | PostgreSQL trên `mc-db1`, firewall chỉ mở từ CICD (§3.4, §6.2) |
| Longhorn làm block storage | Longhorn trên cụm App, mặc định 2 replica (§5.3) |
| Self-signed CA + cert-manager rotate | CA lab cấp bởi cert-manager trên cụm Admin (§4.2) |
| Wildcard cert + reflector | Wildcard `*.mc.lab` từ CA lab, reflector nhân bản theo namespace (§7.1) |
| Argo instance riêng mỗi cụm | 3 ArgoCD, manual sync + server-side apply (§7.2) |

Khác biệt có chủ đích so với bài viết: dùng **Traefik từ lúc cài RKE2** (`ingress-controller:
traefik`) — tránh đúng "kiếp nạn" update/revert mà bài viết mô tả, vì ingress-nginx đã EOL
03/2026 và bị gỡ khỏi RKE2 từ v1.37.

## 2. Quy hoạch

### 2.1. Phiên bản được khóa

Track này có bảng phiên bản riêng, không dùng chung bảng A1.3 của Lab 00 (khác distro).
Giá trị đối chiếu ngày 17/08/2026; nếu kho phát hành đã đổi thì dừng, đối chiếu release notes
rồi cập nhật bảng — không âm thầm cài bản khác.

| Thành phần | Phiên bản | Ghi chú |
| --- | --- | --- |
| Ubuntu Server | 24.04.x LTS amd64 | ISO và cách verify SHA256: [Lab 00 §A1.3](../k8s-docs/labs/LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) |
| RKE2 | `v1.35.7+rke2r1` | Kubernetes v1.35.7; ship Traefik v3.7.8; **không lên 1.36** khi còn Rancher 2.14.3 (chart khai `kubeVersion: < 1.36.0-0`) |
| Rancher | chart `2.14.3` | repo `rancher-stable`; support matrix phủ K8s 1.33–1.35 |
| cert-manager | chart `v1.21.1` | repo `jetstack` |
| Helm | `v4.2.3` | GitLab chart 10.x yêu cầu Helm ≥ 4.0; cài bằng script Helm 4 chính thức với `DESIRED_VERSION` pin cứng |
| Longhorn | chart `1.12.1` | yêu cầu K8s ≥ 1.25; V1 data engine |
| PostgreSQL | **17** (repo PGDG chính thức) | GitLab 19.x chỉ hỗ trợ PostgreSQL **17.x** (min = max = 17); gói 16 của Ubuntu Noble **không dùng được** |
| Redis | 7.x (gói Ubuntu Noble) | GitLab chart 10.x **bắt buộc** external Redis |
| MinIO server | `RELEASE.2025-09-07T16-13-09Z` | binary + SHA256 chính thức; GitLab chart 10.x **bắt buộc** external object storage |
| MinIO client `mc` | `RELEASE.2025-08-13T08-35-41Z` | binary + SHA256 chính thức; tạo và verify bucket |
| GitLab chart | `10.2.2` (GitLab `19.2.2`) | repo `https://charts.gitlab.io`; đối chiếu [version mappings](https://docs.gitlab.com/charts/installation/version_mappings/) nếu repo đã sang bản khác |
| Argo CD | `v3.5.1` | manifest install pin theo tag, không dùng `stable` trôi nổi |
| local-path-provisioner | `v0.0.37` | StorageClass cho cụm CICD (Gitaly cần PVC); trùng bản khóa ở [Lab 00 §A1.4](../k8s-docs/labs/LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) |
| gitlab-runner chart | `0.91.2` | repo `gitlab`; pin đúng chart version ở §6.4 |
| Reflector chart | `10.0.65` | repo `emberstack`; pin đúng chart version ở §7.1 |

### 2.2. VM, mạng và DNS

| VM | Dải / VMnet | IP | vCPU | RAM | Disk | Vai trò |
| --- | --- | --- | --- | --- | --- | --- |
| `mc-router` | NAT (VMnet8) + VMnet2/3/4/5 | `.1` mỗi dải | 1 | 1 GB | 20 GB | NAT, DNS, firewall giữa các dải |
| `mc-admin1` | ADMIN / VMnet2 | `10.20.10.11` | 4 | 6 GB | 50 GB | RKE2 server cụm Admin |
| `mc-cicd1` | CICD / VMnet3 | `10.20.20.11` | 4 | 8 GB | 60 GB | RKE2 server cụm CICD |
| `mc-app1` | APP / VMnet4 | `10.20.30.11` | 2 | 4 GB | 40 + 20 GB | RKE2 server cụm App |
| `mc-app2` | APP / VMnet4 | `10.20.30.12` | 2 | 4 GB | 40 + 20 GB | RKE2 agent cụm App |
| `mc-db1` | DATA / VMnet5 | `10.20.40.11` | 2 | 4 GB | 40 GB | PostgreSQL + Redis + MinIO |

Tổng 27 GB RAM cho VM. Host **tối thiểu 32 GB** (phải đóng ứng dụng nặng khi chạy đủ stack),
khuyến nghị 48 GB. Kiểm tra host bằng block PowerShell ở
[Lab 00 §A1.1](../k8s-docs/labs/LAB-00-MOI-TRUONG-1.35.7.md#a11-máy-host-windows).

Disk thứ hai 20 GB trên `mc-app1`/`mc-app2` dành riêng cho Longhorn (mount `/var/lib/longhorn`).

DNS nội bộ (dnsmasq trên `mc-router`, domain `mc.lab`):

| Tên | Trỏ về |
| --- | --- |
| `rancher.mc.lab` | `10.20.10.11` |
| `gitlab.mc.lab`, `registry.mc.lab` | `10.20.20.11` |
| `minio.mc.lab` | `10.20.40.11` |
| `app.mc.lab` | `10.20.30.11` |
| `mc-admin1.mc.lab` … `mc-db1.mc.lab` | IP node tương ứng |

**Nếu thiếu RAM** (host 32 GB): trong §8 bỏ monitoring; nếu vẫn thiếu, chấp nhận chạy cụm App
1 node + Longhorn 1 replica — ghi rõ vào hồ sơ lab rằng cấu hình này mất tính chất "2 replica
trên 2 node" và mọi kết luận về drain/eviction không còn giá trị.

#### Vì sao dùng host-only + NAT thay vì Bridged

Track này cố ý mô phỏng bốn dải mạng riêng ADMIN, CICD, APP và DATA, nên các VM workload
`mc-admin1`, `mc-cicd1`, `mc-app1`, `mc-app2` và `mc-db1` **không** gắn Bridged. Mỗi VM chỉ
gắn vào VMnet host-only đúng theo bảng trên; traffic liên dải phải đi qua `mc-router` để
`nftables` thực thi chính sách ở §3.4 và `dnsmasq` cung cấp DNS `*.mc.lab`.

Chỉ `mc-router` có một NIC NAT: adapter 1 nối VMnet8 làm uplink ra Internet; bốn adapter còn
lại nối VMnet2–VMnet5 và giữ địa chỉ `.1` của từng dải. Vì vậy NAT **không** phải network mode
của năm VM workload. Đường egress của chúng là `VM workload → gateway .1 → mc-router →
VMnet8 → Internet`. Windows host vẫn SSH trực tiếp vào từng dải qua host virtual adapter
`.254` ở §3.1, không đi qua firewall liên dải.

Khác với [Lab 00](../k8s-docs/labs/LAB-00-MOI-TRUONG-1.35.7.md#a12-ba-vm) và
[runbook VMware](../runbook-k8s-vmware.md#3-tạo-3-vm-ubuntu-2404-trên-vmware), nơi một cluster
phẳng dùng Bridged để các node xuất hiện trực tiếp trên LAN vật lý, topology này cần giữ bốn
dải tách biệt và buộc traffic liên dải đi qua router của lab. Theo tài liệu VMware, Bridged
nối VM trực tiếp vào LAN vật lý, host-only tạo private LAN giữa host và các VM, còn NAT dùng
địa chỉ của host để đi ra mạng ngoài: [Broadcom — Understanding networking types in hosted
products](https://knowledge.broadcom.com/external/article?legacyId=1006480).

#### Quan hệ với các lab Bridged đã có và vì sao không dùng VMnet1

Trong cấu hình mẫu của [Lab 00](../k8s-docs/labs/LAB-00-MOI-TRUONG-1.35.7.md#a12-ba-vm)
và [runbook VMware](../runbook-k8s-vmware.md#3-tạo-3-vm-ubuntu-2404-trên-vmware), cả hai hạ
tầng cũ đều dùng Bridged và đặt IP node trên LAN `192.168.100.0/24`. VMware Workstation mặc
định dùng VMnet0 cho Bridged. Tuy nhiên, chỉ runbook VMware pin rõ tên VMnet0 và yêu cầu bind
nó vào card vật lý; Lab 00 chỉ ghi `Network: Bridged`, không pin tên VMnet. Vì vậy, khi Lab 00
đi qua VMnet0 thì đó là ánh xạ Bridged của VMware host, không phải một giá trị được Lab 00
quy định.

VMnet1 mặc định là host-only: nó tạo private LAN giữa Windows host và các VM cùng mạng, nhưng
không nối trực tiếp VM vào LAN vật lý và không tự cung cấp Internet egress. Do đó không thay
Bridged của hai lab cũ bằng VMnet1; nếu làm vậy, topology LAN/gateway và các gate Internet sẽ
không còn đúng nếu không bổ sung một router/NAT khác nằm ngoài hai tài liệu đó.

Track M1 cũng không thể biểu diễn bốn zone ADMIN, CICD, APP và DATA bằng một VMnet1 duy nhất:
bốn zone cần bốn VMnet host-only tách biệt để traffic liên dải buộc phải đi qua `mc-router`.
Số hiệu VMnet không bắt buộc phải dùng liên tục, nên track này không dùng hoặc thay đổi VMnet1
và dành VMnet2–VMnet5 cho bốn zone. Không xóa hay cấu hình lại VMnet1 chỉ để làm lab này.

Tạo các VM của track này **không tự động thay đổi** hạ tầng Bridged cũ nếu giữ nguyên VMnet0
và cấu hình VMnet8. Track chỉ dùng VMnet8 hiện hữu làm uplink NAT của `mc-router`, không yêu
cầu đổi subnet hoặc DHCP của VMnet8. Trước khi tạo VMnet2–VMnet5, xác nhận chúng chưa được
VM/lab khác sử dụng và các dải `10.20.10.0/24`–`10.20.40.0/24` không trùng LAN, VPN,
WSL/Hyper-V hoặc route hiện có trên Windows — [bước 0 của §3.1](#31-tạo-4-vmnet-host-only)
là gate kiểm bằng lệnh cho chính việc này. Thay subnet, type hoặc DHCP của một VMnet đang
dùng sẽ ảnh hưởng mọi VM nối vào VMnet đó.

Các lab không xung đột IP theo giá trị mẫu, nhưng nếu chạy đồng thời thì vẫn cộng dồn nhu cầu
CPU, RAM và disk; riêng một cluster cũ 20 GB RAM cộng track này 27 GB RAM đã là 47 GB cho VM,
chưa tính Windows và VMware.

### 2.3. Biến đầu vào

Mọi giá trị người học phải tự điền được liệt kê ở đây, một chỗ duy nhất. **Trước khi bắt đầu
§4** (lúc `mc-admin1` đã SSH được), sinh sẵn toàn bộ nhóm "tự sinh" bằng `openssl` và lưu
vào password manager hoặc secret note mã hóa trên Windows host; không lưu trong repo, VM hay
transcript. Các mục sau chỉ tra nguồn này. Một secret điền lệch nhau giữa hai mục
(ví dụ mật khẩu DB ở §6.2 và secret ở §6.3) sẽ fail ở gate với thông báo không nói thẳng
nguyên nhân — quy ước một-chỗ-duy-nhất này tồn tại để chặn đúng lỗi đó.

| Placeholder trong lab | Là gì | Cách sinh / lấy | Dùng ở |
| --- | --- | --- | --- |
| `<SINH-CHUỖI-NGẪU-NHIÊN-DÀI>` | token RKE2 cụm Admin | `openssl rand -hex 16` | §4.1 |
| `<TOKEN-RIÊNG-CỦA-CỤM-APP>` | token RKE2 cụm App | `openssl rand -hex 16` — **khác** token Admin | §5.1 (server và agent phải trùng nhau) |
| `<TOKEN-RIÊNG-CỦA-CỤM-CICD>` | token RKE2 cụm CICD | `openssl rand -hex 16` | §6.1 |
| `<ĐẶT-MẬT-KHẨU-BOOTSTRAP>` | mật khẩu đăng nhập Rancher lần đầu | tự đặt; đổi ngay sau lần login đầu | §4.3 |
| `<TOKEN-UI-SINH-RA>` | token import cluster | Rancher UI sinh khi tạo Import Existing | §5.4, §6.1 |
| `<MẬT-KHẨU-DB>` | mật khẩu role `gitlab` của PostgreSQL | `openssl rand -base64 24` | §6.2 và secret `gitlab-psql` ở §6.3 — **phải trùng** |
| `<MẬT-KHẨU-REDIS>` | requirepass của Redis | `openssl rand -base64 24` | §6.2 và secret `gitlab-redis` ở §6.3 — **phải trùng** |
| `<MẬT-KHẨU-MINIO>` | MINIO_ROOT_PASSWORD | `openssl rand -base64 24` | §6.2 (`minio.env` + `mc alias`) và hai secret object-storage/registry-storage ở §6.3 — **phải trùng** |
| `glrt-<TOKEN>` | runner authentication token | GitLab UI → Admin → Runners → New instance runner | §6.4 |
| `<TOKEN-argocd-read>` | deploy token đọc repo | GitLab UI → group `platform` → Deploy tokens, scope `read_repository` | §7.2 |
| `<TOKEN-registry-pull>` | deploy token pull image | như trên, scope `read_registry` | §7.2 |
| `<TOKEN-git-push>` | Personal Access Token của `root` để push code và đọc trạng thái pipeline | GitLab UI → avatar root → Edit profile → Access tokens, scope `api` và `write_repository` | §7.2, §9 |
| `<tag từ §6.4>` | tag image demo | chính là `CI_COMMIT_SHORT_SHA` của pipeline xanh đầu tiên | §7.2 |

Trên `mc-admin1`, chạy từng lệnh dưới đây, chép output thẳng vào password manager rồi xóa
terminal (`clear`); không redirect output ra file:

```bash
printf 'RKE2_ADMIN_TOKEN='; openssl rand -hex 16
printf 'RKE2_APP_TOKEN='; openssl rand -hex 16
printf 'RKE2_CICD_TOKEN='; openssl rand -hex 16
printf 'RANCHER_BOOTSTRAP='; openssl rand -base64 24
printf 'POSTGRES_PASSWORD='; openssl rand -base64 24
printf 'REDIS_PASSWORD='; openssl rand -base64 24
printf 'MINIO_PASSWORD='; openssl rand -base64 24
clear
```

**Gate:** password manager có đủ bảy giá trị không rỗng; ba token RKE2 khác nhau. Nếu thiếu
hoặc trùng, sinh lại trước §4. Các token UI/PAT ở nửa dưới bảng chỉ tạo khi UI tương ứng đã
tồn tại.

Sang [Lab M2](LAB-M2-CAPSTONE-PRODUCTION-HA.md), các biến cùng vai trò không nằm rải trong
tài liệu nữa mà dồn vào `inventories/prod/group_vars/all_vault.yml` (ansible-vault):
token RKE2 ba cụm → `vault_rke2_token_{admin,cicd,app}`, mật khẩu DB/Redis/MinIO →
`vault_db_password`, `vault_redis_password`, `vault_minio_*`. Bảng trên chính là danh sách
khởi tạo vault của M2.

### 2.4. Quy ước khi một gate FAIL

Ba bước, áp cho mọi gate trong lab:

1. **Dừng — không làm bước kế tiếp.** Mọi mục sau đều giả định gate trước đã PASS; đi tiếp
   trên nền fail chỉ làm lỗi lan rộng và khó lần ngược.
2. **Chẩn đoán tại chỗ** theo [bảng troubleshooting §10](#10-troubleshooting-của-lab-này) và
   log của thành phần liên quan. Sửa được thì chạy lại đúng gate đó cho tới PASS.
3. **Trạng thái đã bẩn hoặc không chẩn đoán ra trong ~30 phút:** restore snapshot đầu vào
   của mục đang làm (bảng dưới) rồi làm lại **từ đầu mục đó**. Chuỗi snapshot tồn tại chính
   là để bước này rẻ.

| Gate fail ở mục | Restore về | Ghi chú |
| --- | --- | --- |
| §3 | snapshot golden của VM lỗi | hạ tầng chưa có mốc chung; VM nào hỏng làm lại VM đó |
| §4 | `m1-infra-ready` | |
| §5 | `m1-admin-ready` | cụm Admin và Rancher giữ nguyên, chỉ làm lại cụm App |
| §6 | `m1-app-ready` | |
| §7–§9 | `m1-cicd-ready` | các mục này chỉ đổi trạng thái trong cluster, thường chỉ cần xóa resource lỗi (`kubectl delete`) thay vì restore cả VM |

Restore snapshot là thao tác trên **toàn bộ** các VM có trong mốc đó (trạng thái ba cụm đan
nhau qua Rancher import — restore lệch một VM là lệch mốc).

**Protocol snapshot nhất quán — áp dụng cho mọi mốc `m1-*`:** không chụp riêng lẻ sáu VM
đang ghi dữ liệu. Trước khi shutdown: không còn pipeline/Helm/Argo sync hoặc Longhorn test
đang chạy; đã đóng mọi `kubectl port-forward`/SSH tunnel; đã xóa `wildcard.key`, AskPass
helper và các file secret tạm. `pgrep -af 'kubectl.*port-forward'` phải không có output và
`test ! -e /tmp/wildcard.key` phải PASS trên Admin/CICD/App. Sau khi gate của mục PASS,
shutdown guest sạch theo thứ tự
`mc-cicd1 → mc-app2 → mc-app1 → mc-admin1 → mc-db1 → mc-router`; chờ VMware hiển thị cả sáu
VM `Powered Off`, rồi chụp cùng một tên snapshot trên **đủ sáu VM**. Khởi động lại theo thứ
tự `mc-router → mc-db1 → mc-admin1 → mc-app1 → mc-app2 → mc-cicd1`; chờ mỗi máy lên trước
máy kế và chạy lại gate vừa PASS. Shutdown CICD trước DB làm dừng writer GitLab trước khi
PostgreSQL/Redis/MinIO flush và tắt; powered-off snapshot tránh lệch filesystem.

Khi restore: power off đủ sáu VM, revert **cả sáu** về cùng tên mốc, khởi động theo thứ tự
trên và chạy lại gate của mốc. Nếu bất kỳ VM nào thiếu snapshot cùng tên hoặc gate sau restore
fail, **STOP**; không tiếp tục trên một bộ VM lệch mốc. Snapshot là rollback của lab, không
thay thế backup ứng dụng production.

## 3. Hạ tầng: mạng, router và các VM

### 3.1. Tạo 4 VMnet host-only

**Bước 0 — gate tiền đề, chạy TRƯỚC khi tạo bất kỳ VMnet nào** (chạy sau khi đã tạo thì
điều kiện (3) mất giá trị, vì chính các host adapter mới sẽ sinh route `10.20.x`):

```powershell
# (1) Subnet NAT của VMnet8 — mc-golden (§3.2.2) và uplink của mc-router dùng mạng này:
Get-NetIPAddress -AddressFamily IPv4 |
  Where-Object InterfaceAlias -like '*VMnet8*' |
  Select-Object InterfaceAlias, IPAddress
# ví dụ ra 192.168.229.1 → subnet NAT là 192.168.229.0/24

# (2) Dải LAN thật của host:
ipconfig | findstr /i "Default Gateway"

# (3) Route 10.20.x đã tồn tại từ trước (VPN, WSL, Hyper-V, lab khác):
Get-NetRoute -AddressFamily IPv4 |
  Where-Object DestinationPrefix -like '10.20.*' |
  Select-Object DestinationPrefix, InterfaceAlias
```

**PASS (đủ cả ba):** subnet VMnet8 ở (1) **không** bắt đầu bằng `10.20.` và **khác** dải
gateway ở (2); lệnh (3) không in ra route nào.

**FAIL ở (1) hoặc (3):** đừng sửa VMnet8 — đổi subnet/DHCP của nó ảnh hưởng mọi VM NAT hiện
có trên máy. Thay vào đó đổi quy hoạch `10.20.x` của lab này sang dải trống khác, sửa đồng
bộ: bảng §2.2, netplan §3.3, nftables + dnsmasq §3.4 và hosts file ở §3.5. **FAIL ở (2)**
(LAN trùng subnet VMnet8) nghĩa là máy bạn vốn đã có xung đột NAT tiềm ẩn ngoài phạm vi lab
— xử lý xong mới tiếp tục.

Sau khi bước 0 PASS: trên VMware Workstation mở **Edit → Virtual Network Editor → Change
Settings** (cần quyền admin), thêm 4 mạng host-only. Với **từng** VMnet: **bỏ tick "Use local DHCP service"** (IP
đặt tĩnh, DHCP của VMware chỉ gây nhiễu) và đặt subnet:

| VMnet | Subnet IP | Subnet mask | Host virtual adapter (tick Connect a host virtual adapter) |
| --- | --- | --- | --- |
| VMnet2 | `10.20.10.0` | `255.255.255.0` | host nhận `10.20.10.254` |
| VMnet3 | `10.20.20.0` | `255.255.255.0` | host nhận `10.20.20.254` |
| VMnet4 | `10.20.30.0` | `255.255.255.0` | host nhận `10.20.30.254` |
| VMnet5 | `10.20.40.0` | `255.255.255.0` | host nhận `10.20.40.254` |

VMware thường gán địa chỉ `.1` cho host virtual adapter mới, trong khi `.1` của bốn dải
được dành cho `mc-router`. Vì vậy phải đặt `.254` tĩnh trên Windows; chỉ tick host adapter
trong Virtual Network Editor **không** bảo đảm có đúng địa chỉ này. Chạy PowerShell **Run as
Administrator** sau khi đã tạo đủ bốn VMnet:

```powershell
$m1HostAdapters = [ordered]@{
  'VMnet2' = '10.20.10.254'
  'VMnet3' = '10.20.20.254'
  'VMnet4' = '10.20.30.254'
  'VMnet5' = '10.20.40.254'
}

foreach ($vmnet in $m1HostAdapters.Keys) {
  $alias = "VMware Network Adapter $vmnet"
  $adapter = Get-NetAdapter -Name $alias -ErrorAction Stop
  Set-NetIPInterface -InterfaceIndex $adapter.ifIndex -AddressFamily IPv4 -Dhcp Disabled

  $wanted = $m1HostAdapters[$vmnet]
  $current = @(Get-NetIPAddress -InterfaceIndex $adapter.ifIndex -AddressFamily IPv4 -ErrorAction SilentlyContinue | Where-Object PrefixOrigin -ne 'WellKnown')
  $current | Where-Object { $_.IPAddress -ne $wanted -and $_.IPAddress -notlike '169.254.*' } |
    Remove-NetIPAddress -Confirm:$false
  if ($current.IPAddress -notcontains $wanted) {
    New-NetIPAddress -InterfaceIndex $adapter.ifIndex -IPAddress $wanted -PrefixLength 24 |
      Out-Null
  }
}
```

Host virtual adapter cho phép Windows host SSH thẳng vào VM của từng dải **không qua router**,
nên firewall giữa các dải ở §3.4 không bao giờ khóa mất đường quản trị của bạn.

Verify trên PowerShell của host:

```powershell
Get-NetIPAddress -AddressFamily IPv4 |
  Where-Object InterfaceAlias -like 'VMware Network Adapter VMnet*' |
  Where-Object IPAddress -like '10.20.*' |
  Select-Object InterfaceAlias, IPAddress
$actual = foreach ($vmnet in $m1HostAdapters.Keys) {
  Get-NetIPAddress -InterfaceAlias "VMware Network Adapter $vmnet" -AddressFamily IPv4 |
    Where-Object IPAddress -eq $m1HostAdapters[$vmnet]
}
if ($actual.Count -ne 4 -or ($actual | Where-Object PrefixLength -ne 24)) {
  throw 'FAIL: VMnet2..5 chưa có đúng bốn địa chỉ .254/24'
}
$badDefault = foreach ($vmnet in $m1HostAdapters.Keys) {
  Get-NetRoute -InterfaceAlias "VMware Network Adapter $vmnet" `
    -DestinationPrefix '0.0.0.0/0' -ErrorAction SilentlyContinue
}
if ($badDefault) { throw 'FAIL: host-only adapter không được có default route' }
'PASS: VMnet2..5 = .254/24 và không có default route'
```

**PASS:** thấy đúng 4 adapter `VMware Network Adapter VMnet2..5`, lần lượt mang
`10.20.10.254`, `10.20.20.254`, `10.20.30.254`, `10.20.40.254`; không adapter nào mang `.1`.
Nếu sai, **STOP** tại đây — chưa tạo VM cho tới khi bốn địa chỉ khớp tuyệt đối.

### 3.2. Dựng VM gốc Ubuntu 24.04 và nhân bản 6 VM

Toàn bộ quy trình viết trọn tại đây, không tham chiếu "làm như runbook khác". Nguyên tắc:
cài Ubuntu **một lần duy nhất** trên VM gốc `mc-golden`, làm sạch và chuẩn bị chung, snapshot,
rồi full-clone ra 6 VM và tách identity từng bản. Mọi VM của lab đều là hậu duệ của một
snapshot — môi trường đồng nhất, dựng lại rẻ.

#### 3.2.1. Tải và kiểm ISO

Tải `ubuntu-24.04.x-live-server-amd64.iso` từ <https://ubuntu.com/download/server> cùng file
`SHA256SUMS` của Canonical. Kiểm trên PowerShell của host **trước khi** gắn vào VM:

```powershell
$isoPath = 'D:\ISO\ubuntu-24.04.4-live-server-amd64.iso'   # đổi theo nơi lưu
$expected = '<giá trị dòng tương ứng trong SHA256SUMS>'     # với 24.04.4, đối chiếu thêm
                                                            # bảng A1.3 của Lab 00
$actual = (Get-FileHash -Algorithm SHA256 -LiteralPath $isoPath).Hash.ToLowerInvariant()
if ($actual -ne $expected) { throw "FAIL: ISO SHA256 mismatch" }
'PASS: ISO hợp lệ'
```

#### 3.2.2. Tạo VM gốc `mc-golden`

Trên VMware Workstation:

1. **File → New Virtual Machine → Typical**.
2. Ở bước chọn nguồn cài, chọn **"I will install the operating system later"** — KHÔNG trỏ
   ISO ngay tại đây. Lý do: trỏ ISO lúc tạo VM sẽ kích hoạt **Easy Install** của VMware, nó
   tự trả lời installer bằng đáp án riêng (user, partition) và phá các lựa chọn ở bước 6.
3. Guest OS: **Linux → Ubuntu 64-bit**. Tên VM: `mc-golden`.
4. Disk: **40 GB**, *Store virtual disk as a single file* (thin — chỉ chiếm chỗ thật theo
   dung lượng dùng).
5. **Customize Hardware**: 2 vCPU, 4096 MB RAM (chỉ cho lúc cài; clone sẽ đặt lại theo bảng
   §2.2); **Network Adapter → NAT (VMnet8)** — golden dùng NAT để có DHCP + Internet lúc
   cài, tuyệt đối không đụng Bridged/VMnet0 của các cluster cũ; Firmware: **UEFI** (tab
   Options → Advanced nếu không thấy ở Hardware). Close → Finish.
6. VM Settings → CD/DVD → *Use ISO image file* → trỏ file ISO đã kiểm → tick *Connect at
   power on* → **Power On** và cài Ubuntu Server với các lựa chọn sau (mục nào không nhắc
   thì giữ mặc định):
   - *Try or Install Ubuntu Server* → ngôn ngữ/bàn phím tùy bạn.
   - Type of installation: **Ubuntu Server** (bản chuẩn, không chọn *minimized* — cần đủ
     công cụ chẩn đoán).
   - Network: để DHCP trên NIC duy nhất — PASS tại chỗ: NIC hiện IP dải NAT của VMnet8.
   - Proxy: để trống. Mirror: giữ mặc định.
   - Storage: **Use an entire disk** và **BỎ TICK "Set up this disk as an LVM group"** —
     không LVM thì bước mở rộng disk ở 3.2.5 chỉ cần `growpart` + `resize2fs`, ít tầng để
     sai hơn.
   - Profile: name `ubuntu`, server name `mc-golden`, username `ubuntu`, mật khẩu tự đặt
     (ghi vào file secrets của §2.3).
   - Ubuntu Pro: Skip. **SSH Setup: tick "Install OpenSSH server"**. Featured snaps: không
     chọn gì.
7. Cài xong → *Reboot Now* → khi VM lên lại, VM Settings → CD/DVD → bỏ tick *Connect at
   power on* (nhả ISO).

```bash
# Đăng nhập console (hoặc SSH ubuntu@<IP DHCP mà console hiển thị>) và verify nền:
lsb_release -d          # PASS: Ubuntu 24.04.x LTS
ip -br a                # PASS: một NIC (thường ens33) có IP dải NAT
lsblk                   # PASS: sda1 = ESP, sda2 = / (không có lvm)
```

#### 3.2.3. Chuẩn bị chung trên golden — làm MỘT lần, mọi clone thừa hưởng

```bash
sudo apt-get update && sudo apt-get dist-upgrade -y
# Công cụ nền: open-vm-tools (tương tác VMware), cloud-guest-utils (growpart cho 3.2.5):
sudo apt-get install -y open-vm-tools cloud-guest-utils curl
# Tắt swap vĩnh viễn — kubelet của RKE2 không chấp nhận swap:
sudo swapoff -a
sudo sed -i.bak '/\sswap\s/s/^/#/' /etc/fstab
# Đồng bộ giờ — lệch giờ làm TLS giữa các cụm chết ngầm:
sudo timedatectl set-ntp true
# Chặn cloud-init quản mạng — thiếu dòng này, sau reboot cloud-init sinh lại netplan DHCP
# và ghi đè IP tĩnh của §3.3 trên các bản clone:
echo 'network: {config: disabled}' | \
  sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg

swapon --show | wc -l                            # PASS: 0
timedatectl show -p NTPSynchronized --value      # PASS: yes
systemctl is-active open-vm-tools                # PASS: active
```

**Không cài** containerd/kubeadm/kubelet như Lab 00 — RKE2 tự mang containerd riêng; golden
càng ít thứ càng ít lệch.

#### 3.2.4. Snapshot mốc gốc

```bash
sudo shutdown -h now
```

VM tắt hẳn → chuột phải `mc-golden` → **Snapshot → Take Snapshot** → tên `golden-base`.
Snapshot lúc **powered off** để mọi clone khởi đầu từ trạng thái disk sạch, không kèm RAM
state. Từ đây không bật lại `mc-golden` nữa — nó chỉ là nguồn clone.

#### 3.2.5. Full-clone 6 VM và tùy chỉnh phần cứng từng bản

Với **từng** VM trong bảng §2.2: chuột phải `mc-golden` → **Manage → Clone** → nguồn
**An existing snapshot: `golden-base`** → **Create a full clone** (không dùng linked clone —
6 VM phải độc lập disk) → đặt đúng tên (`mc-router`, `mc-admin1`, `mc-cicd1`, `mc-app1`,
`mc-app2`, `mc-db1`).

Clone xong, **chưa bật vội** — mở VM Settings của từng bản và chỉnh theo bảng:

| VM | RAM/vCPU (theo §2.2) | Network Adapter | Disk |
| --- | --- | --- | --- |
| `mc-router` | 1 GB / 1 | adapter 1 giữ **NAT**; **Add** 4 adapter mới → *Custom* → VMnet2, VMnet3, VMnet4, VMnet5 (đúng thứ tự) | giữ 40 GB |
| `mc-admin1` | 6 GB / 4 | đổi sang *Custom* → **VMnet2** | **Expand** → 50 GB |
| `mc-cicd1` | 8 GB / 4 | *Custom* → **VMnet3** | **Expand** → 60 GB |
| `mc-app1` | 4 GB / 2 | *Custom* → **VMnet4** | giữ 40 GB + **Add → Hard Disk → SCSI → 20 GB** (disk Longhorn) |
| `mc-app2` | 4 GB / 2 | *Custom* → **VMnet4** | như `mc-app1` |
| `mc-db1` | 4 GB / 2 | *Custom* → **VMnet5** | giữ 40 GB |

Ghi chú: nút **Expand** (Hard Disk → Expand) chỉ bấm được khi VM đang tắt và bản clone chưa
có snapshot riêng — đúng trạng thái hiện tại. `mc-router`/`mc-db1` thừa hưởng disk 40 GB
thin lớn hơn con số tối thiểu ở bảng §2.2 — chấp nhận, thin không chiếm chỗ thật.

#### 3.2.6. Tách identity từng clone — qua CONSOLE VMware, từng máy một

Các VMnet host-only đã **tắt DHCP** (§3.1) nên clone chưa có IP, chưa SSH được — mọi lệnh
mục này gõ qua console VMware. Bật **từng VM một**, đặt `TARGET_NAME` đúng tên VM đang mở rồi
chạy trọn block. Block tự tạo netplan; không cần nhảy tới §3.3 để tự sửa YAML.

```bash
# ĐỔI đúng một dòng này trên từng clone:
TARGET_NAME=mc-router
# Giá trị hợp lệ: mc-router mc-admin1 mc-cicd1 mc-app1 mc-app2 mc-db1

case "$TARGET_NAME" in
  mc-router) ROLE=router ;;
  mc-admin1) ROLE=node; ADDR=10.20.10.11; GW=10.20.10.1 ;;
  mc-cicd1)  ROLE=node; ADDR=10.20.20.11; GW=10.20.20.1 ;;
  mc-app1)   ROLE=node; ADDR=10.20.30.11; GW=10.20.30.1 ;;
  mc-app2)   ROLE=node; ADDR=10.20.30.12; GW=10.20.30.1 ;;
  mc-db1)    ROLE=node; ADDR=10.20.40.11; GW=10.20.40.1 ;;
  *) echo "STOP: TARGET_NAME không hợp lệ: $TARGET_NAME"; exit 1 ;;
esac

# 1) Tách identity của clone:
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo systemd-machine-id-setup
sudo rm -f /etc/ssh/ssh_host_*
sudo dpkg-reconfigure openssh-server
sudo hostnamectl set-hostname "$TARGET_NAME"

# 2) Bỏ netplan DHCP của golden và tạo file xác định:
sudo mv /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.bak 2>/dev/null || true
sudo rm -f /etc/netplan/01-static.yaml /etc/netplan/01-router.yaml

if [ "$ROLE" = router ]; then
  ip -br link
  echo 'Đối chiếu MAC trong VMware: ens33=NAT, ens34=VMnet2, ens35=VMnet3, ens36=VMnet4, ens37=VMnet5.'
  echo 'Nếu thứ tự khác, STOP và sửa đúng tên interface trong YAML trước khi netplan apply.'
  sudo tee /etc/netplan/01-router.yaml >/dev/null <<'EOF'
network:
  version: 2
  ethernets:
    ens33: {dhcp4: true}
    ens34: {dhcp4: false, addresses: [10.20.10.1/24]}
    ens35: {dhcp4: false, addresses: [10.20.20.1/24]}
    ens36: {dhcp4: false, addresses: [10.20.30.1/24]}
    ens37: {dhcp4: false, addresses: [10.20.40.1/24]}
EOF
  NETPLAN=/etc/netplan/01-router.yaml
else
  NIC=$(ip -o link show | awk -F': ' '$2 != "lo" {sub(/@.*/, "", $2); print $2; exit}')
  test -n "$NIC" || { echo 'STOP: không tìm thấy NIC workload'; exit 1; }
  sudo tee /etc/netplan/01-static.yaml >/dev/null <<EOF
network:
  version: 2
  ethernets:
    $NIC:
      dhcp4: false
      addresses: [$ADDR/24]
      routes: [{to: default, via: $GW}]
      nameservers:
        addresses: [$GW]
        search: [mc.lab]
EOF
  NETPLAN=/etc/netplan/01-static.yaml
fi

sudo chmod 600 "$NETPLAN"
sudo netplan generate
sudo netplan apply

# 3) Gate tại console trước reboot:
ip -br addr
ip route
# PASS node: đúng ADDR và default via GW. PASS router: đủ bốn địa chỉ .1 và default qua ens33.
# Nếu sai, STOP tại console; sửa netplan, chạy lại generate/apply, không reboot sang VM kế.

# 4) Reboot để identity, hostname và network được kiểm lại từ trạng thái sạch:
sudo reboot
```

#### 3.2.7. Verify từng VM sau reboot

SSH từ host vào từng máy qua host virtual adapter (ví dụ `ssh ubuntu@10.20.10.11`) — SSH
được chính là bằng chứng netplan đúng:

```bash
hostnamectl | head -1            # PASS: đúng tên máy
ip -br a                         # PASS: đúng IP tĩnh theo bảng §2.2, không còn IP DHCP
swapon --show | wc -l            # PASS: 0 (thừa hưởng từ golden)
timedatectl show -p NTPSynchronized --value   # PASS: yes
cat /etc/machine-id              # ghi lại — so 6 VM phải ra 6 giá trị KHÁC nhau
```

Gate tổng identity, chạy trên Windows host sau khi cả sáu VM đã lên:

```powershell
$m1Ips = '10.20.10.1','10.20.10.11','10.20.20.11','10.20.30.11','10.20.30.12','10.20.40.11'
$remoteIdentity = 'printf "%s|%s|" "$(cat /etc/machine-id)" "$(cat /sys/class/dmi/id/product_uuid)"; ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub | cut -d " " -f 2'
$identity = foreach ($ip in $m1Ips) {
  $parts = (ssh "ubuntu@$ip" $remoteIdentity).Trim() -split '\|'
  [pscustomobject]@{ IP=$ip; MachineId=$parts[0]; ProductUuid=$parts[1]; SshFingerprint=$parts[2] }
}
$identity | Format-Table -AutoSize
foreach ($field in 'MachineId','ProductUuid','SshFingerprint') {
  if (($identity.$field | Sort-Object -Unique).Count -ne 6) {
    throw "FAIL: $field không unique trên đủ sáu VM"
  }
}
'PASS: machine-id, product UUID và SSH host fingerprint đều unique'
```

Riêng hai máy expand disk, nới partition rồi filesystem (không LVM nên chỉ hai lệnh):

```bash
# mc-admin1 và mc-cicd1:
lsblk                            # xác nhận: sda2 là partition / và disk đã là 50G/60G
M1_GROW=$(sudo growpart /dev/sda 2 2>&1) || \
  printf '%s\n' "$M1_GROW" | grep -q 'NOCHANGE' || { printf '%s\n' "$M1_GROW"; exit 1; }
sudo resize2fs /dev/sda2
df -h / | awk 'NR==2 {print $2}' # PASS: ~49G (admin1) / ~59G (cicd1)
```

Riêng hai node App, xác nhận disk Longhorn tồn tại (chưa format — §5.2 sẽ làm):

```bash
lsblk | grep -c '^sdb'           # PASS: 1 — disk 20 GB thứ hai hiện diện
```

### 3.3. Bảng đối chiếu Netplan — không thao tác lại

§3.2.6 đã tạo và apply file thật. Mục này chỉ để đối chiếu khi gate mạng fail; nếu gate
§3.2.7 đã PASS thì không sửa hoặc apply lại Netplan ở đây.

Mẫu cho VM một NIC (thay IP theo bảng §2.2; gateway và DNS luôn là `.1` của dải đó):

```yaml
# /etc/netplan/01-static.yaml — ví dụ cho mc-admin1
network:
  version: 2
  ethernets:
    ens33:                      # xác nhận tên NIC bằng: ip -br link
      dhcp4: false
      addresses: [10.20.10.11/24]
      routes: [{to: default, via: 10.20.10.1}]
      nameservers:
        addresses: [10.20.10.1]
        search: [mc.lab]
```

`mc-router` có 5 NIC — NIC NAT giữ DHCP, 4 NIC còn lại đặt `.1` tĩnh **không có** route
default (default đi qua NAT):

```yaml
# /etc/netplan/01-router.yaml trên mc-router
network:
  version: 2
  ethernets:
    ens33: {dhcp4: true}                       # NAT ra Internet
    ens34: {dhcp4: false, addresses: [10.20.10.1/24]}
    ens35: {dhcp4: false, addresses: [10.20.20.1/24]}
    ens36: {dhcp4: false, addresses: [10.20.30.1/24]}
    ens37: {dhcp4: false, addresses: [10.20.40.1/24]}
```

Thứ tự `ens33..ens37` phải đối chiếu với thứ tự adapter trong VMware (`ip -br link` +
so MAC trong VM Settings).

**PASS trên mỗi VM:** `ip -br addr` đúng IP; `ip route | grep default` đúng gateway.

### 3.4. Router: NAT, DNS và firewall giữa các dải

Trên `mc-router`:

```bash
sudo apt-get install -y nftables dnsmasq
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-forward.conf
sudo sysctl --system | grep ip_forward     # PASS: net.ipv4.ip_forward = 1
```

Ghi trọn `/etc/nftables.conf` — NAT ra Internet, các dải nói chuyện với nhau tự do **trừ**
dải DATA; chỉ CICD được vào PostgreSQL/Redis/MinIO:

```bash
sudo tee /etc/nftables.conf >/dev/null <<'EOF'
#!/usr/sbin/nft -f
flush ruleset

define IF_NAT   = "ens33"
define NET_DATA = 10.20.40.0/24
define NET_CICD = 10.20.20.0/24

table inet filter {
  chain forward {
    type filter hook forward priority 0; policy accept;
    ct state established,related accept
    # Dải DATA là dải nội bộ: chặn mọi dải khác, trừ CICD → PostgreSQL/Redis/MinIO.
    ip saddr $NET_CICD ip daddr $NET_DATA tcp dport { 5432, 6379, 9000 } accept
    ip daddr $NET_DATA drop
  }
}
table ip nat {
  chain postrouting {
    type nat hook postrouting priority 100;
    oifname $IF_NAT masquerade
  }
}
EOF

sudo nft -c -f /etc/nftables.conf             # syntax check; không đổi ruleset
sudo systemctl enable --now nftables
sudo nft list ruleset | grep -c masquerade    # PASS: 1
sudo nft list chain inet filter forward | grep -F 'ip daddr 10.20.40.0/24 drop'
# PASS: thấy rule drop DATA và rule allow CICD tới {5432,6379,9000}.
```

Ghi trọn `/etc/dnsmasq.conf` — DNS cho toàn bộ lab, forward phần còn lại ra ngoài:

```bash
sudo cp -an /etc/dnsmasq.conf /etc/dnsmasq.conf.pre-m1
sudo tee /etc/dnsmasq.conf >/dev/null <<'EOF'
domain-needed
bogus-priv
listen-address=127.0.0.1,10.20.10.1,10.20.20.1,10.20.30.1,10.20.40.1
server=1.1.1.1
local=/mc.lab/
host-record=mc-admin1.mc.lab,10.20.10.11
host-record=mc-cicd1.mc.lab,10.20.20.11
host-record=mc-app1.mc.lab,10.20.30.11
host-record=mc-app2.mc.lab,10.20.30.12
host-record=mc-db1.mc.lab,10.20.40.11
address=/rancher.mc.lab/10.20.10.11
address=/gitlab.mc.lab/10.20.20.11
address=/registry.mc.lab/10.20.20.11
address=/minio.mc.lab/10.20.40.11
address=/app.mc.lab/10.20.30.11
EOF
sudo dnsmasq --test                            # PASS: syntax check OK
```

Ubuntu đang chạy `systemd-resolved` chiếm cổng 53 — tắt stub listener trước khi start
dnsmasq. **Bắt buộc sửa cả `/etc/resolv.conf`**: mặc định nó là symlink tới stub
`127.0.0.53`; tắt stub listener mà giữ symlink thì chính router mất khả năng phân giải:

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
test -L /etc/resolv.conf || {
  echo 'STOP: /etc/resolv.conf không phải symlink mặc định; chẩn đoán trước khi thay'; exit 1;
}
readlink /etc/resolv.conf | sudo tee /etc/resolv.conf.pre-m1-target >/dev/null
printf '[Resolve]\nDNSStubListener=no\n' |
  sudo tee /etc/systemd/resolved.conf.d/dnsmasq.conf
sudo systemctl restart systemd-resolved
# Thay symlink stub bằng file tĩnh trỏ vào dnsmasq của chính router:
sudo rm -f /etc/resolv.conf
printf 'nameserver 127.0.0.1\nsearch mc.lab\n' | sudo tee /etc/resolv.conf
sudo systemctl enable --now dnsmasq

# Gate resolver của router (lặp lại sau khi reboot router một lần để chắc cấu hình bền):
sudo ss -lntup 'sport = :53' | grep -c dnsmasq   # PASS: >= 1 — dnsmasq đang giữ cổng 53
getent hosts rancher.mc.lab                       # PASS: 10.20.10.11
getent hosts get.rke2.io | head -1                # PASS: có IP — forward ra Internet chạy
```

Nếu gate DNS fail và cần hoàn tác tại chỗ, chạy nguyên block sau rồi chẩn đoán trước khi thử
lại; block phục hồi symlink resolver đã ghi ở trên, không để router mất DNS:

```bash
sudo systemctl disable --now dnsmasq || true
sudo rm -f /etc/resolv.conf
sudo ln -s "$(sudo cat /etc/resolv.conf.pre-m1-target)" /etc/resolv.conf
sudo rm -f /etc/systemd/resolved.conf.d/dnsmasq.conf
sudo systemctl restart systemd-resolved
getent hosts get.rke2.io | head -1                # PASS rollback: có IP
```

### 3.5. Gate hạ tầng

Chạy từ `mc-admin1` (đại diện; lặp lại nhanh trên các VM khác nếu nghi ngờ):

```bash
ping -c2 10.20.20.11 && ping -c2 10.20.30.11        # PASS: liên dải đi qua router
getent hosts rancher.mc.lab gitlab.mc.lab app.mc.lab # PASS: đúng IP bảng §2.2
curl -fsSL -m 10 -o /dev/null https://get.rke2.io && echo PASS-egress
# PASS: in "PASS-egress" — `-f` fail với mã >= 400, `-L` theo redirect, `-m 10` chặn treo;
# không so khớp chuỗi "HTTP/2 200" vì phía website đổi protocol/redirect là gate hỏng oan
```

Chưa kiểm port DATA ở đây vì PostgreSQL/Redis/MinIO chưa được cài; một port chưa có listener
luôn fail kể cả firewall sai và sẽ tạo false PASS. Gate hành vi allow/deny thật nằm ở §6.2,
sau khi ba service đã nghe cổng. Tại mốc này chỉ kiểm cấu trúc ruleset trên `mc-router`:

```bash
sudo nft list chain inet filter forward | grep -F 'ip saddr 10.20.20.0/24 ip daddr 10.20.40.0/24'
sudo nft list chain inet filter forward | grep -F 'ip daddr 10.20.40.0/24 drop'
# PASS: cả hai lệnh đều có output; nếu thiếu một rule thì STOP, chưa snapshot.
```

Từ Windows host, thêm bản ghi hosts để trình duyệt truy cập được các UI
(PowerShell **Run as Administrator**):

```powershell
$hostsPath = 'C:\Windows\System32\drivers\etc\hosts'
$begin = '# BEGIN M1 LAB'
$end = '# END M1 LAB'
$raw = Get-Content -LiteralPath $hostsPath -Raw
$pattern = '(?ms)^# BEGIN M1 LAB\r?\n.*?^# END M1 LAB\r?\n?'
$clean = [regex]::Replace($raw, $pattern, '').TrimEnd()
$nl = [Environment]::NewLine
$block = @(
  $begin
  '10.20.10.11 rancher.mc.lab mc-admin1.mc.lab'
  '10.20.20.11 gitlab.mc.lab registry.mc.lab mc-cicd1.mc.lab'
  '10.20.40.11 minio.mc.lab mc-db1.mc.lab'
  '10.20.30.11 app.mc.lab mc-app1.mc.lab'
  '10.20.30.12 mc-app2.mc.lab'
  $end
) -join $nl
Set-Content -LiteralPath $hostsPath -Encoding ascii -Value ($clean + $nl + $block + $nl)
ipconfig /flushdns | Out-Null
[System.Net.Dns]::GetHostAddresses('mc-app1.mc.lab').IPAddressToString
# PASS: 10.20.30.11; chạy lại block không tạo dòng trùng.
```

Chụp snapshot đủ sáu VM với tên `m1-infra-ready` theo protocol §2.4; khởi động lại và chạy
lại gate §3.5 trước khi sang §4.

## 4. Cụm Admin: RKE2 + cert-manager + CA lab + Rancher

### 4.1. RKE2 server với Traefik ngay từ đầu

Trên `mc-admin1`. Đây là chỗ bài viết gốc vấp: cài mặc định rồi phải đổi ingress sau. Ta khai
`ingress-controller: traefik` **trước khi start** — RKE2 v1.35 ship sẵn cả hai controller và
cho chọn bằng đúng key này:

```bash
sudo install -d -m 0755 /etc/rancher/rke2 /var/lib/rancher/rke2/server/manifests
sudo tee /etc/rancher/rke2/config.yaml >/dev/null <<'EOF'
token: <SINH-CHUỖI-NGẪU-NHIÊN-DÀI>          # openssl rand -hex 16
tls-san:
  - mc-admin1.mc.lab
  - 10.20.10.11
ingress-controller: traefik
EOF

# Phải có trước lần start đầu tiên để Traefik không lên với cấu hình tạm khác.
sudo tee /var/lib/rancher/rke2/server/manifests/rke2-traefik-config.yaml >/dev/null <<'EOF'
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-traefik
  namespace: kube-system
spec:
  valuesContent: |-
    ports:
      web:
        hostPort: 80
      websecure:
        hostPort: 443
    additionalArguments:
      - "--providers.kubernetesingress.ingressendpoint.ip=10.20.10.11"
EOF

curl -sfL https://get.rke2.io -o /tmp/rke2-install.sh
sudo INSTALL_RKE2_VERSION="v1.35.7+rke2r1" INSTALL_RKE2_TYPE="server" \
  sh /tmp/rke2-install.sh
rm -f /tmp/rke2-install.sh
sudo systemctl enable --now rke2-server.service
```

`rke2-server` lần đầu kéo image mất vài phút. Chuẩn bị kubectl:

```bash
sudo systemctl is-active --quiet rke2-server || {
  sudo journalctl -u rke2-server -n 100 --no-pager
  exit 1
}
for i in $(seq 1 120); do
  sudo test -s /etc/rancher/rke2/rke2.yaml && break
  sleep 5
done
sudo test -s /etc/rancher/rke2/rke2.yaml || {
  echo 'STOP: kubeconfig chưa được tạo sau 10 phút' >&2
  exit 1
}
mkdir -p ~/.kube && sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
sudo chown "$(id -u):$(id -g)" ~/.kube/config
grep -qxF 'export PATH=$PATH:/var/lib/rancher/rke2/bin' ~/.bashrc || \
  echo 'export PATH=$PATH:/var/lib/rancher/rke2/bin' >> ~/.bashrc
export PATH="$PATH:/var/lib/rancher/rke2/bin"

kubectl wait --for=condition=Ready node/mc-admin1 --timeout=600s
kubectl -n kube-system wait --for=condition=Ready pod \
  -l app.kubernetes.io/name=traefik --timeout=600s
test "$(kubectl -n kube-system get pods --no-headers 2>/dev/null | grep -c ingress-nginx)" -eq 0
kubectl get node mc-admin1 -o wide
# PASS: ba lệnh không lỗi; mc-admin1 Ready, Traefik Ready, ingress-nginx = 0.
```

Ghim hành vi Traefik bằng `HelmChartConfig` (cơ chế tùy biến add-on chuẩn của RKE2 — file đặt
trong `manifests/` được tự apply): bind cổng 80/443 lên node và **khai tĩnh địa chỉ điền vào
`Ingress.status`** — fix gốc rễ cho vụ "Argo báo Progressing" trong bài viết, làm ngay từ đầu
thay vì đợi sự cố. Lưu ý vì sao dùng `ingressendpoint.ip` chứ không phải `publishedService`:
RKE2 **tắt ServiceLB mặc định**, nên Service của Traefik không bao giờ có LoadBalancer IP —
`publishedService` chỉ copy một status đang rỗng và Argo vẫn treo Progressing; khai IP tĩnh
của node cho kết quả xác định trên bare-metal:

Gate hostPort, không dùng `sleep` cố định:

```bash
for i in $(seq 1 60); do
  CODE=$(curl -sS -o /dev/null -w '%{http_code}' --connect-timeout 2 http://10.20.10.11 || true)
  [ "$CODE" = 404 ] && break
  sleep 5
done
[ "$CODE" = 404 ] || { echo "STOP: Traefik HTTP code=$CODE, cần 404" >&2; exit 1; }
echo 'PASS: Traefik trả HTTP 404 trên hostPort 80'
```

### 4.2. Helm, cert-manager và CA của lab

```bash
# Helm 4 pin cứng; script chính thức tự verify checksum archive tải về:
HELM_VERSION=v4.2.3
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4 -o /tmp/get-helm-4
chmod 700 /tmp/get-helm-4
DESIRED_VERSION="$HELM_VERSION" /tmp/get-helm-4
rm -f /tmp/get-helm-4
helm version --short | grep -F "$HELM_VERSION"
# PASS: output chứa đúng v4.2.3; khác version thì STOP.

helm repo add jetstack https://charts.jetstack.io --force-update
helm repo update
helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --version v1.21.1 --set crds.enabled=true --wait --timeout 10m
for d in cert-manager cert-manager-cainjector cert-manager-webhook; do
  kubectl -n cert-manager rollout status deployment/$d --timeout=300s
done
# PASS: cả ba Deployment successfully rolled out.
```

CA gốc của lab — cert-manager tự ký và **tự gia hạn các cert lá** (Rancher, wildcard), đúng
"chiến lược quản lý cert" trong bài viết. Nói cho chính xác phạm vi tự động: chỉ leaf cert
xoay tự động; khi **root CA** hết hạn hoặc phải thay, trust store trên 6 VM, Windows host và
`registries.yaml` của containerd không tự cập nhật — phải phát hành CA mới song song CA cũ
rồi phân phối lại trust bundle bằng tay (đó là lý do CA gốc dưới đây đặt hạn 5 năm):

```bash
kubectl apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata: {name: selfsigned-bootstrap}
spec: {selfSigned: {}}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata: {name: mc-lab-root-ca, namespace: cert-manager}
spec:
  isCA: true
  commonName: mc-lab-root-ca
  secretName: mc-lab-root-ca
  duration: 43800h            # 5 năm cho CA gốc của lab
  privateKey: {algorithm: ECDSA, size: 256}
  issuerRef: {name: selfsigned-bootstrap, kind: ClusterIssuer}
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata: {name: mc-lab-ca}
spec:
  ca: {secretName: mc-lab-root-ca}
EOF

kubectl -n cert-manager wait certificate/mc-lab-root-ca \
  --for=condition=Ready --timeout=300s
kubectl wait clusterissuer/mc-lab-ca --for=condition=Ready --timeout=300s
test "$(kubectl get clusterissuer mc-lab-ca -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')" = True
# PASS: cả Certificate và ClusterIssuer Ready=True; khác thì STOP.
```

Xuất CA và **trust trên cả 6 VM** — điều kiện bài viết đã nêu để node các cụm khác join về
Rancher; đồng thời giúp lệnh import ở §5.4 dùng được bản `kubectl apply` thường thay vì bản
`curl --insecure`:

```bash
kubectl -n cert-manager get secret mc-lab-root-ca \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > /tmp/mc-lab-ca.crt
# Cài trên chính mc-admin1, rồi copy tới đúng năm VM còn lại.
sudo install -m 0644 /tmp/mc-lab-ca.crt /usr/local/share/ca-certificates/mc-lab-ca.crt
sudo update-ca-certificates
for host in 10.20.10.1 10.20.20.11 10.20.30.11 10.20.30.12 10.20.40.11; do
  scp /tmp/mc-lab-ca.crt "ubuntu@$host:/tmp/mc-lab-ca.crt"
  ssh "ubuntu@$host" 'sudo install -m 0644 /tmp/mc-lab-ca.crt /usr/local/share/ca-certificates/mc-lab-ca.crt && sudo update-ca-certificates && rm -f /tmp/mc-lab-ca.crt && test -s /usr/local/share/ca-certificates/mc-lab-ca.crt'
done
for host in 10.20.10.1 10.20.10.11 10.20.20.11 10.20.30.11 10.20.30.12 10.20.40.11; do
  ssh "ubuntu@$host" 'test -s /usr/local/share/ca-certificates/mc-lab-ca.crt' || exit 1
done
rm -f /tmp/mc-lab-ca.crt
echo 'PASS-ca: đủ sáu VM có CA tại đường dẫn chuẩn'
# Nếu username Ubuntu không phải `ubuntu`, thay username ở cả hai vòng lặp trước khi chạy.
# `/usr/local/share/ca-certificates/mc-lab-ca.crt` là ĐƯỜNG DẪN CHUẨN của CA trên MỌI node —
# tất cả các mục sau tham chiếu đúng path này, không dùng lại /tmp.
```

Trên Windows host, muốn trình duyệt hết cảnh báo thì import `mc-lab-ca.crt` vào *Trusted Root
Certification Authorities* (tùy chọn; không ảnh hưởng gate nào).

### 4.3. Rancher với tls.source=secret + privateCA

Cấp cert cho Rancher từ CA lab rồi cài đúng mô hình bài viết dùng:

```bash
kubectl create namespace cattle-system --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: Certificate
metadata: {name: tls-rancher-ingress, namespace: cattle-system}
spec:
  secretName: tls-rancher-ingress
  dnsNames: [rancher.mc.lab]
  issuerRef: {name: mc-lab-ca, kind: ClusterIssuer}
EOF
kubectl -n cattle-system wait certificate/tls-rancher-ingress \
  --for=condition=Ready --timeout=300s
# privateCA=true yêu cầu secret tls-ca chứa cacerts.pem là CA đã ký cert trên:
kubectl -n cert-manager get secret mc-lab-root-ca -o jsonpath='{.data.ca\.crt}' | \
  base64 -d | kubectl -n cattle-system create secret generic tls-ca \
  --from-file=cacerts.pem=/dev/stdin --dry-run=client -o yaml | kubectl apply -f -

helm repo add rancher-stable https://releases.rancher.com/server-charts/stable --force-update
helm repo update
read -rsp 'Rancher bootstrap password: ' M1_RANCHER_BOOTSTRAP; echo
helm upgrade --install rancher rancher-stable/rancher \
  --namespace cattle-system --version 2.14.3 \
  --set hostname=rancher.mc.lab \
  --set replicas=1 \
  --set-string bootstrapPassword="$M1_RANCHER_BOOTSTRAP" \
  --set ingress.ingressClassName=traefik \
  --set ingress.tls.source=secret \
  --set privateCA=true --wait --timeout 10m
unset M1_RANCHER_BOOTSTRAP
```

`replicas=1` là cấu hình homelab có chủ đích — đây chính là SPOF mà nhận xét đã chỉ ra ở bài
viết gốc; [Lab M2](LAB-M2-CAPSTONE-PRODUCTION-HA.md) sửa nó bằng Rancher HA 3 replica.

```bash
kubectl -n cattle-system rollout status deploy/rancher --timeout=600s
# PASS: successfully rolled out
curl -s --cacert /usr/local/share/ca-certificates/mc-lab-ca.crt https://rancher.mc.lab/healthz
# PASS: ok
```

Mở `https://rancher.mc.lab` từ host, đăng nhập bằng bootstrap password, đổi mật khẩu, xác
nhận **Server URL** là `https://rancher.mc.lab`. **PASS §4:** cụm `local` Active trong UI.

Snapshot đủ sáu VM: `m1-admin-ready`, theo protocol §2.4; sau khởi động lại, gate healthz và
Rancher `local` Active phải PASS lại.

## 5. Cụm App: RKE2 hai node + Longhorn + import vào Rancher

### 5.1. RKE2 server và agent

Trên `mc-app1` — thêm `deployment.kind: DaemonSet` cho Traefik để cả 2 node đều nhận traffic
cổng 80/443:

```bash
sudo install -d -m 0755 /etc/rancher/rke2 /var/lib/rancher/rke2/server/manifests
sudo tee /etc/rancher/rke2/config.yaml >/dev/null <<'EOF'
token: <TOKEN-RIÊNG-CỦA-CỤM-APP>
tls-san: [mc-app1.mc.lab, 10.20.30.11]
ingress-controller: traefik
EOF
curl -sfL https://get.rke2.io -o /tmp/rke2-install.sh
sudo INSTALL_RKE2_VERSION="v1.35.7+rke2r1" INSTALL_RKE2_TYPE="server" sh /tmp/rke2-install.sh
rm -f /tmp/rke2-install.sh

# HelmChartConfig Traefik của cụm App — khác §4.1 ở khối `deployment` (DaemonSet để CẢ HAI
# node đều nhận traffic 80/443) và ingressendpoint trỏ IP của mc-app1 (địa chỉ mà DNS
# app.mc.lab đang trỏ về):
sudo tee /var/lib/rancher/rke2/server/manifests/rke2-traefik-config.yaml >/dev/null <<'EOF'
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-traefik
  namespace: kube-system
spec:
  valuesContent: |-
    deployment:
      kind: DaemonSet
    ports:
      web:
        hostPort: 80
      websecure:
        hostPort: 443
    additionalArguments:
      - "--providers.kubernetesingress.ingressendpoint.ip=10.20.30.11"
EOF
sudo systemctl enable --now rke2-server.service
```

Trên `mc-app2` — agent join qua cổng đăng ký 9345:

```bash
sudo install -d -m 0755 /etc/rancher/rke2 /var/lib/rancher/rke2/server/manifests
sudo tee /etc/rancher/rke2/config.yaml >/dev/null <<'EOF'
server: https://10.20.30.11:9345
token: <TOKEN-RIÊNG-CỦA-CỤM-APP>
EOF
curl -sfL https://get.rke2.io -o /tmp/rke2-install.sh
sudo INSTALL_RKE2_TYPE="agent" INSTALL_RKE2_VERSION="v1.35.7+rke2r1" sh /tmp/rke2-install.sh
rm -f /tmp/rke2-install.sh
sudo systemctl enable --now rke2-agent.service
sudo systemctl is-active --quiet rke2-agent || {
  sudo journalctl -u rke2-agent -n 100 --no-pager
  exit 1
}
```

Trên `mc-app1` (kubectl cấu hình như §4.1):

```bash
sudo systemctl is-active --quiet rke2-server || {
  sudo journalctl -u rke2-server -n 100 --no-pager
  exit 1
}
for i in $(seq 1 120); do
  sudo test -s /etc/rancher/rke2/rke2.yaml && break
  sleep 5
done
sudo test -s /etc/rancher/rke2/rke2.yaml || { echo 'STOP: thiếu kubeconfig' >&2; exit 1; }
mkdir -p ~/.kube && sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
sudo chown "$(id -u):$(id -g)" ~/.kube/config
export PATH="$PATH:/var/lib/rancher/rke2/bin"
kubectl wait --for=condition=Ready node/mc-app1 node/mc-app2 --timeout=600s
kubectl -n kube-system rollout status daemonset/rke2-traefik --timeout=600s
test "$(kubectl -n kube-system get pods --no-headers 2>/dev/null | grep -c ingress-nginx)" -eq 0
kubectl get nodes -o wide
# PASS: hai node Ready, Traefik DaemonSet rolled out trên cả hai, ingress-nginx = 0.
```

### 5.2. Chuẩn bị node cho Longhorn

Trên **cả** `mc-app1` và `mc-app2`:

```bash
sudo apt-get install -y open-iscsi nfs-common multipath-tools
sudo systemctl enable --now iscsid
systemctl is-active iscsid            # PASS: active

# Disk thứ hai cho Longhorn. Tuyệt đối không format nếu preflight không khớp.
M1_DISK=/dev/sdb
[ -b "$M1_DISK" ] || { echo 'STOP: không có /dev/sdb' >&2; exit 1; }
lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,MOUNTPOINTS "$M1_DISK"
[ "$(lsblk -dn -o TYPE "$M1_DISK")" = disk ] || {
  echo 'STOP: /dev/sdb không phải whole disk' >&2; exit 1;
}
M1_FSTYPE=$(lsblk -dn -o FSTYPE "$M1_DISK")
M1_LABEL=$(lsblk -dn -o LABEL "$M1_DISK")
case "$M1_FSTYPE:$M1_LABEL" in
  :) sudo mkfs.ext4 -L m1-longhorn "$M1_DISK" ;;
  ext4:m1-longhorn) echo 'INFO: filesystem M1 đã tồn tại; không format lại' ;;
  *) echo "STOP: $M1_DISK có FSTYPE=$M1_FSTYPE LABEL=$M1_LABEL; không được ghi đè" >&2; exit 1 ;;
esac
M1_UUID=$(sudo blkid -s UUID -o value "$M1_DISK")
[ -n "$M1_UUID" ] || { echo 'STOP: không đọc được UUID /dev/sdb' >&2; exit 1; }
sudo install -d -m 0755 /var/lib/longhorn
M1_FSTAB="UUID=$M1_UUID /var/lib/longhorn ext4 defaults 0 2"
grep -qxF "$M1_FSTAB" /etc/fstab || printf '%s\n' "$M1_FSTAB" | sudo tee -a /etc/fstab
mountpoint -q /var/lib/longhorn || sudo mount /var/lib/longhorn
findmnt -rn -S "UUID=$M1_UUID" -T /var/lib/longhorn | grep -q '/var/lib/longhorn' || {
  echo 'STOP: Longhorn mount không đúng UUID' >&2; exit 1;
}

# multipathd chiếm device của Longhorn → BLACKLIST theo KB chính thức, không disable service:
sudo tee /etc/multipath.conf >/dev/null <<'EOF'
blacklist {
    devnode "^sd[a-z0-9]+"
}
EOF
sudo systemctl restart multipathd 2>/dev/null || true
sudo multipath -t | grep -A3 'blacklist {' | grep -q 'sd\[a-z0-9\]' || {
  echo 'STOP: multipath chưa nhận blacklist sd*' >&2; exit 1;
}
echo 'PASS: disk Longhorn mount theo UUID và multipath blacklist đã áp dụng'
```

Bài viết gốc disable hẳn `multipathd`; KB của Longhorn khuyến nghị blacklist — giữ được
multipath cho hệ thống nào thật sự cần nó.

### 5.3. Cài Longhorn với mặc định 2 replica

Trên `mc-app1` (cài Helm 4 bằng đúng khối lệnh ở §4.2):

```bash
helm repo add longhorn https://charts.longhorn.io --force-update && helm repo update
helm upgrade --install longhorn longhorn/longhorn \
  --namespace longhorn-system --create-namespace --version 1.12.1 \
  --set defaultSettings.defaultReplicaCount=2 \
  --set persistence.defaultClassReplicaCount=2 --wait --timeout 10m

kubectl -n longhorn-system rollout status deploy/longhorn-driver-deployer --timeout=600s
kubectl get storageclass
# PASS: longhorn (default)
```

Gate hành vi mà bài viết gặp sự cố — PVC RWO detach/re-attach khi Pod đổi node:

```bash
kubectl create ns lh-test --dry-run=client -o yaml | kubectl apply -f -
cleanup_lh_test() { kubectl delete ns lh-test --ignore-not-found --wait --timeout=300s; }
trap cleanup_lh_test EXIT
kubectl -n lh-test apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: lh-pvc}
spec:
  accessModes: [ReadWriteOnce]
  resources: {requests: {storage: 1Gi}}
---
apiVersion: v1
kind: Pod
metadata: {name: lh-writer}
spec:
  nodeName: mc-app1              # ghim node để phép thử re-attach CHẮC CHẮN đổi node
  containers:
  - name: w
    image: busybox:1.37
    command: [sh, -c, "echo m1-longhorn-ok > /data/proof && sleep 3600"]
    volumeMounts: [{name: d, mountPath: /data}]
  volumes: [{name: d, persistentVolumeClaim: {claimName: lh-pvc}}]
EOF
kubectl -n lh-test wait pod/lh-writer --for=condition=Ready --timeout=300s
kubectl -n lh-test get pod lh-writer -o wide     # ghi lại NODE = mc-app1
kubectl -n lh-test delete pod lh-writer --wait
kubectl -n lh-test apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata: {name: lh-reader}
spec:
  nodeName: mc-app2
  containers:
  - name: r
    image: busybox:1.37
    command: [sh, -c, "cat /data/proof && sleep 600"]
    volumeMounts: [{name: d, mountPath: /data}]
  volumes: [{name: d, persistentVolumeClaim: {claimName: lh-pvc}}]
EOF
kubectl -n lh-test wait pod/lh-reader --for=condition=Ready --timeout=300s
test "$(kubectl -n lh-test get pod lh-reader -o jsonpath='{.spec.nodeName}')" = mc-app2
kubectl -n lh-test logs lh-reader | grep -qx m1-longhorn-ok
echo 'PASS: volume detach mc-app1, attach mc-app2 và giữ nguyên dữ liệu'
# PASS — volume detach khỏi mc-app1 và re-attach sang mc-app2 sạch,
# không dính MountVolume.SetUp failed
cleanup_lh_test
trap - EXIT
```

Ghi nhớ vận hành (nợ của lab, trả ở M2 §12): khi rolling update node, volume 1 replica sẽ chặn
drain bởi policy mặc định `block-if-contains-last-replica` — lý do lab ép mặc định 2 replica.

### 5.4. Import cụm App vào Rancher

Rancher UI → **Cluster Management → Import Existing → Generic**, đặt tên `mc-app`. UI đưa hai
lệnh; vì CA lab đã được trust trên mọi node (§4.2), dùng bản thường trên `mc-app1`:

```bash
read -rsp 'Dán URL import mc-app do Rancher UI sinh: ' M1_IMPORT_URL; echo
kubectl apply -f "$M1_IMPORT_URL"
unset M1_IMPORT_URL
kubectl -n cattle-system rollout status deploy/cattle-cluster-agent --timeout=300s
# PASS: cattle-cluster-agent rolled out
kubectl -n cattle-system logs deploy/cattle-cluster-agent | grep -i "connect" | tail -3
# PASS: log báo Connecting/Connected tới wss://rancher.mc.lab — agent mở outbound, không cần
# mở cổng inbound nào ở cụm App
```

**PASS §5:** Rancher UI hiện `mc-app` **Active** với 2 node. Đây chính là mảnh "downstream
import" mà cả runbook cũ lẫn nhận xét đã đánh dấu là chưa hoàn tất.

Snapshot đủ sáu VM: `m1-app-ready`, theo protocol §2.4; sau khởi động lại, hai node App Ready,
Longhorn healthy và Rancher `mc-app` Active phải PASS lại.

## 6. Cụm CICD: PostgreSQL ngoài + GitLab + runner

### 6.1. RKE2 server, Helm và import

Toàn bộ chạy trên `mc-cicd1` — khối đầy đủ, không tham chiếu "làm như §4.1":

```bash
sudo install -d -m 0755 /etc/rancher/rke2 /var/lib/rancher/rke2/server/manifests
sudo tee /etc/rancher/rke2/config.yaml >/dev/null <<'EOF'
token: <TOKEN-RIÊNG-CỦA-CỤM-CICD>          # từ bảng biến §2.3, KHÁC token Admin/App
tls-san: [mc-cicd1.mc.lab, 10.20.20.11]
ingress-controller: traefik
EOF
curl -sfL https://get.rke2.io -o /tmp/rke2-install.sh
sudo INSTALL_RKE2_VERSION="v1.35.7+rke2r1" INSTALL_RKE2_TYPE="server" sh /tmp/rke2-install.sh
rm -f /tmp/rke2-install.sh

sudo tee /var/lib/rancher/rke2/server/manifests/rke2-traefik-config.yaml >/dev/null <<'EOF'
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-traefik
  namespace: kube-system
spec:
  valuesContent: |-
    ports:
      web:
        hostPort: 80
      websecure:
        hostPort: 443
    additionalArguments:
      - "--providers.kubernetesingress.ingressendpoint.ip=10.20.20.11"
EOF
sudo systemctl enable --now rke2-server.service

sudo systemctl is-active --quiet rke2-server || {
  sudo journalctl -u rke2-server -n 100 --no-pager
  exit 1
}
for i in $(seq 1 120); do
  sudo test -s /etc/rancher/rke2/rke2.yaml && break
  sleep 5
done
sudo test -s /etc/rancher/rke2/rke2.yaml || { echo 'STOP: thiếu kubeconfig' >&2; exit 1; }
mkdir -p ~/.kube && sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
sudo chown "$(id -u):$(id -g)" ~/.kube/config
grep -qxF 'export PATH=$PATH:/var/lib/rancher/rke2/bin' ~/.bashrc || \
  echo 'export PATH=$PATH:/var/lib/rancher/rke2/bin' >> ~/.bashrc
export PATH="$PATH:/var/lib/rancher/rke2/bin"

kubectl wait --for=condition=Ready node/mc-cicd1 --timeout=600s
kubectl -n kube-system wait --for=condition=Ready pod \
  -l app.kubernetes.io/name=traefik --timeout=600s
test "$(kubectl -n kube-system get pods --no-headers 2>/dev/null | grep -c ingress-nginx)" -eq 0
kubectl get node mc-cicd1 -o wide
# PASS: node và Traefik Ready; ingress-nginx = 0.

# Helm 4 — §6.3 cần đúng bản pin trên chính node này:
HELM_VERSION=v4.2.3
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4 -o /tmp/get-helm-4
chmod 700 /tmp/get-helm-4
DESIRED_VERSION="$HELM_VERSION" /tmp/get-helm-4
rm -f /tmp/get-helm-4
helm version --short | grep -F "$HELM_VERSION"   # PASS: v4.2.3
```

Import vào Rancher: UI → **Cluster Management → Import Existing → Generic**, tên `mc-cicd`,
rồi trên `mc-cicd1`:

```bash
read -rsp 'Dán URL import mc-cicd do Rancher UI sinh: ' M1_IMPORT_URL; echo
kubectl apply -f "$M1_IMPORT_URL"
unset M1_IMPORT_URL
kubectl -n cattle-system rollout status deploy/cattle-cluster-agent --timeout=300s
# PASS: rolled out
```

**PASS §6.1:** Rancher UI hiện 3 cụm: `local`, `mc-app`, `mc-cicd` — đều Active.

### 6.2. PostgreSQL + Redis + MinIO trên mc-db1

GitLab chart 10.x **bắt buộc** cả ba dependency này nằm ngoài chart. Cả ba đặt trên `mc-db1`
(4 GB RAM) — vẫn đúng tinh thần "dịch vụ dữ liệu trên máy chủ riêng, dải nội bộ" của bài
viết; production tách mỗi thứ một máy (ghi ở sổ SPOF của M2).

**PostgreSQL:**

```bash
# GitLab 19.x CHỈ hỗ trợ PostgreSQL 17.x; Ubuntu Noble chỉ đóng gói 16 → cài 17 từ repo
# PGDG chính thức của postgresql.org:
sudo apt-get install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh -y
sudo apt-get install -y postgresql-17
psql --version        # PASS: 17.x

read -rsp 'PostgreSQL password for gitlab: ' M1_DB_PASSWORD; echo
sudo -u postgres psql -v ON_ERROR_STOP=1 -v db_password="$M1_DB_PASSWORD" <<'SQL'
SELECT format('CREATE ROLE gitlab LOGIN PASSWORD %L', :'db_password')
WHERE NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'gitlab')
\gexec
SELECT format('ALTER ROLE gitlab PASSWORD %L', :'db_password')
\gexec
SELECT 'CREATE DATABASE gitlabhq_production OWNER gitlab'
WHERE NOT EXISTS (SELECT 1 FROM pg_database WHERE datname = 'gitlabhq_production')
\gexec
SQL
unset M1_DB_PASSWORD
sudo -u postgres psql -v ON_ERROR_STOP=1 -d gitlabhq_production <<'SQL'
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS btree_gist;
CREATE EXTENSION IF NOT EXISTS amcheck;
SQL

# Drop-in xác định, không bắt người học tự sửa postgresql.conf.
sudo install -d -m 0755 /etc/postgresql/17/main/conf.d
sudo tee /etc/postgresql/17/main/conf.d/99-m1.conf >/dev/null <<'EOF'
listen_addresses = '10.20.40.11'
max_connections = 500
shared_buffers = 1GB
EOF
M1_HBA='host gitlabhq_production gitlab 10.20.20.0/24 scram-sha-256'
grep -qxF "$M1_HBA" /etc/postgresql/17/main/pg_hba.conf || \
  printf '%s\n' "$M1_HBA" | sudo tee -a /etc/postgresql/17/main/pg_hba.conf
sudo systemctl restart postgresql
sudo systemctl is-active --quiet postgresql || {
  sudo journalctl -u postgresql -n 100 --no-pager; exit 1;
}
sudo -u postgres psql -Atqc 'SHOW listen_addresses; SHOW max_connections; SHOW shared_buffers;'
# PASS theo thứ tự: 10.20.40.11, 500, 1GB.
```

Gate hai lớp, chạy đúng nơi ghi trong comment:

```bash
# Trên mc-cicd1:
sudo apt-get install -y postgresql-client
read -rsp 'PostgreSQL password for gitlab: ' M1_DB_PASSWORD; echo
PGPASSWORD="$M1_DB_PASSWORD" psql -h 10.20.40.11 -U gitlab \
  -d gitlabhq_production -Atqc 'SHOW max_connections' | grep -qx 500
unset M1_DB_PASSWORD
echo 'PASS: CICD kết nối PostgreSQL và max_connections=500'

# Trên mc-app1: chỉ dùng netcat; không nhầm lỗi thiếu psql thành firewall PASS.
if nc -vzw3 10.20.40.11 5432; then
  echo 'STOP: APP đi được tới PostgreSQL; firewall DATA sai' >&2
  exit 1
fi
echo 'PASS: APP bị chặn tại firewall DATA'
```

**Redis:**

```bash
sudo apt-get install -y redis-server
read -rsp 'Redis password: ' M1_REDIS_PASSWORD; echo
printf 'requirepass %s\n' "$M1_REDIS_PASSWORD" | sudo tee /etc/redis/m1-secret.conf >/dev/null
unset M1_REDIS_PASSWORD
sudo chown root:redis /etc/redis/m1-secret.conf
sudo chmod 0640 /etc/redis/m1-secret.conf
sudo sed -i -E 's/^bind .*/bind 10.20.40.11 127.0.0.1/' /etc/redis/redis.conf
grep -qxF 'include /etc/redis/m1-secret.conf' /etc/redis/redis.conf || \
  echo 'include /etc/redis/m1-secret.conf' | sudo tee -a /etc/redis/redis.conf
sudo systemctl restart redis-server
sudo systemctl is-active --quiet redis-server || {
  sudo journalctl -u redis-server -n 100 --no-pager; exit 1;
}
# Từ mc-cicd1:
sudo apt-get install -y redis-tools
read -rsp 'Redis password: ' M1_REDIS_PASSWORD; echo
REDISCLI_AUTH="$M1_REDIS_PASSWORD" redis-cli -h 10.20.40.11 ping | grep -qx PONG
unset M1_REDIS_PASSWORD
# PASS: PONG; mật khẩu không xuất hiện trong history hay argv.
```

**MinIO (object storage):**

```bash
id -u minio >/dev/null 2>&1 || sudo useradd -r -s /usr/sbin/nologin minio
sudo install -d -o minio -g minio -m 0750 /srv/minio
sudo install -d -o root -g minio -m 0750 /etc/minio
MINIO_RELEASE=RELEASE.2025-09-07T16-13-09Z
curl -fLo "/tmp/minio.$MINIO_RELEASE" \
  "https://dl.min.io/community/server/minio/release/linux-amd64/minio.$MINIO_RELEASE"
curl -fLo "/tmp/minio.$MINIO_RELEASE.sha256sum" \
  "https://dl.min.io/community/server/minio/release/linux-amd64/minio.$MINIO_RELEASE.sha256sum"
(cd /tmp && sha256sum -c "minio.$MINIO_RELEASE.sha256sum")
sudo install -m 0755 "/tmp/minio.$MINIO_RELEASE" /usr/local/bin/minio
rm -f "/tmp/minio.$MINIO_RELEASE" "/tmp/minio.$MINIO_RELEASE.sha256sum"
minio --version | grep -F 'RELEASE.2025-09-07T16-13-09Z'

read -rp 'MinIO root user [minio-admin]: ' M1_MINIO_USER
M1_MINIO_USER=${M1_MINIO_USER:-minio-admin}
read -rsp 'MinIO root password: ' M1_MINIO_PASSWORD; echo
{
  printf 'MINIO_ROOT_USER=%q\n' "$M1_MINIO_USER"
  printf 'MINIO_ROOT_PASSWORD=%q\n' "$M1_MINIO_PASSWORD"
} | sudo tee /etc/minio/minio.env >/dev/null
sudo chown root:minio /etc/minio/minio.env
sudo chmod 0640 /etc/minio/minio.env

sudo tee /etc/systemd/system/minio.service >/dev/null <<'EOF'
[Unit]
Description=MinIO
Wants=network-online.target
After=network-online.target
[Service]
User=minio
Group=minio
EnvironmentFile=/etc/minio/minio.env
ExecStart=/usr/local/bin/minio server /srv/minio --address 10.20.40.11:9000
Restart=always
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload && sudo systemctl enable --now minio
sudo systemctl is-active --quiet minio || {
  sudo journalctl -u minio -n 100 --no-pager; exit 1;
}
for i in $(seq 1 30); do
  curl -fsS http://10.20.40.11:9000/minio/health/live >/dev/null && break
  sleep 2
done
curl -fsS http://10.20.40.11:9000/minio/health/live >/dev/null || {
  echo 'STOP: MinIO health chưa PASS' >&2; exit 1;
}

# Tạo bucket bằng mc (MinIO client) ngay trên mc-db1:
MC_RELEASE=RELEASE.2025-08-13T08-35-41Z
curl -fLo "/tmp/mc.$MC_RELEASE" \
  "https://dl.min.io/client/mc/release/linux-amd64/mc.$MC_RELEASE"
curl -fLo "/tmp/mc.$MC_RELEASE.sha256sum" \
  "https://dl.min.io/client/mc/release/linux-amd64/mc.$MC_RELEASE.sha256sum"
(cd /tmp && sha256sum -c "mc.$MC_RELEASE.sha256sum")
sudo install -m 0755 "/tmp/mc.$MC_RELEASE" /usr/local/bin/mc
rm -f "/tmp/mc.$MC_RELEASE" "/tmp/mc.$MC_RELEASE.sha256sum"
mc --version | grep -F 'RELEASE.2025-08-13T08-35-41Z'
mc alias set lab http://10.20.40.11:9000 "$M1_MINIO_USER" "$M1_MINIO_PASSWORD"
for b in gitlab-artifacts gitlab-lfs gitlab-uploads gitlab-packages gitlab-mr-diffs \
         gitlab-terraform-state gitlab-dependency-proxy gitlab-ci-secure-files \
         gitlab-registry gitlab-backups gitlab-tmp; do mc mb --ignore-existing "lab/$b"; done
test "$(mc ls lab | wc -l)" -eq 11 || { echo 'STOP: cần đúng 11 bucket' >&2; exit 1; }
mc alias remove lab
unset M1_MINIO_USER M1_MINIO_PASSWORD
echo 'PASS: MinIO healthy, đúng checksum/version và đủ 11 bucket'
```

### 6.3. GitLab qua Helm, ingress Traefik, DB ngoài

Trên `mc-cicd1`. Wildcard cert cho `*.mc.lab` lấy từ CA lab — cấp trên cụm Admin rồi copy
secret sang (mô phỏng "cert wildcard mua một lần" của bài viết; reflector nhân bản nội cụm ở
§7.1):

```bash
# Trên mc-admin1:
kubectl apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: Certificate
metadata: {name: mc-lab-wildcard, namespace: cert-manager}
spec:
  secretName: mc-lab-wildcard-tls
  dnsNames: ["*.mc.lab", "mc.lab"]
  issuerRef: {name: mc-lab-ca, kind: ClusterIssuer}
EOF
kubectl -n cert-manager wait certificate/mc-lab-wildcard \
  --for=condition=Ready --timeout=300s
# Xuất cert/key ra file — KHÔNG copy YAML của secret (khỏi phải gọt metadata bằng tay):
kubectl -n cert-manager get secret mc-lab-wildcard-tls \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > /tmp/wildcard.crt
kubectl -n cert-manager get secret mc-lab-wildcard-tls \
  -o jsonpath='{.data.tls\.key}' | base64 -d > /tmp/wildcard.key
chmod 0600 /tmp/wildcard.key
for host in 10.20.20.11 10.20.30.11; do
  scp /tmp/wildcard.crt /tmp/wildcard.key "ubuntu@$host:/tmp/"
  ssh "ubuntu@$host" 'test -s /tmp/wildcard.crt && test -s /tmp/wildcard.key' || exit 1
done
shred -u /tmp/wildcard.key 2>/dev/null || rm -f /tmp/wildcard.key
rm -f /tmp/wildcard.crt
# PASS: CICD và App nhận đủ file; Admin đã xóa private key tạm.

# Trên mc-cicd1.
# BƯỚC 0 — StorageClass: RKE2 không có default StorageClass; không có nó, PVC của Gitaly sẽ
# Pending vĩnh viễn và GitLab không bao giờ lên. Homelab dùng local-path (dữ liệu gắn node,
# không HA — chấp nhận cho lab, ghi vào sổ SPOF của M2):
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.37/deploy/local-path-storage.yaml
kubectl patch storageclass local-path \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl get sc local-path -o jsonpath='{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}' | grep -qx true
# PASS: local-path là default; khác thì STOP.

kubectl create ns gitlab --dry-run=client -o yaml | kubectl apply -f -
kubectl -n gitlab create secret tls mc-lab-wildcard-tls \
  --cert=/tmp/wildcard.crt --key=/tmp/wildcard.key \
  --dry-run=client -o yaml | kubectl apply -f -

# Secrets cho ba dependency ngoài (giá trị khớp §6.2):
read -rsp 'PostgreSQL password: ' M1_DB_PASSWORD; echo
kubectl -n gitlab create secret generic gitlab-psql \
  --from-literal=password="$M1_DB_PASSWORD" --dry-run=client -o yaml | kubectl apply -f -
unset M1_DB_PASSWORD
read -rsp 'Redis password: ' M1_REDIS_PASSWORD; echo
kubectl -n gitlab create secret generic gitlab-redis \
  --from-literal=password="$M1_REDIS_PASSWORD" --dry-run=client -o yaml | kubectl apply -f -
unset M1_REDIS_PASSWORD
read -rp 'MinIO root user [minio-admin]: ' M1_MINIO_USER
M1_MINIO_USER=${M1_MINIO_USER:-minio-admin}
read -rsp 'MinIO root password: ' M1_MINIO_PASSWORD; echo
{
cat <<EOF
provider: AWS
region: us-east-1
aws_access_key_id: $M1_MINIO_USER
aws_secret_access_key: $M1_MINIO_PASSWORD
endpoint: "http://10.20.40.11:9000"
path_style: true
EOF
} | kubectl -n gitlab create secret generic object-storage \
  --from-file=connection=/dev/stdin --dry-run=client -o yaml | kubectl apply -f -
{
cat <<EOF
s3:
  bucket: gitlab-registry
  accesskey: $M1_MINIO_USER
  secretkey: $M1_MINIO_PASSWORD
  regionendpoint: http://10.20.40.11:9000
  region: us-east-1
  v4auth: true
  pathstyle: true
EOF
} | kubectl -n gitlab create secret generic registry-storage \
  --from-file=config=/dev/stdin --dry-run=client -o yaml | kubectl apply -f -
unset M1_MINIO_USER M1_MINIO_PASSWORD

helm repo add gitlab https://charts.gitlab.io && helm repo update
helm show chart gitlab/gitlab --version 10.2.2 | grep -E '^(version|appVersion):'
# PASS: version 10.2.2, appVersion v19.2.2 — nếu repo không còn bản này, tra
# https://docs.gitlab.com/charts/installation/version_mappings/ và cập nhật bảng §2.1
# trước khi đi tiếp, không cài bản trôi nổi.

helm upgrade --install gitlab gitlab/gitlab --namespace gitlab --version 10.2.2 \
  --wait --timeout 15m -f - <<'EOF'
global:
  edition: ce
  hosts: {domain: mc.lab, https: true}
  gatewayApi:
    enabled: false
  ingress:
    enabled: true
    class: traefik
    configureCertmanager: false
    tls: {secretName: mc-lab-wildcard-tls}
  psql:
    host: 10.20.40.11
    database: gitlabhq_production
    username: gitlab
    password: {secret: gitlab-psql, key: password}
  redis:
    host: 10.20.40.11
    auth: {enabled: true, secret: gitlab-redis, key: password}
  minio: {enabled: false}                # chart 10.x: object storage bắt buộc external
  appConfig:
    object_store:
      enabled: true
      connection: {secret: object-storage, key: connection}
    artifacts: {bucket: gitlab-artifacts}
    lfs: {bucket: gitlab-lfs}
    uploads: {bucket: gitlab-uploads}
    packages: {bucket: gitlab-packages}
    externalDiffs: {bucket: gitlab-mr-diffs}
    terraformState: {bucket: gitlab-terraform-state}
    dependencyProxy: {bucket: gitlab-dependency-proxy}
    ciSecureFiles: {bucket: gitlab-ci-secure-files}
    backups: {bucket: gitlab-backups, tmpBucket: gitlab-tmp}
  registry:
    bucket: gitlab-registry
  kas: {enabled: false}
postgresql: {install: false}
nginx-ingress: {enabled: false}          # thay ingress-nginx của bundle bằng Traefik của RKE2
gitlab-runner: {install: false}          # runner cài riêng ở §6.4 theo cấu hình bài viết
prometheus: {install: false}
gitlab:
  gitaly:
    persistence: {size: 20Gi, storageClass: local-path}
  webservice: {minReplicas: 1, maxReplicas: 1}
  sidekiq: {minReplicas: 1, maxReplicas: 1}
  gitlab-shell: {minReplicas: 1, maxReplicas: 1}
registry:
  storage: {secret: registry-storage, key: config}
  hpa: {minReplicas: 1, maxReplicas: 1}
EOF
```

```bash
kubectl -n gitlab get pvc
# PASS: PVC của Gitaly Bound trên StorageClass local-path
M1_MIGRATION_JOB=$(kubectl -n gitlab get jobs -o name | grep migrations | tail -1)
[ -n "$M1_MIGRATION_JOB" ] || { echo 'STOP: không thấy migration Job' >&2; exit 1; }
kubectl -n gitlab wait --for=condition=complete "$M1_MIGRATION_JOB" --timeout=900s
for r in $(kubectl -n gitlab get deployment,statefulset -o name); do
  kubectl -n gitlab rollout status "$r" --timeout=900s || exit 1
done
kubectl -n gitlab get pods
kubectl -n gitlab get pods --no-headers | \
  awk '$3 !~ /^(Running|Completed)$/ {print; bad=1} END {exit bad}'
# PASS: migration Completed và mọi workload rollout thành công; không pod lỗi.
curl -s -o /dev/null -w '%{http_code}\n' https://gitlab.mc.lab/users/sign_in
# PASS: 200 — không có chuỗi 5xx chập chờn vì max_connections đã nâng từ trước

# Gate memory — mc-cicd1 8 GB là mức "memory-constrained" theo requirements chính thức
# (baseline là 16 GB); hai gate này phát hiện sớm OOM thay vì để nó thành 5xx ngẫu nhiên:
test "$(kubectl -n gitlab get pods \
  -o jsonpath='{range .items[*]}{.status.containerStatuses[*].lastState.terminated.reason}{"\n"}{end}' \
  | grep -c OOMKilled)" -eq 0
# PASS: 0 — chưa container nào bị OOMKilled
M1_AVAILABLE_MB=$(free -m | awk '/Mem:/ {print $7}')
echo "available_MB=$M1_AVAILABLE_MB"
[ "$M1_AVAILABLE_MB" -ge 500 ] || {
  echo 'STOP: RAM available < 500 MB; tăng RAM mc-cicd1 rồi chạy lại gate' >&2
  exit 1
}
# PASS: available >= 500 MB sau khi GitLab lên đủ; thấp hơn thì tăng RAM VM (host cho phép)
# hoặc dừng ở đây chẩn đoán — đừng chạy tiếp pipeline trên node đang cạn RAM

kubectl -n gitlab get secret gitlab-gitlab-initial-root-password \
  -o jsonpath='{.data.password}' | base64 -d; echo
shred -u /tmp/wildcard.key 2>/dev/null || rm -f /tmp/wildcard.key
rm -f /tmp/wildcard.crt
# PASS cleanup trên mc-cicd1: test ! -e /tmp/wildcard.key -a ! -e /tmp/wildcard.crt
```

Trên `mc-app1`, lưu cặp cert vào Kubernetes ngay và xóa file tạm **trước** snapshot
`m1-cicd-ready`; §7.1 sẽ chỉ thêm annotation cho Secret này:

```bash
kubectl create ns certs --dry-run=client -o yaml | kubectl apply -f -
kubectl -n certs create secret tls mc-lab-wildcard-tls \
  --cert=/tmp/wildcard.crt --key=/tmp/wildcard.key \
  --dry-run=client -o yaml | kubectl apply -f -
shred -u /tmp/wildcard.key 2>/dev/null || rm -f /tmp/wildcard.key
rm -f /tmp/wildcard.crt
test ! -e /tmp/wildcard.key -a ! -e /tmp/wildcard.crt
# PASS: Secret tồn tại trong namespace certs; không còn private key dạng file.
```

Đăng nhập `root`, tạo group `platform`, project `platform/demo-app` và project
`platform/deploy-config` (chứa manifest cho ArgoCD). Tạo cả hai project ở trạng thái **blank**,
không chọn Initialize repository with a README; transcript §7.2 sẽ tạo branch `main`.

### 6.4. gitlab-runner với Kubernetes executor

GitLab UI → **Admin → CI/CD → Runners → New instance runner** → lấy token `glrt-…`. Cấu hình
dưới đây giữ tinh thần bài viết nhưng sửa các lỗi đã nhận xét: `concurrent` một chỗ duy nhất,
không bật `log_level = debug`, token khai qua `runnerToken` (field registration cũ đã
deprecated), và ghi rõ cái giá của `privileged`:

```bash
kubectl create ns gitlab-runner --dry-run=client -o yaml | kubectl apply -f -
kubectl -n gitlab-runner create secret generic mc-lab-ca-cert \
  --from-file=gitlab.mc.lab.crt=/usr/local/share/ca-certificates/mc-lab-ca.crt \
  --dry-run=client -o yaml | kubectl apply -f -
# runner trust GitLab qua CA lab — path chuẩn có sẵn trên mc-cicd1 từ §4.2

helm show chart gitlab/gitlab-runner --version 0.91.2 | grep -E '^(version|appVersion):'
read -rsp 'GitLab runner token glrt-...: ' M1_RUNNER_TOKEN; echo
helm upgrade --install gitlab-runner gitlab/gitlab-runner -n gitlab-runner \
  --version 0.91.2 --set-string runnerToken="$M1_RUNNER_TOKEN" \
  --wait --timeout 10m -f - <<'EOF'
gitlabUrl: https://gitlab.mc.lab
certsSecretName: mc-lab-ca-cert        # CA cho RUNNER MANAGER xác thực TLS của GitLab
rbac: {create: true}
concurrent: 4
runners:
  config: |
    [[runners]]
      executor = "kubernetes"
      [runners.kubernetes]
        namespace = "gitlab-runner"
        image = "quay.io/podman/stable:v5.8.2"
        privileged = true
        [runners.kubernetes.pod_security_context]
          run_as_user = 0
        # certsSecretName KHÔNG tự xuất hiện trong job pod — nó chỉ mount vào manager.
        # Job pod cần CA riêng để push vào registry, mount tường minh (đúng thủ thuật
        # /custom-certs của bài viết gốc):
        [[runners.kubernetes.volumes.secret]]
          name = "mc-lab-ca-cert"
          mount_path = "/custom-certs"
          read_only = true
EOF
unset M1_RUNNER_TOKEN
```

> ⚠️ `privileged = true` nghĩa là job CI chạy container đặc quyền trên node CICD — rộng quyền
> hơn cả mount docker socket. Chấp nhận được trong lab một-tenant; production phải cô lập
> runner ra node/cluster riêng hoặc chuyển buildah/kaniko không đặc quyền.

```bash
kubectl -n gitlab-runner rollout status deployment/gitlab-runner --timeout=300s
kubectl -n gitlab-runner get pods
# PASS: Deployment rolled out và pod gitlab-runner Running.
# GitLab UI → Admin → Runners: PASS: runner Online
```

Pipeline demo trong `platform/demo-app` — build image bằng podman, push vào registry của
GitLab (CA lab đã nằm trong `/etc/ssl/certs` của image? — chưa; nạp CA cho podman qua biến
mount của runner):

```yaml
# .gitlab-ci.yml
build-image:
  stage: build
  image: quay.io/podman/stable:v5.8.2
  script:
    - mkdir -p /etc/containers/certs.d/registry.mc.lab
    - cp /custom-certs/gitlab.mc.lab.crt /etc/containers/certs.d/registry.mc.lab/ca.crt
    - podman login -u gitlab-ci-token -p "$CI_JOB_TOKEN" registry.mc.lab
    - podman build -t registry.mc.lab/platform/demo-app:$CI_COMMIT_SHORT_SHA .
    - podman push registry.mc.lab/platform/demo-app:$CI_COMMIT_SHORT_SHA
```

Dockerfile demo được tạo chính xác ở transcript §7.2 bằng image pin `nginx:1.27.5-alpine`.

**PASS §6:** job `build-image` xanh; GitLab UI → Packages → Container Registry hiện tag mới.

Snapshot đủ sáu VM: `m1-cicd-ready`, theo protocol §2.4; sau khởi động lại, ba cụm Active,
GitLab health/pipeline và runner Online phải PASS lại.

## 7. Cert wildcard + reflector và ArgoCD mỗi cụm

### 7.1. Reflector nhân bản wildcard cert theo namespace

Đúng thủ thuật bài viết: đổi cert một lần, mọi namespace nhận theo. Trên `mc-app1`:

```bash
helm repo add emberstack https://emberstack.github.io/helm-charts --force-update
helm repo update
helm show chart emberstack/reflector --version 10.0.65 | grep -E '^(version|appVersion):'
helm upgrade --install reflector emberstack/reflector -n kube-system \
  --version 10.0.65 --wait --timeout 5m
kubectl -n kube-system rollout status deployment/reflector --timeout=300s

# Secret nguồn đã được tạo và file private key đã xóa ở cuối §6.3; chỉ gắn annotation:
kubectl -n certs get secret mc-lab-wildcard-tls >/dev/null || {
  echo 'STOP: thiếu Secret nguồn từ §6.3' >&2; exit 1;
}
kubectl -n certs annotate secret mc-lab-wildcard-tls \
  reflector.v1.k8s.emberstack.com/reflection-allowed="true" \
  reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces="demo-app,.*-prod" \
  reflector.v1.k8s.emberstack.com/reflection-auto-enabled="true" --overwrite

kubectl create ns demo-app --dry-run=client -o yaml | kubectl apply -f -   # idempotent
for i in $(seq 1 60); do
  kubectl -n demo-app get secret mc-lab-wildcard-tls >/dev/null 2>&1 && break
  sleep 2
done
kubectl -n demo-app get secret mc-lab-wildcard-tls >/dev/null || {
  echo 'STOP: Reflector chưa tạo secret sau 120 giây' >&2; exit 1;
}
# PASS: secret xuất hiện trong demo-app mà không phải copy tay
```

Giới hạn cần biết của mô hình cert này (ghi vào hồ sơ lab): reflector chỉ tự động **bên
trong một cụm**. Việc đưa wildcard từ Admin sang CICD/App ở trên là **bootstrap thủ công một
lần** — khi cert-manager trên Admin renew wildcard, secret ở CICD/App **không tự cập nhật**;
phải lặp lại bước xuất/scp/tạo secret (hoặc dựng cơ chế phân phối xuyên cụm — ngoài phạm vi
M1). Lời hứa "update một lần, đổi cert mọi nơi" của bài viết chỉ đúng trong phạm vi từng cụm.

### 7.2. ArgoCD instance riêng cho từng cụm

Cài trên **cả ba cụm** (lệnh giống nhau; chạy trên node server của từng cụm):

```bash
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.5.1/manifests/install.yaml
kubectl -n argocd get deploy argocd-server \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
# PASS: image tag v3.5.1 — đúng bản pin ở bảng §2.1
kubectl -n argocd wait --for=condition=Available deployment --all --timeout=600s
# PASS: mọi Argo CD Deployment Available.
```

Cho ArgoCD tin GitLab (CA tự ký) — trên từng cụm:

```bash
kubectl -n argocd create configmap argocd-tls-certs-cm \
  --from-file=gitlab.mc.lab=/usr/local/share/ca-certificates/mc-lab-ca.crt \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl -n argocd rollout restart deploy argocd-repo-server
kubectl -n argocd rollout status deploy/argocd-repo-server --timeout=300s
```

**Credential trước, Application sau.** Project GitLab mặc định là private: ArgoCD không
clone được repo và cụm App không pull được image nếu chỉ có CA. Tạo hai deploy token trong
GitLab (group `platform` → Settings → Repository → Deploy tokens):

- `argocd-read` với scope `read_repository`;
- `registry-pull` với scope `read_registry`.

Rồi khai báo trên cụm App:

```bash
# Repo credential cho ArgoCD (secret kiểu declarative của Argo):
read -rsp 'Deploy token argocd-read: ' M1_ARGO_TOKEN; echo
kubectl -n argocd create secret generic repo-deploy-config \
  --from-literal=type=git \
  --from-literal=url=https://gitlab.mc.lab/platform/deploy-config.git \
  --from-literal=username=argocd-read \
  --from-literal=password="$M1_ARGO_TOKEN" --dry-run=client -o yaml | \
  kubectl label --local -f - argocd.argoproj.io/secret-type=repository -o yaml | \
  kubectl apply -f -
unset M1_ARGO_TOKEN

# Pull secret cho workload (namespace đã tồn tại từ phép thử reflector §7.1 — dùng dạng
# idempotent để chạy lại không lỗi AlreadyExists):
kubectl create ns demo-app --dry-run=client -o yaml | kubectl apply -f -
read -rsp 'Deploy token registry-pull: ' M1_REGISTRY_TOKEN; echo
kubectl -n demo-app create secret docker-registry regcred \
  --docker-server=registry.mc.lab \
  --docker-username=registry-pull \
  --docker-password="$M1_REGISTRY_TOKEN" \
  --dry-run=client -o yaml | kubectl apply -f -
unset M1_REGISTRY_TOKEN
```

Application mẫu trên **cụm App** — trỏ về `platform/deploy-config`, **manual sync +
server-side apply** đúng như bài viết:

```bash
# Chạy trên mc-app1:
kubectl -n argocd apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: {name: demo-app, namespace: argocd}
spec:
  project: default
  source:
    repoURL: https://gitlab.mc.lab/platform/deploy-config.git
    targetRevision: main
    path: demo-app
  destination: {server: https://kubernetes.default.svc, namespace: demo-app}
  syncPolicy:
    syncOptions: [ServerSideApply=true]
    # KHÔNG có automated: sync thủ công qua UI/CLI, đổi tham số = commit lên GitLab
EOF
kubectl -n argocd get application demo-app
# PASS: Application tồn tại; SYNC STATUS có thể là Unknown/OutOfSync cho tới khi repo
# deploy-config có nội dung (transcript ở dưới) và bạn bấm sync
```

Nội dung đầy đủ của `platform/deploy-config/demo-app/`. Lưu ý mô hình storage: **web
stateless 2 replica** (trải 2 node) và **phần stateful tách riêng thành StatefulSet 1
replica** — vì PVC Longhorn mặc định là RWO, hai Pod trên hai node không thể cùng mount một
volume; đòi hỏi "2 replica + 1 PVC chung" là mâu thuẫn thiết kế, không phải bài để cố làm:

```yaml
# deployment.yaml — tầng web stateless
apiVersion: apps/v1
kind: Deployment
metadata: {name: demo-web}
spec:
  replicas: 2
  selector: {matchLabels: {app: demo-web}}
  template:
    metadata: {labels: {app: demo-web}}
    spec:
      imagePullSecrets: [{name: regcred}]
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector: {matchLabels: {app: demo-web}}
      containers:
      - name: web
        image: registry.mc.lab/platform/demo-app:<tag từ §6.4>
        ports: [{containerPort: 80}]
        resources:
          requests: {cpu: 50m, memory: 64Mi}
          limits: {memory: 128Mi}
        readinessProbe:
          httpGet: {path: /, port: 80}
        livenessProbe:
          httpGet: {path: /, port: 80}
          initialDelaySeconds: 10
---
# service.yaml
apiVersion: v1
kind: Service
metadata: {name: demo-web}
spec:
  selector: {app: demo-web}
  ports: [{port: 80, targetPort: 80}]
---
# statefulset.yaml — phần stateful minh họa Longhorn: mỗi replica một PVC riêng
apiVersion: apps/v1
kind: StatefulSet
metadata: {name: demo-data}
spec:
  serviceName: demo-data
  replicas: 1
  selector: {matchLabels: {app: demo-data}}
  template:
    metadata: {labels: {app: demo-data}}
    spec:
      containers:
      - name: recorder
        image: busybox:1.37
        command: [sh, -c, "while true; do date >> /data/heartbeat; sleep 30; done"]
        resources:
          requests: {cpu: 10m, memory: 16Mi}
          limits: {memory: 32Mi}
        volumeMounts: [{name: data, mountPath: /data}]
  volumeClaimTemplates:
  - metadata: {name: data}
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: longhorn
      resources: {requests: {storage: 1Gi}}
---
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-app
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  ingressClassName: traefik
  tls: [{hosts: [app.mc.lab], secretName: mc-lab-wildcard-tls}]
  rules:
  - host: app.mc.lab
    http:
      paths:
      - {path: /, pathType: Prefix, backend: {service: {name: demo-web, port: {number: 80}}}}
```

Registry cũng tự ký → cụm App cần trust khi pull image. CA đã nằm trong trust store OS của
node (§4.2), RKE2 dùng containerd riêng nên khai tường minh trên **cả 2 node App** rồi restart:

```bash
sudo mkdir -p /etc/rancher/rke2
sudo tee /etc/rancher/rke2/registries.yaml >/dev/null <<'EOF'
configs:
  registry.mc.lab:
    tls:
      ca_file: /usr/local/share/ca-certificates/mc-lab-ca.crt
EOF
sudo systemctl restart rke2-server   # mc-app1; mc-app2 restart rke2-agent
# Trên mc-app1 sau khi cả hai service restart:
kubectl wait --for=condition=Ready node/mc-app1 node/mc-app2 --timeout=600s
```

**Transcript đưa code và manifest lên GitLab** — chạy trên `mc-admin1` (hoặc bất kỳ VM nào
đã trust CA), dùng `<TOKEN-git-push>` từ bảng §2.3:

```bash
sudo apt-get install -y git
git config --global user.name pilot
git config --global user.email pilot@mc.lab
read -rsp 'GitLab PAT (api/write_repository): ' M1_GIT_PAT; echo
M1_ASKPASS=$(mktemp)
cat > "$M1_ASKPASS" <<'EOF'
#!/bin/sh
case "$1" in
  *Username*) printf '%s\n' root ;;
  *Password*) printf '%s\n' "$M1_GIT_PAT" ;;
esac
EOF
chmod 0700 "$M1_ASKPASS"
export M1_GIT_PAT M1_ASKPASS GIT_ASKPASS="$M1_ASKPASS" GIT_TERMINAL_PROMPT=0

# 1) Repo demo-app: source + Dockerfile + CI
git clone https://root@gitlab.mc.lab/platform/demo-app.git
cd demo-app
git symbolic-ref HEAD refs/heads/main
printf '%s\n' '<h1>m1 demo v1</h1>' > index.html
cat > Dockerfile <<'EOF'
FROM nginx:1.27.5-alpine
COPY index.html /usr/share/nginx/html/index.html
EOF
cat > .gitlab-ci.yml <<'EOF'
build-image:
  stage: build
  image: quay.io/podman/stable:v5.8.2
  script:
    - mkdir -p /etc/containers/certs.d/registry.mc.lab
    - cp /custom-certs/gitlab.mc.lab.crt /etc/containers/certs.d/registry.mc.lab/ca.crt
    - podman login -u gitlab-ci-token -p "$CI_JOB_TOKEN" registry.mc.lab
    - podman build -t registry.mc.lab/platform/demo-app:$CI_COMMIT_SHORT_SHA .
    - podman push registry.mc.lab/platform/demo-app:$CI_COMMIT_SHORT_SHA
EOF
git add -A
git commit -m 'demo-app v1'
git push -u origin main
git ls-remote --exit-code --heads origin refs/heads/main >/dev/null

# 2) Chờ pipeline xanh (GitLab UI → project → Pipelines), rồi chốt tag một cách xác định:
TAG=$(git rev-parse --short=8 HEAD)   # CI_COMMIT_SHORT_SHA của GitLab = 8 ký tự đầu SHA
echo "$TAG"
# PASS: GitLab UI → Packages → Container Registry có image tag đúng bằng $TAG

# 3) Repo deploy-config: tạo chính xác bốn file manifest, không copy/paste thủ công.
cd ..
git clone https://root@gitlab.mc.lab/platform/deploy-config.git
cd deploy-config
git symbolic-ref HEAD refs/heads/main
mkdir -p demo-app
cat > demo-app/deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata: {name: demo-web}
spec:
  replicas: 2
  selector: {matchLabels: {app: demo-web}}
  template:
    metadata: {labels: {app: demo-web}}
    spec:
      imagePullSecrets: [{name: regcred}]
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector: {matchLabels: {app: demo-web}}
      containers:
      - name: web
        image: registry.mc.lab/platform/demo-app:${TAG}
        ports: [{containerPort: 80}]
        resources:
          requests: {cpu: 50m, memory: 64Mi}
          limits: {memory: 128Mi}
        readinessProbe: {httpGet: {path: /, port: 80}}
        livenessProbe:
          httpGet: {path: /, port: 80}
          initialDelaySeconds: 10
EOF
cat > demo-app/service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata: {name: demo-web}
spec:
  selector: {app: demo-web}
  ports: [{port: 80, targetPort: 80}]
EOF
cat > demo-app/statefulset.yaml <<'EOF'
apiVersion: apps/v1
kind: StatefulSet
metadata: {name: demo-data}
spec:
  serviceName: demo-data
  replicas: 1
  selector: {matchLabels: {app: demo-data}}
  template:
    metadata: {labels: {app: demo-data}}
    spec:
      containers:
      - name: recorder
        image: busybox:1.37
        command: [sh, -c, "while true; do date >> /data/heartbeat; sleep 30; done"]
        resources:
          requests: {cpu: 10m, memory: 16Mi}
          limits: {memory: 32Mi}
        volumeMounts: [{name: data, mountPath: /data}]
  volumeClaimTemplates:
  - metadata: {name: data}
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: longhorn
      resources: {requests: {storage: 1Gi}}
EOF
cat > demo-app/ingress.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-app
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  ingressClassName: traefik
  tls: [{hosts: [app.mc.lab], secretName: mc-lab-wildcard-tls}]
  rules:
  - host: app.mc.lab
    http:
      paths:
      - {path: /, pathType: Prefix, backend: {service: {name: demo-web, port: {number: 80}}}}
EOF
grep -q "demo-app:${TAG}" demo-app/deployment.yaml
git add -A
git commit -m "demo-app ${TAG}"
git push -u origin main
git ls-remote --exit-code --heads origin refs/heads/main >/dev/null

# Không để PAT trong .git/config, command history hoặc snapshot.
for repo in ../demo-app .; do
  git -C "$repo" remote -v | grep -F "$M1_GIT_PAT" && {
    echo "STOP: PAT còn trong remote của $repo" >&2; exit 1;
  }
done
rm -f "$M1_ASKPASS"
unset M1_GIT_PAT M1_ASKPASS GIT_ASKPASS GIT_TERMINAL_PROMPT
echo 'PASS: hai repo có branch main; PAT không nằm trong remote'
```

**Sync thủ công** (đúng mô hình bài viết — không automated):

1. Trên `mc-app1`: `kubectl -n argocd port-forward svc/argocd-server 8443:443 --address 127.0.0.1`.
   Từ Windows mở SSH tunnel
   `ssh -L 8443:127.0.0.1:8443 ubuntu@10.20.30.11`, rồi mở `https://127.0.0.1:8443`.
2. Đăng nhập `admin`, mật khẩu lấy bằng:
   `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d`
3. Mở app `demo-app` → **SYNC** → Synchronize. (Tương đương CLI nếu đã cài `argocd`:
   `argocd app sync demo-app`.)
4. Gate xong phải nhấn `Ctrl-C` ở cả terminal port-forward và SSH tunnel; xác nhận không còn
   process: `pgrep -af 'kubectl.*port-forward'` phải không có output trước snapshot.

Gates:

```bash
kubectl -n demo-app get pods -o wide
# PASS: 2 pod demo-web Running trải trên mc-app1 và mc-app2 (topologySpreadConstraints ép),
# và pod demo-data-0 Running — không có ImagePullBackOff (regcred đúng)
kubectl -n demo-app get pvc
# PASS: data-demo-data-0 Bound, STORAGECLASS longhorn
kubectl -n demo-app get ingress demo-app -o jsonpath='{.status.loadBalancer}{"\n"}'
# PASS: {"ingress":[{"ip":"10.20.30.11"}]} — ingressendpoint (§5.1) đã điền status;
# ArgoCD báo Healthy, không cần cách "ép Healthy" bằng Lua như workaround trong bài viết
curl -s --cacert /usr/local/share/ca-certificates/mc-lab-ca.crt https://app.mc.lab \
  | grep -qi html && echo PASS
# PASS — luồng đầy đủ: GitLab CI build → registry → ArgoCD sync → Traefik → app
```

## 8. Monitoring tập trung qua Rancher (tùy chọn theo RAM)

Chỉ làm khi host còn ≥ 4 GB RAM trống thật sự (kiểm tra Task Manager trước).

1. Rancher UI → cụm `local` → **Apps → Charts → Monitoring** → Install (giữ mặc định).
2. Lặp lại trên cụm `mc-app`.
3. Gate: cụm → Monitoring → Grafana mở được qua proxy của Rancher; Prometheus targets Up.

**PASS:** xem được CPU/memory node của cả cụm Admin lẫn cụm App **từ một UI Rancher duy
nhất** — tính chất "quản lý tập trung" của bài viết, giờ có số liệu chứng minh.

Nói đúng phạm vi: đây là **xem tập trung** qua proxy của Rancher — Prometheus và dữ liệu
metric vẫn nằm trên từng cụm, chưa phải "monitoring/logging đặt trên cụm Admin" đúng nghĩa
lưu trữ tập trung như bài viết mô tả, và **logging tập trung không nằm trong M1**. Cả hai
phần đó thuộc [M2 §9](LAB-M2-CAPSTONE-PRODUCTION-HA.md). Không đủ RAM thì bỏ qua mục này và
ghi vào hồ sơ lab.

## 9. Gate cuối của Lab M1

Chạy tuần tự, tất cả phải PASS:

```bash
# 1) Ba cụm, một UI:      Rancher UI hiện local + mc-app + mc-cicd đều Active
# 2) Ingress đúng bài học: trên CẢ 3 cụm không tồn tại pod ingress-nginx nào
test "$(kubectl -n kube-system get pods --no-headers 2>/dev/null | grep -c ingress-nginx)" -eq 0 || exit 1
# Chạy trên server của từng cụm; cả ba lần đều phải exit 0.
# 3) Firewall dải DATA:    từ mc-app1
if nc -vzw3 10.20.40.11 5432; then
  echo 'STOP: APP vẫn vào được PostgreSQL' >&2; exit 1
fi
echo 'PASS-firewall: APP bị chặn khỏi DATA:5432'
# 4) DB ngoài cluster:     từ mc-db1
M1_PROJECTS=$(sudo -u postgres psql -At -d gitlabhq_production \
  -c 'SELECT count(*) FROM projects;')
[ "$M1_PROJECTS" -ge 2 ] || { echo "STOP: projects=$M1_PROJECTS, cần >=2" >&2; exit 1; }
# 5) Storage — PVC sống qua đời pod (chạy trên mc-app1):
BEFORE=$(kubectl -n demo-app exec demo-data-0 -- sh -c 'wc -l < /data/heartbeat')
kubectl -n demo-app delete pod demo-data-0 --wait
kubectl -n demo-app wait pod/demo-data-0 --for=condition=Ready --timeout=300s
AFTER=$(kubectl -n demo-app exec demo-data-0 -- sh -c 'wc -l < /data/heartbeat')
[ "$AFTER" -ge "$BEFORE" ] || { echo 'STOP: dữ liệu heartbeat bị mất' >&2; exit 1; }
echo PASS-storage
# PASS: in "PASS-storage" — pod thay thế (StatefulSet tạo lại cùng tên) đọc được dữ liệu cũ
```

Gate 6 kiểm tra end-to-end bằng **một thay đổi thật**, không dùng placeholder. Chạy trên
`mc-admin1`, từ thư mục home đã chứa hai repo ở §7.2:

```bash
read -rsp 'GitLab PAT (api/write_repository): ' M1_GIT_PAT; echo
M1_ASKPASS=$(mktemp)
cat > "$M1_ASKPASS" <<'EOF'
#!/bin/sh
case "$1" in
  *Username*) printf '%s\n' root ;;
  *Password*) printf '%s\n' "$M1_GIT_PAT" ;;
esac
EOF
chmod 0700 "$M1_ASKPASS"
export M1_GIT_PAT M1_ASKPASS GIT_ASKPASS="$M1_ASKPASS" GIT_TERMINAL_PROMPT=0

cd "$HOME/demo-app"
git pull --ff-only origin main
printf '%s\n' '<h1>m1 demo v2</h1>' > index.html
git add index.html
git commit -m 'demo-app v2 final gate'
git push origin main
M1_SOURCE_SHA=$(git rev-parse HEAD)
M1_NEW_TAG=$(git rev-parse --short=8 HEAD)

# Chờ pipeline của đúng SHA; timeout 15 phút. Python 3 có sẵn trong Ubuntu 24.04.
M1_PIPELINE_STATUS=missing
for i in $(seq 1 90); do
  M1_PIPELINE_STATUS=$(curl -fsS \
    --header "PRIVATE-TOKEN: $M1_GIT_PAT" \
    "https://gitlab.mc.lab/api/v4/projects/platform%2Fdemo-app/pipelines?sha=$M1_SOURCE_SHA" | \
    python3 -c 'import json,sys; x=json.load(sys.stdin); print(x[0]["status"] if x else "missing")')
  [ "$M1_PIPELINE_STATUS" = success ] && break
  case "$M1_PIPELINE_STATUS" in failed|canceled|skipped)
    echo "STOP: pipeline=$M1_PIPELINE_STATUS" >&2; exit 1 ;;
  esac
  sleep 10
done
[ "$M1_PIPELINE_STATUS" = success ] || {
  echo "STOP: pipeline chưa success sau 15 phút: $M1_PIPELINE_STATUS" >&2; exit 1;
}

cd "$HOME/deploy-config"
git pull --ff-only origin main
sed -i -E \
  "s|(image: registry\.mc\.lab/platform/demo-app:).*|\\1${M1_NEW_TAG}|" \
  demo-app/deployment.yaml
grep -qx "        image: registry.mc.lab/platform/demo-app:${M1_NEW_TAG}" \
  demo-app/deployment.yaml
git add demo-app/deployment.yaml
git commit -m "deploy demo-app ${M1_NEW_TAG}"
git push origin main
M1_DEPLOY_SHA=$(git rev-parse HEAD)

for repo in "$HOME/demo-app" "$HOME/deploy-config"; do
  git -C "$repo" remote -v | grep -F "$M1_GIT_PAT" && {
    echo "STOP: PAT còn trong remote của $repo" >&2; exit 1;
  }
done
rm -f "$M1_ASKPASS"
unset M1_GIT_PAT M1_ASKPASS GIT_ASKPASS GIT_TERMINAL_PROMPT M1_SOURCE_SHA
echo "PASS-pipeline: image tag $M1_NEW_TAG; deploy revision $M1_DEPLOY_SHA"
```

Trên `mc-app1`, yêu cầu Argo CD refresh và sync chính revision vừa push, rồi kiểm workload và
nội dung qua Traefik:

```bash
read -rp 'Nhập image tag từ dòng PASS-pipeline trên mc-admin1: ' M1_NEW_TAG
[ -n "$M1_NEW_TAG" ] || { echo 'STOP: thiếu image tag' >&2; exit 1; }
read -rp 'Nhập deploy revision từ cùng dòng PASS-pipeline: ' M1_DEPLOY_SHA
[ -n "$M1_DEPLOY_SHA" ] || { echo 'STOP: thiếu deploy revision' >&2; exit 1; }
kubectl -n argocd annotate application demo-app argocd.argoproj.io/refresh=hard --overwrite
kubectl -n argocd patch application demo-app --type merge \
  -p '{"operation":{"sync":{"revision":"main"}}}'
for i in $(seq 1 90); do
  M1_SYNC=$(kubectl -n argocd get application demo-app \
    -o jsonpath='{.status.sync.status}/{.status.health.status}/{.status.sync.revision}')
  [ "$M1_SYNC" = "Synced/Healthy/$M1_DEPLOY_SHA" ] && break
  sleep 10
done
[ "$M1_SYNC" = "Synced/Healthy/$M1_DEPLOY_SHA" ] || {
  echo "STOP: Argo CD=$M1_SYNC sau 15 phút" >&2; exit 1;
}
kubectl -n demo-app rollout status deployment/demo-web --timeout=300s
test "$(kubectl -n demo-app get deployment demo-web \
  -o jsonpath='{.spec.template.spec.containers[0].image}')" = \
  "registry.mc.lab/platform/demo-app:${M1_NEW_TAG}"
curl -fsS --cacert /usr/local/share/ca-certificates/mc-lab-ca.crt https://app.mc.lab | \
  grep -q 'm1 demo v2'
echo 'PASS-E2E: commit -> pipeline -> registry -> Argo CD -> Traefik -> m1 demo v2'
unset M1_NEW_TAG M1_DEPLOY_SHA M1_SYNC
```

Snapshot cuối `m1-complete` trên đủ sáu VM theo protocol §2.4. Sau khi khởi động lại, chạy lại
toàn bộ gate §9; chỉ khi gate PASS lần nữa mới đánh dấu lab hoàn tất. Hồ sơ lab phải có bảng
version thực cài (GitLab chart, runner chart, reflector, ArgoCD); token/mật khẩu lưu ngoài git
và không được chép vào transcript.

### 9.1. Pilot là chính lượt chạy end-to-end của Lab M1

Không có một "pilot setup" riêng trước §3. Pilot nghĩa là lấy host chưa có VM của track này và
chạy **đúng một lượt từ §2 đến §9**. Người chạy đầu tiên là maintainer/pilot operator; không
được dùng lệnh ngoài file để cứu một gate mà không ghi deviation. Một lượt chỉ được ghi
`PILOT PASS` khi đủ tất cả bằng chứng sau:

| Bằng chứng phải lưu (đã redact secret) | Gate PASS |
| --- | --- |
| Ngày chạy, CPU/RAM/disk host, VMware version | đủ tài nguyên theo §2.2 |
| Bảng version thực tế | khớp toàn bộ pin ở §2.1 |
| Output network/identity/router | toàn bộ gate §3 |
| Nodes, Traefik, cert-manager, Rancher | toàn bộ gate §4 |
| Longhorn detach/attach + cleanup | toàn bộ gate §5 |
| PostgreSQL/Redis/MinIO, migration, OOM/headroom, runner | toàn bộ gate §6 |
| Reflector, Argo revision, app/PVC/Ingress | toàn bộ gate §7 |
| `PASS-storage` và `PASS-E2E ... m1 demo v2` | gate §9 |
| Reboot từ snapshot `m1-complete` | chạy lại §9 vẫn PASS |

Sau `PILOT PASS`, bắt buộc thử rollback: power off và restore **đủ sáu VM** về
`m1-cicd-ready` theo §2.4, chạy lại §6 gates phù hợp với mốc đó, rồi làm lại §7–§9 tới
`m1-complete`. Nếu restore hoặc bất kỳ gate nào fail, trạng thái vẫn là `PILOT FAIL`; ghi
command, output, thời gian và mục bị lệch vào `review.md`, sửa file rồi chạy lại từ snapshot
đầu vào của mục — không xóa toàn bộ lab nếu snapshot còn hợp lệ.

Runbook chỉ đổi nhãn từ `READY FOR PILOT` thành runtime `READY` sau khi: (1) pilot trên PASS,
(2) mọi deviation đã được sửa trong file, và (3) một người học khác dựng lại từ đầu chỉ bằng
tài liệu này và cũng PASS. Không chép output giả định vào hồ sơ để thay cho hai lượt chạy.

Những gì lab này **cố ý chưa làm** (đúng danh sách nhận xét, trả ở
[Lab M2](LAB-M2-CAPSTONE-PRODUCTION-HA.md)): HA control plane và Rancher HA, NetworkPolicy
enforcement, backup/restore thật, nâng cấp + rollback, IaC hóa toàn bộ, alerting.

## 10. Troubleshooting của lab này

| Triệu chứng | Hướng xử lý |
| --- | --- |
| VM không có Internet | `ip route` trên VM phải default via `.1`; trên router kiểm `nft list ruleset` còn masquerade; NIC NAT của router có IP DHCP không |
| `rke2-server` không lên | `journalctl -u rke2-server -e`; sai YAML trong `config.yaml` là lỗi phổ biến nhất |
| Agent không join | token lệch, hoặc chưa mở được `https://<server>:9345` — `nc -vz <server> 9345` |
| Import Rancher treo Pending | trên cụm con: log `cattle-cluster-agent`; thường là DNS `rancher.mc.lab` hoặc CA chưa trust trên node |
| GitLab pod `Pending` | `kubectl -n gitlab describe pvc` — thường do thiếu StorageClass; xác nhận bước 0 của §6.3 (local-path default) đã chạy |
| GitLab 5xx | `kubectl -n gitlab logs deploy/gitlab-webservice-default -c webservice`; kiểm connection PostgreSQL từ §6.2; xem số connection: `SELECT count(*) FROM pg_stat_activity;` |
| Lỗi artifacts/LFS/registry push | log sidekiq/registry — thường sai secret `object-storage`/`registry-storage`, thiếu bucket, hoặc firewall CICD→DATA:9000; thử `mc ls lab` trên `mc-db1` |
| Pod không mount lại PVC Longhorn | đúng kịch bản KB multipath — xác nhận blacklist §5.2 đã áp trên **cả hai** node |
| ArgoCD app Progressing mãi | `kubectl get ingress -o yaml` xem `status.loadBalancer` — nếu rỗng, HelmChartConfig `ingressendpoint.ip` (§4.1/§5.1) chưa được apply trên cụm đó; kiểm job `helm-install-rke2-traefik` |

## 11. Nguồn chính thức

- RKE2 install & HA: <https://docs.rke2.io/install/quickstart>, <https://docs.rke2.io/install/ha>
- RKE2 ingress (chọn Traefik, HelmChartConfig): <https://docs.rke2.io/networking/networking_services>
- RKE2 v1.35 release notes: <https://docs.rke2.io/release-notes/v1.35.X>
- Migration ingress-nginx → Traefik: <https://docs.rke2.io/reference/ingress_migration>
- Rancher cài qua Helm + privateCA: <https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster>
- Rancher import cluster: <https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-clusters-in-rancher-setup/register-existing-clusters>
- cert-manager: <https://cert-manager.io/docs/installation/helm/>
- Longhorn install + requirements: <https://longhorn.io/docs/1.12.1/deploy/install/>
- Longhorn multipath KB: <https://longhorn.io/kb/troubleshooting-volume-with-multipath/>
- GitLab chart: <https://docs.gitlab.com/charts/>, external DB: <https://docs.gitlab.com/charts/advanced/external-db/>, external Redis: <https://docs.gitlab.com/charts/advanced/external-redis/>, external object storage: <https://docs.gitlab.com/charts/advanced/external-object-storage/>, storage: <https://docs.gitlab.com/charts/installation/storage/>, version mappings: <https://docs.gitlab.com/charts/installation/version_mappings/>
- PostgreSQL tuning cho GitLab: <https://docs.gitlab.com/administration/postgresql/tune/>
- GitLab requirements (PostgreSQL 17, memory): <https://docs.gitlab.com/install/requirements/>
- GitLab chart prerequisites (Helm 4): <https://docs.gitlab.com/charts/installation/tools/>
- PGDG apt repo: <https://www.postgresql.org/download/linux/ubuntu/>
- RKE2 server config reference (ServiceLB mặc định tắt): <https://docs.rke2.io/reference/server_config>
- Traefik kubernetesIngress provider (`ingressEndpoint`): <https://doc.traefik.io/traefik/reference/install-configuration/providers/kubernetes/kubernetes-ingress/>
- local-path-provisioner: <https://github.com/rancher/local-path-provisioner>
- MinIO server: <https://min.io/docs/minio/linux/index.html>
- gitlab-runner chart + Kubernetes executor: <https://docs.gitlab.com/runner/install/kubernetes/>, <https://docs.gitlab.com/runner/executors/kubernetes/>
- Argo CD: <https://argo-cd.readthedocs.io/en/stable/getting_started/>
- Reflector: <https://github.com/emberstack/kubernetes-reflector>
