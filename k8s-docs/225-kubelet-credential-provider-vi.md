# Cấu hình một kubelet image credential provider (Configure a kubelet image credential provider)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/kubelet-credential-provider/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.26 [stable]`

Bắt đầu từ Kubernetes v1.20, kubelet có thể lấy thông tin xác thực (credential) cho một
container image registry một cách động bằng các exec plugin. Kubelet và exec plugin giao tiếp
với nhau qua stdio (stdin, stdout và stderr) bằng các API có phiên bản của Kubernetes. Các
plugin này cho phép kubelet yêu cầu credential cho một container registry một cách động thay vì
lưu trữ credential tĩnh trên đĩa. Ví dụ, plugin có thể liên lạc với một metadata server cục bộ
để lấy credential có thời hạn ngắn cho image mà kubelet đang kéo (pull).

Bạn có thể quan tâm tới việc sử dụng khả năng này nếu bất kỳ điều nào dưới đây là đúng:

* Cần gọi API tới dịch vụ của một nhà cung cấp cloud để lấy thông tin xác thực cho một registry.
* Credential có thời gian hết hạn ngắn và cần thường xuyên yêu cầu credential mới.
* Việc lưu trữ credential của registry trên đĩa hoặc trong imagePullSecrets là không chấp nhận
  được.

Hướng dẫn này trình bày cách cấu hình cơ chế plugin image credential provider của kubelet.

## Service account token cho việc pull image (Service Account Token for Image Pulls)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.34 [beta]` (được bật mặc định: true)

Bắt đầu từ Kubernetes v1.33, kubelet có thể được cấu hình để gửi tới credential provider plugin
một service account token được gắn (bound) với Pod mà việc pull image đang được thực hiện cho
nó.

Điều này cho phép plugin đổi token đó lấy credential để truy cập image registry.

Để bật tính năng này, feature gate `KubeletServiceAccountTokenForCredentialProviders` phải được
bật trên kubelet, và trường `tokenAttributes` phải được đặt trong file `CredentialProviderConfig`
cho plugin đó.

Trường `tokenAttributes` chứa thông tin về service account token sẽ được truyền cho plugin, bao
gồm audience dự kiến của token và việc plugin có yêu cầu Pod phải có service account hay không.

Sử dụng credential dạng service account token có thể phục vụ các trường hợp sử dụng sau:

* Tránh việc phải dùng danh tính (identity) dựa trên kubelet/node để pull image từ một registry.
* Cho phép các workload pull image dựa trên danh tính runtime của chính chúng mà không cần các
  secret tồn tại lâu dài/được lưu trữ bền vững.

## Trước khi bạn bắt đầu (Before you begin)

* Bạn cần một cluster Kubernetes với các node hỗ trợ kubelet credential provider plugin. Hỗ trợ
  này có sẵn trong Kubernetes v1.36; Kubernetes v1.24 và v1.25 đưa vào khả năng này ở dạng tính
  năng beta, được bật mặc định.
* Nếu bạn cấu hình một credential provider plugin cần service account token, bạn cần một cluster
  Kubernetes với các node chạy Kubernetes v1.33 trở lên và feature gate
  `KubeletServiceAccountTokenForCredentialProviders` được bật trên kubelet.
* Một bản triển khai (implementation) hoạt động được của credential provider exec plugin. Bạn có
  thể tự xây dựng plugin của riêng mình hoặc dùng plugin do các nhà cung cấp cloud cung cấp.

Máy chủ Kubernetes của bạn phải ở phiên bản bằng hoặc mới hơn v1.26. Để kiểm tra phiên bản,
hãy chạy `kubectl version`.

## Cài đặt plugin trên các node (Installing Plugins on Nodes)

Một credential provider plugin là một file thực thi (executable binary) sẽ được kubelet chạy.
Hãy đảm bảo rằng file thực thi của plugin tồn tại trên mọi node trong cluster của bạn và được
lưu trong một thư mục xác định. Thư mục này sẽ cần tới sau này khi cấu hình các cờ (flag) của
kubelet.

## Cấu hình kubelet (Configuring the Kubelet)

Để sử dụng tính năng này, kubelet cần được đặt hai cờ:

* `--image-credential-provider-config` - đường dẫn tới file cấu hình của credential provider
  plugin.
* `--image-credential-provider-bin-dir` - đường dẫn tới thư mục chứa các file thực thi của
  credential provider plugin.

### Cấu hình một kubelet credential provider (Configure a kubelet credential provider)

