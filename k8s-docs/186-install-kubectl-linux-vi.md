# Cài đặt và thiết lập kubectl trên Linux (Install and Set Up kubectl on Linux)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/>
>
> Hướng dẫn cài đặt công cụ dòng lệnh kubectl trên Linux, xác minh cấu hình,
> và thiết lập các tùy chọn như tự động hoàn thành cho shell và plugin `kubectl convert`.

## Trước khi bạn bắt đầu (Before you begin)

Bạn phải sử dụng một phiên bản kubectl chỉ lệch trong phạm vi một phiên bản minor so với
cluster của bạn. Ví dụ, một client v1.36 có thể giao tiếp với các control plane
v1.35, v1.36 và v1.37.
Việc dùng phiên bản kubectl tương thích mới nhất giúp tránh các sự cố không lường trước.

## Cài đặt kubectl trên Linux (Install kubectl on Linux) {#install-kubectl-on-linux}

Có các phương pháp sau để cài đặt kubectl trên Linux:

- [Cài đặt kubectl binary bằng curl trên Linux](#install-kubectl-binary-with-curl-on-linux)
- [Cài đặt bằng trình quản lý gói native](#install-using-native-package-management)
- [Cài đặt bằng trình quản lý gói khác](#install-using-other-package-management)

### Cài đặt kubectl binary bằng curl trên Linux (Install kubectl binary with curl on Linux) {#install-kubectl-binary-with-curl-on-linux}

1. Tải bản phát hành (release) mới nhất bằng lệnh:

   **x86-64:**

   ```bash
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   ```

   **ARM64:**

   ```bash
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/arm64/kubectl"
   ```

   > **Ghi chú:** Để tải một phiên bản cụ thể, thay phần `$(curl -L -s https://dl.k8s.io/release/stable.txt)`
   > của lệnh bằng phiên bản cụ thể đó.
   >
   > Ví dụ, để tải phiên bản 1.36.0 trên Linux x86-64, gõ:
   >
   > ```bash
   > curl -LO https://dl.k8s.io/release/v1.36.0/bin/linux/amd64/kubectl
   > ```
   >
   > Và cho Linux ARM64, gõ:
   >
   > ```bash
   > curl -LO https://dl.k8s.io/release/v1.36.0/bin/linux/arm64/kubectl
   > ```

2. Xác thực binary (tùy chọn)

   Tải file checksum của kubectl:

   **x86-64:**

   ```bash
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
   ```

   **ARM64:**

   ```bash
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/arm64/kubectl.sha256"
   ```

   Xác thực kubectl binary bằng file checksum:

   ```bash
   echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
   ```

   Nếu hợp lệ, output sẽ là:

   ```console
   kubectl: OK
   ```

   Nếu việc kiểm tra thất bại, `sha256` thoát với trạng thái khác không (nonzero) và in ra
   output tương tự như:

   ```console
   kubectl: FAILED
   sha256sum: WARNING: 1 computed checksum did NOT match
   ```

   > **Ghi chú:** Hãy tải binary và checksum của cùng một phiên bản.

3. Cài đặt kubectl

   ```bash
   sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
   ```

   > **Ghi chú:** Nếu bạn không có quyền root trên hệ thống đích, bạn vẫn có thể cài đặt
   > kubectl vào thư mục `~/.local/bin`:
   >
   > ```bash
   > chmod +x kubectl
   > mkdir -p ~/.local/bin
   > mv ./kubectl ~/.local/bin/kubectl
   > # và sau đó thêm ~/.local/bin vào cuối (hoặc đầu) $PATH
   > ```

4. Kiểm tra để đảm bảo phiên bản bạn đã cài là mới nhất:

   ```bash
   kubectl version --client
   ```

   Hoặc dùng lệnh sau để xem chi tiết về phiên bản:

   ```cmd
   kubectl version --client --output=yaml
   ```

### Cài đặt bằng trình quản lý gói native (Install using native package management) {#install-using-native-package-management}

#### Các bản phân phối dựa trên Debian (Debian-based distributions)

1. Cập nhật chỉ mục gói `apt` và cài đặt các gói cần thiết để dùng kho (repository) `apt`
   của Kubernetes:

   ```shell
   sudo apt-get update
   # apt-transport-https có thể là một gói giả (dummy); nếu vậy, bạn có thể bỏ qua gói đó
   sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
   ```

2. Tải khóa ký công khai (public signing key) cho các kho gói của Kubernetes.
   Cùng một khóa ký được dùng cho tất cả các kho nên bạn có thể bỏ qua phiên bản trong URL:

   ```shell
   # Nếu thư mục `/etc/apt/keyrings` chưa tồn tại, nó cần được tạo trước lệnh curl, hãy đọc ghi chú bên dưới.
   # sudo mkdir -p -m 755 /etc/apt/keyrings
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
   sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg # cho phép các chương trình APT không đặc quyền đọc keyring này
   ```

   > **Ghi chú:** Trên các bản phát hành cũ hơn Debian 12 và Ubuntu 22.04, thư mục
   > `/etc/apt/keyrings` mặc định không tồn tại, và nó cần được tạo trước lệnh curl.

3. Thêm kho `apt` phù hợp của Kubernetes. Nếu bạn muốn dùng phiên bản Kubernetes khác
   v1.36, hãy thay v1.36 bằng phiên bản minor mong muốn trong lệnh bên dưới:

   ```shell
   # Lệnh này ghi đè mọi cấu hình hiện có trong /etc/apt/sources.list.d/kubernetes.list
   echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
   sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list   # giúp các công cụ như command-not-found hoạt động đúng
   ```

   > **Ghi chú:** Để nâng cấp kubectl lên một bản phát hành minor khác, bạn cần tăng phiên bản
   > trong `/etc/apt/sources.list.d/kubernetes.list` trước khi chạy `apt-get update` và
   > `apt-get upgrade`. Quy trình này được mô tả chi tiết hơn tại
   > [Thay đổi kho gói Kubernetes](217-change-package-repository-vi.md).

4. Cập nhật chỉ mục gói `apt`, sau đó cài đặt kubectl:

   ```shell
   sudo apt-get update
   sudo apt-get install -y kubectl
   ```

#### Các bản phân phối dựa trên Red Hat (Red Hat-based distributions)

1. Thêm kho `yum` của Kubernetes. Nếu bạn muốn dùng phiên bản Kubernetes khác
   v1.36, hãy thay v1.36 bằng phiên bản minor mong muốn trong lệnh bên dưới.

   ```bash
   # Lệnh này ghi đè mọi cấu hình hiện có trong /etc/yum.repos.d/kubernetes.repo
   cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
   [kubernetes]
   name=Kubernetes
   baseurl=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/
   enabled=1
   gpgcheck=1
   gpgkey=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/repodata/repomd.xml.key
   EOF
   ```

   > **Ghi chú:** Để nâng cấp kubectl lên một bản phát hành minor khác, bạn cần tăng phiên bản
   > trong `/etc/yum.repos.d/kubernetes.repo` trước khi chạy `yum update`. Quy trình này được
   > mô tả chi tiết hơn tại
   > [Thay đổi kho gói Kubernetes](217-change-package-repository-vi.md).

2. Cài đặt kubectl bằng `yum`:

   ```bash
   sudo yum install -y kubectl
   ```

#### Các bản phân phối dựa trên SUSE (SUSE-based distributions)

1. Thêm kho `zypper` của Kubernetes. Nếu bạn muốn dùng phiên bản Kubernetes khác
   v1.36, hãy thay v1.36 bằng phiên bản minor mong muốn trong lệnh bên dưới.

   ```bash
   # Lệnh này ghi đè mọi cấu hình hiện có trong /etc/zypp/repos.d/kubernetes.repo
   cat <<EOF | sudo tee /etc/zypp/repos.d/kubernetes.repo
   [kubernetes]
   name=Kubernetes
   baseurl=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/
   enabled=1
   gpgcheck=1
   gpgkey=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/repodata/repomd.xml.key
   EOF
   ```

   > **Ghi chú:** Để nâng cấp kubectl lên một bản phát hành minor khác, bạn cần tăng phiên bản
   > trong `/etc/zypp/repos.d/kubernetes.repo` trước khi chạy `zypper update`. Quy trình này
   > được mô tả chi tiết hơn tại
   > [Thay đổi kho gói Kubernetes](217-change-package-repository-vi.md).

2. Cập nhật `zypper` và xác nhận việc thêm kho mới:

   ```bash
   sudo zypper update
   ```

   Khi thông báo sau xuất hiện, nhấn 't' hoặc 'a':

   ```
   New repository or package signing key received:

   Repository:       Kubernetes
   Key Fingerprint:  1111 2222 3333 4444 5555 6666 7777 8888 9999 AAAA
   Key Name:         isv:kubernetes OBS Project <isv:kubernetes@build.opensuse.org>
   Key Algorithm:    RSA 2048
   Key Created:      Thu 25 Aug 2022 01:21:11 PM -03
   Key Expires:      Sat 02 Nov 2024 01:21:11 PM -03 (expires in 85 days)
   Rpm Name:         gpg-pubkey-9a296436-6307a177

   Note: Signing data enables the recipient to verify that no modifications occurred after the data
   were signed. Accepting data with no, wrong or unknown signature can lead to a corrupted system
   and in extreme cases even to a system compromise.

   Note: A GPG pubkey is clearly identified by its fingerprint. Do not rely on the key's name. If
   you are not sure whether the presented key is authentic, ask the repository provider or check
   their web site. Many providers maintain a web page showing the fingerprints of the GPG keys they
   are using.

   Do you want to reject the key, trust temporarily, or trust always? [r/t/a/?] (r): a
   ```

3. Cài đặt kubectl bằng `zypper`:

   ```bash
   sudo zypper install -y kubectl
   ```

### Cài đặt bằng trình quản lý gói khác (Install using other package management) {#install-using-other-package-management}

#### Snap

Nếu bạn đang dùng Ubuntu hoặc một bản phân phối Linux khác hỗ trợ trình quản lý gói
[snap](https://snapcraft.io/docs/core/install), kubectl có sẵn dưới dạng
một ứng dụng [snap](https://snapcraft.io/).

```shell
snap install kubectl --classic
kubectl version --client
```

#### Homebrew

Nếu bạn đang dùng Linux với trình quản lý gói [Homebrew](https://docs.brew.sh/Homebrew-on-Linux),
kubectl có sẵn để [cài đặt](https://docs.brew.sh/Homebrew-on-Linux#install).

```shell
brew install kubectl
kubectl version --client
```

## Xác minh cấu hình kubectl (Verify kubectl configuration) {#verify-kubectl-configuration}

Để kubectl tìm thấy và truy cập được một cluster Kubernetes, nó cần một
[file kubeconfig](111-kubeconfig-vi.md),
file này được tạo tự động khi bạn tạo cluster bằng
[kube-up.sh](https://github.com/kubernetes/kubernetes/blob/master/cluster/kube-up.sh)
hoặc triển khai thành công một cluster Minikube.
Mặc định, cấu hình của kubectl nằm tại `~/.kube/config`.

Kiểm tra kubectl đã được cấu hình đúng hay chưa bằng cách lấy trạng thái của cluster:

```shell
kubectl cluster-info
```

Nếu bạn thấy một URL trong phản hồi, kubectl đã được cấu hình đúng để truy cập cluster của bạn.

Nếu bạn thấy một thông báo tương tự như sau, kubectl chưa được cấu hình đúng
hoặc không thể kết nối tới một cluster Kubernetes.

```
The connection to the server <server-name:port> was refused - did you specify the right host or port?
```

Ví dụ, nếu bạn định chạy một cluster Kubernetes trên laptop của mình (cục bộ),
bạn cần cài đặt trước một công cụ như [Minikube](https://minikube.sigs.k8s.io/docs/start/)
rồi chạy lại các lệnh nêu trên.

Nếu `kubectl cluster-info` trả về phản hồi có url nhưng bạn vẫn không truy cập được cluster,
hãy kiểm tra xem nó đã được cấu hình đúng chưa bằng lệnh:

```shell
kubectl cluster-info dump
```

### Khắc phục thông báo lỗi 'No Auth Provider Found' (Troubleshooting the 'No Auth Provider Found' error message) {#no-auth-provider-found}

Trong Kubernetes 1.26, kubectl đã loại bỏ cơ chế xác thực (authentication) tích hợp sẵn cho
các dịch vụ Kubernetes được quản lý (managed) của những nhà cung cấp cloud sau. Các nhà cung cấp
này đã phát hành các plugin kubectl để cung cấp cơ chế xác thực riêng cho từng cloud.
Để biết hướng dẫn, hãy tham khảo tài liệu của nhà cung cấp tương ứng:

* Azure AKS: [kubelogin plugin](https://azure.github.io/kubelogin/)
* Google Kubernetes Engine: [gke-gcloud-auth-plugin](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl#install_plugin)

Cũng có thể có những nguyên nhân khác gây ra cùng thông báo lỗi này
mà không liên quan tới thay đổi đó.

## Các cấu hình và plugin tùy chọn của kubectl (Optional kubectl configurations and plugins) {#optional-kubectl-configurations-and-plugins}

### Bật tự động hoàn thành cho shell (Enable shell autocompletion) {#enable-shell-autocompletion}

kubectl cung cấp hỗ trợ tự động hoàn thành (autocompletion) cho Bash, Zsh, Fish và PowerShell,
giúp bạn tiết kiệm rất nhiều thao tác gõ lệnh.

Dưới đây là quy trình thiết lập tự động hoàn thành cho Bash, Fish và Zsh.

#### Bash

##### Giới thiệu (Introduction)

Script hoàn thành lệnh (completion script) của kubectl cho Bash có thể được sinh ra bằng lệnh
`kubectl completion bash`. Việc source script này trong shell của bạn sẽ bật tính năng
tự động hoàn thành của kubectl.

Tuy nhiên, script hoàn thành này phụ thuộc vào
[**bash-completion**](https://github.com/scop/bash-completion),
nghĩa là bạn phải cài đặt phần mềm này trước
(bạn có thể kiểm tra xem bash-completion đã được cài chưa bằng cách chạy `type _init_completion`).

##### Cài đặt bash-completion (Install bash-completion)

bash-completion được cung cấp bởi nhiều trình quản lý gói
(xem [tại đây](https://github.com/scop/bash-completion#installation)).
Bạn có thể cài đặt nó bằng `apt-get install bash-completion` hoặc `yum install bash-completion`, v.v.

Các lệnh trên tạo ra file `/usr/share/bash-completion/bash_completion`,
là script chính của bash-completion. Tùy theo trình quản lý gói của bạn,
bạn có thể phải tự source file này trong file `~/.bashrc` của mình.

Để biết chắc, hãy nạp lại (reload) shell và chạy `type _init_completion`.
Nếu lệnh thành công, bạn đã sẵn sàng; nếu không, hãy thêm dòng sau vào file `~/.bashrc`:

```bash
source /usr/share/bash-completion/bash_completion
```

Nạp lại shell và xác minh bash-completion đã được cài đúng bằng cách gõ `type _init_completion`.

##### Bật tự động hoàn thành cho kubectl (Enable kubectl autocompletion)

**Bash**

Bây giờ bạn cần đảm bảo script hoàn thành lệnh của kubectl được source trong tất cả
các phiên (session) shell của bạn. Có hai cách để làm việc này:

**Người dùng hiện tại (User):**

```bash
echo 'source <(kubectl completion bash)' >>~/.bashrc
```

**Toàn hệ thống (System):**

```bash
kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null
sudo chmod a+r /etc/bash_completion.d/kubectl
```

Nếu bạn có alias cho kubectl, bạn có thể mở rộng tính năng hoàn thành lệnh của shell
để hoạt động với alias đó:

```bash
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
```

> **Ghi chú:** bash-completion source tất cả các script hoàn thành lệnh trong `/etc/bash_completion.d`.

Cả hai cách tiếp cận là tương đương. Sau khi nạp lại shell, tính năng tự động hoàn thành
của kubectl sẽ hoạt động.
Để bật tự động hoàn thành của bash trong phiên shell hiện tại, hãy source file ~/.bashrc:

```bash
source ~/.bashrc
```

#### Fish

> **Ghi chú:** Tự động hoàn thành cho Fish yêu cầu kubectl 1.23 trở lên.

Script hoàn thành lệnh của kubectl cho Fish có thể được sinh ra bằng lệnh
`kubectl completion fish`. Việc source script này trong shell của bạn sẽ bật tính năng
tự động hoàn thành của kubectl.

Để làm việc này trong tất cả các phiên shell của bạn, hãy thêm dòng sau vào file
`~/.config/fish/config.fish`:

```shell
kubectl completion fish | source
```

Sau khi nạp lại shell, tính năng tự động hoàn thành của kubectl sẽ hoạt động.

#### Zsh

Script hoàn thành lệnh của kubectl cho Zsh có thể được sinh ra bằng lệnh
`kubectl completion zsh`. Việc source script này trong shell của bạn sẽ bật tính năng
tự động hoàn thành của kubectl.

Để làm việc này trong tất cả các phiên shell của bạn, hãy thêm dòng sau vào file `~/.zshrc`:

```zsh
source <(kubectl completion zsh)
```

Nếu bạn có alias cho kubectl, tính năng tự động hoàn thành của kubectl sẽ tự động
hoạt động với alias đó.

Sau khi nạp lại shell, tính năng tự động hoàn thành của kubectl sẽ hoạt động.

Nếu bạn gặp lỗi kiểu `2: command not found: compdef`, hãy thêm đoạn sau vào đầu file `~/.zshrc`:

```zsh
autoload -Uz compinit
compinit
```

### Cấu hình kuberc (Configure kuberc) {#configure-kuberc}

Xem [kuberc](https://kubernetes.io/docs/reference/kubectl/kuberc) để biết thêm thông tin.

### Cài đặt plugin `kubectl convert` (Install `kubectl convert` plugin) {#install-kubectl-convert-plugin}

Đây là một plugin cho công cụ dòng lệnh `kubectl` của Kubernetes, cho phép bạn chuyển đổi
manifest giữa các phiên bản API khác nhau. Điều này đặc biệt hữu ích khi di chuyển (migrate)
manifest sang một phiên bản API không bị loại bỏ (non-deprecated) trong bản phát hành
Kubernetes mới hơn.
Để biết thêm thông tin, xem [migrate to non deprecated apis](https://kubernetes.io/docs/reference/using-api/deprecation-guide/#migrate-to-non-deprecated-apis).

1. Tải bản phát hành mới nhất bằng lệnh:

   **x86-64:**

   ```bash
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl-convert"
   ```

   **ARM64:**

   ```bash
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/arm64/kubectl-convert"
   ```

2. Xác thực binary (tùy chọn)

   Tải file checksum của kubectl-convert:

   **x86-64:**

   ```bash
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl-convert.sha256"
   ```

   **ARM64:**

   ```bash
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/arm64/kubectl-convert.sha256"
   ```

   Xác thực kubectl-convert binary bằng file checksum:

   ```bash
   echo "$(cat kubectl-convert.sha256) kubectl-convert" | sha256sum --check
   ```

   Nếu hợp lệ, output sẽ là:

   ```console
   kubectl-convert: OK
   ```

   Nếu việc kiểm tra thất bại, `sha256` thoát với trạng thái khác không (nonzero) và in ra
   output tương tự như:

   ```console
   kubectl-convert: FAILED
   sha256sum: WARNING: 1 computed checksum did NOT match
   ```

   > **Ghi chú:** Hãy tải binary và checksum của cùng một phiên bản.

3. Cài đặt kubectl-convert

   ```bash
   sudo install -o root -g root -m 0755 kubectl-convert /usr/local/bin/kubectl-convert
   ```

4. Xác minh plugin đã được cài đặt thành công

   ```shell
   kubectl convert --help
   ```

   Nếu bạn không thấy lỗi nào, nghĩa là plugin đã được cài đặt thành công.

5. Sau khi cài đặt plugin, dọn dẹp các file cài đặt:

   ```bash
   rm kubectl-convert kubectl-convert.sha256
   ```

## Tiếp theo (What's next)

* Tìm hiểu về [kubectl](26-kubectl-vi.md) và vai trò của nó trong hệ sinh thái Kubernetes.
* [Cài đặt Minikube](https://minikube.sigs.k8s.io/docs/start/)
* Xem [các hướng dẫn bắt đầu](https://kubernetes.io/docs/setup/) để biết thêm về việc tạo cluster.
* [Tìm hiểu cách khởi chạy và expose ứng dụng của bạn.](370-service-access-application-cluster-vi.md)
* Nếu bạn cần truy cập một cluster mà bạn không tạo ra, hãy xem
  [tài liệu Chia sẻ quyền truy cập cluster](361-configure-access-multiple-clusters-vi.md).
* Đọc [tài liệu tham khảo kubectl](https://kubernetes.io/docs/reference/kubectl/kubectl/)
