# Ansible Zero → Hero — học qua chính codebase này

> Mục tiêu: cung cấp **toàn bộ kiến thức nền tảng về Ansible**, và quan trọng hơn — giải thích **mọi khái niệm bằng chính code có trong dự án này** (thư mục [ansible/](ansible/)). Đọc xong bạn vừa nắm Ansible từ đầu, vừa hiểu được từng dòng playbook/role trong repo.
>
> Dự án này là một bộ Ansible dùng để **dựng cluster Kubernetes** (containerd + kubeadm). Ta sẽ mượn chính nó làm "sách bài tập".

---

## Mục lục

1. [Ansible là gì & vì sao dùng](#1-ansible-là-gì--vì-sao-dùng)
2. [Kiến trúc & cách hoạt động](#2-kiến-trúc--cách-hoạt-động)
3. [Cài đặt & lệnh ad-hoc đầu tiên](#3-cài-đặt--lệnh-ad-hoc-đầu-tiên)
4. [Inventory — danh sách máy](#4-inventory--danh-sách-máy)
5. [`ansible.cfg` — file cấu hình](#5-ansiblecfg--file-cấu-hình)
6. [Playbook, Play, Task — đơn vị thực thi](#6-playbook-play-task--đơn-vị-thực-thi)
7. [Module & tính idempotent](#7-module--tính-idempotent)
8. [Biến (Variables), Facts, magic vars](#8-biến-variables-facts-magic-vars)
9. [Điều kiện & vòng lặp (`when`, `loop`)](#9-điều-kiện--vòng-lặp-when-loop)
10. [Handlers & `notify`](#10-handlers--notify)
11. [Templates & Jinja2 (`.j2`)](#11-templates--jinja2-j2)
12. [Roles — đóng gói & tái sử dụng](#12-roles--đóng-gói--tái-sử-dụng)
13. [Privilege escalation (`become`/sudo)](#13-privilege-escalation-becomesudo)
14. [Delegation & chạy cục bộ (`delegate_to`, `local`)](#14-delegation--chạy-cục-bộ-delegate_to-local)
15. [Điều khiển task (`register`, `changed_when`, `creates`…)](#15-điều-khiển-task-register-changed_when-creates)
16. [`import` vs `include` — ghép playbook/role/task](#16-import-vs-include--ghép-playbookroletask)
17. [`fetch` / `copy` — chuyển file qua lại](#17-fetch--copy--chuyển-file-qua-lại)
18. [Bản đồ tổng thể dự án (ai làm gì)](#18-bản-đồ-tổng-thể-dự-án-ai-làm-gì)
19. [Các anti-pattern trong code dự án (học từ lỗi)](#19-các-anti-pattern-trong-code-dự-án-học-từ-lỗi)
20. [Cheat sheet & lệnh hay dùng](#20-cheat-sheet--lệnh-hay-dùng)
21. [Tài liệu tham khảo chính thức](#21-tài-liệu-tham-khảo-chính-thức)

---

## 1. Ansible là gì & vì sao dùng

**Ansible** là công cụ **tự động hóa cấu hình & triển khai** (configuration management + orchestration). Bạn mô tả **trạng thái mong muốn** của máy chủ bằng file YAML, Ansible đăng nhập SSH vào các máy đó và làm cho chúng đạt trạng thái đó.

4 đặc tính cốt lõi:

| Đặc tính | Nghĩa | Trong dự án này |
|---|---|---|
| **Agentless** | Không cần cài agent trên máy đích; chỉ cần **SSH + Python**. | Bạn chỉ cần SSH tới các node K8s là chạy được |
| **Push-based** | Máy điều khiển "đẩy" cấu hình xuống. | Bạn chạy `ansible-playbook` từ bastion/WSL |
| **Idempotent** | Chạy lại nhiều lần cho cùng kết quả, không phá. | Vd dùng `creates:` để `kubeadm init` không chạy 2 lần |
| **Khai báo (declarative)** | Mô tả "muốn gì", không phải "làm từng bước shell". | `apt: name=containerd state=present` thay vì gõ apt thủ công |

> Vì sao hợp với việc dựng K8s? Một cụm có nhiều node giống nhau (cài containerd, kubelet, kubeadm…). Ansible giúp làm **đồng loạt, lặp lại được, có version control** thay vì SSH gõ tay từng máy.

---

## 2. Kiến trúc & cách hoạt động

```
   ┌──────────────────────────┐         SSH (cổng 22)        ┌────────────┐
   │   CONTROL NODE           │ ───────────────────────────▶ │  master1   │  (managed node)
   │   (bastion / WSL)        │ ───────────────────────────▶ │  worker1   │
   │   - ansible, ansible-     │ ───────────────────────────▶ │  worker2   │
   │     playbook              │                              └────────────┘
   │   - inventory + playbooks │   Ansible copy "module" (Python) lên máy đích,
   └──────────────────────────┘   chạy, lấy kết quả JSON về, rồi xóa. Không để lại agent.
```

- **Control node:** máy chạy `ansible` (phải là Linux/WSL). Trong dự án này là nhóm `[local]`.
- **Managed nodes:** các máy đích (master/worker). Chỉ cần Python + SSH.
- **Cách chạy 1 task:** Ansible sinh một module (đoạn code Python), copy lên máy đích, thực thi, nhận lại JSON (`changed`/`ok`/`failed`), rồi dọn. → đó là lý do "agentless".

---

## 3. Cài đặt & lệnh ad-hoc đầu tiên

```bash
# Trên control node (Linux/WSL):
sudo apt update && sudo apt install -y ansible
ansible --version
```

**Lệnh ad-hoc** = chạy 1 module nhanh, không cần viết playbook. Cú pháp: `ansible <pattern> -m <module> -a "<args>"`.

```bash
# "ping" mọi host trong inventory (kiểm tra SSH + Python sẵn sàng)
ansible all -i ansible/inventories/inventory.ini -m ping

# Chạy 1 lệnh shell trên tất cả control-plane
ansible controlplane -m command -a "uptime"
```

> Trong repo có file [ansible/inventory](ansible/inventory) và [ansible/inventories/inventory.ini](ansible/inventories/inventory.ini) — đó là **inventory**, ta tìm hiểu ngay dưới đây.

---

## 4. Inventory — danh sách máy

**Inventory** liệt kê máy đích và **gom nhóm** chúng. Dự án dùng định dạng INI tại [ansible/inventories/inventory.ini](ansible/inventories/inventory.ini):

```ini
[k8s_node:children]      # nhóm-của-nhóm (group of groups)
controlplane
node
local

[k8s_worker_node:children]
node

[controlplane]           # nhóm rỗng — bạn điền host vào đây
[local]
[node]
[nfs-server]
```

Khái niệm cần nắm:

| Khái niệm | Giải thích | Ví dụ trong repo |
|---|---|---|
| **Host** | một máy đích (có tên + `ansible_host=IP`) | `master1 ansible_host=10.0.0.11` |
| **Group** | nhóm host, đặt trong `[tên_nhóm]` | `[controlplane]`, `[node]` |
| **`:children`** | nhóm chứa các nhóm con | `[k8s_node:children]` gồm controlplane+node+local |
| **Host vars** | biến gắn theo host | `ansible_user=ansible-user` |
| **Group vars** | biến gắn theo nhóm (`[grp:vars]` hoặc `group_vars/`) | `[all:vars]` |
| **Pattern** | cách chọn host khi chạy | `all`, `controlplane`, `master2:master3` |

Một host muốn dùng được cần các **biến kết nối**:
```ini
[all:vars]
ansible_user=ansible-user                       # user SSH
ansible_ssh_private_key_file=~/.ssh/id_rsa       # khóa SSH
ansible_python_interpreter=/usr/bin/python3      # python trên máy đích
```

> ⚠️ **Quan sát thực tế:** inventory trong repo đang **để trống host** (chỉ có tên nhóm). Đây là "template" — bạn phải tự điền IP. Ngoài ra repo tham chiếu các host tên `master1/master2/master3` (trong [first-master.yaml](ansible/first-master.yaml), [join_masters.yaml](ansible/join_masters.yaml)) nhưng inventory chưa định nghĩa chúng → là một thiếu sót cần bổ sung.

---

## 5. `ansible.cfg` — file cấu hình

`ansible.cfg` đặt mặc định cho dự án (đường dẫn inventory, user, tắt host-key checking…). Ansible tìm nó theo thứ tự: biến môi trường `ANSIBLE_CONFIG` → `./ansible.cfg` → `~/.ansible.cfg` → `/etc/ansible/ansible.cfg`.

Ví dụ tối thiểu (nên thêm vào dự án):
```ini
[defaults]
inventory = inventory.ini
roles_path = roles
remote_user = ansible-user
host_key_checking = False

[privilege_escalation]
become = True
become_method = sudo
```

> ⚠️ **Quan sát:** repo **chưa có `ansible.cfg`** → mỗi lần chạy phải truyền `-i inventories/inventory.ini`. Thêm `ansible.cfg` sẽ gọn hơn (chỉ cần `ansible-playbook cluster.yaml`).

---

## 6. Playbook, Play, Task — đơn vị thực thi

- **Playbook** = 1 file YAML chứa nhiều **play**.
- **Play** = ánh xạ "**một nhóm host** ↔ **một danh sách task/role**".
- **Task** = một hành động gọi **một module**.

Ví dụ play đơn giản nhất trong repo — [ansible/k8s-setup.yaml](ansible/k8s-setup.yaml):

```yaml
---
- name: Prepare Nodes for Running Kubernetes   # tên play
  hosts: k8s_node                              # chạy trên nhóm nào
  become: yes                                  # sudo
  tasks:
    - name: Import K8s-setup role              # task
      import_role:
        name: k8s-setup
```

Một play đầy đủ task — trích [ansible/time-sync.yaml](ansible/time-sync.yaml) gọi role, còn task "trần" thì xem [ansible/roles/time-sync/tasks/main.yaml](ansible/roles/time-sync/tasks/main.yaml):

```yaml
- name: Install chrony package      # mỗi gạch đầu dòng = 1 task
  package:
    name: "{{ chrony_package }}"
    state: present

- name: Start and enable chrony service
  service:
    name: "{{ chrony_service }}"
    state: started
    enabled: yes
```

**Cú pháp YAML cần nhớ:** 2 dấu cách thụt lề (không tab), `-` cho phần tử list, `key: value` cho map, `"{{ biến }}"` để chèn biến.

### Orchestrator: playbook gọi playbook

File "nhạc trưởng" [ansible/k8s_installer.yaml](ansible/k8s_installer.yaml) gọi lần lượt nhiều playbook bằng `import_playbook`:

```yaml
---
- import_playbook: k8s-setup.yaml
- import_playbook: time-sync.yaml
- import_playbook: kubernetes_role.yaml
- import_playbook: first-master.yaml
- import_playbook: join_workers.yaml
- import_playbook: install_cni.yaml
...
```

> ⚠️ Trong repo các dòng này còn có đuôi `hostlist=k8s_node`. **`import_playbook` KHÔNG nhận tham số đó** → nó vô tác dụng; phạm vi host thật nằm ở `hosts:` bên trong mỗi playbook con. Đây là điểm dễ gây hiểu nhầm.

---

## 7. Module & tính idempotent

**Module** là "đơn vị công việc" Ansible biết làm (cài gói, copy file, khởi động service…). Phần lớn module **idempotent**: tự kiểm tra trạng thái, chỉ thay đổi khi cần (báo `changed`), nếu đã đúng thì `ok`.

Các module **xuất hiện trong dự án này** (rất nên thuộc):

| Module | Công dụng | Ví dụ trong repo |
|---|---|---|
| `apt` | cài/gỡ gói Debian/Ubuntu | cài containerd ở [k8s-setup/tasks](ansible/roles/k8s-setup/tasks/main.yaml) |
| `package` | cài gói (đa distro) | cài chrony ở [time-sync](ansible/roles/time-sync/tasks/main.yaml) |
| `service` / `systemd` | quản lý dịch vụ | bật kubelet ở [kubernetes_role](ansible/roles/kubernetes_role/tasks/main.yaml) |
| `copy` | copy nội dung/file lên máy đích | ghi `/etc/modules-load.d/containerd.conf` |
| `template` | render file Jinja2 rồi copy | render kubeadm config ([first-master](ansible/roles/first-master/tasks/main.yaml)) |
| `file` | tạo thư mục/đặt quyền | tạo `/etc/kubernetes`, `~/.kube` |
| `lineinfile` | sửa/thêm 1 dòng trong file | bật `SystemdCgroup = true` |
| `command` | chạy lệnh (không qua shell) | `kubeadm init`, `sysctl --system` |
| `shell` | chạy lệnh **qua shell** (có pipe `|`, `>`) | thêm GPG key bằng `curl … | gpg …` |
| `stat` | lấy thông tin file (tồn tại?) | kiểm tra `admin.conf` đã có chưa |
| `get_url` | tải file từ URL | tải Helm ([helm role](ansible/roles/helm/tasks/main.yaml)) |
| `unarchive` | giải nén | giải nén Helm tarball |
| `fetch` | **kéo** file từ máy đích về control node | copy cert giữa các master ([deploy.yaml](ansible/deploy.yaml)) |
| `set_fact` | tạo biến runtime | lưu join command |
| `debug` | in biến/thông báo | in trạng thái |
| `hostname` | đặt hostname | [kubernetes_role](ansible/roles/kubernetes_role/tasks/main.yaml) |
| `wait_for` / `reboot` | chờ điều kiện / khởi động lại | [set_hostnames.yml](ansible/set_hostnames.yml) |

**`command` vs `shell`:** `command` an toàn hơn nhưng **không** hiểu `|`, `>`, `&&`. Khi cần pipe (vd `curl … | gpg …`) phải dùng `shell` — xem [kubernetes_role/tasks](ansible/roles/kubernetes_role/tasks/main.yaml):

```yaml
- name: Add kubernetes Apt key
  ansible.builtin.shell: |
    curl -fsSL {{ kubernetes_repo_url }}Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
  args:
    creates: /etc/apt/keyrings/kubernetes-apt-keyring.gpg   # idempotent: bỏ qua nếu file đã có
```

> 💡 `command`/`shell` **không tự idempotent** (Ansible không biết lệnh làm gì). Ta tự làm idempotent bằng `args: creates:`/`removes:` hoặc `when:`/`changed_when:`.

---

## 8. Biến (Variables), Facts, magic vars

### 8.1. Khai báo & dùng biến
Chèn biến bằng `{{ }}`. Repo dùng biến khắp nơi — vd [kubernetes_role/defaults/main.yaml](ansible/roles/kubernetes_role/defaults/main.yaml):

```yaml
kubernetes_package:
  - kubelet=1.28.7-1.1
  - kubeadm=1.28.7-1.1
  - kubectl=1.28.7-1.1
kubernetes_version: "v1.28"
kubernetes_repo_url: "https://pkgs.k8s.io/core:/stable:/{{ kubernetes_version }}/deb/"
```
→ `kubernetes_repo_url` tham chiếu lại `kubernetes_version` (biến lồng biến).

### 8.2. Nơi đặt biến & độ ưu tiên (precedence)
Từ **thấp → cao** (cao đè thấp), những loại repo này dùng:

| Nơi | Độ ưu tiên | Trong repo |
|---|---|---|
| `roles/<role>/defaults/main.yaml` | thấp nhất (dễ override) | [k8s-setup/defaults](ansible/roles/k8s-setup/defaults/main.yaml), [first-master/defaults](ansible/roles/first-master/defaults/main.yaml) |
| `group_vars/` , `host_vars/` | trung bình | *(repo chưa dùng — nên thêm)* |
| `vars:` trong play / `roles/<role>/vars/` | cao | [kubectl_role/vars/main.yaml](ansible/roles/kubectl_role/vars/main.yaml) |
| `set_fact` | cao | lưu join command |
| `-e` (extra-vars CLI) | **cao nhất** | `ansible-playbook … -e kubernetes_version=v1.34` |

> **`defaults` vs `vars`:** đặt giá trị **mặc định, dễ cho người dùng đổi** vào `defaults/`; đặt giá trị **nội bộ role, không nên đổi** vào `vars/`.

### 8.3. Facts — thông tin tự thu thập
Đầu mỗi play, Ansible chạy "gather_facts" để lấy thông tin máy đích (IP, hostname, OS…) thành biến `ansible_*`. Repo dùng:
- `ansible_hostname` → đặt lại hostname ([kubernetes_role](ansible/roles/kubernetes_role/tasks/main.yaml)): `hostname: name: "{{ ansible_hostname }}"`
- `ansible_env.HOME`, `ansible_env.USER` → ở [kubectl_alias role](ansible/roles/kubectl_alias/tasks/main.yaml)

Tắt thu thập facts để chạy nhanh hơn: `gather_facts: false` (repo dùng trong [join_workers.yaml](ansible/join_workers.yaml)).

### 8.4. Magic variables — biến "ma thuật"
Ansible cấp sẵn vài biến đặc biệt:

| Biến | Nghĩa | Ví dụ repo |
|---|---|---|
| `inventory_hostname` | tên host hiện tại trong inventory | `when: inventory_hostname == 'master1'` ([set_hostnames.yml](ansible/set_hostnames.yml)) |
| `hostvars` | truy biến của host **khác** | `hostvars['master1'].join_command` ([join_workers.yaml](ansible/join_workers.yaml)) |
| `groups` | các nhóm & host trong inventory | dùng để lấy host đầu của 1 nhóm |
| `ansible_*` | facts của máy đích | `ansible_default_ipv4.address` |

Ví dụ "ngôi sao" — truyền dữ liệu **giữa các host** qua `hostvars` ([join_workers.yaml](ansible/join_workers.yaml)):
```yaml
# Play 1: chạy trên master1, tạo token, lưu vào fact
- hosts: master1
  tasks:
    - shell: kubeadm token create --print-join-command --ttl 24h
      register: join_command_output
    - set_fact:
        join_command: "{{ join_command_output.stdout }}"

# Play 2: chạy trên worker, đọc fact CỦA master1 qua hostvars
- hosts: node
  vars:
    join_command: "{{ hostvars['master1'].join_command }}"
  tasks:
    - command: "{{ join_command }}"
```

---

## 9. Điều kiện & vòng lặp (`when`, `loop`)

### `when:` — chạy task có điều kiện
[set_hostnames.yml](ansible/set_hostnames.yml):
```yaml
- name: Set hostname for master1
  when: inventory_hostname == 'master1'
  command: hostnamectl set-hostname master1
```
[first-master/tasks](ansible/roles/first-master/tasks/main.yaml) — chỉ init khi config tồn tại:
```yaml
- command: kubeadm init --config ...
  when: kubeadm_config.stat.exists
```

### `loop` / `with_items` — lặp
[kubernetes_role/tasks](ansible/roles/kubernetes_role/tasks/main.yaml):
```yaml
- name: Install dependencies package
  apt:
    name: "{{ item }}"     # "item" = phần tử hiện tại
    state: present
  loop:
    - "apt-transport-https"
    - "ca-certificates"
    - "curl"
    - "gpg"
```
[deploy.yaml](ansible/deploy.yaml) dùng `with_items` (tên cũ của `loop`) để copy nhiều manifest. Lặp trên kết quả lệnh — [add_labels-master.yaml](ansible/add_labels-master.yaml):
```yaml
- shell: kubectl get nodes | awk '{print $1}'
  register: node_list
- set_fact:
    nodes: "{{ node_list.stdout_lines[1:] }}"   # bỏ dòng tiêu đề
- command: kubectl label node {{ item }} node-role.kubernetes.io/control-plane=true --overwrite
  loop: "{{ nodes }}"
```

---

## 10. Handlers & `notify`

**Handler** là task **chỉ chạy khi được `notify`**, và chạy **một lần ở cuối play** (dù được notify nhiều lần). Dùng điển hình: *đổi config → restart service*.

[k8s-setup](ansible/roles/k8s-setup/tasks/main.yaml) báo handler khi đổi config containerd:
```yaml
- name: Copy config.toml to /etc/containerd
  copy:
    src: /tmp/containerd_config.toml
    dest: /etc/containerd/config.toml
  notify: restart containerd          # ← kích hoạt handler
```
Handler nằm ở [k8s-setup/handlers/main.yaml](ansible/roles/k8s-setup/handlers/main.yaml):
```yaml
- name: restart containerd
  service:
    name: containerd
    state: restarted
```
[kubernetes_role/handlers](ansible/roles/kubernetes_role/handlers/main.yaml) có handler "Enable and start kubelet"; [set_hostnames.yml](ansible/set_hostnames.yml) có handler `reboot`.

> ⚠️ **Bẫy trong repo:** role `time-sync` đặt handler vào thư mục **`handler/`** (số ít). Ansible chỉ tự nạp **`handlers/`** (số nhiều) → handler đó không được nạp. Đây là lỗi đặt tên thư mục.

---

## 11. Templates & Jinja2 (`.j2`)

Module `template` render file **Jinja2** (`.j2`) — chèn biến/vòng lặp/điều kiện — rồi copy lên máy đích. Khác `copy` ở chỗ nội dung **động**.

[first-master/tasks](ansible/roles/first-master/tasks/main.yaml):
```yaml
- name: Copy kubeadm configuration
  template:
    src: roles/first-master/templates/config.yaml.j2
    dest: "{{ kubernetes_dir }}/kubeadm-config.yaml"
```
Template [config.yaml.j2](ansible/roles/first-master/templates/config.yaml.j2):
```jinja
localAPIEndpoint:
  advertiseAddress: {{ advertise_address }}     # chèn biến
  bindPort: 6443
...
controlPlaneEndpoint: "{{ LOAD_BALANCER_IP }}:6443"
networking:
  podSubnet: "192.168.0.0/16"
```

Cú pháp Jinja2 hay dùng:
- `{{ biến }}` — chèn giá trị
- `{% for x in list %} … {% endfor %}` — vòng lặp
- `{% if cond %} … {% endif %}` — điều kiện
- **filter** với `|`: `{{ name | default('x') }}`, `{{ list | length }}`, `{{ s | upper }}`

> Repo có 2 template `.j2` khác trong role [ebs-csi-driver/templates](ansible/roles/ebs-csi-driver/templates/) — cùng nguyên lý.

---

## 12. Roles — đóng gói & tái sử dụng

**Role** là cách đóng gói task + biến + template + handler thành 1 đơn vị tái dùng. Ansible **tự nạp** file `main.yaml` trong các thư mục con theo quy ước:

```
roles/<tên_role>/
├── tasks/main.yaml        # các task (BẮT BUỘC nếu có logic)
├── defaults/main.yaml     # biến mặc định (ưu tiên thấp)
├── vars/main.yaml         # biến nội bộ (ưu tiên cao)
├── handlers/main.yaml     # handlers
├── templates/             # file .j2
├── files/                 # file tĩnh để copy
└── meta/main.yaml         # metadata, dependencies
```

Ví dụ điển hình — role [k8s-setup](ansible/roles/k8s-setup/) có đủ `tasks/` + `defaults/` + `handlers/`. Role [first-master](ansible/roles/first-master/) có thêm `templates/`. Role [kubectl_alias](ansible/roles/kubectl_alias/) có `files/` (chứa [kubectl_aliases.sh](ansible/roles/kubectl_alias/files/kubectl_aliases.sh)).

### Gọi role: `roles:` vs `import_role`/`include_role`
```yaml
# Cách 1 — khai báo ở cấp play (chạy trước tasks của play):
- hosts: k8s_node
  roles:
    - kubectl_alias        # xem kubectl_alias.yaml

# Cách 2 — gọi như một task (chạy đúng vị trí trong tasks):
- hosts: k8s_node
  tasks:
    - import_role:
        name: k8s-setup    # xem k8s-setup.yaml
```
Repo dùng cả hai kiểu (`import_role` ở [k8s-setup.yaml](ansible/k8s-setup.yaml), `roles:` ở [kubectl_alias.yaml](ansible/kubectl_alias.yaml)).

### Chia nhỏ tasks: `include_tasks`
Role [kubectl_role](ansible/roles/kubectl_role/tasks/main.yaml) tách logic thành nhiều file:
```yaml
- include_tasks: install_kubectl.yml
- include_tasks: fetch_admin_config.yml
- include_tasks: permissions.yml
```

---

## 13. Privilege escalation (`become`/sudo)

Hầu hết việc dựng K8s cần **root**. `become: yes` bảo Ansible chạy task bằng `sudo`.

```yaml
- hosts: k8s_node
  become: yes              # cả play chạy bằng sudo
  tasks:
    - apt: { name: containerd, state: present }
```
Có thể bật ở mức **task** (xem nhiều chỗ trong [k8s-setup/tasks](ansible/roles/k8s-setup/tasks/main.yaml): `become: yes`), hoặc đổi user đích bằng `become_user`:
- [first-master/tasks](ansible/roles/first-master/tasks/main.yaml) tạo `.kube` cho user `ansible-user` với `owner: ansible-user`.
- [kubectl_alias](ansible/roles/kubectl_alias/tasks/main.yaml) dùng `become_user: "{{ ansible_user | default(ansible_env.USER) }}"`.

> Điều kiện: user SSH (`ansible-user`) phải có quyền `sudo` (lý tưởng là NOPASSWD) trên máy đích.

---

## 14. Delegation & chạy cục bộ (`delegate_to`, `local`)

### `delegate_to` — chạy task trên host khác
Bình thường task chạy trên host của play. `delegate_to` ép task chạy trên **một host cụ thể**. Repo dùng rất nhiều trong [first-master/tasks](ansible/roles/first-master/tasks/main.yaml):
```yaml
- name: Initialize the Kubernetes master
  delegate_to: master1                 # luôn chạy trên master1
  command: kubeadm init --config ...
```
Và trong [deploy.yaml](ansible/deploy.yaml), `fetch … delegate_to: master1` để kéo cert **từ** master1.

### Chạy trên control node
- `connection: local` hoặc nhóm `[local]` (như [kubectl_role.yaml](ansible/kubectl_role.yaml) chạy `hosts: local`) → thao tác ngay trên máy điều khiển (cài kubectl, lấy kubeconfig về).
- `run_once: true` → task chỉ chạy 1 lần dù play có nhiều host (hữu ích khi sinh token).

> ⚠️ **Bẫy trong repo:** [first-master/tasks](ansible/roles/first-master/tasks/main.yaml) có `delegate_to: bastion` nhưng inventory **không định nghĩa host `bastion`** → task đó sẽ lỗi. Cần đổi thành `localhost` hoặc thêm host `bastion`.

---

## 15. Điều khiển task (`register`, `changed_when`, `creates`…)

| Cơ chế | Công dụng | Ví dụ repo |
|---|---|---|
| `register` | lưu kết quả task vào biến (`.stdout`, `.rc`, `.stat`…) | `register: join_command` |
| `.stdout` / `.stdout_lines` | output lệnh (chuỗi / list dòng) | `node_list.stdout_lines[1:]` |
| `.rc` | mã thoát lệnh | `failed_when: kubeadm_init.rc != 0` |
| `args: creates:` | bỏ qua lệnh nếu file đã tồn tại (idempotent thủ công) | thêm GPG key, repo |
| `changed_when:` | tự định nghĩa khi nào coi là "changed" | `changed_when: false` cho lệnh chỉ đọc |
| `failed_when:` | tự định nghĩa khi nào coi là "failed" | `failed_when: validate_output.rc != 0` |
| `ignore_errors: yes` | bỏ qua lỗi, chạy tiếp | `kubeadm init` trong first-master |
| `gather_facts: false` | tắt thu thập facts | join_workers |

Ví dụ kết hợp — [first-master/tasks](ansible/roles/first-master/tasks/main.yaml):
```yaml
- command: kubeadm init --config "{{ kubernetes_dir }}/kubeadm-config.yaml" --ignore-preflight-errors=NumCPU,Mem
  when: kubeadm_config.stat.exists     # chỉ chạy nếu có config
  register: kubeadm_init               # lưu kết quả
  failed_when: kubeadm_init.rc != 0    # tự xét fail
  ignore_errors: yes                   # nhưng vẫn chạy tiếp (⚠️ che lỗi - xem §19)
```

---

## 16. `import` vs `include` — ghép playbook/role/task

| Cú pháp | Tĩnh/Động | Khi nào nạp |
|---|---|---|
| `import_playbook`, `import_role`, `import_tasks` | **Tĩnh** | nạp lúc parse (trước khi chạy) |
| `include_role`, `include_tasks` | **Động** | nạp lúc chạy (cho phép dùng trong `loop`, `when` runtime) |

Repo dùng:
- `import_playbook` ở orchestrator [k8s_installer.yaml](ansible/k8s_installer.yaml)
- `import_role` ở [k8s-setup.yaml](ansible/k8s-setup.yaml)
- `include_tasks` ở [kubectl_role/tasks/main.yaml](ansible/roles/kubectl_role/tasks/main.yaml)

> Mẹo chọn: cần `tags`/thấy task khi `--list-tasks` → dùng `import`. Cần điều kiện/loop động → dùng `include`.

---

## 17. `fetch` / `copy` — chuyển file qua lại

- `copy`: **đẩy** file từ control node (hoặc nội dung inline) **lên** máy đích.
- `fetch`: **kéo** file **từ** máy đích **về** control node.
- `remote_src: yes` (trong `copy`): nguồn nằm sẵn trên máy đích (copy nội bộ remote).

Repo minh họa rõ trong [deploy.yaml](ansible/deploy.yaml) — đồng bộ cert/manifest giữa các master:
```yaml
- name: Fetch pki files from master1 to the controller
  fetch:                                # KÉO từ master1 về /tmp local
    src: "/etc/kubernetes/pki/{{ item }}"
    dest: "/tmp/pki/{{ item }}"
    flat: yes
  delegate_to: master1

- name: Copy fetched pki files to remote masters
  copy:                                 # ĐẨY từ /tmp local LÊN master2/3
    src: "/tmp/pki/{{ item }}"
    dest: "/etc/kubernetes/pki/{{ item }}"
```
→ Mẫu "fetch về controller rồi copy sang host khác" là cách Ansible chuyển file **giữa hai máy đích** (Ansible không copy trực tiếp host‑to‑host).

---

## 18. Bản đồ tổng thể dự án (ai làm gì)

Toàn bộ luồng do [k8s_installer.yaml](ansible/k8s_installer.yaml) điều phối. Bảng dưới gắn **playbook → role → khái niệm Ansible** để bạn đọc code không lạc:

| Thứ tự | Playbook | Role/Logic | Khái niệm Ansible nổi bật |
|---|---|---|---|
| 1 | [k8s-setup.yaml](ansible/k8s-setup.yaml) | [k8s-setup](ansible/roles/k8s-setup/) | `apt`, `copy`, `lineinfile`, `shell`, **handler** `restart containerd` |
| 2 | [set_hostnames.yml](ansible/set_hostnames.yml) | inline tasks | `when` theo `inventory_hostname`, **handler** `reboot` |
| 3 | [time-sync.yaml](ansible/time-sync.yaml) | [time-sync](ansible/roles/time-sync/) | `package`, `service`, biến `defaults` |
| 4 | [helm-install.yaml](ansible/helm-install.yaml) | [helm](ansible/roles/helm/) | `get_url`, `unarchive`, `register`+`debug` |
| 5 | [kubernetes_role.yaml](ansible/kubernetes_role.yaml) | [kubernetes_role](ansible/roles/kubernetes_role/) | `loop`, `shell+creates`, `apt-mark hold`, `systemd` |
| 6 | [first-master.yaml](ansible/first-master.yaml) | [first-master](ansible/roles/first-master/) | `template` (.j2), `delegate_to`, `stat`, `register`/`when`, `set_fact` |
| 7 | [join_masters.yaml](ansible/join_masters.yaml) | inline | `hostvars`, `set_fact` (⚠️ cách join HA không chuẩn) |
| 8 | [join_workers.yaml](ansible/join_workers.yaml) | inline | truyền biến giữa play qua `hostvars`, `environment` |
| 9 | [install_cni.yaml](ansible/install_cni.yaml) | inline | `shell` chạy `helm install` |
| 10 | [kubectl_role.yaml](ansible/kubectl_role.yaml) | [kubectl_role](ansible/roles/kubectl_role/) | `include_tasks`, `fetch`, `connection: local` |
| 11 | [add_labels-master.yaml](ansible/add_labels-master.yaml) / [add_labels_worker.yaml](ansible/add_labels_worker.yaml) | inline | `register`→`stdout_lines`→`loop` |
| 12 | [nfs.yaml](ansible/nfs.yaml) / [nfs-setup.yaml](ansible/nfs-setup.yaml) | [nfs](ansible/roles/nfs/) / [nfs-setup](ansible/roles/nfs-setup/) | role có `templates/` (`.j2`), `handlers/` |
| 13 | [deploy.yaml](ansible/deploy.yaml) / [all.yaml](ansible/all.yaml) | inline | `fetch`+`copy` đồng bộ cert giữa master |

> Đọc theo thứ tự này, đối chiếu từng khái niệm ở các mục 4–17, là bạn hiểu **toàn bộ phần Ansible** của dự án.

---

## 19. Các anti-pattern trong code dự án (học từ lỗi)

Code thật luôn có chỗ chưa chuẩn — nhận ra chúng giúp bạn "hero" thật sự:

1. **`hostlist=...` sau `import_playbook`** ([k8s_installer.yaml](ansible/k8s_installer.yaml)): vô tác dụng, gây hiểu nhầm. Phạm vi host nằm ở `hosts:` trong playbook con.
2. **`ignore_errors: yes` che lỗi** ([first-master/tasks](ansible/roles/first-master/tasks/main.yaml)): `kubeadm init` fail nhưng playbook vẫn chạy tiếp → khó debug. Nên xử lý lỗi có chủ đích.
3. **Lạm dụng `shell`/`command` thay cho module**: vd `swapoff -a` + `sed` thay vì module chuyên dụng; mất tính idempotent. Nên dùng module hoặc thêm `creates:`/`changed_when:`.
4. **Thiếu `ansible.cfg` và `group_vars/`**: biến rải rác trong nhiều `defaults/`, version K8s/containerd không tập trung → khó đổi đồng bộ.
5. **Đường dẫn & user hardcode** (`/home/ansible-user`, host `master1`, `bastion`): kém linh hoạt, dễ vỡ nếu đổi tên.
6. **Thư mục handler đặt sai tên** (`time-sync/handler` thay vì `handlers`): handler không được nạp.
7. **Inventory để trống** + tham chiếu host chưa định nghĩa (`master1/2/3`, `bastion`): chạy sẽ lỗi nếu không bổ sung.
8. **File trùng/rác**: nhiều bản `add_labels*` (`-master`, `_woker`, `_worker`…), 2 inventory (`inventory` và `inventories/inventory.ini`).

> 👉 Hai file hướng dẫn kèm theo — [K8S-1CP-NWORKER-GUIDE.md](K8S-1CP-NWORKER-GUIDE.md) — đã **sửa hầu hết các điểm trên** (thêm `ansible.cfg`, `group_vars/all.yml`, idempotent bằng `creates:`, bỏ HA copy cert…). Đối chiếu 2 bên là bài học refactor rất tốt.

---

## 20. Cheat sheet & lệnh hay dùng

```bash
# Kiểm tra kết nối
ansible all -m ping

# Liệt kê host khớp pattern (không chạy gì)
ansible controlplane --list-hosts

# Chạy playbook
ansible-playbook k8s_installer.yaml

# Chạy thử (dry-run) — xem sẽ đổi gì mà không thực thi
ansible-playbook k8s-setup.yaml --check

# In ra các thay đổi chi tiết
ansible-playbook k8s-setup.yaml --diff

# Giới hạn host
ansible-playbook k8s-setup.yaml --limit worker1

# Truyền biến từ CLI (ưu tiên cao nhất)
ansible-playbook kubernetes_role.yaml -e kubernetes_version=v1.34

# Chỉ chạy task có tag / liệt kê task
ansible-playbook site.yaml --tags "containerd"
ansible-playbook site.yaml --list-tasks

# Verbose để debug (thêm v càng nhiều càng chi tiết)
ansible-playbook first-master.yaml -vvv
```

**Khái niệm "phải thuộc lòng":** inventory → group → playbook → play → task → module; biến (`defaults`/`vars`/`-e`) → precedence; `register`/`when`/`loop`; `become`; `template`+Jinja2; `handler`+`notify`; `delegate_to`; idempotent (`creates`/`changed_when`).

---

## 21. Tài liệu tham khảo chính thức

- Ansible — *Getting started*: <https://docs.ansible.com/ansible/latest/getting_started/index.html>
- Ansible — *How to build inventory*: <https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html>
- Ansible — *Playbooks*: <https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_intro.html>
- Ansible — *Roles*: <https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html>
- Ansible — *Variables & precedence*: <https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html>
- Ansible — *Conditionals & loops*: <https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_conditionals.html>
- Ansible — *Handlers*: <https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html>
- Ansible — *Templating (Jinja2)*: <https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_templating.html>
- Ansible — *Module Index* (tra cứu module): <https://docs.ansible.com/ansible/latest/collections/index_module.html>
- Ansible — *Privilege escalation (become)*: <https://docs.ansible.com/ansible/latest/playbook_guide/become.html>
- Ansible — *Delegation*: <https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_delegation.html>

---

*Tài liệu học Ansible bám theo codebase `ansible_playbook_k8s-installation`. Mọi đường dẫn trong ngoặc đều click được để mở đúng file trong repo.*
