# Thay đổi Container Runtime trên Node từ Docker Engine sang containerd (Changing the Container Runtime on a Node from Docker Engine to containerd)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/change-runtime-containerd/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 27 — Di chuyển khỏi dockershim (cluster cũ)](00-ALO-TRINH-ADMIN.md#giai-đoạn-27--di-chuyển-khỏi-dockershim-cluster-cũ),
bài 4/6 · Phần II không có lab riêng: tự chấm bằng **Checkpoint** ghi ở cuối mục giai đoạn 27 trong
lộ trình. Bài này **không kiểm chứng được trên cluster lab**: ba node lab đã chạy containerd từ
[Lab 00 phần A4.2](labs/LAB-00-MOI-TRUONG-1.35.7.md#a42-cài-containerd-và-runc-đúng-version), nên
không có Docker daemon nào để dừng và không có gì để chuyển đổi.

**Đây là runbook cho cluster của người khác.** Lộ trình cho phép *bỏ qua toàn bộ giai đoạn 27 nếu
cluster của bạn đã dùng containerd*. Tuyệt đối **không** cài Docker Engine lên node lab để "có cái
mà chuyển": việc đó phá baseline của Lab 00 và không dạy thêm gì. Điều đáng lấy ở đây là **trình tự**
và **những chỗ dễ chết người** trong trình tự đó. Một nửa khung ngoài thì bạn đã chạy tay rồi: vòng
`drain → bảo trì → uncordon` là bài [255](255-safely-drain-node-vi.md) của giai đoạn 16 và
[Lab 12 phần B7](labs/LAB-12-VAN-HANH-VONG-DOI-NODE.md#b7-vòng-bảo-trì-node-cordon--drain--tắt--bật--uncordon).
Phần ruột — dừng Docker, cài containerd, trỏ kubelet sang socket mới — thì chỉ đọc.

**Phải hiểu ở lần đọc này:**

- Toàn bộ trang là **một vòng bảo trì node**, làm **từng node một**: mở đầu bằng
  `kubectl drain <node> --ignore-daemonsets`, kết thúc bằng `kubectl uncordon <node>`. Node không
  được uncordon là node bị bỏ quên ngoài vòng lập lịch.
- Thứ tự dừng dịch vụ có ý nghĩa: `systemctl stop kubelet` **trước**, rồi mới
  `systemctl disable docker.service --now`. Kubelet phải im trước khi runtime dưới nó biến mất.
- Chỗ khai báo runtime mới cho kubelet: thêm
  `--container-runtime-endpoint=unix:///run/containerd/containerd.sock` vào
  `/var/lib/kubelet/kubeadm-flags.env`. Với người dùng kubeadm, CRI socket của host còn được lưu ở
  `/var/lib/kubelet/instance-config.yaml` qua tham số `containerRuntimeEndpoint`.
- Cách xác minh trước khi đi tiếp: `kubectl get nodes -o wide` phải hiển thị containerd là runtime
  của node vừa đổi — **đúng cột mà bài [239](239-find-out-runtime-vi.md) dạy đọc**. Chỉ khi node đã
  có vẻ healthy mới sang bước gỡ Docker.
- Hai cảnh báo của bước gỡ Docker: lệnh `remove`/`purge` gói `docker-ce`, `docker-ce-cli`
  **không** xóa image, container, volume hay file cấu hình tùy chỉnh trên host; và hướng dẫn gỡ cài
  đặt của Docker **tiềm ẩn rủi ro xóa nhầm containerd** — tức xóa mất chính runtime bạn vừa dựng.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Nhánh **Windows (PowerShell)** của mục *Cài đặt containerd* | node Windows là một nhánh riêng của lộ trình, không thuộc giai đoạn 27 | [giai đoạn 15](00-ALO-TRINH-ADMIN.md#giai-đoạn-15--windows-nếu-môi-trường-có-node-windows) và [Lab 15](labs/LAB-15-NODE-WINDOWS.md) |
| Chi tiết thiết lập kho Docker và cài gói `containerd.io` cho từng bản phân phối Linux | chỉ áp dụng khi tiếp quản cluster dockershim; ba node lab đã có containerd và runc đúng version từ trước | [Lab 00 phần A4.2](labs/LAB-00-MOI-TRUONG-1.35.7.md#a42-cài-containerd-và-runc-đúng-version) đã làm phần này một lần rồi |
| Bốn khối lệnh gỡ Docker theo CentOS / Debian / Fedora / Ubuntu | chỉ áp dụng khi tiếp quản cluster dockershim; là thao tác của gói phần mềm, không phải của Kubernetes | chỉ cần khi thực sự có Docker Engine trên node phải gỡ |
| Hai ghi chú về nguyên tắc nội dung bên thứ ba của website CNCF | là chú thích biên tập của kubernetes.io, không mang nội dung kỹ thuật | không cần |

---

Trang này trình bày các bước cần thiết để chuyển container runtime của bạn từ Docker sang containerd. Nội dung áp dụng cho những người vận hành cluster đang chạy Kubernetes 1.23 trở về trước. Trang này cũng bao gồm một kịch bản ví dụ về việc di chuyển (migrate) từ dockershim sang containerd. Bạn có thể chọn container runtime thay thế khác từ [trang này](00-container-runtimes-vi.md).

## Trước khi bạn bắt đầu (Before you begin)

> **Ghi chú:** Mục này liên kết tới các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án đó. Trang này tuân theo [nguyên tắc nội dung của website CNCF](https://github.com/cncf/foundation/blob/master/website-guidelines.md), liệt kê các mục theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc [hướng dẫn nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content) trước khi gửi thay đổi.

Cài đặt containerd. Để biết thêm thông tin, xem
[tài liệu cài đặt của containerd](https://containerd.io/docs/getting-started/)
và với các điều kiện tiên quyết cụ thể, hãy làm theo
[hướng dẫn containerd](00-container-runtimes-vi.md#containerd).

## Drain node (Drain the node)

```shell
kubectl drain <node-to-drain> --ignore-daemonsets
```

Thay `<node-to-drain>` bằng tên của node mà bạn đang drain.

## Dừng Docker daemon (Stop the Docker daemon)

```shell
systemctl stop kubelet
systemctl disable docker.service --now
```

## Cài đặt containerd (Install Containerd)

Làm theo [hướng dẫn](00-container-runtimes-vi.md#containerd)
để biết các bước chi tiết cài đặt containerd.

#### Linux

1. Cài đặt gói `containerd.io` từ các kho (repository) chính thức của Docker.
   Hướng dẫn thiết lập kho Docker cho từng bản phân phối Linux tương ứng và
   cài đặt gói `containerd.io` có tại
   [Getting started with containerd](https://github.com/containerd/containerd/blob/main/docs/getting-started.md).

2. Cấu hình containerd:

   ```shell
   sudo mkdir -p /etc/containerd
   containerd config default | sudo tee /etc/containerd/config.toml
   ```

3. Khởi động lại containerd:

   ```shell
   sudo systemctl restart containerd
   ```

#### Windows (PowerShell)

Mở một phiên PowerShell, đặt `$Version` thành phiên bản mong muốn (ví dụ: `$Version="1.4.3"`), rồi chạy các lệnh sau:

1. Tải containerd:

   ```powershell
   curl.exe -L https://github.com/containerd/containerd/releases/download/v$Version/containerd-$Version-windows-amd64.tar.gz -o containerd-windows-amd64.tar.gz
   tar.exe xvf .\containerd-windows-amd64.tar.gz
   ```

2. Giải nén và cấu hình:

   ```powershell
   Copy-Item -Path ".\bin\" -Destination "$Env:ProgramFiles\containerd" -Recurse -Force
   cd $Env:ProgramFiles\containerd\
   .\containerd.exe config default | Out-File config.toml -Encoding ascii

   # Xem lại cấu hình. Tùy vào thiết lập, bạn có thể muốn điều chỉnh:
   # - sandbox_image (image pause của Kubernetes)
   # - vị trí bin_dir và conf_dir của cni
   Get-Content config.toml

   # (Tùy chọn - nhưng rất nên làm) Loại trừ containerd khỏi quét của Windows Defender
   Add-MpPreference -ExclusionProcess "$Env:ProgramFiles\containerd\containerd.exe"
   ```

3. Khởi động containerd:

   ```powershell
   .\containerd.exe --register-service
   Start-Service containerd
   ```

## Cấu hình kubelet sử dụng containerd làm container runtime (Configure the kubelet to use containerd as its container runtime)

Chỉnh sửa file `/var/lib/kubelet/kubeadm-flags.env` và thêm containerd runtime vào các flag:
`--container-runtime-endpoint=unix:///run/containerd/containerd.sock`.

Người dùng kubeadm cần lưu ý rằng công cụ kubeadm lưu CRI socket của host trong

file `/var/lib/kubelet/instance-config.yaml` trên mỗi node. Bạn có thể tạo file `/var/lib/kubelet/instance-config.yaml` này trên node.

File `/var/lib/kubelet/instance-config.yaml` cho phép thiết lập tham số `containerRuntimeEndpoint`.

Bạn có thể đặt giá trị của tham số này thành đường dẫn của CRI socket mà bạn đã chọn (ví dụ `unix:///run/containerd/containerd.sock`).

## Khởi động lại kubelet (Restart the kubelet)

```shell
systemctl start kubelet
```

## Xác minh node ở trạng thái healthy (Verify that the node is healthy)

Chạy `kubectl get nodes -o wide` và containerd sẽ hiển thị là runtime của node mà chúng ta vừa thay đổi.

## Gỡ bỏ Docker Engine (Remove Docker Engine)

> **Ghi chú:** Mục này liên kết tới các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án đó. Trang này tuân theo [nguyên tắc nội dung của website CNCF](https://github.com/cncf/foundation/blob/master/website-guidelines.md), liệt kê các mục theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc [hướng dẫn nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content) trước khi gửi thay đổi.

Nếu node có vẻ đã healthy, hãy gỡ bỏ Docker.

#### CentOS

```shell
sudo yum remove docker-ce docker-ce-cli
```

#### Debian

```shell
sudo apt-get purge docker-ce docker-ce-cli
```

#### Fedora

```shell
sudo dnf remove docker-ce docker-ce-cli
```

#### Ubuntu

```shell
sudo apt-get purge docker-ce docker-ce-cli
```

Các lệnh trên không xóa image, container, volume hay các file cấu hình tùy chỉnh trên host của bạn.
Để xóa chúng, hãy làm theo hướng dẫn của Docker tại [Uninstall Docker Engine](https://docs.docker.com/engine/install/ubuntu/#uninstall-docker-engine).

> **Thận trọng:** Hướng dẫn gỡ cài đặt Docker Engine của Docker tiềm ẩn rủi ro xóa nhầm containerd. Hãy cẩn thận khi thực thi các lệnh.

## Uncordon node (Uncordon the node)

```shell
kubectl uncordon <node-to-uncordon>
```

Thay `<node-to-uncordon>` bằng tên của node mà bạn đã drain trước đó.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 27. Cả bốn câu trả lời
được bằng lập luận, không cần cluster dockershim:

1. Bài bắt đầu bằng `kubectl drain --ignore-daemonsets` và kết thúc bằng `kubectl uncordon`, làm
   từng node một. Vì sao không thể bỏ bước drain và cứ thế dừng Docker trên node đang chạy, và vì
   sao bước uncordon không được quên?
2. Trình tự bài đưa ra là `systemctl stop kubelet` trước, `systemctl disable docker.service --now`
   sau. Đảo thứ tự hai lệnh đó thì hỏng ở đâu?
3. **Câu bẫy.** Bạn đã cài containerd, đã restart nó, và `crictl` trên node chạy ngon. Vậy kubelet
   đã dùng containerd rồi chứ? Theo bài, còn phải sửa thứ gì nữa, và sửa ở file nào?
4. Bài xếp bước gỡ Docker Engine xuống **sau** bước xác minh node healthy, và kèm một cảnh báo.
   Cảnh báo đó là gì, và vì sao nó nguy hiểm hơn nghe qua? Nếu ba node lab của bạn
   (`lab-k8s-master`, `lab-k8s-worker1`, `lab-k8s-worker2`) dính đúng lỗi đó thì mất cái gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Vì đây là **một vòng bảo trì node** đúng nghĩa: bước tiếp theo sau drain là **dừng kubelet và
   dừng Docker daemon**, tức node mất khả năng chạy container. Không drain trước thì mọi Pod đang
   chạy trên node bị cắt đột ngột thay vì được evict an toàn. Vế sau: **drain đánh dấu node
   unschedulable**, và trạng thái đó nằm trên object Node trong cluster nên **không tự mất đi** khi
   node khỏe lại — quên `kubectl uncordon` thì node đã chuyển runtime xong vẫn nằm ngoài vòng lập
   lịch, không nhận Pod mới.
2. Nếu tắt Docker trước, **kubelet vẫn đang chạy và vẫn đang nói chuyện với runtime cũ**: runtime
   biến mất dưới chân nó, kubelet sẽ báo lỗi và cố xử lý container trên một daemon không còn tồn
   tại. Trình tự của bài đưa node về trạng thái tĩnh trước: **kubelet im, rồi runtime cũ mới bị
   tắt**, sau đó mới cài runtime mới và trỏ kubelet sang nó.
3. **Chưa.** `crictl` chạy được chỉ chứng minh containerd đang phục vụ socket của nó — nó **không**
   nói gì về việc kubelet nối vào đâu, vì kubelet là một client khác. Phải khai báo tường minh cho
   kubelet: thêm `--container-runtime-endpoint=unix:///run/containerd/containerd.sock` vào
   **`/var/lib/kubelet/kubeadm-flags.env`**. Với kubeadm, CRI socket của host còn được lưu ở
   **`/var/lib/kubelet/instance-config.yaml`** qua tham số `containerRuntimeEndpoint`. Rồi mới
   `systemctl start kubelet`, và xác minh bằng `kubectl get nodes -o wide`.
4. Cảnh báo: **hướng dẫn gỡ cài đặt Docker Engine của Docker tiềm ẩn rủi ro xóa nhầm containerd.**
   Nguy hiểm vì nó xóa đúng thứ bạn vừa dựng lên để thay thế — node mất luôn container runtime, tức
   mất khả năng chạy Pod, ngay sau khi bạn vừa xác minh nó healthy. Đó cũng là lý do bài đặt bước gỡ
   Docker **sau** bước `kubectl get nodes -o wide`: gỡ khi chưa chắc chắn thì mất cả hai runtime.
   Ba node lab không có Docker để gỡ, nhưng nếu dính đúng lỗi này thì thứ mất đi là **containerd** —
   runtime duy nhất của cluster lab — và cả ba node sẽ không chạy nổi một container nào.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
