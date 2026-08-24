# Cài đặt và thiết lập kubectl trên Windows (Install and Set Up kubectl on Windows)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/>
>
> Hướng dẫn cài đặt công cụ dòng lệnh kubectl trên Windows bằng cách tải trực tiếp, curl,
> Chocolatey, Scoop hoặc winget; kiểm tra cấu hình, bật tự động hoàn thành (autocompletion)
> cho PowerShell và cài plugin `kubectl convert`.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Tài liệu tra cứu thuộc nhánh Tasks, không nằm trong 15 giai đoạn của lộ trình;
liên quan gần nhất tới nhóm [1b](00-ALO-TRINH-ADMIN.md#1b-làm-việc-với-object-và-kubectl) (bài
[26 — kubectl](26-kubectl-vi.md)). Lab của lộ trình chạy `kubectl` ngay trên VM Linux của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md); chỉ cần bài này khi bạn muốn điều khiển cluster từ máy
Windows cá nhân.

**Phải hiểu ở lần đọc này:**

- Quy tắc version skew của kubectl: client lệch tối đa **một** phiên bản minor so với control
  plane — client v1.36 nói chuyện được với control plane v1.35, v1.36 và v1.37.
- Hai đường cài trên Windows: tải binary trực tiếp/curl (kèm kiểm tra checksum bằng `CertUtil`
  hoặc `Get-FileHash` trong PowerShell) và trình quản lý gói (Chocolatey, Scoop, winget).
- Docker Desktop chèn bản `kubectl` riêng vào `PATH` — thứ tự các mục trong `PATH` quyết định
  bản nào thực sự được chạy.
