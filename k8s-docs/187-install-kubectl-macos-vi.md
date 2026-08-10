# Cài đặt và thiết lập kubectl trên macOS (Install and Set Up kubectl on macOS)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/>
>
> Hướng dẫn cài đặt công cụ dòng lệnh kubectl trên macOS bằng curl, Homebrew hoặc Macports;
> kiểm tra cấu hình, bật tự động hoàn thành (autocompletion) cho shell và cài plugin
> `kubectl convert`.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** Tài liệu tra cứu thuộc nhánh Tasks, không nằm trong 15 giai đoạn của lộ trình;
liên quan gần nhất tới nhóm [1b](LO-TRINH-ADMIN.md#1b-làm-việc-với-object-và-kubectl) (bài
[26 — kubectl](26-kubectl-vi.md)). Lab của lộ trình chạy `kubectl` ngay trên VM Linux của
[Lab 00](labs/LAB-00-MOI-TRUONG.md); chỉ cần bài này khi bạn muốn điều khiển cluster từ máy
macOS cá nhân.

**Phải hiểu ở lần đọc này:**

- Quy tắc version skew của kubectl: client lệch tối đa **một** phiên bản minor so với control
  plane — client v1.36 nói chuyện được với control plane v1.35, v1.36 và v1.37.
- Ba cách cài trên macOS (binary bằng curl, Homebrew, Macports) và vì sao khi tải binary thủ
  công phải kiểm tra checksum bằng `shasum -a 256 --check` với file `.sha256` **cùng phiên bản**.
- `kubectl version --client` chỉ chứng minh binary chạy được; muốn truy cập cluster thì cần
  file kubeconfig (`~/.kube/config`) và kiểm chứng bằng `kubectl cluster-info` — xem mục
  *Kiểm tra cấu hình kubectl*.