File cấu hình được truyền vào `--image-credential-provider-config` được kubelet đọc để xác định
exec plugin nào nên được gọi cho những container image nào. Dưới đây là một file cấu hình ví dụ
mà bạn có thể dùng nếu bạn sử dụng
[plugin dựa trên ECR](https://github.com/kubernetes/cloud-provider-aws/tree/master/cmd/ecr-credential-provider):

```yaml
apiVersion: kubelet.config.k8s.io/v1
kind: CredentialProviderConfig
# providers là danh sách các credential provider helper plugin sẽ được kubelet bật.
# Nhiều provider có thể cùng khớp với một image, trong trường hợp đó credential
# từ tất cả các provider sẽ được trả về cho kubelet. Nếu nhiều provider được gọi
# cho một image, các kết quả sẽ được gộp lại. Nếu các provider trả về các
# auth key trùng nhau, giá trị từ provider đứng trước trong danh sách này sẽ được dùng.
providers:
  # name là tên bắt buộc của credential provider. Nó phải khớp với tên của
  # file thực thi provider mà kubelet nhìn thấy. File thực thi phải nằm trong
  # thư mục bin của kubelet (đặt bằng cờ --image-credential-provider-bin-dir).
  - name: ecr-credential-provider
    # matchImages là danh sách chuỗi bắt buộc, dùng để so khớp với các image nhằm
    # xác định xem provider này có nên được gọi hay không. Nếu một trong các chuỗi
    # khớp với image mà kubelet yêu cầu, plugin sẽ được gọi và có cơ hội
    # cung cấp credential. Image được kỳ vọng chứa domain của registry
    # và đường dẫn URL.
    #
    # Mỗi mục trong matchImages là một pattern, có thể tùy chọn chứa port và path.
    # Glob có thể được dùng trong phần domain, nhưng không được dùng trong port hoặc path.
    # Glob được hỗ trợ dưới dạng subdomain như '*.k8s.io' hay 'k8s.*.io', và top-level-domain
    # như 'k8s.*'. So khớp subdomain một phần như 'app*.k8s.io' cũng được hỗ trợ. Mỗi glob
    # chỉ có thể khớp một phân đoạn subdomain duy nhất, nên `*.io` KHÔNG khớp `*.k8s.io`.
    #
    # Một image và một matchImage được coi là khớp khi tất cả các điều dưới đây đều đúng:
    # - Cả hai chứa cùng số lượng phần domain và từng phần đều khớp.
    # - Đường dẫn URL của matchImages phải là tiền tố (prefix) của đường dẫn URL image đích.
    # - Nếu matchImages chứa port, thì port đó cũng phải khớp trong image.
    #
    # Các giá trị ví dụ của matchImages:
    # - 123456789.dkr.ecr.us-east-1.amazonaws.com
    # - *.azurecr.io
    # - gcr.io
    # - *.*.registry.io
    # - registry.io:8080/path
    matchImages:
      - "*.dkr.ecr.*.amazonaws.com"
      - "*.dkr.ecr.*.amazonaws.com.cn"
      - "*.dkr.ecr-fips.*.amazonaws.com"
      - "*.dkr.ecr.us-iso-east-1.c2s.ic.gov"
      - "*.dkr.ecr.us-isob-east-1.sc2s.sgov.gov"
    # defaultCacheDuration là khoảng thời gian mặc định mà plugin sẽ cache credential trong
    # bộ nhớ nếu thời gian cache không được cung cấp trong phản hồi của plugin. Trường này là bắt buộc.
    defaultCacheDuration: "12h"
    # Phiên bản đầu vào bắt buộc của exec CredentialProviderRequest. CredentialProviderResponse
    # trả về BẮT BUỘC dùng cùng phiên bản mã hóa với đầu vào. Các giá trị hiện được hỗ trợ:
    # - credentialprovider.kubelet.k8s.io/v1
    apiVersion: credentialprovider.kubelet.k8s.io/v1
    # Các đối số truyền cho lệnh khi thực thi nó.
    # +optional
    # args:
    #   - --example-argument
    # Env định nghĩa các biến môi trường bổ sung cần cung cấp cho tiến trình. Chúng
    # được hợp nhất với môi trường của host, cũng như các biến mà client-go dùng
    # để truyền đối số cho plugin.
    # +optional
    env:
      - name: AWS_PROFILE
        value: example_profile

    # tokenAttributes là cấu hình cho service account token sẽ được truyền cho plugin.
    # Credential provider chọn tham gia (opt in) việc dùng service account token cho pull image
    # bằng cách đặt trường này.
    # Nếu trường này được đặt mà feature gate `KubeletServiceAccountTokenForCredentialProviders`
    # không được bật, kubelet sẽ không khởi động được với lỗi cấu hình không hợp lệ.
    # +optional
    tokenAttributes:
      # serviceAccountTokenAudience là audience dự kiến của projected service account token.
      # +required
      serviceAccountTokenAudience: "<audience for the token>"
      # cacheType cho biết loại khóa cache dùng để cache credential mà plugin trả về
      # khi service account token được sử dụng.
      # Lựa chọn thận trọng nhất là đặt giá trị này thành "Token", nghĩa là kubelet sẽ cache
      # credential trả về theo từng token. Nên đặt như vậy nếu thời hạn của credential
      # trả về bị giới hạn theo thời hạn của service account token.
      # Nếu logic lấy credential của plugin chỉ phụ thuộc vào service account chứ không
      # phụ thuộc vào các claim riêng của pod, thì plugin có thể đặt giá trị này thành
      # "ServiceAccount". Trong trường hợp đó, kubelet sẽ cache credential trả về theo từng
      # service account. Dùng lựa chọn này khi credential trả về hợp lệ cho mọi pod
      # dùng cùng một service account.
      # +required
      cacheType: "<Token or ServiceAccount>"
      # requireServiceAccount cho biết plugin có yêu cầu pod phải có service account hay không.
      # Nếu đặt là true, kubelet sẽ chỉ gọi plugin nếu pod có service account.
      # Nếu đặt là false, kubelet sẽ gọi plugin ngay cả khi pod không có service account
      # và sẽ không kèm token trong CredentialProviderRequest. Điều này hữu ích cho các plugin
      # dùng để pull image cho các pod không có service account (ví dụ: static pod).
      # +required
      requireServiceAccount: true
      # requiredServiceAccountAnnotationKeys là danh sách các khóa annotation mà plugin quan tâm
      # và bắt buộc phải có trong service account.
      # Các khóa định nghĩa trong danh sách này sẽ được trích xuất từ service account tương ứng
      # và truyền cho plugin như một phần của CredentialProviderRequest. Nếu bất kỳ khóa nào
      # trong danh sách này không có trong service account, kubelet sẽ không gọi plugin và sẽ
      # trả về lỗi.
      # Trường này là tùy chọn và có thể để trống. Plugin có thể dùng trường này để trích xuất
      # thông tin bổ sung cần thiết cho việc lấy credential, hoặc cho phép workload chọn tham gia
      # việc dùng service account token cho pull image.
      # Nếu không rỗng, requireServiceAccount phải được đặt là true.
      # Các khóa định nghĩa trong danh sách này phải là duy nhất và không trùng với các khóa
      # định nghĩa trong danh sách optionalServiceAccountAnnotationKeys.
      # +optional
      requiredServiceAccountAnnotationKeys:
      - "example.com/required-annotation-key-1"
      - "example.com/required-annotation-key-2"
      # optionalServiceAccountAnnotationKeys là danh sách các khóa annotation mà plugin quan tâm
      # và không bắt buộc phải có trong service account.
      # Các khóa định nghĩa trong danh sách này sẽ được trích xuất từ service account tương ứng
      # và truyền cho plugin như một phần của CredentialProviderRequest. Plugin chịu trách nhiệm
      # kiểm tra sự tồn tại của các annotation và giá trị của chúng. Trường này là tùy chọn và
      # có thể để trống.
      # Plugin có thể dùng trường này để trích xuất thông tin bổ sung cần thiết cho việc lấy
      # credential.
      # Các khóa định nghĩa trong danh sách này phải là duy nhất và không trùng với các khóa
      # định nghĩa trong danh sách requiredServiceAccountAnnotationKeys.
      # +optional
      optionalServiceAccountAnnotationKeys:
      - "example.com/optional-annotation-key-1"
      - "example.com/optional-annotation-key-2"
```

Trường `providers` là danh sách các plugin được bật mà kubelet sử dụng. Mỗi mục có một vài
trường bắt buộc:

* `name`: tên của plugin, BẮT BUỘC phải khớp với tên của file thực thi tồn tại trong thư mục
  được truyền vào `--image-credential-provider-bin-dir`.
* `matchImages`: danh sách chuỗi dùng để so khớp với các image nhằm xác định xem provider này
  có nên được gọi hay không. Chi tiết ở phần dưới.
* `defaultCacheDuration`: khoảng thời gian mặc định mà kubelet sẽ cache credential trong bộ nhớ
  nếu plugin không chỉ định thời gian cache.
* `apiVersion`: phiên bản API mà kubelet và exec plugin sẽ dùng khi giao tiếp.

Mỗi credential provider cũng có thể được cung cấp thêm các đối số (args) và biến môi trường tùy
chọn. Hãy tham khảo bên triển khai plugin để xác định tập đối số và biến môi trường cần thiết
cho một plugin cụ thể.

Nếu bạn đang dùng feature gate KubeletServiceAccountTokenForCredentialProviders và cấu hình
plugin dùng service account token bằng cách đặt trường tokenAttributes, các trường sau là bắt
buộc:

* `serviceAccountTokenAudience`: audience dự kiến của projected service account token. Giá trị
  này không được là chuỗi rỗng. Khi
  [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
  `ServiceAccountNodeAudienceRestriction` được bật, kubelet phải được ủy quyền (authorized) để
  yêu cầu token cho audience này; nếu không, credential provider sẽ không được gọi. Bạn phải cấp
  cho nhóm `system:nodes` quyền dùng verb `request-serviceaccounts-token-audience` trên audience
  này thông qua RBAC. Để biết chi tiết và ví dụ, xem
  [Giới hạn audience của service account token](https://kubernetes.io/docs/reference/access-authn-authz/node/#service-account-token-audience-restriction).
* `cacheType`: loại khóa cache dùng để cache credential mà plugin trả về khi service account
  token được sử dụng. Lựa chọn thận trọng nhất là đặt giá trị này thành `Token`, nghĩa là
  kubelet sẽ cache credential trả về theo từng token. Nên đặt như vậy nếu thời hạn của
  credential trả về bị giới hạn theo thời hạn của service account token. Nếu logic lấy
  credential của plugin chỉ phụ thuộc vào service account chứ không phụ thuộc vào các claim
  riêng của Pod, thì plugin có thể đặt giá trị này thành `ServiceAccount`. Trong trường hợp đó,
  kubelet sẽ cache credential trả về theo từng service account. Dùng lựa chọn này khi credential
  trả về hợp lệ cho mọi Pod dùng cùng một service account.
* `requireServiceAccount`: plugin có yêu cầu Pod phải có service account hay không.
  * Nếu đặt là `true`, kubelet sẽ chỉ gọi plugin nếu Pod có service account.
  * Nếu đặt là `false`, kubelet sẽ gọi plugin ngay cả khi Pod không có service account và sẽ
    không kèm token trong `CredentialProviderRequest`.

Điều này hữu ích cho các plugin dùng để pull image cho các Pod không có service account (ví dụ:
static pod).

#### Cấu hình so khớp image (Configure image matching)

Trường `matchImages` của mỗi credential provider được kubelet dùng để xác định xem một plugin có
nên được gọi cho một image mà Pod đang sử dụng hay không. Mỗi mục trong `matchImages` là một
pattern image, có thể tùy chọn chứa port và path. Glob có thể được dùng trong phần domain, nhưng
không được dùng trong port hoặc path. Glob được hỗ trợ dưới dạng subdomain như `*.k8s.io` hay
`k8s.*.io`, và top-level domain như `k8s.*`. So khớp subdomain một phần như `app*.k8s.io` cũng
được hỗ trợ. Mỗi glob chỉ có thể khớp một phân đoạn subdomain duy nhất, nên `*.io` KHÔNG khớp
`*.k8s.io`.

Một tên image và một mục `matchImage` được coi là khớp khi tất cả các điều dưới đây đều đúng:

* Cả hai chứa cùng số lượng phần domain và từng phần đều khớp.
* Đường dẫn URL của match image phải là tiền tố (prefix) của đường dẫn URL image đích.
* Nếu matchImages chứa port, thì port đó cũng phải khớp trong image.

Một số giá trị ví dụ của các pattern `matchImages`:

* `123456789.dkr.ecr.us-east-1.amazonaws.com`
* `*.azurecr.io`
* `gcr.io`
* `*.*.registry.io`
* `foo.registry.io:8080/path`

## Tiếp theo (What's next)

* Đọc chi tiết về `CredentialProviderConfig` trong
  [tài liệu tham khảo API cấu hình kubelet (v1)](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1/).
* Đọc [tài liệu tham khảo API kubelet credential provider (v1)](https://kubernetes.io/docs/reference/config-api/kubelet-credentialprovider.v1/).