- kubectl cần file kubeconfig tại `~/.kube/config`; file `config` tạo bằng `New-Item` chỉ là
  file rỗng, phải điền nội dung rồi kiểm chứng bằng `kubectl cluster-info` — xem mục *Kiểm tra
  cấu hình kubectl*.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Cài plugin `kubectl convert` | chỉ cần khi di chuyển manifest khỏi API version bị loại bỏ lúc nâng cấp | [CP2 — Nâng cấp cluster](00-ALO-TRINH-ADMIN.md#cp2--nâng-cấp-cluster) |
| Lỗi "No Auth Provider Found" | chỉ gặp với cluster cloud-managed (AKS, GKE) | ngoài phạm vi lộ trình — đọc khi làm việc với cluster cloud |

---

## Trước khi bạn bắt đầu (Before you begin)

Bạn phải dùng phiên bản kubectl lệch không quá một phiên bản minor so với cluster của bạn.
Ví dụ, một client v1.36 có thể giao tiếp với control plane v1.35, v1.36 và v1.37.
Dùng phiên bản kubectl tương thích mới nhất giúp tránh những sự cố không lường trước.

## Cài đặt kubectl trên Windows (Install kubectl on Windows)

Có các phương pháp sau để cài đặt kubectl trên Windows:

- [Cài đặt kubectl binary trên Windows (tải trực tiếp hoặc dùng curl)](#install-kubectl-binary-on-windows-via-direct-download-or-curl)
- [Cài đặt trên Windows bằng Chocolatey, Scoop hoặc winget](#install-nonstandard-package-tools)

### Cài đặt kubectl binary trên Windows (tải trực tiếp hoặc dùng curl) (Install kubectl binary on Windows (via direct download or curl)) {#install-kubectl-binary-on-windows-via-direct-download-or-curl}

1. Bạn có hai lựa chọn để cài kubectl trên thiết bị Windows của mình

   - Tải trực tiếp:

     Tải trực tiếp binary của bản vá (patch release) mới nhất thuộc dòng 1.36 cho đúng kiến
     trúc máy của bạn từ [trang phát hành Kubernetes](https://kubernetes.io/releases/download/#binaries).
     Nhớ chọn đúng binary cho kiến trúc của bạn (ví dụ: amd64, arm64, v.v.).

   - Dùng curl:

     Nếu bạn đã cài `curl`, dùng lệnh sau:

     ```powershell
     curl.exe -LO "https://dl.k8s.io/release/v1.36.0/bin/windows/amd64/kubectl.exe"
     ```

   > **Ghi chú:** Để biết phiên bản stable mới nhất (ví dụ để dùng trong script), xem
   > [https://dl.k8s.io/release/stable.txt](https://dl.k8s.io/release/stable.txt).

2. Xác thực binary (tùy chọn)

   Tải file checksum của `kubectl`:

   ```powershell
   curl.exe -LO "https://dl.k8s.io/v1.36.0/bin/windows/amd64/kubectl.exe.sha256"
   ```

   Xác thực `kubectl` binary bằng file checksum:

   - Dùng Command Prompt để so sánh thủ công đầu ra của `CertUtil` với file checksum đã tải:

     ```cmd
     CertUtil -hashfile kubectl.exe SHA256
     type kubectl.exe.sha256
     ```

   - Dùng PowerShell để tự động hóa việc xác minh bằng toán tử `-eq`, cho ra kết quả
     `True` hoặc `False`:

     ```powershell
      $(Get-FileHash -Algorithm SHA256 .\kubectl.exe).Hash -eq $(Get-Content .\kubectl.exe.sha256)
     ```

3. Thêm thư mục chứa `kubectl` binary vào (đầu hoặc cuối) biến môi trường `PATH` của bạn.

4. Kiểm tra để đảm bảo phiên bản của `kubectl` đúng với bản đã tải:

   ```cmd
   kubectl version --client
   ```

   Hoặc dùng lệnh sau để xem chi tiết về phiên bản:

   ```cmd
   kubectl version --client --output=yaml
   ```

> **Ghi chú:** [Docker Desktop for Windows](https://docs.docker.com/docker-for-windows/#kubernetes)
> thêm bản `kubectl` riêng của nó vào `PATH`. Nếu bạn đã cài Docker Desktop trước đó, bạn có
> thể cần đặt mục `PATH` của mình trước mục do trình cài đặt Docker Desktop thêm vào, hoặc gỡ
> bỏ `kubectl` của Docker Desktop.

### Cài đặt trên Windows bằng Chocolatey, Scoop hoặc winget (Install on Windows using Chocolatey, Scoop, or winget) {#install-nonstandard-package-tools}

1. Để cài kubectl trên Windows, bạn có thể dùng trình quản lý gói
   [Chocolatey](https://chocolatey.org), trình cài đặt dòng lệnh [Scoop](https://scoop.sh),
   hoặc trình quản lý gói [winget](https://learn.microsoft.com/en-us/windows/package-manager/winget/).

   **choco**

   ```powershell
   choco install kubernetes-cli
   ```

   **scoop**

   ```powershell
   scoop install kubectl
   ```

   **winget**

   ```powershell
   winget install -e --id Kubernetes.kubectl
   ```

2. Kiểm tra để đảm bảo phiên bản bạn vừa cài là mới nhất:

   ```powershell
   kubectl version --client
   ```

3. Di chuyển tới thư mục home của bạn:

   ```powershell
   # Nếu bạn dùng cmd.exe, chạy: cd %USERPROFILE%
   cd ~
   ```

4. Tạo thư mục `.kube`:

   ```powershell
   mkdir .kube
   ```

5. Chuyển vào thư mục `.kube` vừa tạo:

   ```powershell
   cd .kube
   ```

6. Cấu hình kubectl để sử dụng một cluster Kubernetes từ xa:

   ```powershell
   New-Item config -type file
   ```

> **Ghi chú:** Chỉnh sửa file config bằng trình soạn thảo văn bản mà bạn chọn, chẳng hạn Notepad.

## Kiểm tra cấu hình kubectl (Verify kubectl configuration)

Để kubectl tìm và truy cập được một cluster Kubernetes, nó cần một
[file kubeconfig](111-kubeconfig-vi.md), file này được tạo tự động khi bạn tạo cluster bằng
[kube-up.sh](https://github.com/kubernetes/kubernetes/blob/master/cluster/kube-up.sh)
hoặc triển khai thành công một cluster Minikube.
Mặc định, cấu hình của kubectl nằm tại `~/.kube/config`.

Kiểm tra xem kubectl đã được cấu hình đúng chưa bằng cách lấy trạng thái cluster:

```shell
kubectl cluster-info
```

Nếu bạn thấy phản hồi dạng URL, kubectl đã được cấu hình đúng để truy cập cluster của bạn.

Nếu bạn thấy một thông báo tương tự như sau, kubectl chưa được cấu hình đúng hoặc không thể
kết nối tới một cluster Kubernetes.

```
The connection to the server <server-name:port> was refused - did you specify the right host or port?
```

Ví dụ, nếu bạn định chạy một cluster Kubernetes trên laptop của mình (cục bộ), bạn cần cài
trước một công cụ như [Minikube](https://minikube.sigs.k8s.io/docs/start/) rồi chạy lại các
lệnh nêu trên.

Nếu `kubectl cluster-info` trả về phản hồi dạng URL nhưng bạn vẫn không truy cập được cluster,
hãy kiểm tra xem nó đã được cấu hình đúng chưa bằng lệnh:

```shell
kubectl cluster-info dump
```

### Xử lý thông báo lỗi 'No Auth Provider Found' (Troubleshooting the 'No Auth Provider Found' error message) {#no-auth-provider-found}

Ở Kubernetes 1.26, kubectl đã gỡ bỏ cơ chế xác thực tích hợp sẵn cho các dịch vụ Kubernetes
được quản lý (managed) của những nhà cung cấp cloud sau đây. Các nhà cung cấp này đã phát hành
plugin kubectl để cung cấp cơ chế xác thực riêng cho cloud của họ. Xem hướng dẫn trong tài
liệu của từng nhà cung cấp:

* Azure AKS: [kubelogin plugin](https://azure.github.io/kubelogin/)
* Google Kubernetes Engine: [gke-gcloud-auth-plugin](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl#install_plugin)

Cũng có thể có những nguyên nhân khác, không liên quan tới thay đổi trên, dẫn đến cùng thông
báo lỗi này.

## Cấu hình tùy chọn và plugin cho kubectl (Optional kubectl configurations and plugins)

### Bật tự động hoàn thành cho shell (Enable shell autocompletion)

kubectl cung cấp hỗ trợ tự động hoàn thành (autocompletion) cho Bash, Zsh, Fish và PowerShell,
giúp bạn tiết kiệm rất nhiều thao tác gõ.

Dưới đây là quy trình thiết lập tự động hoàn thành cho PowerShell.

Script hoàn thành lệnh (completion script) của kubectl cho PowerShell có thể được sinh ra bằng
lệnh `kubectl completion powershell`.

Để áp dụng trong mọi phiên shell, thêm dòng sau vào file `$PROFILE` của bạn:

```powershell
kubectl completion powershell | Out-String | Invoke-Expression
```

Lệnh này sẽ sinh lại script tự động hoàn thành mỗi lần PowerShell khởi động. Bạn cũng có thể
thêm trực tiếp script đã sinh ra vào file `$PROFILE` của mình.

Để thêm script đã sinh ra vào file `$PROFILE`, chạy dòng sau trong prompt powershell:

```powershell
kubectl completion powershell >> $PROFILE
```

Sau khi tải lại shell, tự động hoàn thành của kubectl sẽ hoạt động.

### Cấu hình kuberc (Configure kuberc)

Xem [kuberc](https://kubernetes.io/docs/reference/kubectl/kuberc) để biết thêm thông tin.

### Cài đặt plugin `kubectl convert` (Install `kubectl convert` plugin)

Đây là một plugin cho công cụ dòng lệnh `kubectl` của Kubernetes, cho phép bạn chuyển đổi
manifest giữa các phiên bản API khác nhau. Nó đặc biệt hữu ích khi cần di chuyển manifest sang
một phiên bản API không bị deprecated (khuyến cáo ngừng sử dụng) ở bản phát hành Kubernetes
mới hơn. Để biết thêm thông tin, xem
[migrate to non deprecated apis](https://kubernetes.io/docs/reference/using-api/deprecation-guide/#migrate-to-non-deprecated-apis)

1. Tải bản phát hành mới nhất bằng lệnh:

   ```powershell
   curl.exe -LO "https://dl.k8s.io/release/v1.36.0/bin/windows/amd64/kubectl-convert.exe"
   ```

2. Xác thực binary (tùy chọn).

   Tải file checksum của `kubectl-convert`:

   ```powershell
   curl.exe -LO "https://dl.k8s.io/v1.36.0/bin/windows/amd64/kubectl-convert.exe.sha256"
   ```

   Xác thực `kubectl-convert` binary bằng file checksum:

   - Dùng Command Prompt để so sánh thủ công đầu ra của `CertUtil` với file checksum đã tải:

     ```cmd
     CertUtil -hashfile kubectl-convert.exe SHA256
     type kubectl-convert.exe.sha256
     ```

   - Dùng PowerShell để tự động hóa việc xác minh bằng toán tử `-eq`, cho ra kết quả
     `True` hoặc `False`:

     ```powershell
     $($(CertUtil -hashfile .\kubectl-convert.exe SHA256)[1] -replace " ", "") -eq $(type .\kubectl-convert.exe.sha256)
     ```

3. Thêm thư mục chứa `kubectl-convert` binary vào (đầu hoặc cuối) biến môi trường `PATH` của bạn.

4. Xác minh plugin đã được cài thành công.

   ```shell
   kubectl convert --help
   ```

   Nếu bạn không thấy lỗi nào, nghĩa là plugin đã được cài đặt thành công.

5. Sau khi cài plugin xong, dọn dẹp các file cài đặt:

   ```powershell
   del kubectl-convert.exe
   del kubectl-convert.exe.sha256
   ```

## Tiếp theo (What's next)

* Tìm hiểu về [kubectl](26-kubectl-vi.md) và vai trò của nó trong hệ sinh thái Kubernetes.
* [Cài đặt Minikube](https://minikube.sigs.k8s.io/docs/start/)
* Xem [các hướng dẫn bắt đầu](https://kubernetes.io/docs/setup/) để biết thêm về cách tạo cluster.
* [Tìm hiểu cách khởi chạy và expose ứng dụng của bạn.](370-service-access-application-cluster-vi.md)
* Nếu bạn cần truy cập một cluster mà bạn không tự tạo, xem
  [tài liệu Chia sẻ quyền truy cập cluster](361-configure-access-multiple-clusters-vi.md).
* Đọc [tài liệu tham khảo kubectl](https://kubernetes.io/docs/reference/kubectl/kubectl/)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc này.

1. Máy Windows của bạn cài kubectl v1.36 bằng winget, còn cluster lab của bạn chạy control
   plane v1.35. Tổ hợp này có được hỗ trợ không? Vì sao?
2. Bạn vừa cài kubectl mới nhất nhưng `kubectl version --client` lại in ra một phiên bản cũ
   hơn hẳn. Trên máy đã cài Docker Desktop, bạn nghi ngờ điều gì đầu tiên và xử lý thế nào?
3. Bạn đã tạo file `~/.kube/config` bằng `New-Item config -type file`. Chạy
   `kubectl cluster-info` ngay lúc này thì kubectl đã truy cập được cluster chưa? Vì sao?
4. Bài đưa ra hai cách xác thực checksum của binary trên Windows. Chúng khác nhau ở điểm nào
   và cách nào phù hợp để tự động hóa?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Được.** Quy tắc version skew cho phép client lệch tối đa một phiên bản minor so với
   control plane: client v1.36 làm việc được với control plane v1.35, v1.36 và v1.37.
2. Nghi ngờ **bản `kubectl` mà Docker Desktop đã thêm vào `PATH`** đang che mất bản bạn vừa
   cài. Bài ghi chú rõ: cần đặt mục `PATH` của bạn **trước** mục do trình cài đặt Docker
   Desktop thêm vào, hoặc gỡ bỏ `kubectl` của Docker Desktop.
3. **Chưa.** `New-Item config -type file` chỉ tạo một **file rỗng**; bạn còn phải điền nội
   dung kubeconfig của cluster (chỉnh bằng trình soạn thảo như Notepad). Với file rỗng,
   `kubectl cluster-info` sẽ báo lỗi kiểu "connection refused" — đây chính là dấu hiệu
   "kubectl chưa được cấu hình đúng" mà mục *Kiểm tra cấu hình kubectl* mô tả.
4. Cách 1 dùng `CertUtil` trong Command Prompt và **so sánh bằng mắt** đầu ra với nội dung
   file `.sha256`; cách 2 dùng PowerShell (`Get-FileHash` + toán tử `-eq`) trả về thẳng
   **`True`/`False`**, nên phù hợp để **tự động hóa** trong script.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
