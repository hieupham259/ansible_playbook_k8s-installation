# Cấu hình truy cập nhiều cluster (Configure Access to Multiple Clusters)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/>
>
> Trang này hướng dẫn cách cấu hình truy cập nhiều cluster bằng các file cấu hình
> (kubeconfig), và cách chuyển đổi nhanh giữa các cluster bằng lệnh
> `kubectl config use-context`.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 1 — Mô hình Kubernetes](00-ALO-TRINH-ADMIN.md#giai-đoạn-1--mô-hình-kubernetes)
→ nhóm [1b. Làm việc với object và kubectl](00-ALO-TRINH-ADMIN.md#1b-làm-việc-với-object-và-kubectl),
bài 7/8 của dòng **Thực hành** · Kiểm chứng ở
[Lab 1b — Object, label, kubectl và kubeconfig](labs/LAB-1B-OBJECT-LABEL-KUBECTL-VA-KUBECONFIG.md)
phần B4, nơi lab tạo một context mới trên **bản sao** kubeconfig và chứng minh context thật của
`lab-k8s-master` không đổi.

Đây là bản thực hành của bài [111 — Tổ chức quyền truy cập cluster bằng file kubeconfig](111-kubeconfig-vi.md).
Bạn chỉ có một cluster lab, nên hai cluster `development` và `test` trong bài là **giả**: các
`server`, `fake-ca-file`, `fake-cert-file` không trỏ tới đâu cả. Điều đó không sao — mọi bước
đều chạy được, vì bài chỉ thao tác trên file cấu hình chứ không gọi tới API server. Làm theo
trên file tạm trong thư mục `config-exercise`, đúng như Lab 1b làm, và **không sửa
`$HOME/.kube/config`**.

**Phải hiểu ở lần đọc này:**

- Một context là một **bộ ba (cluster, user, namespace)**, và `kubectl config use-context` đổi cả
  ba cùng lúc. Bài nói thẳng ý nghĩa của `dev-frontend`: "dùng thông tin đăng nhập của user
  `developer` để truy cập namespace `frontend` của cluster `development`".
- Sửa kubeconfig bằng lệnh chứ không bằng tay: `kubectl config --kubeconfig=<file>` với
  `set-cluster`, `set-credentials`, `set-context`, `use-context`, `view`; xóa bằng
  `config unset users.<name>`, `clusters.<name>`, `contexts.<name>`. Cờ `--kubeconfig` quyết định
  file nào bị sửa, nên đây là cách làm mà không đụng tới file cấu hình đang dùng thật.
- `kubectl config view` in toàn bộ; thêm `--minify` thì **chỉ in phần gắn với context hiện tại**
  — mục *Định nghĩa cluster, user và context* dùng nó để chứng minh mỗi lần `use-context` là một
  bộ ba khác.
- Biến môi trường `KUBECONFIG` là một **danh sách đường dẫn** (ngăn bằng `:` trên Linux/Mac, `;`
  trên Windows), và `kubectl config view` in ra bản **hợp nhất** của mọi file trong danh sách —
  đó là lý do context `dev-ramp-up` của `config-demo-2` xuất hiện cạnh ba context của
  `config-demo`. Nhớ lưu giá trị cũ (`KUBECONFIG_SAVED`) và khôi phục ở mục *Dọn dẹp*.
- Hai điều về an toàn và danh tính: cảnh báo đầu bài nói **chỉ dùng kubeconfig từ nguồn đáng tin
  cậy**, vì một file được chế tác có thể dẫn tới thực thi mã độc hoặc lộ file — hãy soi nó như
  soi một shell script. Và khi không chắc mình đang là ai trên cluster nào, `kubectl auth whoami`
  trả lời đúng câu đó (mục *Kiểm tra chủ thể (subject) mà kubeconfig đại diện*).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| `set-credentials --username --password` cho cluster `test`, và ghi chú về client-go credential plugin | ở đây chỉ cần thấy user là một mục trong kubeconfig; cơ chế xác thực thật chưa học | [Giai đoạn 9 — Bảo mật và multi-tenancy](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy) |
| Hậu tố `-data` (`certificate-authority-data`, `client-certificate-data`, `client-key-data`) | chỉ là cách nhúng nội dung certificate thay cho đường dẫn file | [Giai đoạn 18 — Vòng đời chứng chỉ](00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ) |
| Quy tắc chi tiết khi hợp nhất nhiều file kubeconfig | ở đây chỉ cần thấy kết quả hợp nhất | bài [111 — kubeconfig](111-kubeconfig-vi.md) của chính nhóm 1b |
| Các biến thể lệnh cho Windows PowerShell | mọi lệnh của lộ trình chạy trên VM Linux `lab-k8s-master` | ngoài phạm vi lộ trình — đọc khi máy trạm của bạn là Windows |

---

Trang này hướng dẫn cách cấu hình truy cập nhiều cluster bằng cách sử dụng
các file cấu hình. Sau khi các cluster, user và context của bạn đã được định nghĩa
trong một hoặc nhiều file cấu hình, bạn có thể chuyển đổi nhanh giữa các cluster bằng
lệnh `kubectl config use-context`.

> **Ghi chú:**
>
> File được dùng để cấu hình truy cập vào một cluster đôi khi được gọi là
> *file kubeconfig*. Đây là cách gọi chung cho các file cấu hình.
> Điều đó không có nghĩa là tồn tại một file có tên `kubeconfig`.

> **Cảnh báo:**
>
> Chỉ sử dụng các file kubeconfig từ những nguồn đáng tin cậy. Việc sử dụng một file
> kubeconfig được chế tác đặc biệt có thể dẫn đến thực thi mã độc hoặc lộ file.
> Nếu bạn buộc phải dùng một file kubeconfig không đáng tin cậy, hãy kiểm tra nó cẩn thận
> trước, giống như cách bạn kiểm tra một shell script.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình
để giao tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất
hai node không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo
một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
hoặc sử dụng một trong các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra kubectl đã được cài đặt hay chưa, hãy chạy `kubectl version --client`.
Phiên bản kubectl nên
[chênh lệch không quá một phiên bản minor](https://kubernetes.io/releases/version-skew-policy/#kubectl)
so với API server của cluster.

## Định nghĩa cluster, user và context (Define clusters, users, and contexts)

Giả sử bạn có hai cluster, một cluster dành cho công việc phát triển (development) và một
cluster dành cho công việc kiểm thử (test). Trong cluster `development`, các lập trình viên
frontend làm việc trong một namespace tên là `frontend`, còn các lập trình viên storage làm
việc trong một namespace tên là `storage`. Trong cluster `test`, các lập trình viên làm việc
trong namespace default, hoặc họ tự tạo các namespace phụ trợ khi thấy cần. Truy cập vào
cluster development yêu cầu xác thực (authentication) bằng certificate. Truy cập vào cluster
test yêu cầu xác thực bằng username và password.

Tạo một thư mục tên là `config-exercise`. Trong thư mục
`config-exercise`, tạo một file tên là `config-demo` với nội dung sau:

```yaml
apiVersion: v1
kind: Config
preferences: {}

clusters:
- cluster:
  name: development
- cluster:
  name: test

users:
- name: developer
- name: experimenter

contexts:
- context:
  name: dev-frontend
- context:
  name: dev-storage
- context:
  name: exp-test
```

Một file cấu hình mô tả các cluster, user và context. File `config-demo` của bạn
có bộ khung để mô tả hai cluster, hai user và ba context.

Đi tới thư mục `config-exercise`. Nhập các lệnh sau để thêm thông tin chi tiết về cluster
vào file cấu hình của bạn:

```shell
kubectl config --kubeconfig=config-demo set-cluster development --server=https://1.2.3.4 --certificate-authority=fake-ca-file
kubectl config --kubeconfig=config-demo set-cluster test --server=https://5.6.7.8 --insecure-skip-tls-verify
```

Thêm thông tin chi tiết về user vào file cấu hình của bạn:

> **Thận trọng:**
>
> Lưu trữ password trong cấu hình client của Kubernetes là rủi ro. Một lựa chọn tốt hơn là
> dùng một credential plugin và lưu trữ chúng riêng biệt. Xem:
> [client-go credential plugins](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#client-go-credential-plugins)

```shell
kubectl config --kubeconfig=config-demo set-credentials developer --client-certificate=fake-cert-file --client-key=fake-key-seefile
kubectl config --kubeconfig=config-demo set-credentials experimenter --username=exp --password=some-password
```

> **Ghi chú:**
>
> - Để xóa một user, bạn có thể chạy `kubectl --kubeconfig=config-demo config unset users.<name>`
> - Để xóa một cluster, bạn có thể chạy `kubectl --kubeconfig=config-demo config unset clusters.<name>`
> - Để xóa một context, bạn có thể chạy `kubectl --kubeconfig=config-demo config unset contexts.<name>`

Thêm thông tin chi tiết về context vào file cấu hình của bạn:

```shell
kubectl config --kubeconfig=config-demo set-context dev-frontend --cluster=development --namespace=frontend --user=developer
kubectl config --kubeconfig=config-demo set-context dev-storage --cluster=development --namespace=storage --user=developer
kubectl config --kubeconfig=config-demo set-context exp-test --cluster=test --namespace=default --user=experimenter
```

Mở file `config-demo` của bạn để xem các thông tin đã được thêm vào. Thay vì mở file
`config-demo`, bạn cũng có thể dùng lệnh `config view`.

```shell
kubectl config --kubeconfig=config-demo view
```

Kết quả hiển thị hai cluster, hai user và ba context:

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority: fake-ca-file
    server: https://1.2.3.4
  name: development
- cluster:
    insecure-skip-tls-verify: true
    server: https://5.6.7.8
  name: test
contexts:
- context:
    cluster: development
    namespace: frontend
    user: developer
  name: dev-frontend
- context:
    cluster: development
    namespace: storage
    user: developer
  name: dev-storage
- context:
    cluster: test
    namespace: default
    user: experimenter
  name: exp-test
current-context: ""
kind: Config
preferences: {}
users:
- name: developer
  user:
    client-certificate: fake-cert-file
    client-key: fake-key-file
- name: experimenter
  user:
    # Ghi chú của tài liệu (comment này KHÔNG phải là một phần của output lệnh).
    # Lưu trữ password trong cấu hình client của Kubernetes là rủi ro.
    # Một lựa chọn tốt hơn là dùng một credential plugin
    # và lưu trữ thông tin đăng nhập (credentials) riêng biệt.
    # Xem https://kubernetes.io/docs/reference/access-authn-authz/authentication/#client-go-credential-plugins
    password: some-password
    username: exp
```

Các giá trị `fake-ca-file`, `fake-cert-file` và `fake-key-file` ở trên là các placeholder
cho đường dẫn (pathname) của các file certificate. Bạn cần thay chúng bằng đường dẫn thực tế
của các file certificate trong môi trường của bạn.

Đôi khi bạn có thể muốn dùng dữ liệu mã hóa Base64 nhúng trực tiếp tại đây thay vì các file
certificate riêng biệt; trong trường hợp đó bạn cần thêm hậu tố `-data` vào các key, ví dụ,
`certificate-authority-data`, `client-certificate-data`, `client-key-data`.

Mỗi context là một bộ ba (cluster, user, namespace). Ví dụ, context
`dev-frontend` nói rằng, "Dùng thông tin đăng nhập của user `developer`
để truy cập namespace `frontend` của cluster `development`".

Thiết lập context hiện tại:

```shell
kubectl config --kubeconfig=config-demo use-context dev-frontend
```

Từ giờ, mỗi khi bạn nhập một lệnh `kubectl`, hành động sẽ áp dụng cho cluster
và namespace được liệt kê trong context `dev-frontend`. Và lệnh đó sẽ dùng
thông tin đăng nhập của user được liệt kê trong context `dev-frontend`.

Để chỉ xem thông tin cấu hình gắn với context hiện tại, hãy dùng flag `--minify`.

```shell
kubectl config --kubeconfig=config-demo view --minify
```

Kết quả hiển thị thông tin cấu hình gắn với context `dev-frontend`:

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority: fake-ca-file
    server: https://1.2.3.4
  name: development
contexts:
- context:
    cluster: development
    namespace: frontend
    user: developer
  name: dev-frontend
current-context: dev-frontend
kind: Config
preferences: {}
users:
- name: developer
  user:
    client-certificate: fake-cert-file
    client-key: fake-key-file
```

Bây giờ giả sử bạn muốn làm việc một lúc trong cluster test.

Đổi context hiện tại sang `exp-test`:

```shell
kubectl config --kubeconfig=config-demo use-context exp-test
```

Bây giờ, bất kỳ lệnh `kubectl` nào bạn đưa ra sẽ áp dụng cho namespace default của
cluster `test`. Và lệnh đó sẽ dùng thông tin đăng nhập của user được liệt kê
trong context `exp-test`.

Xem cấu hình gắn với context hiện tại mới, `exp-test`.

```shell
kubectl config --kubeconfig=config-demo view --minify
```

Cuối cùng, giả sử bạn muốn làm việc một lúc trong namespace `storage` của
cluster `development`.

Đổi context hiện tại sang `dev-storage`:

```shell
kubectl config --kubeconfig=config-demo use-context dev-storage
```

Xem cấu hình gắn với context hiện tại mới, `dev-storage`.

```shell
kubectl config --kubeconfig=config-demo view --minify
```

## Tạo file cấu hình thứ hai (Create a second configuration file)

Trong thư mục `config-exercise`, tạo một file tên là `config-demo-2` với nội dung sau:

```yaml
apiVersion: v1
kind: Config
preferences: {}

contexts:
- context:
    cluster: development
    namespace: ramp
    user: developer
  name: dev-ramp-up
```

File cấu hình trên định nghĩa một context mới có tên `dev-ramp-up`.

## Thiết lập biến môi trường KUBECONFIG (Set the KUBECONFIG environment variable) {#set-the-kubeconfig-environment-variable}

Kiểm tra xem bạn có biến môi trường tên là `KUBECONFIG` hay không. Nếu có, hãy lưu lại
giá trị hiện tại của biến môi trường `KUBECONFIG`, để bạn có thể khôi phục nó sau này.
Ví dụ:

### Linux

```shell
export KUBECONFIG_SAVED="$KUBECONFIG"
```

### Windows PowerShell

```powershell
$Env:KUBECONFIG_SAVED=$ENV:KUBECONFIG
```

Biến môi trường `KUBECONFIG` là một danh sách các đường dẫn tới các file cấu hình.
Danh sách này phân tách bằng dấu hai chấm trên Linux và Mac, và phân tách bằng dấu
chấm phẩy trên Windows. Nếu bạn có biến môi trường `KUBECONFIG`, hãy làm quen với
các file cấu hình trong danh sách đó.

Tạm thời nối thêm hai đường dẫn vào biến môi trường `KUBECONFIG` của bạn. Ví dụ:

### Linux

```shell
export KUBECONFIG="${KUBECONFIG}:config-demo:config-demo-2"
```

### Windows PowerShell

```powershell
$Env:KUBECONFIG=("config-demo;config-demo-2")
```

Trong thư mục `config-exercise`, nhập lệnh sau:

```shell
kubectl config view
```

Kết quả hiển thị thông tin đã được hợp nhất (merge) từ tất cả các file được liệt kê trong
biến môi trường `KUBECONFIG` của bạn. Đặc biệt, hãy chú ý rằng thông tin hợp nhất có
context `dev-ramp-up` từ file `config-demo-2` và ba context từ
file `config-demo`:

```yaml
contexts:
- context:
    cluster: development
    namespace: frontend
    user: developer
  name: dev-frontend
- context:
    cluster: development
    namespace: ramp
    user: developer
  name: dev-ramp-up
- context:
    cluster: development
    namespace: storage
    user: developer
  name: dev-storage
- context:
    cluster: test
    namespace: default
    user: experimenter
  name: exp-test
```

Để biết thêm thông tin về cách các file kubeconfig được hợp nhất, xem
[Tổ chức quyền truy cập cluster bằng file kubeconfig](111-kubeconfig-vi.md)
(đã có [bản dịch tiếng Việt](111-kubeconfig-vi.md)).

## Khám phá thư mục $HOME/.kube (Explore the $HOME/.kube directory)

Nếu bạn đã có sẵn một cluster, và bạn có thể dùng `kubectl` để tương tác với
cluster đó, thì nhiều khả năng bạn có một file tên là `config` trong thư mục
`$HOME/.kube`.

Đi tới `$HOME/.kube` và xem trong đó có những file nào. Thông thường sẽ có một file tên là
`config`. Cũng có thể có các file cấu hình khác trong thư mục này. Hãy làm quen sơ qua với
nội dung của các file đó.

## Nối $HOME/.kube/config vào biến môi trường KUBECONFIG (Append $HOME/.kube/config to your KUBECONFIG environment variable)

Nếu bạn có file `$HOME/.kube/config`, và nó chưa được liệt kê trong biến môi trường
`KUBECONFIG` của bạn, hãy nối nó vào biến môi trường `KUBECONFIG` ngay bây giờ.
Ví dụ:

### Linux

```shell
export KUBECONFIG="${KUBECONFIG}:${HOME}/.kube/config"
```

### Windows Powershell

```powershell
$Env:KUBECONFIG="$Env:KUBECONFIG;$HOME\.kube\config"
```

Xem thông tin cấu hình đã hợp nhất từ tất cả các file hiện được liệt kê
trong biến môi trường `KUBECONFIG` của bạn. Trong thư mục config-exercise, nhập:

```shell
kubectl config view
```

## Dọn dẹp (Clean up)

Đưa biến môi trường `KUBECONFIG` của bạn trở về giá trị ban đầu. Ví dụ:

### Linux

```shell
export KUBECONFIG="$KUBECONFIG_SAVED"
```

### Windows PowerShell

```powershell
$Env:KUBECONFIG=$ENV:KUBECONFIG_SAVED
```

## Kiểm tra chủ thể (subject) mà kubeconfig đại diện (Check the subject represented by the kubeconfig)

Không phải lúc nào cũng dễ thấy rõ bạn sẽ nhận được những thuộc tính nào (username, groups)
sau khi xác thực với cluster. Việc này thậm chí còn khó hơn nếu bạn đang quản lý nhiều hơn
một cluster cùng lúc.

Có một lệnh con của `kubectl` để kiểm tra các thuộc tính chủ thể, chẳng hạn như username,
cho context Kubernetes client mà bạn đã chọn: `kubectl auth whoami`.

Đọc [Truy cập API để lấy thông tin xác thực của một client](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#self-subject-review)
để tìm hiểu chi tiết hơn về điều này.

## Tiếp theo (What's next)

* [Tổ chức quyền truy cập cluster bằng file kubeconfig](111-kubeconfig-vi.md) — đã có [bản dịch tiếng Việt](111-kubeconfig-vi.md)
* [kubectl config](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#config)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở nhóm 1b:

1. Bạn chạy `kubectl config --kubeconfig=config-demo use-context dev-storage`. Ba thứ nào vừa
   đổi cùng lúc, và lệnh `kubectl get pods` chạy ngay sau đó sẽ hỏi cluster nào, namespace nào,
   với danh tính nào?
2. **Câu bẫy.** Trong suốt bài, `$HOME/.kube/config` của bạn có bị sửa không? Cụ thể, cái gì
   khiến các lệnh `kubectl config ...` của bài không đụng tới file đó — và mục nào của bài mới
   thực sự kéo file đó vào cuộc?
3. Bạn đặt `KUBECONFIG="config-demo:config-demo-2"` rồi chạy `kubectl config view`. Kết quả có
   bốn context trong khi mỗi file chỉ định nghĩa một phần trong số đó. Cơ chế nào tạo ra kết
   quả đó, và vì sao bài bắt bạn lưu `KUBECONFIG_SAVED` trước khi làm?
4. Trên `lab-k8s-master`, bạn nhận một file kubeconfig từ đồng nghiệp và muốn biết mình sẽ là ai
   khi dùng nó. Lệnh nào của bài trả lời được, và bài cảnh báo gì trước khi bạn dùng một file
   kubeconfig lạ?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Đổi cùng lúc **cluster, user và namespace** — vì một context chính là bộ ba đó. Với
   `dev-storage`, lệnh `kubectl get pods` sẽ hỏi cluster **`development`**, namespace
   **`storage`**, bằng thông tin đăng nhập của user **`developer`**. Đây là lý do bài mô tả
   context bằng một câu tiếng Việt gọn: dùng credential của user X để truy cập namespace Y của
   cluster Z.
2. **Không bị sửa.** Nguyên nhân là **mọi lệnh trong bài đều có cờ `--kubeconfig=config-demo`**,
   nên chúng chỉ ghi vào file trong thư mục `config-exercise`. Chỗ dễ nhầm là nghĩ
   `kubectl config` luôn tác động lên file cấu hình mặc định. Chỉ có hai mục cuối kéo
   `$HOME/.kube/config` vào cuộc, và cũng chỉ **đọc**: mục *Khám phá thư mục $HOME/.kube* xem
   trong đó có gì, và mục *Nối $HOME/.kube/config vào biến môi trường KUBECONFIG* thêm đường dẫn
   của nó vào danh sách để `kubectl config view` hợp nhất — chứ không ghi đè nó.
3. Vì `KUBECONFIG` là một **danh sách đường dẫn**, và `kubectl config view` in ra bản **hợp
   nhất** của tất cả các file trong danh sách. Ba context của `config-demo` cộng với
   `dev-ramp-up` của `config-demo-2` thành bốn. Bài bắt lưu `KUBECONFIG_SAVED` vì bước này
   **ghi đè giá trị đang có** của biến môi trường; mục *Dọn dẹp* trả nó về nguyên trạng bằng
   `export KUBECONFIG="$KUBECONFIG_SAVED"`.
4. Lệnh là **`kubectl auth whoami`**: nó cho biết các thuộc tính chủ thể — username, groups — mà
   bạn nhận được sau khi xác thực với context đang chọn; bài nói rõ điều này khó thấy khi bạn
   quản lý nhiều hơn một cluster. Cảnh báo đi kèm: **chỉ dùng kubeconfig từ nguồn đáng tin cậy**,
   vì một file được chế tác đặc biệt có thể dẫn tới **thực thi mã độc hoặc lộ file**; buộc phải
   dùng file lạ thì kiểm tra nó cẩn thận trước, giống như kiểm tra một shell script.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
