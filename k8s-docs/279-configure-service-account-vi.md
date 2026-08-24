# Cấu hình Service Account cho Pod (Configure Service Accounts for Pods)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/>

Kubernetes cung cấp hai cách riêng biệt để các client chạy bên trong cluster của bạn, hoặc có
quan hệ với control plane của cluster, xác thực (authenticate) với API server.

Một _service account_ cung cấp danh tính (identity) cho các tiến trình chạy trong Pod, và ánh xạ
tới một object ServiceAccount. Khi bạn xác thực với API server, bạn tự nhận diện mình là một
_user_ (người dùng) cụ thể. Kubernetes công nhận khái niệm user, tuy nhiên bản thân Kubernetes
**không** có API User.

Hướng dẫn tác vụ này nói về ServiceAccount — thứ thực sự tồn tại trong API của Kubernetes.
Hướng dẫn chỉ cho bạn một số cách cấu hình ServiceAccount cho Pod.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Sử dụng service account mặc định để truy cập API server (Use the default service account to access the API server) {#use-the-default-service-account-to-access-the-api-server}

Khi các Pod liên hệ với API server, Pod xác thực dưới danh nghĩa một ServiceAccount cụ thể (ví
dụ: `default`). Luôn có ít nhất một ServiceAccount trong mỗi namespace.

Mỗi namespace của Kubernetes chứa ít nhất một ServiceAccount: ServiceAccount mặc định của
namespace đó, có tên `default`. Nếu bạn không chỉ định ServiceAccount khi tạo Pod, Kubernetes
tự động gán ServiceAccount có tên `default` trong namespace đó.

Bạn có thể lấy chi tiết của một Pod bạn đã tạo. Ví dụ:

```shell
kubectl get pods/<podname> -o yaml
```

Trong kết quả, bạn thấy một field `spec.serviceAccountName`. Kubernetes tự động đặt giá trị đó
nếu bạn không chỉ định khi tạo Pod.

Một ứng dụng chạy bên trong Pod có thể truy cập API của Kubernetes bằng thông tin xác thực
(credentials) của service account được tự động mount. Xem
[truy cập Cluster](359-access-cluster-vi.md)
để tìm hiểu thêm.

Khi một Pod xác thực dưới danh nghĩa một ServiceAccount, mức độ truy cập của nó phụ thuộc vào
[plugin và chính sách phân quyền (authorization)](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#authorization-modules)
đang được sử dụng.

Thông tin xác thực API bị thu hồi tự động khi Pod bị xóa, ngay cả khi có finalizer. Cụ thể,
thông tin xác thực API bị thu hồi 60 giây sau thời điểm `.metadata.deletionTimestamp` được đặt
trên Pod (deletion timestamp thường là thời điểm yêu cầu **delete** được chấp nhận cộng với
thời gian ân hạn kết thúc (termination grace period) của Pod).

### Từ chối tự động mount thông tin xác thực API (Opt out of API credential automounting)

Nếu bạn không muốn kubelet tự động mount thông tin xác thực API của một ServiceAccount, bạn có
thể từ chối (opt out) hành vi mặc định này. Bạn có thể từ chối việc tự động mount thông tin xác
thực API tại `/var/run/secrets/kubernetes.io/serviceaccount/token` cho một service account bằng
cách đặt `automountServiceAccountToken: false` trên ServiceAccount:

Ví dụ:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
automountServiceAccountToken: false
...
```

Bạn cũng có thể từ chối tự động mount thông tin xác thực API cho một Pod cụ thể:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: build-robot
  automountServiceAccountToken: false
  ...
```

Nếu cả ServiceAccount lẫn `.spec` của Pod đều chỉ định giá trị cho
`automountServiceAccountToken`, thì spec của Pod được ưu tiên.

## Sử dụng nhiều hơn một ServiceAccount {#use-multiple-service-accounts}

Mỗi namespace có ít nhất một ServiceAccount: tài nguyên ServiceAccount mặc định, tên là
`default`. Bạn có thể liệt kê tất cả các tài nguyên ServiceAccount trong
[namespace hiện tại](19-namespaces-vi.md#setting-the-namespace-preference)
của mình bằng:

```shell
kubectl get serviceaccounts
```

Kết quả tương tự như sau:

```
NAME      SECRETS    AGE
default   1          1d
```

Bạn có thể tạo thêm các object ServiceAccount như sau:

```shell
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
EOF
```

Tên của một object ServiceAccount phải là một
[tên miền con DNS (DNS subdomain name)](17-names-vi.md#dns-subdomain-names)
hợp lệ.

Nếu bạn lấy toàn bộ nội dung (dump) của object service account, như sau:

```shell
kubectl get serviceaccounts/build-robot -o yaml
```

Kết quả tương tự như sau:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2019-06-16T00:12:34Z
  name: build-robot
  namespace: default
  resourceVersion: "272500"
  uid: 721ab723-13bc-11e5-aec2-42010af0021e
```

Bạn có thể dùng các plugin phân quyền để
[đặt quyền cho service account](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#service-account-permissions).

Để dùng một service account không phải mặc định, hãy đặt field `spec.serviceAccountName` của
Pod thành tên của ServiceAccount mà bạn muốn dùng.

Bạn chỉ có thể đặt field `serviceAccountName` khi tạo Pod, hoặc trong template của một Pod mới.
Bạn không thể cập nhật field `.spec.serviceAccountName` của một Pod đã tồn tại.

> **Ghi chú:**
>
> Field `.spec.serviceAccount` là bí danh (alias) đã lỗi thời (deprecated) của
> `.spec.serviceAccountName`. Nếu bạn muốn loại bỏ các field này khỏi một tài nguyên workload,
> hãy đặt cả hai field thành rỗng một cách tường minh trên
> [pod template](https://kubernetes.io/docs/concepts/workloads/pods#pod-templates).

### Dọn dẹp {#cleanup-use-multiple-service-accounts}

Nếu bạn đã thử tạo ServiceAccount `build-robot` từ ví dụ trên, bạn có thể dọn dẹp bằng cách
chạy:

```shell
kubectl delete serviceaccount/build-robot
```

## Tạo API token thủ công cho một ServiceAccount (Manually create an API token for a ServiceAccount) {#manually-create-an-api-token-for-a-serviceaccount}

Giả sử bạn có một service account tên "build-robot" như đã nhắc tới ở trên.

Bạn có thể lấy một API token có thời hạn cho ServiceAccount đó bằng `kubectl`:

```shell
kubectl create token build-robot
```

Kết quả của lệnh đó là một token mà bạn có thể dùng để xác thực dưới danh nghĩa ServiceAccount
đó. Bạn có thể yêu cầu thời hạn cụ thể cho token bằng đối số dòng lệnh `--duration` của
`kubectl create token` (thời hạn thực tế của token được cấp có thể ngắn hơn, hoặc thậm chí dài
hơn).

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.33 [stable]`

Với `kubectl` v1.31 trở lên, bạn có thể tạo một service account token được gắn (bound) trực
tiếp với một Node:

```shell
kubectl create token build-robot --bound-object-kind Node --bound-object-name node-001 --bound-object-uid 123...456
```

Token sẽ hợp lệ cho đến khi nó hết hạn hoặc Node hay service account liên kết bị xóa.

> **Ghi chú:**
>
> Các phiên bản Kubernetes trước v1.22 tự động tạo thông tin xác thực dài hạn để truy cập API
> của Kubernetes. Cơ chế cũ này dựa trên việc tạo các token Secret mà sau đó có thể được mount
> vào các Pod đang chạy. Trong các phiên bản gần đây hơn, bao gồm Kubernetes v1.36, thông tin
> xác thực API được lấy trực tiếp thông qua API
> [TokenRequest](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-request-v1/),
> và được mount vào Pod bằng một
> [projected volume](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#bound-service-account-token-volume).
> Các token lấy bằng phương thức này có thời hạn giới hạn, và tự động bị vô hiệu hóa khi Pod mà
> chúng được mount vào bị xóa.
>
> Bạn vẫn có thể tạo thủ công một service account token Secret; ví dụ, nếu bạn cần một token
> không bao giờ hết hạn. Tuy nhiên, cách được khuyến nghị là sử dụng subresource
> [TokenRequest](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-request-v1/)
> để lấy token truy cập API.

### Tạo thủ công một API token dài hạn cho ServiceAccount (Manually create a long-lived API token for a ServiceAccount)

Nếu bạn muốn lấy một API token cho một ServiceAccount, bạn tạo một Secret mới với một
annotation đặc biệt, `kubernetes.io/service-account.name`.

```shell
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: build-robot-secret
  annotations:
    kubernetes.io/service-account.name: build-robot
type: kubernetes.io/service-account-token
EOF
```

Nếu bạn xem Secret bằng:

```shell
kubectl get secret/build-robot-secret -o yaml
```

bạn có thể thấy Secret giờ đây chứa một API token cho ServiceAccount "build-robot".

Nhờ annotation bạn đã đặt, control plane tự động sinh token cho ServiceAccount đó và lưu vào
Secret liên kết. Control plane cũng dọn dẹp các token của những ServiceAccount đã bị xóa.

```shell
kubectl describe secrets/build-robot-secret
```

Kết quả tương tự như sau:

```
Name:           build-robot-secret
Namespace:      default
Labels:         <none>
Annotations:    kubernetes.io/service-account.name: build-robot
                kubernetes.io/service-account.uid: da68f9c6-9d26-11e7-b84e-002dc52800da

Type:   kubernetes.io/service-account-token

Data
====
ca.crt:         1338 bytes
namespace:      7 bytes
token:          ...
```

> **Ghi chú:**
>
> Nội dung của `token` được lược bỏ ở đây.
>
> Hãy cẩn thận, đừng hiển thị nội dung của một Secret loại `kubernetes.io/service-account-token`
> ở nơi mà terminal / màn hình máy tính của bạn có thể bị người khác nhìn thấy.

Khi bạn xóa một ServiceAccount có Secret liên kết, control plane của Kubernetes tự động dọn dẹp
token dài hạn khỏi Secret đó.

> **Ghi chú:**
>
> Nếu bạn xem ServiceAccount bằng:
>
> ` kubectl get serviceaccount build-robot -o yaml`
>
> Bạn không thể thấy Secret `build-robot-secret` trong field
> [`.secrets`](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/service-account-v1/)
> của object ServiceAccount trong API, vì field đó chỉ được điền với các Secret được sinh tự
> động.

## Thêm ImagePullSecrets vào một service account (Add ImagePullSecrets to a service account) {#add-imagepullsecrets-to-a-service-account}

Trước tiên, hãy
[tạo một imagePullSecret](40-images-vi.md#specifying-imagepullsecrets-on-a-pod).
Tiếp theo, xác nhận rằng nó đã được tạo. Ví dụ:

- Tạo một imagePullSecret, như mô tả trong
  [Chỉ định ImagePullSecrets trên Pod](40-images-vi.md#specifying-imagepullsecrets-on-a-pod).

  ```shell
  kubectl create secret docker-registry myregistrykey --docker-server=<registry name> \
          --docker-username=DUMMY_USERNAME --docker-password=DUMMY_DOCKER_PASSWORD \
          --docker-email=DUMMY_DOCKER_EMAIL
  ```

- Xác nhận rằng nó đã được tạo.

  ```shell
  kubectl get secrets myregistrykey
  ```

  Kết quả tương tự như sau:

  ```
  NAME             TYPE                              DATA    AGE
  myregistrykey    kubernetes.io/.dockerconfigjson   1       1d
  ```

### Thêm image pull secret vào service account (Add image pull secret to service account)

Tiếp theo, sửa service account mặc định của namespace để dùng Secret này làm imagePullSecret.

```shell
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "myregistrykey"}]}'
```

Bạn có thể đạt kết quả tương tự bằng cách chỉnh sửa object thủ công:

```shell
kubectl edit serviceaccount/default
```

Kết quả của file `sa.yaml` tương tự như sau:

Trình soạn thảo văn bản bạn đã chọn sẽ mở ra với cấu hình trông giống như sau:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2021-07-07T22:02:39Z
  name: default
  namespace: default
  resourceVersion: "243024"
  uid: 052fb0f4-3d50-11e5-b066-42010af0d7b6
```

Trong trình soạn thảo, hãy xóa dòng có khóa `resourceVersion`, thêm các dòng cho
`imagePullSecrets:` rồi lưu lại. Giữ nguyên giá trị `uid` như bạn thấy ban đầu.

Sau khi thực hiện các thay đổi đó, ServiceAccount đã chỉnh sửa trông giống như sau:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2021-07-07T22:02:39Z
  name: default
  namespace: default
  uid: 052fb0f4-3d50-11e5-b066-42010af0d7b6
imagePullSecrets:
  - name: myregistrykey
```

### Xác nhận imagePullSecrets được đặt cho Pod mới (Verify that imagePullSecrets are set for new Pods)

Bây giờ, khi một Pod mới được tạo trong namespace hiện tại và sử dụng ServiceAccount mặc định,
Pod mới sẽ có field `spec.imagePullSecrets` được đặt tự động:

```shell
kubectl run nginx --image=<registry name>/nginx --restart=Never
kubectl get pod nginx -o=jsonpath='{.spec.imagePullSecrets[0].name}{"\n"}'
```

Kết quả là:

```
myregistrykey
```

## Chiếu token của ServiceAccount vào volume (ServiceAccount token volume projection) {#serviceaccount-token-volume-projection}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.20 [stable]`

> **Ghi chú:**
>
> Để bật và sử dụng tính năng chiếu token theo yêu cầu (token request projection), bạn phải chỉ
> định từng đối số dòng lệnh sau cho `kube-apiserver`:
>
> `--service-account-issuer`
> : định nghĩa định danh (Identifier) của bên phát hành (issuer) service account token. Bạn có
>   thể chỉ định đối số `--service-account-issuer` nhiều lần, điều này hữu ích khi muốn thay đổi
>   issuer mà không gây gián đoạn. Khi flag này được chỉ định nhiều lần, giá trị đầu tiên được
>   dùng để sinh token và tất cả các giá trị được dùng để xác định những issuer nào được chấp
>   nhận. Bạn phải chạy Kubernetes v1.22 trở lên để có thể chỉ định `--service-account-issuer`
>   nhiều lần.
>
> `--service-account-key-file`
> : chỉ định đường dẫn tới file chứa các khóa X.509 riêng tư (private) hoặc công khai (public)
>   được mã hóa PEM (RSA hoặc ECDSA), dùng để xác minh các ServiceAccount token. File được chỉ
>   định có thể chứa nhiều khóa, và flag này có thể được chỉ định nhiều lần với các file khác
>   nhau. Nếu được chỉ định nhiều lần, token được ký bởi bất kỳ khóa nào trong số đó đều được
>   API server của Kubernetes coi là hợp lệ.
>
> `--service-account-signing-key-file`
> : chỉ định đường dẫn tới file chứa khóa riêng tư hiện tại của bên phát hành service account
>   token. Issuer ký các ID token được phát hành bằng khóa riêng tư này.
>
> `--api-audiences` (có thể bỏ qua)
> : định nghĩa các audience cho ServiceAccount token. Bộ xác thực service account token kiểm tra
>   rằng các token dùng với API được gắn với ít nhất một trong các audience này. Nếu
>   `api-audiences` được chỉ định nhiều lần, token cho bất kỳ audience nào trong số đó đều được
>   API server của Kubernetes coi là hợp lệ. Nếu bạn chỉ định đối số dòng lệnh
>   `--service-account-issuer` nhưng không đặt `--api-audiences`, control plane mặc định dùng
>   danh sách audience một phần tử chỉ chứa URL của issuer.

Kubelet cũng có thể chiếu một ServiceAccount token vào Pod. Bạn có thể chỉ định các thuộc tính
mong muốn của token, chẳng hạn như audience và thời hạn hợp lệ. Các thuộc tính này _không_ thể
cấu hình được trên ServiceAccount token mặc định. Token cũng sẽ trở nên không hợp lệ đối với API
khi Pod hoặc ServiceAccount bị xóa.

Bạn có thể cấu hình hành vi này cho `spec` của một Pod bằng loại
[projected volume](91-volumes-vi.md#projected) có tên
`ServiceAccountToken`.

Token từ projected volume này là một JSON Web Token (JWT). Phần payload JSON của token này tuân
theo một schema được định nghĩa rõ ràng — ví dụ payload cho một token gắn với Pod:

```yaml
{
  "aud": [  # khớp với các audience được yêu cầu, hoặc các audience mặc định của API server khi không có audience nào được yêu cầu tường minh
    "https://kubernetes.default.svc"
  ],
  "exp": 1731613413,
  "iat": 1700077413,
  "iss": "https://kubernetes.default.svc",  # khớp với giá trị đầu tiên truyền cho flag --service-account-issuer
  "jti": "ea28ed49-2e11-4280-9ec5-bc3d1d84661a", 
  "kubernetes.io": {
    "namespace": "kube-system",
    "node": {
      "name": "127.0.0.1",
      "uid": "58456cb0-dd00-45ed-b797-5578fdceaced"
    },
    "pod": {
      "name": "coredns-69cbfb9798-jv9gn",
      "uid": "778a530c-b3f4-47c0-9cd5-ab018fb64f33"
    },
    "serviceaccount": {
      "name": "coredns",
      "uid": "a087d5a0-e1dd-43ec-93ac-f13d89cd13af"
    },
    "warnafter": 1700081020
  },
  "nbf": 1700077413,
  "sub": "system:serviceaccount:kube-system:coredns"
}
```

### Khởi chạy Pod dùng chiếu service account token (Launch a Pod using service account token projection) {#launch-a-pod-using-service-account-token-projection}

Để cung cấp cho một Pod một token có audience là `vault` và thời hạn hợp lệ hai giờ, bạn có thể
định nghĩa một manifest Pod tương tự như:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    volumeMounts:
    - mountPath: /var/run/secrets/tokens
      name: vault-token
  serviceAccountName: build-robot
  volumes:
  - name: vault-token
    projected:
      sources:
      - serviceAccountToken:
          path: vault-token
          expirationSeconds: 7200
          audience: vault
```

Tạo Pod:

```shell
kubectl create -f https://k8s.io/examples/pods/pod-projected-svc-token.yaml
```

Kubelet sẽ: yêu cầu và lưu trữ token thay cho Pod; làm cho token khả dụng đối với Pod tại một
đường dẫn file có thể cấu hình; và làm mới (refresh) token khi nó gần hết hạn. Kubelet chủ động
yêu cầu xoay vòng (rotation) token nếu token cũ hơn 80% tổng thời gian sống (TTL) của nó, hoặc
nếu token cũ hơn 24 giờ.

Ứng dụng chịu trách nhiệm nạp lại token khi token được xoay vòng. Thông thường, việc ứng dụng
nạp token theo lịch định kỳ (ví dụ: 5 phút một lần) mà không cần theo dõi thời điểm hết hạn thực
tế là đủ tốt.

### Khám phá issuer của service account (Service account issuer discovery)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.21 [stable]`

Nếu bạn đã bật [chiếu token](#chiếu-token-của-serviceaccount-vào-volume-serviceaccount-token-volume-projection)
cho các ServiceAccount trong cluster, bạn cũng có thể tận dụng tính năng khám phá (discovery).
Kubernetes cung cấp một cách để các client liên kết (federate) như một _nhà cung cấp danh tính_
(identity provider), sao cho một hoặc nhiều hệ thống bên ngoài có thể đóng vai trò _bên tin cậy_
(relying party).

> **Ghi chú:**
>
> URL của issuer phải tuân thủ
> [OIDC Discovery Spec](https://openid.net/specs/openid-connect-discovery-1_0.html). Trên thực
> tế, điều này nghĩa là nó phải dùng scheme `https`, và nên phục vụ cấu hình OpenID provider tại
> `{service-account-issuer}/.well-known/openid-configuration`.
>
> Nếu URL không tuân thủ, các endpoint khám phá issuer của ServiceAccount sẽ không được đăng ký
> hoặc không truy cập được.

Khi được bật, API server của Kubernetes công bố một tài liệu OpenID Provider Configuration qua
HTTP. Tài liệu cấu hình này được công bố tại `/.well-known/openid-configuration`. OpenID
Provider Configuration đôi khi được gọi là _discovery document_ (tài liệu khám phá). API server
của Kubernetes cũng công bố JSON Web Key Set (JWKS) liên quan, cũng qua HTTP, tại
`/openid/v1/jwks`.

> **Ghi chú:**
>
> Các phản hồi được phục vụ tại `/.well-known/openid-configuration` và `/openid/v1/jwks` được
> thiết kế để tương thích OIDC, nhưng không tuân thủ OIDC một cách chặt chẽ. Các tài liệu đó chỉ
> chứa những tham số cần thiết để thực hiện việc xác minh các service account token của
> Kubernetes.

Các cluster sử dụng RBAC có sẵn một ClusterRole mặc định tên là
`system:service-account-issuer-discovery`. Một ClusterRoleBinding mặc định gán role này cho
nhóm `system:serviceaccounts`, nhóm mà tất cả các ServiceAccount ngầm định thuộc về. Điều này
cho phép các Pod chạy trên cluster truy cập tài liệu khám phá service account thông qua service
account token đã được mount của chúng. Ngoài ra, quản trị viên có thể chọn gắn (bind) role này
cho `system:authenticated` hoặc `system:unauthenticated` tùy theo yêu cầu bảo mật và các hệ
thống bên ngoài mà họ dự định liên kết.

Phản hồi JWKS chứa các khóa công khai mà một bên tin cậy có thể dùng để xác minh các service
account token của Kubernetes. Bên tin cậy trước tiên truy vấn OpenID Provider Configuration, và
dùng field `jwks_uri` trong phản hồi để tìm JWKS.

Trong nhiều trường hợp, API server của Kubernetes không khả dụng trên Internet công cộng, nhưng
người dùng hoặc nhà cung cấp dịch vụ có thể tạo các endpoint công cộng phục vụ phản hồi được
cache từ API server. Trong những trường hợp này, có thể ghi đè `jwks_uri` trong OpenID Provider
Configuration để nó trỏ tới endpoint công cộng thay vì địa chỉ của API server, bằng cách truyền
flag `--service-account-jwks-uri` cho API server. Giống như URL của issuer, JWKS URI bắt buộc
phải dùng scheme `https`.

## Tiếp theo (What's next)

Xem thêm:

- Đọc [Hướng dẫn quản trị cluster về Service Account](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)
- Đọc về [Phân quyền trong Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
- Đọc về [Secrets](109-secret-vi.md)
  - hoặc học cách [phân phối thông tin xác thực an toàn bằng Secrets](334-distribute-credentials-secure-vi.md)
  - nhưng cũng cần lưu ý rằng việc dùng Secret để xác thực dưới danh nghĩa ServiceAccount đã lỗi
    thời. Cách thay thế được khuyến nghị là
    [chiếu ServiceAccount token vào volume](#chiếu-token-của-serviceaccount-vào-volume-serviceaccount-token-volume-projection).
- Đọc về [projected volumes](277-configure-projected-volume-vi.md).
- Để tìm hiểu bối cảnh về OIDC discovery, đọc Kubernetes Enhancement Proposal
  [ServiceAccount signing key retrieval](https://github.com/kubernetes/enhancements/tree/master/keps/sig-auth/1393-oidc-discovery)
- Đọc [OIDC Discovery Spec](https://openid.net/specs/openid-connect-discovery-1_0.html)
