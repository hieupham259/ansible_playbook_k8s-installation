# Runbook: Lab hệ thống log tập trung trên VMware — Filebeat → Kafka → Logstash → OpenSearch → Dashboards (+ snapshot MinIO)

> **Runbook này hiện thực hoá bản lab của kiến trúc trong [`kien-truc-he-thong-log-tap-trung-on-premise.md`](kien-truc-he-thong-log-tap-trung-on-premise.md)** trên VMware (máy host Windows), dùng Ubuntu Server 24.04, gồm **5 VM**:
>
> - 1 VM **nguồn log** chạy log generator + **Filebeat** (lớp agent, §2.1 kiến trúc);
> - 1 VM **pipeline** chạy **Kafka** (lớp đệm bền) + **Logstash** (lớp xử lý) + **MinIO** (đích snapshot);
> - 3 VM **OpenSearch** thành một cluster thật (quorum 3 node, mỗi shard 2 bản) + **Dashboards**.
>
> Lab giữ nguyên **ba nguyên tắc an toàn** của kiến trúc (§3): ba lớp đệm độc lập, mọi thành phần có trạng thái đều có bản sao, quorum lẻ. Kết thúc bằng **bảy bài phá hệ thống** (§9 kiến trúc) thu nhỏ cho lab — phần mà không khoá học nào dạy.
>
> ✅ Lab dùng dải IP riêng `.121–.125`, không đụng hai lab trong [`runbook-k8s-vmware.md`](runbook-k8s-vmware.md) (`.100/.101/.103` và `.111–.113`). Có thể chạy song song nếu đủ RAM host.
>
> **Đối tượng:** lab/homelab cá nhân, học kiến trúc log tập trung. Các bước có thể copy-paste.
>
> **Phạm vi cam kết:** đây là **lab để hiểu và phá thử**, không phải production. Bảng ánh xạ ở [§1.2](#12-lab-này-giữ-gì--cắt-gì-so-với-production) nói rõ từng khoảng cách so với topology production (§6.3 kiến trúc).

---

## Mục lục

1. [Tổng quan &amp; kiến trúc lab](#1-tổng-quan--kiến-trúc-lab)
2. [Quy hoạch (versions, IP, tài nguyên, mật khẩu)](#2-quy-hoạch)
3. [Tạo 5 VM Ubuntu 24.04 trên VMware](#3-tạo-5-vm-ubuntu-2404-trên-vmware)
4. [Cấu hình OS chung](#4-cấu-hình-os-chung)
5. [Cụm OpenSearch 3 node](#5-cụm-opensearch-3-node-log-os1os2os3)
6. [OpenSearch Dashboards](#6-opensearch-dashboards-log-os1)
7. [Kafka (KRaft, 1 broker)](#7-kafka-kraft-1-broker--log-pipeline)
8. [Logstash](#8-logstash-log-pipeline)
9. [Nguồn log + Filebeat](#9-nguồn-log--filebeat-log-agent)
10. [Verify end-to-end: đếm dòng ↔ đếm doc](#10-verify-end-to-end-đếm-dòng--đếm-doc)
11. [Snapshot ra MinIO](#11-snapshot-ra-minio)
12. [Bảy bài phá hệ thống (bản lab)](#12-bảy-bài-phá-hệ-thống-bản-lab)
13. [Vận hành &amp; troubleshooting](#13-vận-hành--troubleshooting)
14. [Nguồn official](#14-nguồn-official)
15. [Phụ lục](#15-phụ-lục)

---

## 1. Tổng quan & kiến trúc lab

### 1.1. Sơ đồ

```
      log-agent (.121)          log-pipeline (.122)                log-os1/2/3 (.123/.124/.125)
   ┌──────────────────┐      ┌───────────────────────┐          ┌──────────────────────────────┐
   │  loggen (systemd)│      │   Kafka 1 broker      │          │   OpenSearch cluster 3 node  │
   │   ↓ ghi file     │      │   (KRaft, topic       │          │   (master+data trên cả 3,    │
   │ /var/log/lab-app │─────▶│    logs-raw)          │─────────▶│    1 primary + 1 replica)    │
   │   ↑ đọc file     │      │         │             │          │            │                 │
   │  Filebeat        │      │   Logstash            │          │   Dashboards (trên os1)      │
   └──────────────────┘      │   (parse JSON,        │          └────────────┬─────────────────┘
    file + registry           │    kafka → opensearch)│                       │ snapshot (repository-s3)
    = lớp đệm 1               │   MinIO  ◀────────────┼───────────────────────┘
                              └───────────────────────┘
                               Kafka = lớp đệm 2            replica = lớp đệm 3
```

Đúng luồng 5 lớp của kiến trúc: **agent → đệm bền → xử lý → lưu + tìm kiếm → hiển thị**, kèm snapshot ra đích ngoài cluster.

### 1.2. Lab này giữ gì / cắt gì so với production

| Nguyên tắc / thành phần (kiến trúc) | Production (§6.3) | Lab này | Hệ quả cần nhớ |
|---|---|---|---|
| Ba lớp đệm độc lập (§3.1) | file → Kafka → replica | **Giữ nguyên** cả 3 lớp | Bài phá 1–4 ở [§12](#12-bảy-bài-phá-hệ-thống-bản-lab) kiểm chứng được thật |
| Quorum lẻ, tối thiểu 3 (§3.3) | 3 master riêng | **Giữ**: 3 node OpenSearch đều master-eligible | Chịu mất 1 node mà cluster vẫn bầu được master |
| 2 bản mỗi shard (§3.3) | 1 primary + 1 replica | **Giữ**: `number_of_replicas: 1` | Mất 1 node không mất dữ liệu |
| Kafka RF = 3 (§3.3) | 3 broker | **Cắt**: 1 broker, RF = 1 | Broker chết = lớp đệm 2 tạm mất; lớp 1 (file) gánh — đây là khoảng cách lớn nhất so với production |
| Coordinating node (§3.2) | 2 node riêng | **Cắt** | Query nặng đánh thẳng data node — bài phá 5 cho thấy hậu quả |
| Hot/warm + ILM (§6.4) | 2 tầng | **Cắt** tầng warm; giữ vòng đời bằng ISM (xoá sau 3 ngày) | Đủ để hiểu lifecycle, không đủ để đo chi phí tầng |
| Snapshot ngoài cluster (§2.1) | MinIO 4 node EC | **Giữ** nhưng MinIO 1 node | Backup thật, restore thử được (bài 7) |
| Bảo mật | TLS + phân quyền đầy đủ | Demo certs tự sinh của OpenSearch | Đủ để không tắt security như đa số repo mẫu; **không dùng cho production** |

### 1.3. Vì sao lab xếp như vậy

- **3 VM OpenSearch thật** thay vì 1 VM chạy 3 container: mất node = tắt nguồn được từng VM (bài "rút điện" §9 kiến trúc), thấy được cluster re-election và shard recovery thật.
- **Kafka + Logstash + MinIO chung 1 VM**: cả ba đều nhẹ ở tải lab; tách chúng ra không dạy thêm điều gì mà tốn RAM host.
- **Log generator có số thứ tự (`seq`)**: mọi bài kiểm tra quy về đúng một câu hỏi của kiến trúc §9 — *"đếm số dòng trước và sau, có khớp không?"* — trả lời được bằng số liệu, không phải cảm giác.

---

## 2. Quy hoạch

### 2.1. Phiên bản

| Thành phần | Bản dùng trong runbook | Ghi chú |
| --- | --- | --- |
| Ubuntu Server | **24.04.x LTS (Noble), amd64** | Như hai lab k8s |
| OpenSearch | **3.7.0** (APT repo `3.x`) | Gate bằng `apt-cache madison opensearch` trước khi cài — cài đúng bản repo đang có |
| OpenSearch Dashboards | **cùng bản với OpenSearch** | Hai package phải cùng minor |
| Kafka | **4.x (KRaft), build Scala 2.13** — ví dụ `kafka_2.13-4.1.0.tgz` | Lấy bản **stable mới nhất** ở trang downloads ([§14](#14-nguồn-official)); cần **Java 17** |
| Logstash | **logstash-oss-with-opensearch-output-plugin 8.9.0** | Bundle chính chủ do OpenSearch phân phối, đã kèm sẵn output plugin |
| Filebeat | **filebeat-oss 8.9.0** (deb) | Chỉ dùng output Kafka nên bản 8.x nào cũng được; ghim 8.9.0 cho khớp Logstash |
| MinIO | binary mới nhất từ `dl.min.io` | Single-node single-drive |

> ⚠️ **Về version:** OpenSearch/Kafka ra bản mới liên tục. Nguyên tắc của runbook: **luôn chạy lệnh gate xem repo/trang download đang có bản nào rồi mới cài**, và ghi lại bản đã cài vào bảng này. Đừng copy số version mù quáng.

### 2.2. IP, hostname, tài nguyên (ví dụ — đổi theo dải LAN của bạn)

Giả sử LAN `192.168.100.0/24`, gateway `192.168.100.1` — như hai lab k8s.

| Vai trò | Hostname | IP tĩnh | vCPU | RAM | Disk | Heap JVM |
| --- | --- | --- | --- | --- | --- | --- |
| Nguồn log + Filebeat | `log-agent` | `192.168.100.121` | 2 | **2 GB** | 20 GB | — |
| Kafka + Logstash + MinIO | `log-pipeline` | `192.168.100.122` | 4 | **6 GB** | 40 GB | Kafka 1 GB, Logstash 1 GB |
| OpenSearch + Dashboards | `log-os1` | `192.168.100.123` | 2 | **5 GB** | 40 GB | OpenSearch 2 GB |
| OpenSearch | `log-os2` | `192.168.100.124` | 2 | **4 GB** | 40 GB | OpenSearch 2 GB |
| OpenSearch | `log-os3` | `192.168.100.125` | 2 | **4 GB** | 40 GB | OpenSearch 2 GB |

Tổng: **12 vCPU / 21 GB RAM / 180 GB disk**.

> 💡 Heap OpenSearch = 2 GB trên VM 4 GB là **cố ý**: đúng quy tắc *heap ≤ 50% RAM* của kiến trúc (§5, §6.2). Ở production quy tắc còn lại là *heap ≤ 31 GB* — lab không chạm tới nhưng phải nhớ.
>
> ⚠️ **IP tĩnh phải nằm ngoài DHCP pool của router.** Cách kiểm tra và xử lý y hệt [runbook-k8s-vmware.md §2.2.1](runbook-k8s-vmware.md#221-kiểm-tra-ip-tĩnh-không-trùng-dải-dhcp-của-router-làm-1-lần-trước-khi-cài) — làm một lần cho cả dải `.121–.125`, rồi ghi đè IP đã chốt vào bảng trên và dùng xuyên suốt.

### 2.3. Yêu cầu máy host

- RAM host nên **≥ 32 GB** (5 VM dùng 21 GB; còn lại cho Windows/VMware).
- VMware Workstation Pro/Player, ISO Ubuntu Server 24.04 (như lab k8s).
- Nếu chạy đồng thời lab k8s: cộng dồn RAM của cả hai lab trước khi bật.

### 2.4. Mật khẩu dùng xuyên suốt (đổi nếu muốn, nhưng phải nhất quán)

| Biến | Giá trị ví dụ | Dùng ở |
| --- | --- | --- |
| `OS_PASS` — admin OpenSearch | `Lab@Search2026!` | Cài OpenSearch ([§5.1](#51-cài-opensearch-trên-cả-3-node)), Logstash ([§8.2](#82-pipeline-config)), mọi lệnh `curl` |
| `MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD` | `labadmin` / `Lab@Minio2026!` | MinIO ([§11.1](#111-cài-minio-log-pipeline)), keystore OpenSearch ([§11.2](#112-cài-plugin-repository-s3--keystore-cả-3-node)) |

> OpenSearch từ 2.12 **bắt buộc** mật khẩu admin mạnh (≥ 8 ký tự, đủ hoa/thường/số/ký tự đặc biệt) ngay lúc cài — mật khẩu yếu là cài fail.

---

## 3. Tạo 5 VM Ubuntu 24.04 trên VMware

Quy trình **golden VM → snapshot → full clone** giống hệt [runbook-k8s-vmware.md §4](runbook-k8s-vmware.md#4-tạo-và-nhân-bản-3-server-theo-serversmd), chỉ khác bảng đích. Tóm tắt các bước bắt buộc:

1. **Dựng 1 VM golden** Ubuntu 24.04 (2 vCPU / 2 GB / 20 GB — cấu hình nhỏ nhất, clone xong mới nâng từng máy): cài Ubuntu Server, tick **Install OpenSSH server**, network **Bridged** (chỉnh Virtual Network Editor đúng card vật lý như [runbook k8s §3](runbook-k8s-vmware.md#3-tạo-3-vm-ubuntu-2404-trên-vmware)).
2. Trên golden, **chặn cloud-init quản mạng** trước khi snapshot:

```bash
echo 'network: {config: disabled}' | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

3. **Snapshot** `golden-base` → **full clone ×5** theo tên VM ở bảng [§2.2](#22-ip-hostname-tài-nguyên-ví-dụ--đổi-theo-dải-lan-của-bạn).
4. Trên **từng clone** (qua console VMware, đừng SSH khi còn trùng IP) — gỡ trùng identity:

```bash
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo systemd-machine-id-setup
sudo rm -f /etc/ssh/ssh_host_*
sudo dpkg-reconfigure openssh-server
```

5. Trên **từng clone** — hostname + IP tĩnh (đổi 2 chỗ đánh dấu theo bảng [§2.2](#22-ip-hostname-tài-nguyên-ví-dụ--đổi-theo-dải-lan-của-bạn)):

```bash
sudo hostnamectl set-hostname log-agent        # ⚠️ đổi theo từng máy
ip -br a                                        # lấy tên card (ens33/ens160)
sudo mv /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.bak 2>/dev/null || true
sudo tee /etc/netplan/01-static.yaml >/dev/null <<'EOF'
network:
  version: 2
  ethernets:
    ens33:                               # ⚠️ đổi theo tên card thật
      dhcp4: no
      addresses: [192.168.100.121/24]    # ⚠️ .121–.125 theo bảng §2.2
      routes:
        - to: default
          via: 192.168.100.1
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
EOF
sudo chmod 600 /etc/netplan/01-static.yaml
sudo netplan apply
sudo reboot
```

6. **Chỉnh tài nguyên từng VM** (VM Settings, khi VM đang tắt) theo đúng cột vCPU/RAM của bảng [§2.2](#22-ip-hostname-tài-nguyên-ví-dụ--đổi-theo-dải-lan-của-bạn).

**Verify trước khi sang [§4](#4-cấu-hình-os-chung)** — cả 5 VM: `hostnamectl` đúng tên, `ip -br a` đúng IP, `machine-id` và SSH host key **khác nhau từng máy**, ping thông cả 5 IP, và IP **giữ nguyên sau reboot**. Nên snapshot từng VM (`ip-ready`).

---

## 4. Cấu hình OS chung

### 4.1. `/etc/hosts` — giống nhau trên CẢ 5 VM

```bash
sudo tee -a /etc/hosts >/dev/null <<'EOF'
192.168.100.121 log-agent
192.168.100.122 log-pipeline
192.168.100.123 log-os1
192.168.100.124 log-os2
192.168.100.125 log-os3
EOF
```

### 4.2. Update + đồng bộ thời gian — CẢ 5 VM

```bash
sudo apt-get update && sudo apt-get upgrade -y
timedatectl                       # System clock synchronized: yes + NTP service: active
sudo timedatectl set-ntp true     # bật nếu chưa
```

> NTP không phải tuỳ chọn: log là dữ liệu **theo thời gian**. Máy lệch giờ thì log "biến mất" khỏi timeline dù vẫn nằm trong index — bài phá 6 ([§12](#12-bảy-bài-phá-hệ-thống-bản-lab)) diễn lại đúng sự cố này.

### 4.3. Tắt swap + `vm.max_map_count` — CHỈ 3 node OpenSearch (`log-os1/2/3`)

```bash
# swap: OpenSearch/JVM + swap = GC pause không đoán được
sudo swapoff -a
sudo sed -ri '/^[^#].*[[:space:]]swap[[:space:]]/s/^/#/' /etc/fstab
swapon --show                     # không trả dòng nào = đạt

# mmap cho Lucene:
echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-opensearch.conf
sudo sysctl --system
sysctl vm.max_map_count           # PASS: 262144
```

### 4.4. Firewall

Lab trong LAN tin cậy → tắt cho đỡ vướng:

```bash
sudo ufw disable
```

Nếu muốn giữ UFW, bảng cổng của lab:

| VM | Cổng | Ai gọi vào |
| --- | --- | --- |
| `log-os1/2/3` | TCP `9200` (REST), `9300` (transport) | Logstash, Dashboards, curl từ host; `9300` giữa 3 node OS |
| `log-os1` | TCP `5601` (Dashboards) | Trình duyệt trên host |
| `log-pipeline` | TCP `9092` (Kafka) | Filebeat từ `log-agent`, Logstash nội bộ |
| `log-pipeline` | TCP `9000`/`9001` (MinIO API/console) | 3 node OS (`9000`), trình duyệt host (`9001`) |

---

## 5. Cụm OpenSearch 3 node (`log-os1/2/3`)

### 5.1. Cài OpenSearch — trên CẢ 3 node

```bash
sudo apt-get install -y curl gnupg2 ca-certificates lsb-release

# key + repo 3.x (đối chiếu đúng lệnh trên trang docs nếu OpenSearch đổi URL key — link ở §14):
curl -o- https://artifacts.opensearch.org/publickeys/opensearch-release.pgp \
  | sudo gpg --dearmor --batch --yes -o /usr/share/keyrings/opensearch-release-keyring
echo "deb [signed-by=/usr/share/keyrings/opensearch-release-keyring] https://artifacts.opensearch.org/releases/bundle/opensearch/3.x/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/opensearch-3.x.list
sudo apt-get update

# GATE: xem repo đang có bản nào — cài đúng bản nhìn thấy, ghi lại vào bảng §2.1
apt-cache madison opensearch

# cài (mật khẩu admin bắt buộc từ 2.12; demo security config được áp tự động):
sudo env OPENSEARCH_INITIAL_ADMIN_PASSWORD='Lab@Search2026!' \
  apt-get install -y opensearch=3.7.0          # ⚠️ đổi theo bản thấy ở lệnh madison
sudo apt-mark hold opensearch                   # ghim, tránh apt upgrade tự nhảy bản
```

> Cài đặt này tự sinh **demo TLS certs** + user `admin`. Đủ để lab không phải tắt security (điều mà đa số repo docker-compose mẫu làm — xem cảnh báo §11.3 của tài liệu kiến trúc), nhưng certs demo **không dùng cho production**.

### 5.2. Cấu hình từng node

Trên **mỗi** node, thêm vào **đầu** file `/etc/opensearch/opensearch.yml` (giữ nguyên block security demo ở cuối file):

```yaml
cluster.name: lab-logs
node.name: log-os1                 # ⚠️ đổi: log-os1 / log-os2 / log-os3
network.host: 0.0.0.0
discovery.seed_hosts: ["192.168.100.123", "192.168.100.124", "192.168.100.125"]
cluster.initial_cluster_manager_nodes: ["log-os1", "log-os2", "log-os3"]
```

Heap 2 GB (= 50% RAM VM 4 GB) trên **cả 3 node** — sửa `/etc/opensearch/jvm.options`:

```bash
sudo sed -i 's/^-Xms[0-9]\+[gm]/-Xms2g/; s/^-Xmx[0-9]\+[gm]/-Xmx2g/' /etc/opensearch/jvm.options
grep -E '^-Xm[sx]' /etc/opensearch/jvm.options   # PASS: -Xms2g + -Xmx2g
```

### 5.3. Khởi động & verify cluster

Trên cả 3 node:

```bash
sudo systemctl enable --now opensearch
systemctl is-active opensearch          # active
```

Từ `log-os1` (hoặc bất kỳ node nào):

```bash
curl -sk -u admin:'Lab@Search2026!' https://localhost:9200/_cluster/health?pretty
curl -sk -u admin:'Lab@Search2026!' https://localhost:9200/_cat/nodes?v
```

**Quality gate trước khi tiếp:**

- `status` là `green`, `number_of_nodes` là `3`;
- `_cat/nodes` liệt kê đủ `log-os1/2/3`, một node có dấu `*` (cluster manager được bầu);
- node nào không join: xem `sudo journalctl -u opensearch -e` — thường sai `discovery.seed_hosts`, trùng `node.name`, hoặc quên [§4.3](#43-tắt-swap--vmmax_map_count--chỉ-3-node-opensearch-log-os123).

> 💡 Sau khi cluster đã hình thành lần đầu, dòng `cluster.initial_cluster_manager_nodes` hết vai trò (chỉ dùng cho lần bootstrap đầu tiên). Best practice là xoá nó khỏi cả 3 file yml để tránh hiểu nhầm về sau.

### 5.4. Index template + vòng đời dữ liệu (ISM)

Chạy từ `log-os1`. Template cho index log — **1 primary + 1 replica** (2 bản mỗi shard, §3.3 kiến trúc):

```bash
curl -sk -u admin:'Lab@Search2026!' -X PUT https://localhost:9200/_index_template/logs-lab \
  -H 'Content-Type: application/json' -d '{
  "index_patterns": ["logs-*"],
  "template": { "settings": { "number_of_shards": 1, "number_of_replicas": 1 } }
}'
```

Chính sách ISM: xoá index sau **3 ngày** (bản thu nhỏ của vòng đời §6.4 kiến trúc):

```bash
curl -sk -u admin:'Lab@Search2026!' -X PUT https://localhost:9200/_plugins/_ism/policies/logs-retention \
  -H 'Content-Type: application/json' -d '{
  "policy": {
    "description": "lab: xoa log sau 3 ngay",
    "default_state": "hot",
    "states": [
      { "name": "hot", "actions": [],
        "transitions": [{ "state_name": "delete", "conditions": { "min_index_age": "3d" } }] },
      { "name": "delete", "actions": [{ "delete": {} }], "transitions": [] }
    ],
    "ism_template": [{ "index_patterns": ["logs-*"], "priority": 100 }]
  }
}'
```

> ⚠️ Ở production, snapshot phải chạy **trước** mốc xoá (§6.4 kiến trúc). Trong lab, [§11](#11-snapshot-ra-minio) dựng snapshot trước khi index đầu tiên kịp 3 ngày tuổi — đúng thứ tự đó.

---

## 6. OpenSearch Dashboards (`log-os1`)

```bash
# repo dashboards 3.x (cùng key với §5.1):
echo "deb [signed-by=/usr/share/keyrings/opensearch-release-keyring] https://artifacts.opensearch.org/releases/bundle/opensearch-dashboards/3.x/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/opensearch-dashboards-3.x.list
sudo apt-get update
apt-cache madison opensearch-dashboards            # GATE: cài đúng bản = bản OpenSearch
sudo apt-get install -y opensearch-dashboards=3.7.0   # ⚠️ đổi theo bản madison
sudo apt-mark hold opensearch-dashboards
```

Sửa `/etc/opensearch-dashboards/opensearch_dashboards.yml` — hai dòng:

```yaml
server.host: "0.0.0.0"
opensearch.hosts: ["https://192.168.100.123:9200"]
```

(giữ nguyên các dòng `opensearch.username/password: kibanaserver` và `opensearch.ssl.verificationMode: none` mặc định của gói — chúng thuộc demo security config.)

```bash
sudo systemctl enable --now opensearch-dashboards
systemctl is-active opensearch-dashboards
```

**Verify:** từ trình duyệt máy host mở `http://192.168.100.123:5601` → đăng nhập `admin` / `Lab@Search2026!` → vào được UI. (Chưa có index pattern — tạo sau khi có dữ liệu ở [§10](#10-verify-end-to-end-đếm-dòng--đếm-doc).)

---

## 7. Kafka (KRaft, 1 broker) — `log-pipeline`

### 7.1. Java + tải Kafka

```bash
sudo apt-get install -y openjdk-17-jre-headless
java -version                                   # PASS: openjdk 17.x

# GATE: mở https://kafka.apache.org/downloads → lấy bản stable mới nhất (build Scala 2.13),
# thay số version vào biến dưới:
KAFKA_VERSION=4.1.0
curl -fLO "https://downloads.apache.org/kafka/${KAFKA_VERSION}/kafka_2.13-${KAFKA_VERSION}.tgz"
sudo tar -xzf "kafka_2.13-${KAFKA_VERSION}.tgz" -C /opt
sudo mv "/opt/kafka_2.13-${KAFKA_VERSION}" /opt/kafka
sudo useradd -r -s /usr/sbin/nologin kafka
sudo mkdir -p /var/lib/kafka
sudo chown -R kafka:kafka /opt/kafka /var/lib/kafka
```

### 7.2. Cấu hình + format storage (KRaft combined mode)

Mở `/opt/kafka/config/server.properties`, đảm bảo các dòng sau (Kafka 4.x đã có sẵn `process.roles=broker,controller` và `node.id=1` trong file mặc định — chỉ sửa những dòng khác đi):

```properties
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
advertised.listeners=PLAINTEXT://192.168.100.122:9092
log.dirs=/var/lib/kafka
log.retention.hours=72
```

> `advertised.listeners` phải là **IP thật của VM** — để `localhost` là lỗi kinh điển khiến Filebeat từ `log-agent` bắt tay được rồi bị trả về địa chỉ không kết nối nổi. `log.retention.hours=72` = "Kafka giữ 3 ngày, replay được" (§2 kiến trúc).

Format metadata (chạy một lần duy nhất):

```bash
cd /opt/kafka
sudo -u kafka bin/kafka-storage.sh format --standalone \
  -t "$(bin/kafka-storage.sh random-uuid)" -c config/server.properties
```

> Nếu dùng Kafka 3.9.x thay vì 4.x: file cấu hình KRaft nằm ở `config/kraft/server.properties` và lệnh format không có `--standalone` — theo đúng quickstart của bản đó.

### 7.3. systemd unit + khởi động

```bash
sudo tee /etc/systemd/system/kafka.service >/dev/null <<'EOF'
[Unit]
Description=Apache Kafka (KRaft, single broker - lab)
After=network-online.target
Wants=network-online.target

[Service]
User=kafka
Environment="KAFKA_HEAP_OPTS=-Xms1g -Xmx1g"
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now kafka
systemctl is-active kafka            # active
```

### 7.4. Tạo topic + smoke test

```bash
cd /opt/kafka
bin/kafka-topics.sh --create --topic logs-raw --partitions 3 --replication-factor 1 \
  --bootstrap-server localhost:9092
bin/kafka-topics.sh --describe --topic logs-raw --bootstrap-server localhost:9092

# produce 1 message thử rồi đọc lại:
echo '{"ts":"2026-08-04T00:00:00+07:00","seq":0,"level":"info","service":"smoke-test","msg":"hello kafka"}' \
  | bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic logs-raw
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic logs-raw \
  --from-beginning --max-messages 1
```

**PASS:** consumer in ra đúng message vừa produce. `replication-factor 1` là khoảng cách chấp nhận của lab ([§1.2](#12-lab-này-giữ-gì--cắt-gì-so-với-production)); ở production là RF 3 + `min.insync.replicas=2` + producer `acks=all` (§4 kiến trúc).

---

## 8. Logstash (`log-pipeline`)

### 8.1. Tải bundle OSS kèm sẵn opensearch output plugin

```bash
curl -fLO https://artifacts.opensearch.org/logstash/logstash-oss-with-opensearch-output-plugin-8.9.0-linux-x64.tar.gz
sudo tar -xzf logstash-oss-with-opensearch-output-plugin-8.9.0-linux-x64.tar.gz -C /opt
sudo mv /opt/logstash-8.9.0 /opt/logstash
sudo useradd -r -s /usr/sbin/nologin logstash
sudo chown -R logstash:logstash /opt/logstash
```

### 8.2. Pipeline config

```bash
sudo mkdir -p /opt/logstash/config/pipeline
sudo tee /opt/logstash/config/pipeline/logs.conf >/dev/null <<'EOF'
input {
  kafka {
    bootstrap_servers => "localhost:9092"
    topics            => ["logs-raw"]
    group_id          => "logstash-lab"
    auto_offset_reset => "earliest"
    codec             => "json"          # bóc envelope JSON của Filebeat
  }
}

filter {
  # dòng log gốc nằm trong field "message" của envelope Filebeat → parse vào "app.*"
  json {
    source => "message"
    target => "app"
    skip_on_invalid_json => true
  }
  date {
    match  => ["[app][ts]", "ISO8601"]
    target => "@timestamp"
  }
}

output {
  opensearch {
    hosts    => ["https://192.168.100.123:9200", "https://192.168.100.124:9200", "https://192.168.100.125:9200"]
    user     => "admin"
    password => "Lab@Search2026!"
    index    => "logs-%{+YYYY.MM.dd}"
    ssl      => true
    ssl_certificate_verification => false    # demo certs tự ký — chỉ chấp nhận trong lab
  }
}
EOF
sudo chown -R logstash:logstash /opt/logstash/config
```

> Parse nằm ở **Logstash chứ không phải agent** — đúng vai trò lớp xử lý của kiến trúc (§2.1): parse lỗi thì sửa config một chỗ rồi **replay lại từ Kafka**, không phải đụng vào N máy nguồn.

### 8.3. systemd + verify

```bash
sudo tee /etc/systemd/system/logstash.service >/dev/null <<'EOF'
[Unit]
Description=Logstash OSS (kafka -> opensearch, lab)
After=network-online.target kafka.service

[Service]
User=logstash
Environment="LS_JAVA_OPTS=-Xms1g -Xmx1g"
ExecStart=/opt/logstash/bin/logstash -f /opt/logstash/config/pipeline/logs.conf
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now logstash
```

**Verify** (message smoke-test ở [§7.4](#74-tạo-topic--smoke-test) phải chảy vào OpenSearch — Logstash khởi động mất ~30–60 giây):

```bash
curl -sk -u admin:'Lab@Search2026!' 'https://192.168.100.123:9200/_cat/indices/logs-*?v'
curl -sk -u admin:'Lab@Search2026!' 'https://192.168.100.123:9200/logs-*/_count?pretty'
```

**PASS:** có index `logs-YYYY.MM.dd` với `docs.count ≥ 1`, cột `rep` = 1, health `green`. Lỗi thường gặp: sai `OS_PASS` hoặc quên `ssl_certificate_verification => false` → xem `/opt/logstash/logs/logstash-plain.log`.

---

## 9. Nguồn log + Filebeat (`log-agent`)

### 9.1. Log generator có số thứ tự + logrotate

```bash
sudo mkdir -p /var/log/lab-app /var/lib/lab-app
sudo tee /usr/local/bin/loggen.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
# Sinh log JSON có seq tăng dần — seq là "sổ cái" để đối chiếu số dòng ở mọi bài kiểm tra.
LOG=/var/log/lab-app/app.log
SEQF=/var/lib/lab-app/seq
seq_no=$(cat "$SEQF" 2>/dev/null || echo 0)
while true; do
  seq_no=$((seq_no + 1))
  level=info; [ $((seq_no % 50)) -eq 0 ] && level=error
  printf '{"ts":"%s","seq":%d,"level":"%s","service":"lab-app","msg":"sample log line"}\n' \
    "$(date -Is)" "$seq_no" "$level" >> "$LOG"
  echo "$seq_no" > "$SEQF"
  sleep 0.1
done
EOF
sudo chmod +x /usr/local/bin/loggen.sh

sudo tee /etc/systemd/system/loggen.service >/dev/null <<'EOF'
[Unit]
Description=Lab log generator (JSON + seq)

[Service]
ExecStart=/usr/local/bin/loggen.sh
Restart=always

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now loggen

# logrotate: giữ đủ bản rotate để agent tắc vài giờ vẫn không mất file (điểm 2, §4 kiến trúc)
sudo tee /etc/logrotate.d/lab-app >/dev/null <<'EOF'
/var/log/lab-app/app.log {
    size 50M
    rotate 5
    missingok
    notifempty
    create 0644 root root
}
EOF
```

**Verify:** `tail -f /var/log/lab-app/app.log` thấy ~10 dòng JSON/giây, `cat /var/lib/lab-app/seq` tăng dần.

> App ghi **ra file** chứ không bắn UDP syslog — file chính là lớp đệm bền thứ nhất (§3.1 kiến trúc). Script mở-ghi-đóng theo từng dòng nên logrotate xoay file an toàn, không cần `copytruncate` (vốn có thể nuốt dòng).

### 9.2. Filebeat OSS → Kafka

```bash
curl -fLO https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-oss-8.9.0-amd64.deb
sudo dpkg -i filebeat-oss-8.9.0-amd64.deb

sudo tee /etc/filebeat/filebeat.yml >/dev/null <<'EOF'
filebeat.inputs:
  - type: filestream
    id: lab-app
    paths:
      - /var/log/lab-app/app.log*        # cả file đã rotate

output.kafka:
  hosts: ["192.168.100.122:9092"]
  topic: "logs-raw"
  required_acks: -1        # acks=all — giữ thói quen production dù lab RF=1 (§4 kiến trúc)
  compression: gzip
  max_message_bytes: 1000000
EOF

sudo filebeat test config                        # PASS: Config OK
sudo filebeat test output                        # PASS: kết nối Kafka OK
sudo systemctl enable --now filebeat
```

### 9.3. Verify từng chặng

```bash
# chặng agent → Kafka (chạy trên log-pipeline):
/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic logs-raw --max-messages 3
# PASS: thấy JSON envelope của Filebeat, trong field "message" là dòng log gốc có "seq"

# chặng Kafka → Logstash → OpenSearch (chạy trên log-os1) — count phải TĂNG giữa 2 lần gọi:
curl -sk -u admin:'Lab@Search2026!' 'https://localhost:9200/logs-*/_count?pretty'; sleep 10
curl -sk -u admin:'Lab@Search2026!' 'https://localhost:9200/logs-*/_count?pretty'
```

---

## 10. Verify end-to-end: đếm dòng ↔ đếm doc

Đây là **phép đo chuẩn của cả lab** — mọi bài phá ở [§12](#12-bảy-bài-phá-hệ-thống-bản-lab) đều kết thúc bằng nó:

```bash
# (1) trên log-agent — tổng số dòng đã sinh:
cat /var/lib/lab-app/seq

# (2) trên log-os1 — seq lớn nhất đã vào cluster:
curl -sk -u admin:'Lab@Search2026!' -H 'Content-Type: application/json' \
  'https://localhost:9200/logs-*/_search?size=0' -d '{
  "aggs": { "max_seq": { "max": { "field": "app.seq" } } }
}'

# (3) trên log-os1 — tổng số doc:
curl -sk -u admin:'Lab@Search2026!' 'https://localhost:9200/logs-*/_count?pretty'
```

**Đọc kết quả — thuộc lòng ba mệnh đề này:**

- `max_seq` xấp xỉ giá trị ở (1), chỉ chênh vài giây độ trễ pipeline → **không mất log**.
- `count` **≥** `max_seq` là bình thường: pipeline này là *at-least-once*, sự cố có thể tạo **trùng lặp** (Filebeat gửi lại batch chưa được ack) — trùng thì lọc được, **mất thì không cứu được**.
- `max_seq` dừng lại trong khi (1) vẫn tăng → pipeline đứt ở đâu đó: lần theo [§9.3](#93-verify-từng-chặng) từng chặng.

**Dashboards:** `http://192.168.100.123:5601` → *Dashboards Management → Index patterns* → tạo pattern `logs-*` với time field `@timestamp` → *Discover* thấy log chảy realtime; filter thử `app.level: error` (1/50 số dòng).

> ✅ Đến đây toàn bộ đường ống 5 lớp đã chạy. Snapshot ([§11](#11-snapshot-ra-minio)) rồi mới phá ([§12](#12-bảy-bài-phá-hệ-thống-bản-lab)).

---

## 11. Snapshot ra MinIO

### 11.1. Cài MinIO (`log-pipeline`)

```bash
curl -fLO https://dl.min.io/server/minio/release/linux-amd64/minio
sudo install -m 755 minio /usr/local/bin/minio
sudo useradd -r -s /usr/sbin/nologin minio-user
sudo mkdir -p /var/lib/minio
sudo chown minio-user:minio-user /var/lib/minio

sudo tee /etc/default/minio >/dev/null <<'EOF'
MINIO_ROOT_USER=labadmin
MINIO_ROOT_PASSWORD=Lab@Minio2026!
MINIO_VOLUMES=/var/lib/minio
MINIO_OPTS="--console-address :9001"
EOF

sudo tee /etc/systemd/system/minio.service >/dev/null <<'EOF'
[Unit]
Description=MinIO (single node - lab snapshot target)
After=network-online.target

[Service]
User=minio-user
EnvironmentFile=/etc/default/minio
ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now minio
```

**Tạo bucket:** trình duyệt host → `http://192.168.100.122:9001` → đăng nhập `labadmin` / `Lab@Minio2026!` → **Create Bucket** tên `os-snapshots`.

> MinIO 1 node không có erasure coding chịu lỗi — đủ cho lab. Production: 4 node, và nhớ bài học sizing đã sửa trong tài liệu kiến trúc §6.3: set 4 ổ chỉ cho ~50% dung lượng khả dụng.

### 11.2. Cài plugin `repository-s3` + keystore — CẢ 3 node OpenSearch

Trên **từng** node `log-os1/2/3`:

```bash
sudo /usr/share/opensearch/bin/opensearch-plugin install --batch repository-s3

# credentials MinIO vào keystore (nhập giá trị khi được hỏi: labadmin / Lab@Minio2026!):
sudo OPENSEARCH_PATH_CONF=/etc/opensearch \
  /usr/share/opensearch/bin/opensearch-keystore add s3.client.default.access_key
sudo OPENSEARCH_PATH_CONF=/etc/opensearch \
  /usr/share/opensearch/bin/opensearch-keystore add s3.client.default.secret_key
sudo chown opensearch:opensearch /etc/opensearch/opensearch.keystore

# trỏ client s3 vào MinIO — thêm vào /etc/opensearch/opensearch.yml:
sudo tee -a /etc/opensearch/opensearch.yml >/dev/null <<'EOF'
s3.client.default.endpoint: 192.168.100.122:9000
s3.client.default.protocol: http
s3.client.default.path_style_access: true
EOF
```

**Restart CUỐN CHIẾU** — một node một lần, đợi `green` rồi mới sang node kế (đây chính là thao tác "bảo trì không dừng ingest" mà kiến trúc nói tới: trong lúc node restart, Kafka giữ log, replica phục vụ đọc):

```bash
sudo systemctl restart opensearch
# đợi:
curl -sk -u admin:'Lab@Search2026!' https://localhost:9200/_cluster/health?pretty | grep status
# "green" rồi mới sang node tiếp theo
```

### 11.3. Tạo repo + snapshot + restore thử

Từ `log-os1`:

```bash
# repo:
curl -sk -u admin:'Lab@Search2026!' -X PUT https://localhost:9200/_snapshot/minio-repo \
  -H 'Content-Type: application/json' -d '{
  "type": "s3",
  "settings": { "bucket": "os-snapshots", "base_path": "lab" }
}'

# snapshot toàn bộ logs-*:
curl -sk -u admin:'Lab@Search2026!' -X PUT \
  'https://localhost:9200/_snapshot/minio-repo/snap-1?wait_for_completion=true' \
  -H 'Content-Type: application/json' -d '{ "indices": "logs-*" }'
# PASS: "state" : "SUCCESS"
```

**Restore thử ngay** (bài 7 của §9 kiến trúc — *"backup chưa từng restore thử thì chưa phải backup"*): restore về index tên khác để không đụng dữ liệu đang chạy:

```bash
curl -sk -u admin:'Lab@Search2026!' -X POST \
  'https://localhost:9200/_snapshot/minio-repo/snap-1/_restore?wait_for_completion=true' \
  -H 'Content-Type: application/json' -d '{
  "indices": "logs-*",
  "rename_pattern": "logs-(.+)",
  "rename_replacement": "restored-logs-$1"
}'

# đối chiếu count gốc ↔ count restore tại thời điểm snapshot:
curl -sk -u admin:'Lab@Search2026!' 'https://localhost:9200/restored-logs-*/_count?pretty'

# dọn:
curl -sk -u admin:'Lab@Search2026!' -X DELETE 'https://localhost:9200/restored-logs-*'
```

---

## 12. Bảy bài phá hệ thống (bản lab)

> Bản thu nhỏ của §9 tài liệu kiến trúc. **Trước mỗi bài:** ghi lại `seq` hiện tại ([§10](#10-verify-end-to-end-đếm-dòng--đếm-doc), bước 1). **Sau mỗi bài:** chạy lại đủ 3 phép đo §10 — `max_seq` phải đuổi kịp `seq`, không thủng dải nào. Làm tuần tự, xong bài này khôi phục hẳn rồi mới sang bài sau.

| # | Bài (bản lab) | Lệnh phá | Kỳ vọng |
|---|---|---|---|
| 1 | Kill OpenSearch giữa lúc ingest | Trên `log-os2`: `sudo systemctl kill -s KILL opensearch` | Cluster sang `yellow`, ingest **không dừng** (replica + Kafka gánh). Start lại → `green`, count khớp |
| 2 | "Rút điện" một node | VMware → `log-os3` → **Power Off** (không phải Shut Down Guest) | Như bài 1 nhưng kiểm tra độ bền qua mất điện thật của VM. Bật lại → node tự join, shard recovery |
| 3 | Đứt mạng agent ↔ trung tâm | Trên `log-agent`: `sudo iptables -A OUTPUT -d 192.168.100.122 -p tcp --dport 9092 -j DROP` → đợi 10 phút → gỡ bằng `-D` cùng tham số | Filebeat backoff rồi gửi lại từ vị trí registry. `max_seq` đuổi kịp, **không thủng seq** |
| 4 | Trung tâm đệm chết | Trên `log-pipeline`: `sudo systemctl stop kafka` → đợi 10 phút → `start` | Lớp đệm 1 (file + registry) gánh: loggen vẫn ghi, Filebeat giữ vị trí đọc, Kafka sống lại là đuổi kịp |
| 5 | Query nặng khi đang ingest | Dashboards → Discover, khoảng thời gian lớn nhất + query `app.msg: *log*`, refresh liên tục; song song theo dõi `_count` | Ingest chậm lại nhưng không dừng. **Ghi chú:** lab không có coordinating node riêng — cảm nhận rõ vì sao production cần (§3.2 kiến trúc) |
| 6 | Máy nguồn lệch giờ | Trên `log-agent`: `sudo timedatectl set-ntp false && sudo date -s '+2 hours'` → quan sát Discover → khôi phục: `sudo timedatectl set-ntp true` | Log "biến mất" khỏi cửa sổ thời gian hiện tại (nằm ở tương lai). Bài học: **NTP là bắt buộc** |
| 7 | Restore vào chỗ trống | Snapshot mới (`snap-2`) → xoá hẳn một index ngày cũ → restore đúng index đó từ `snap-2` (bỏ `rename_*` trong lệnh restore [§11.3](#113-tạo-repo--snapshot--restore-thử)) | `_count` index sau restore = trước khi xoá. Backup **được chứng minh** là dùng được |

> Bài 2, 4 và 7 là ba bài hay bị bỏ qua nhất — và cũng là ba bài phân biệt hệ thống *chạy được* với hệ thống *chịu được sự cố*. Nếu một bài FAIL: đừng sửa số liệu, sửa kiến trúc/cấu hình rồi chạy lại bài đó.

---

## 13. Vận hành & troubleshooting

### Bảng lỗi thường gặp

| Triệu chứng | Nguyên nhân hay gặp | Xử lý |
|---|---|---|
| `opensearch` không start | `vm.max_map_count` chưa set; heap > RAM; mật khẩu admin yếu lúc cài | `sudo journalctl -u opensearch -e`; kiểm [§4.3](#43-tắt-swap--vmmax_map_count--chỉ-3-node-opensearch-log-os123), [§5.2](#52-cấu-hình-từng-node) |
| Cluster `yellow` kéo dài | Thiếu node cho replica; node chưa join | `curl -sk -u admin:… https://localhost:9200/_cluster/allocation/explain?pretty` |
| Cluster `red` | Mất cả primary lẫn replica của một shard (trong lab: 2 node cùng chết) | Bật đủ node; còn mất thật thì restore từ snapshot ([§11.3](#113-tạo-repo--snapshot--restore-thử)) |
| Filebeat không gửi | `advertised.listeners` của Kafka sai IP; firewall | `sudo filebeat test output`; kiểm [§7.2](#72-cấu-hình--format-storage-kraft-combined-mode) |
| Logstash không đẩy vào OS | Sai mật khẩu; thiếu `ssl_certificate_verification => false`; OS chưa `green` | `/opt/logstash/logs/logstash-plain.log` |
| Dashboards không đăng nhập được | Dùng nhầm user `kibanaserver` để login UI | Login UI bằng `admin` / `OS_PASS`; `kibanaserver` chỉ là user nội bộ trong yml |
| Host Windows cạn RAM | 5 VM + lab k8s chạy song song | Tắt bớt lab kia, hoặc giảm heap OS xuống 1 GB (chấp nhận chậm) |
| Disk VM OpenSearch đầy | Quên ISM, hoặc loggen chạy quá lâu | Kiểm ISM [§5.4](#54-index-template--vòng-đời-dữ-liệu-ism); xoá index cũ bằng tay: `DELETE logs-YYYY.MM.dd` |

### Nhịp vận hành lab

- Mỗi lần bật lab: bật 3 VM OpenSearch trước → `log-pipeline` → `log-agent`; đợi `green` rồi mới phá tiếp.
- Snapshot VMware từng mốc lớn (`os-cluster-ready`, `pipeline-ready`, `e2e-ok`) để làm lại nhanh.
- Muốn tăng tải: giảm `sleep 0.1` trong `loggen.sh` (0.01 = ~100 dòng/giây) — quan sát lại toàn bộ [§10](#10-verify-end-to-end-đếm-dòng--đếm-doc).

---

## 14. Nguồn official

| Nguồn | Dùng cho |
|---|---|
| [OpenSearch — Install on Debian/Ubuntu](https://docs.opensearch.org/latest/install-and-configure/install-opensearch/debian/) | §5.1 — đối chiếu URL key/repo nếu lệnh trong runbook lệch |
| [OpenSearch — Install Dashboards (Debian)](https://docs.opensearch.org/latest/install-and-configure/install-dashboards/debian/) | §6 |
| [OpenSearch — Important system settings](https://docs.opensearch.org/latest/install-and-configure/install-opensearch/index/) | §4.3 (`vm.max_map_count`, swap) |
| [OpenSearch — ISM policies](https://docs.opensearch.org/latest/im-plugin/ism/index/) | §5.4 |
| [OpenSearch — Snapshot & repository-s3](https://docs.opensearch.org/latest/tuning-your-cluster/availability-and-recovery/snapshots/snapshot-restore/) | §11 |
| [OpenSearch — Logstash](https://docs.opensearch.org/latest/tools/logstash/index/) | §8 — bundle `logstash-oss-with-opensearch-output-plugin` |
| [Apache Kafka — Downloads](https://kafka.apache.org/downloads) / [Quickstart (KRaft)](https://kafka.apache.org/quickstart) | §7 |
| [Filebeat — Kafka output](https://www.elastic.co/guide/en/beats/filebeat/current/kafka-output.html) / [filestream input](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-input-filestream.html) | §9.2 |
| [MinIO — Deploy single-node](https://min.io/docs/minio/linux/operations/install-deploy-manage/deploy-minio-single-node-single-drive.html) | §11.1 |
| [`kien-truc-he-thong-log-tap-trung-on-premise.md`](kien-truc-he-thong-log-tap-trung-on-premise.md) | Toàn bộ "vì sao" của runbook này |

---

## 15. Phụ lục

### Phụ lục A — Biến thể bỏ Kafka (ánh xạ §8.2 kiến trúc: dưới ~50 GB/ngày)

Kiến trúc cho phép hoãn Kafka ở quy mô nhỏ **nhưng phải chèn lại được mà không sửa agent hàng loạt**. Lab thử biến thể này bằng cách đổi đúng 2 chỗ:

**Filebeat (`log-agent`)** — thay `output.kafka` bằng:

```yaml
output.logstash:
  hosts: ["192.168.100.122:5044"]
```

**Logstash (`log-pipeline`)** — thay `input.kafka` bằng:

```conf
input {
  beats { port => 5044 }
}
```

Restart hai service. Sau đó chạy lại **bài phá 4** ([§12](#12-bảy-bài-phá-hệ-thống-bản-lab)) — dừng Logstash 10 phút: chỉ còn disk buffer của agent gánh, và thấy rõ thứ đã mất khi bỏ lớp đệm 2 (không còn replay, không còn giữ-3-ngày). Đó chính là lý do kiến trúc yêu cầu *"chèn Kafka vào sau được mà không phải sửa agent"* — ở đây là đổi một block output.

### Phụ lục B — Reset để chạy lại từ đầu

```bash
# xoá toàn bộ dữ liệu log trong OpenSearch (trên log-os1):
curl -sk -u admin:'Lab@Search2026!' -X DELETE 'https://localhost:9200/logs-*,restored-logs-*'

# reset vị trí đọc của Logstash trong Kafka (trên log-pipeline, dừng logstash trước):
sudo systemctl stop logstash
/opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group logstash-lab --reset-offsets --to-earliest --topic logs-raw --execute
sudo systemctl start logstash

# reset Filebeat + bộ đếm seq (trên log-agent):
sudo systemctl stop filebeat loggen
sudo rm -rf /var/lib/filebeat/registry /var/log/lab-app/* /var/lib/lab-app/seq
sudo systemctl start loggen filebeat
```

### Phụ lục C — Lab này còn thiếu gì so với một hệ thật (để học tiếp)

1. **Kafka 3 broker, RF=3, `min.insync.replicas=2`** — nâng cấp tự nhiên đầu tiên nếu host đủ RAM (mỗi broker thêm ~1,5 GB).
2. **Coordinating node riêng** — thêm 1 VM OpenSearch với `node.roles: []`, trỏ Dashboards + mọi query vào đó, chạy lại bài phá 5 để so sánh.
3. **Hot/warm** — gắn thêm disk cho 2 node, gán `node.attr.temp: hot|warm`, mở rộng ISM policy thêm state `warm`.
4. **TLS thật + phân quyền** — thay demo certs, tạo user chỉ-đọc cho Dashboards, user chỉ-ghi cho Logstash.
5. **Alerting** — plugin Alerting của OpenSearch: cảnh báo khi `app.level: error` vượt ngưỡng, khi disk qua 70% (§4 kiến trúc).
