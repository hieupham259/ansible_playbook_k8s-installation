# Khắc phục sự cố kubectl (Troubleshooting kubectl)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/debug-cluster/troubleshoot-kubectl/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Checkpoint tiếp nối, CP9 — Xử lý sự cố](LO-TRINH-ADMIN.md#cp9--xử-lý-sự-cố),
bài 4/10 · nối tiếp bài [308 — Debug node bằng Kubectl](308-kubectl-node-debug-vi.md);
giai đoạn này không có lab riêng, thực hành trực tiếp trên cluster lab
(xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài này giải quyết tình huống ngược với các bài trước của CP9: không phải cluster hỏng, mà
là **bạn không nói chuyện được với cluster**. Giá trị của bài nằm ở trình tự loại trừ, không
ở lệnh nào phức tạp.

**Phải hiểu ở lần đọc này:**

- Trình tự chẩn đoán khi `kubectl` không kết nối được: kiểm tra cài đặt và version → kiểm
  tra kubeconfig → kiểm tra context → mạng/VPN → API server và bộ cân bằng tải (load
  balancer) → TLS certificate → helper xác thực.
- `kubectl version` là phép thử đầu tiên: nếu thấy
  `Unable to connect to the server: dial tcp <server-ip>:8443: i/o timeout` thay cho
  `Server Version` thì vấn đề là **kết nối tới cluster**, không phải bản cài kubectl.
- kubeconfig: mặc định ở `~/.kube/config`; có thể chép từ `/etc/kubernetes/admin.conf` trên
  control plane; chỉ định file khác bằng biến môi trường `$KUBECONFIG` hoặc flag
  `--kubeconfig`.
- Context: `kubectl config get-contexts` để liệt kê và `kubectl config use-context` để
  chuyển — đứng nhầm context là nguyên nhân rất phổ biến khi làm việc với nhiều cluster.
- Cách kiểm tra hạn certificate ngay từ kubeconfig: `kubectl config view --flatten` +
  jsonpath lấy `certificate-authority-data` / `client-certificate-data`, giải mã `base64 -d`
  rồi đọc bằng `openssl x509 -noout -dates`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Quản lý nhiều cluster và context trong một kubeconfig | cluster lab chỉ có một cluster, một context | bài *Configure Access to Multiple Clusters* của nhánh `/docs/tasks/` (số 361 trong hàng đợi dịch) |
| Helper xác thực như `kubectl-oidc-login` | cluster lab xác thực bằng client certificate trong `admin.conf`, chưa dùng OIDC | nhóm bài xác thực ở [Giai đoạn 9 — Bảo mật](LO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), khi cluster của bạn nối với IdP ngoài |

---

Tài liệu này nói về việc điều tra và chẩn đoán các vấn đề liên quan đến kubectl. Nếu bạn gặp
sự cố khi truy cập `kubectl` hoặc khi kết nối tới cluster của mình, tài liệu này trình bày
các kịch bản phổ biến cùng những giải pháp khả dĩ để giúp xác định và xử lý nguyên nhân có
khả năng nhất.

## Trước khi bạn bắt đầu (Before you begin)

* Bạn cần có một cluster Kubernetes.
* Bạn cũng cần cài đặt sẵn `kubectl` — xem
  [cài đặt công cụ](https://kubernetes.io/docs/tasks/tools/#kubectl).

## Xác minh thiết lập kubectl (Verify kubectl setup)

Hãy chắc chắn rằng bạn đã cài đặt và cấu hình `kubectl` đúng trên máy cục bộ của mình. Kiểm
tra phiên bản `kubectl` để đảm bảo nó đủ mới và tương thích với cluster của bạn.

Kiểm tra phiên bản kubectl:

```shell
kubectl version
```

Bạn sẽ thấy output tương tự:

```console
Client Version: version.Info{Major:"1", Minor:"27", GitVersion:"v1.27.4",GitCommit:"fa3d7990104d7c1f16943a67f11b154b71f6a132", GitTreeState:"clean",BuildDate:"2023-07-19T12:20:54Z", GoVersion:"go1.20.6", Compiler:"gc", Platform:"linux/amd64"}
Kustomize Version: v5.0.1
Server Version: version.Info{Major:"1", Minor:"27", GitVersion:"v1.27.3",GitCommit:"25b4e43193bcda6c7328a6d147b1fb73a33f1598", GitTreeState:"clean",BuildDate:"2023-06-14T09:47:40Z", GoVersion:"go1.20.5", Compiler:"gc", Platform:"linux/amd64"}

```

Nếu bạn thấy `Unable to connect to the server: dial tcp <server-ip>:8443: i/o timeout` thay
cho `Server Version`, bạn cần khắc phục sự cố kết nối giữa kubectl và cluster của mình.

Hãy chắc chắn rằng bạn đã cài kubectl theo
[tài liệu chính thức về cài đặt kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl),
và đã cấu hình đúng biến môi trường `$PATH`.

## Kiểm tra kubeconfig (Check kubeconfig)

`kubectl` cần một file `kubeconfig` để kết nối tới cluster Kubernetes. File `kubeconfig`
thường nằm ở đường dẫn `~/.kube/config`. Hãy chắc chắn rằng bạn có một file `kubeconfig`
hợp lệ. Nếu bạn không có file `kubeconfig`, bạn có thể xin nó từ admin Kubernetes của mình,
hoặc chép nó từ đường dẫn `/etc/kubernetes/admin.conf` trên control plane của Kubernetes.
Nếu bạn triển khai cluster Kubernetes trên một nền tảng cloud và làm mất file `kubeconfig`,
bạn có thể tạo lại nó bằng công cụ của nhà cung cấp cloud. Tham khảo tài liệu của nhà cung
cấp cloud về cách tạo lại file `kubeconfig`.

Kiểm tra xem biến môi trường `$KUBECONFIG` đã được cấu hình đúng chưa. Bạn có thể đặt biến
môi trường `$KUBECONFIG` hoặc dùng tham số `--kubeconfig` với kubectl để chỉ định vị trí của
file `kubeconfig`.

## Kiểm tra kết nối VPN (Check VPN connectivity)

Nếu bạn dùng mạng riêng ảo (Virtual Private Network — VPN) để truy cập cluster Kubernetes,
hãy chắc chắn rằng kết nối VPN của bạn đang hoạt động và ổn định. Đôi khi, việc VPN bị ngắt
có thể dẫn đến sự cố kết nối với cluster. Kết nối lại VPN và thử truy cập cluster lần nữa.

## Xác thực và phân quyền (Authentication and authorization)

Nếu bạn dùng phương thức xác thực dựa trên token và kubectl trả về lỗi liên quan đến token
xác thực hoặc địa chỉ máy chủ xác thực, hãy xác nhận rằng token xác thực Kubernetes và địa
chỉ máy chủ xác thực đã được cấu hình đúng.

Nếu kubectl trả về lỗi liên quan đến phân quyền (authorization), hãy chắc chắn rằng bạn đang
dùng thông tin đăng nhập (credential) hợp lệ, và bạn có quyền truy cập tài nguyên mà bạn đã
yêu cầu.

## Xác minh context (Verify contexts)

Kubernetes hỗ trợ
[nhiều cluster và context](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/).
Hãy chắc chắn rằng bạn đang dùng đúng context để tương tác với cluster của mình.

Liệt kê các context hiện có:

```shell
kubectl config get-contexts
```

Chuyển sang context phù hợp:

```shell
kubectl config use-context <context-name>
```

## API server và bộ cân bằng tải (API server and load balancer)

kube-apiserver là thành phần trung tâm của một cluster Kubernetes. Nếu API server hoặc bộ
cân bằng tải (load balancer) đứng trước các API server không thể truy cập được hoặc không
phản hồi, bạn sẽ không thể tương tác với cluster.

Kiểm tra xem host của API server có truy cập được không bằng lệnh `ping`. Kiểm tra kết nối
mạng và firewall của cluster. Nếu bạn dùng một nhà cung cấp cloud để triển khai cluster, hãy
kiểm tra trạng thái health check của nhà cung cấp cloud đối với API server của cluster.

Xác minh trạng thái của bộ cân bằng tải (nếu có dùng) để đảm bảo nó khỏe mạnh và đang chuyển
tiếp traffic tới API server.

## Các vấn đề TLS (TLS problems)

* Cần công cụ bổ sung — `base64` và `openssl` phiên bản 3.0 trở lên.

Theo mặc định, Kubernetes API server chỉ phục vụ các request HTTPS. Khi đó, các vấn đề TLS
có thể xảy ra vì nhiều lý do khác nhau, chẳng hạn certificate hết hạn hoặc chuỗi tin cậy
(chain of trust) không còn hợp lệ.

Bạn có thể tìm thấy TLS certificate trong file kubeconfig, nằm ở đường dẫn `~/.kube/config`.
Thuộc tính `certificate-authority` chứa CA certificate và thuộc tính `client-certificate`
chứa client certificate.

Xác minh hạn của các certificate này:

```shell
kubectl config view --flatten --output 'jsonpath={.clusters[0].cluster.certificate-authority-data}' | base64 -d | openssl x509 -noout -dates
```

output:
```console
notBefore=Feb 13 05:57:47 2024 GMT
notAfter=Feb 10 06:02:47 2034 GMT
```

```shell
kubectl config view --flatten --output 'jsonpath={.users[0].user.client-certificate-data}'| base64 -d | openssl x509 -noout -dates
```

output:
```console
notBefore=Feb 13 05:57:47 2024 GMT
notAfter=Feb 12 06:02:50 2025 GMT
```

## Xác minh các helper của kubectl (Verify kubectl helpers)

Một số helper xác thực của kubectl giúp truy cập cluster Kubernetes một cách thuận tiện. Nếu
bạn từng dùng những helper như vậy và đang gặp sự cố kết nối, hãy chắc chắn rằng các cấu
hình cần thiết vẫn còn nguyên.

Kiểm tra cấu hình kubectl để xem chi tiết xác thực:

```shell
kubectl config view
```

Nếu trước đây bạn dùng một công cụ helper (ví dụ `kubectl-oidc-login`), hãy chắc chắn rằng
nó vẫn được cài đặt và cấu hình đúng.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở CP9:

1. Trên máy trạm, `kubectl get nodes` treo rồi báo
   `Unable to connect to the server: dial tcp <server-ip>:8443: i/o timeout`. Lệnh nào bạn
   chạy đầu tiên để phân định "kubectl cài hỏng" với "không kết nối được cluster", và kết
   quả nào của lệnh đó cho phép loại trừ vế thứ nhất?
2. Máy trạm mới chưa có file `~/.kube/config`. Với cluster lab dựng bằng kubeadm, bạn lấy
   kubeconfig từ đâu, và có những cách nào để trỏ kubectl tới một file kubeconfig không nằm
   ở vị trí mặc định?
3. **Câu bẫy.** Đồng nghiệp than "kubectl kết nối được nhưng toàn thấy tài nguyên lạ, không
   phải cluster của mình". Kubeconfig của họ hợp lệ và API server sống. Khả năng cao vấn đề
   nằm ở đâu và hai lệnh nào để chẩn đoán rồi sửa?
4. Nghi ngờ client certificate trong kubeconfig đã hết hạn, viết (đại ý) pipeline lệnh để in
   ra `notBefore`/`notAfter` của nó mà không cần mở file bằng tay. Cần công cụ bổ sung nào?
5. Nếu tất cả các bước phía client đều sạch (version, kubeconfig, context, VPN), bài hướng
   dẫn kiểm tra tiếp những gì ở phía hạ tầng?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Chạy **`kubectl version`**. Nếu lệnh in được `Client Version` (và `Kustomize Version`)
   thì bản cài kubectl và `$PATH` không phải vấn đề — lỗi `i/o timeout` thay cho
   `Server Version` cho biết vấn đề nằm ở **đường kết nối tới cluster**: kubeconfig, mạng,
   VPN, hoặc API server.
2. Chép từ **`/etc/kubernetes/admin.conf` trên control plane** (node `k8s-master` của
   cluster lab), hoặc xin từ admin cluster; trên cloud thì tạo lại bằng công cụ của nhà cung
   cấp. Trỏ kubectl tới file khác bằng **biến môi trường `$KUBECONFIG`** hoặc **flag
   `--kubeconfig`**.
3. Họ đang **đứng nhầm context** — kubeconfig có thể chứa nhiều cluster/context và kubectl
   làm việc với context hiện hành, nên "kết nối được" không có nghĩa "đúng cluster". Chẩn
   đoán bằng **`kubectl config get-contexts`**, sửa bằng
   **`kubectl config use-context <context-name>`**. Trực giác "kết nối OK tức cấu hình OK"
   sai vì mọi thứ đều hợp lệ, chỉ trỏ sai đích.
4. `kubectl config view --flatten --output 'jsonpath={.users[0].user.client-certificate-data}' | base64 -d | openssl x509 -noout -dates` —
   lấy trường `client-certificate-data` từ kubeconfig, giải mã base64, rồi đọc hạn bằng
   openssl. Cần **`base64` và `openssl` phiên bản 3.0 trở lên**. (Với CA certificate thì
   thay bằng `{.clusters[0].cluster.certificate-authority-data}`.)
5. Kiểm tra **API server và bộ cân bằng tải**: host của API server có `ping` được không,
   mạng và firewall của cluster, health check của nhà cung cấp cloud (nếu có), và trạng thái
   của load balancer đứng trước API server; sau đó là **TLS** (hạn certificate, chuỗi tin
   cậy) và các **helper xác thực** còn cài và cấu hình đúng hay không.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau của
[CP9](LO-TRINH-ADMIN.md#cp9--xử-lý-sự-cố).