- Tự động hoàn thành Bash trên macOS đòi hỏi **Bash 4.1+ và bash-completion v2**; Bash 3.2 mặc
  định của macOS không dùng được.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Cài plugin `kubectl convert` | chỉ cần khi di chuyển manifest khỏi API version bị loại bỏ lúc nâng cấp | [CP2 — Nâng cấp cluster](LO-TRINH-ADMIN.md#cp2--nâng-cấp-cluster) |
| Lỗi "No Auth Provider Found" | chỉ gặp với cluster cloud-managed (AKS, GKE) | ngoài phạm vi lộ trình — đọc khi làm việc với cluster cloud |

---

## Trước khi bạn bắt đầu (Before you begin)

Bạn phải dùng phiên bản kubectl lệch không quá một phiên bản minor so với cluster của bạn.
Ví dụ, một client v1.36 có thể giao tiếp với control plane v1.35, v1.36 và v1.37.
Dùng phiên bản kubectl tương thích mới nhất giúp tránh những sự cố không lường trước.

## Cài đặt kubectl trên macOS (Install kubectl on macOS) {#install-kubectl-on-macos}

Có các phương pháp sau để cài đặt kubectl trên macOS:

- [Cài đặt kubectl trên macOS](#install-kubectl-on-macos)
  - [Cài đặt kubectl binary bằng curl trên macOS](#install-kubectl-binary-with-curl-on-macos)
  - [Cài đặt bằng Homebrew trên macOS](#install-with-homebrew-on-macos)
  - [Cài đặt bằng Macports trên macOS](#install-with-macports-on-macos)
- [Kiểm tra cấu hình kubectl](#verify-kubectl-configuration)
- [Cấu hình tùy chọn và plugin cho kubectl](#optional-kubectl-configurations-and-plugins)
  - [Bật tự động hoàn thành cho shell](#enable-shell-autocompletion)
  - [Cài đặt plugin `kubectl convert`](#install-kubectl-convert-plugin)

### Cài đặt kubectl binary bằng curl trên macOS (Install kubectl binary with curl on macOS) {#install-kubectl-binary-with-curl-on-macos}

1. Tải bản phát hành mới nhất:

   **Intel**

   ```bash
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
   ```

   **Apple Silicon**

   ```bash
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl"
   ```

   > **Ghi chú:** Để tải một phiên bản cụ thể, hãy thay phần
   > `$(curl -L -s https://dl.k8s.io/release/stable.txt)` trong lệnh bằng phiên bản cụ thể đó.
   >
   > Ví dụ, để tải phiên bản 1.36.0 trên macOS Intel, gõ:
   >
   > ```bash
   > curl -LO "https://dl.k8s.io/release/v1.36.0/bin/darwin/amd64/kubectl"
   > ```
   >
   > Còn với macOS trên Apple Silicon, gõ:
   >
   > ```bash
   > curl -LO "https://dl.k8s.io/release/v1.36.0/bin/darwin/arm64/kubectl"
   > ```

2. Xác thực binary (tùy chọn)

   Tải file checksum của kubectl:

   **Intel**

   ```bash
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl.sha256"
   ```

   **Apple Silicon**

   ```bash
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl.sha256"
   ```

   Xác thực kubectl binary bằng file checksum:

   ```bash
   echo "$(cat kubectl.sha256)  kubectl" | shasum -a 256 --check
   ```

   Nếu hợp lệ, đầu ra là:

   ```console
   kubectl: OK
   ```

   Nếu kiểm tra thất bại, `shasum` thoát với trạng thái khác 0 và in ra kết quả tương tự:

   ```console
   kubectl: FAILED
   shasum: WARNING: 1 computed checksum did NOT match
   ```

   > **Ghi chú:** Hãy tải binary và checksum của cùng một phiên bản.

3. Cấp quyền thực thi cho kubectl binary.

   ```bash
   chmod +x ./kubectl
   ```

4. Di chuyển kubectl binary tới một vị trí nằm trong biến môi trường `PATH` của hệ thống.

   ```bash
   sudo mv ./kubectl /usr/local/bin/kubectl
   sudo chown root: /usr/local/bin/kubectl
   ```

   > **Ghi chú:** Hãy chắc chắn `/usr/local/bin` nằm trong biến môi trường PATH của bạn.

5. Kiểm tra để đảm bảo phiên bản bạn vừa cài là mới nhất:

   ```bash
   kubectl version --client
   ```

   Hoặc dùng lệnh sau để xem chi tiết về phiên bản:

   ```cmd
   kubectl version --client --output=yaml
   ```

6. Sau khi cài đặt và xác thực kubectl xong, xóa file checksum:

   ```bash
   rm kubectl.sha256
   ```

### Cài đặt bằng Homebrew trên macOS (Install with Homebrew on macOS) {#install-with-homebrew-on-macos}

Nếu bạn dùng macOS với trình quản lý gói [Homebrew](https://brew.sh/),
bạn có thể cài kubectl bằng Homebrew.

1. Chạy lệnh cài đặt:

   ```bash
   brew install kubectl
   ```

   hoặc

   ```bash
   brew install kubernetes-cli
   ```

2. Kiểm tra để đảm bảo phiên bản bạn vừa cài là mới nhất:

   ```bash
   kubectl version --client
   ```

### Cài đặt bằng Macports trên macOS (Install with Macports on macOS) {#install-with-macports-on-macos}

Nếu bạn dùng macOS với trình quản lý gói [Macports](https://macports.org/),
bạn có thể cài kubectl bằng Macports.

1. Chạy lệnh cài đặt:

   ```bash
   sudo port selfupdate
   sudo port install kubectl
   ```

2. Kiểm tra để đảm bảo phiên bản bạn vừa cài là mới nhất:

   ```bash
   kubectl version --client
   ```

## Kiểm tra cấu hình kubectl (Verify kubectl configuration) {#verify-kubectl-configuration}

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

## Cấu hình tùy chọn và plugin cho kubectl (Optional kubectl configurations and plugins) {#optional-kubectl-configurations-and-plugins}

### Bật tự động hoàn thành cho shell (Enable shell autocompletion) {#enable-shell-autocompletion}

kubectl cung cấp hỗ trợ tự động hoàn thành (autocompletion) cho Bash, Zsh, Fish và PowerShell,
giúp bạn tiết kiệm rất nhiều thao tác gõ.

Dưới đây là quy trình thiết lập tự động hoàn thành cho Bash, Fish và Zsh.

#### Bash

##### Giới thiệu (Introduction)

Script hoàn thành lệnh (completion script) của kubectl cho Bash có thể được sinh ra bằng
`kubectl completion bash`. Source script này trong shell của bạn sẽ bật tính năng tự động
hoàn thành của kubectl.

Tuy nhiên, script hoàn thành của kubectl phụ thuộc vào
[**bash-completion**](https://github.com/scop/bash-completion), do đó bạn phải cài đặt nó trước.

> **Cảnh báo:** Có hai phiên bản bash-completion là v1 và v2. V1 dành cho Bash 3.2
> (phiên bản mặc định trên macOS), còn v2 dành cho Bash 4.1+. Script hoàn thành của kubectl
> **không hoạt động** đúng với bash-completion v1 và Bash 3.2. Nó yêu cầu
> **bash-completion v2** và **Bash 4.1+**. Vì vậy, để dùng được kubectl completion đúng cách
> trên macOS, bạn phải cài và dùng Bash 4.1+
> ([*hướng dẫn*](https://apple.stackexchange.com/a/292760)). Các hướng dẫn dưới đây giả định
> bạn dùng Bash 4.1+ (tức là bất kỳ phiên bản Bash nào từ 4.1 trở lên).

##### Nâng cấp Bash (Upgrade Bash)

Các hướng dẫn ở đây giả định bạn dùng Bash 4.1+. Bạn có thể kiểm tra phiên bản Bash của mình
bằng cách chạy:

```bash
echo $BASH_VERSION
```

Nếu phiên bản quá cũ, bạn có thể cài đặt/nâng cấp bằng Homebrew:

```bash
brew install bash
```

Tải lại shell và xác minh phiên bản mong muốn đang được sử dụng:

```bash
echo $BASH_VERSION $SHELL
```

Homebrew thường cài nó tại `/usr/local/bin/bash`.

##### Cài đặt bash-completion (Install bash-completion)

> **Ghi chú:** Như đã đề cập, các hướng dẫn này giả định bạn dùng Bash 4.1+, nghĩa là bạn sẽ
> cài bash-completion v2 (khác với trường hợp Bash 3.2 và bash-completion v1, khi đó kubectl
> completion sẽ không hoạt động).

Bạn có thể kiểm tra xem bash-completion v2 đã được cài chưa bằng `type _init_completion`.
Nếu chưa, bạn có thể cài bằng Homebrew:

```bash
brew install bash-completion@2
```

Như thông báo trong đầu ra của lệnh này, hãy thêm dòng sau vào file `~/.bash_profile`:

```bash
brew_etc="$(brew --prefix)/etc" && [[ -r "${brew_etc}/profile.d/bash_completion.sh" ]] && . "${brew_etc}/profile.d/bash_completion.sh"
```

Tải lại shell và xác minh bash-completion v2 đã được cài đúng bằng `type _init_completion`.

##### Bật tự động hoàn thành cho kubectl (Enable kubectl autocompletion)

Giờ bạn phải đảm bảo script hoàn thành của kubectl được source trong mọi phiên shell của
bạn. Có nhiều cách để đạt được điều này:

- Source script hoàn thành trong file `~/.bash_profile`:

    ```bash
    echo 'source <(kubectl completion bash)' >>~/.bash_profile
    ```

- Thêm script hoàn thành vào thư mục `/usr/local/etc/bash_completion.d`:

    ```bash
    kubectl completion bash >/usr/local/etc/bash_completion.d/kubectl
    ```

- Nếu bạn có alias cho kubectl, bạn có thể mở rộng shell completion để hoạt động với alias đó:

    ```bash
    echo 'alias k=kubectl' >>~/.bash_profile
    echo 'complete -o default -F __start_kubectl k' >>~/.bash_profile
    ```

- Nếu bạn cài kubectl bằng Homebrew (như hướng dẫn
  [ở đây](#install-with-homebrew-on-macos)),
  thì script hoàn thành của kubectl hẳn đã nằm sẵn trong `/usr/local/etc/bash_completion.d/kubectl`.
  Trường hợp đó bạn không cần làm gì thêm.

   > **Ghi chú:** Bản cài bash-completion v2 bằng Homebrew source tất cả các file trong thư
   > mục `BASH_COMPLETION_COMPAT_DIR`, đó là lý do hai cách sau cùng hoạt động.

Dù theo cách nào, sau khi tải lại shell, kubectl completion sẽ hoạt động.

#### Fish

> **Ghi chú:** Tự động hoàn thành cho Fish yêu cầu kubectl 1.23 trở lên.

Script hoàn thành lệnh của kubectl cho Fish có thể được sinh ra bằng lệnh
`kubectl completion fish`. Source script hoàn thành này trong shell của bạn sẽ bật tính năng
tự động hoàn thành của kubectl.

Để áp dụng trong mọi phiên shell, thêm dòng sau vào file `~/.config/fish/config.fish`:

```shell
kubectl completion fish | source
```

Sau khi tải lại shell, tự động hoàn thành của kubectl sẽ hoạt động.

#### Zsh

Script hoàn thành lệnh của kubectl cho Zsh có thể được sinh ra bằng lệnh
`kubectl completion zsh`. Source script hoàn thành này trong shell của bạn sẽ bật tính năng
tự động hoàn thành của kubectl.

Để áp dụng trong mọi phiên shell, thêm dòng sau vào file `~/.zshrc`:

```zsh
source <(kubectl completion zsh)
```

Nếu bạn có alias cho kubectl, tự động hoàn thành của kubectl sẽ tự động hoạt động với alias đó.

Sau khi tải lại shell, tự động hoàn thành của kubectl sẽ hoạt động.

Nếu bạn gặp lỗi kiểu `2: command not found: compdef`, hãy thêm đoạn sau vào đầu file
`~/.zshrc`:

```zsh
autoload -Uz compinit
compinit
```

### Cấu hình kuberc (Configure kuberc)

Xem [kuberc](https://kubernetes.io/docs/reference/kubectl/kuberc) để biết thêm thông tin.

### Cài đặt plugin `kubectl convert` (Install `kubectl convert` plugin) {#install-kubectl-convert-plugin}

Đây là một plugin cho công cụ dòng lệnh `kubectl` của Kubernetes, cho phép bạn chuyển đổi
manifest giữa các phiên bản API khác nhau. Nó đặc biệt hữu ích khi cần di chuyển manifest sang
một phiên bản API không bị deprecated (khuyến cáo ngừng sử dụng) ở bản phát hành Kubernetes
mới hơn. Để biết thêm thông tin, xem
[migrate to non deprecated apis](https://kubernetes.io/docs/reference/using-api/deprecation-guide/#migrate-to-non-deprecated-apis)

1. Tải bản phát hành mới nhất bằng lệnh:

   **Intel**

   ```bash
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl-convert"
   ```

   **Apple Silicon**

   ```bash
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl-convert"
   ```

2. Xác thực binary (tùy chọn)

   Tải file checksum của kubectl-convert:

   **Intel**

   ```bash
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl-convert.sha256"
   ```

   **Apple Silicon**

   ```bash
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl-convert.sha256"
   ```

   Xác thực kubectl-convert binary bằng file checksum:

   ```bash
   echo "$(cat kubectl-convert.sha256)  kubectl-convert" | shasum -a 256 --check
   ```

   Nếu hợp lệ, đầu ra là:

   ```console
   kubectl-convert: OK
   ```

   Nếu kiểm tra thất bại, `shasum` thoát với trạng thái khác 0 và in ra kết quả tương tự:

   ```console
   kubectl-convert: FAILED
   shasum: WARNING: 1 computed checksum did NOT match
   ```

   > **Ghi chú:** Hãy tải binary và checksum của cùng một phiên bản.

3. Cấp quyền thực thi cho kubectl-convert binary

   ```bash
   chmod +x ./kubectl-convert
   ```

4. Di chuyển kubectl-convert binary tới một vị trí nằm trong biến môi trường `PATH` của hệ thống.

   ```bash
   sudo mv ./kubectl-convert /usr/local/bin/kubectl-convert
   sudo chown root: /usr/local/bin/kubectl-convert
   ```

   > **Ghi chú:** Hãy chắc chắn `/usr/local/bin` nằm trong biến môi trường PATH của bạn.

5. Xác minh plugin đã được cài thành công

   ```shell
   kubectl convert --help
   ```

   Nếu bạn không thấy lỗi nào, nghĩa là plugin đã được cài đặt thành công.

6. Sau khi cài plugin xong, dọn dẹp các file cài đặt:

   ```bash
   rm kubectl-convert kubectl-convert.sha256
   ```

### Gỡ cài đặt kubectl trên macOS (Uninstall kubectl on macOS)

Tùy theo cách bạn đã cài `kubectl`, dùng một trong các phương pháp sau.

### Gỡ cài đặt kubectl bằng dòng lệnh (Uninstall kubectl using the command-line)

1.  Xác định vị trí `kubectl` binary trên hệ thống của bạn:

    ```bash
    which kubectl
    ```

2.  Xóa `kubectl` binary:

    ```bash
    sudo rm <path>
    ```

    Thay `<path>` bằng đường dẫn tới `kubectl` binary từ bước trước. Ví dụ, `sudo rm /usr/local/bin/kubectl`.

### Gỡ cài đặt kubectl bằng Homebrew (Uninstall kubectl using homebrew)

Nếu bạn đã cài `kubectl` bằng Homebrew, chạy lệnh sau:

```bash
brew remove kubectl
```

## Tiếp theo (What's next)

* Tìm hiểu về [kubectl](26-kubectl-vi.md) và vai trò của nó trong hệ sinh thái Kubernetes.
* [Cài đặt Minikube](https://minikube.sigs.k8s.io/docs/start/)
* Xem [các hướng dẫn bắt đầu](https://kubernetes.io/docs/setup/) để biết thêm về cách tạo cluster.
* [Tìm hiểu cách khởi chạy và expose ứng dụng của bạn.](https://kubernetes.io/docs/tasks/access-application-cluster/service-access-application-cluster/)
* Nếu bạn cần truy cập một cluster mà bạn không tự tạo, xem
  [tài liệu Chia sẻ quyền truy cập cluster](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/).
* Đọc [tài liệu tham khảo kubectl](https://kubernetes.io/docs/reference/kubectl/kubectl/)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc này.

1. Máy Mac của bạn cài kubectl v1.36 (bản stable mới nhất), còn cluster lab của bạn chạy
   control plane v1.35. Tổ hợp này có được hỗ trợ không? Vì sao?
2. Bạn chạy `kubectl version --client` và thấy đúng phiên bản vừa cài. Điều đó có chứng minh
   kubectl đã truy cập được cluster của bạn chưa? Nếu chưa, còn thiếu gì và lệnh nào dùng để
   kiểm chứng?
3. Khi tải kubectl binary bằng curl, vì sao phải tải thêm file `kubectl.sha256` **cùng phiên
   bản** và chạy `shasum -a 256 --check`? Nếu kết quả là `FAILED` thì bạn làm gì?
4. Đồng nghiệp của bạn bật completion cho kubectl trên macOS với Bash mặc định của hệ điều
   hành nhưng không chạy. Nguyên nhân nhiều khả năng nhất là gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Được.** Quy tắc version skew cho phép client lệch tối đa một phiên bản minor so với
   control plane: client v1.36 làm việc được với control plane v1.35, v1.36 và v1.37.
2. **Chưa.** `kubectl version --client` chỉ chứng minh binary chạy được trên máy. Muốn truy
   cập cluster, kubectl còn cần **file kubeconfig** (mặc định `~/.kube/config`) trỏ đúng
   cluster; kiểm chứng bằng **`kubectl cluster-info`** — thấy phản hồi dạng URL là cấu hình
   đúng, thấy "connection refused" là chưa.
3. Để **xác minh tính toàn vẹn** của binary đã tải: checksum chỉ có ý nghĩa khi so với đúng
   phiên bản binary tương ứng, nên bài nhắc riêng "tải binary và checksum của cùng một phiên
   bản". Nếu `FAILED`, **không dùng binary đó** — tải lại đúng cặp binary + checksum rồi kiểm
   tra lại.
4. Bash mặc định của macOS là **3.2**, trong khi script completion của kubectl yêu cầu
   **Bash 4.1+ và bash-completion v2** — bài cảnh báo thẳng là nó *không hoạt động* với
   bash-completion v1 và Bash 3.2. Phải nâng cấp Bash (ví dụ qua Homebrew) rồi cài
   `bash-completion@2`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
