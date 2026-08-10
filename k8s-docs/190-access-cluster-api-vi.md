# Truy cập cluster bằng Kubernetes API (Access Clusters Using the Kubernetes API)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/access-cluster-api/>
>
> Trang này hướng dẫn cách truy cập cluster bằng Kubernetes API.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình
để giao tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít
nhất hai node không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có
thể tạo một cluster bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
hoặc dùng một trong các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản v1.36 hoặc mới hơn. Để kiểm tra phiên bản,
hãy chạy `kubectl version`.

## Truy cập Kubernetes API (Accessing the Kubernetes API)

### Truy cập lần đầu với kubectl (Accessing for the first time with kubectl)

Khi truy cập Kubernetes API lần đầu tiên, hãy dùng công cụ dòng lệnh của Kubernetes
là `kubectl`.

Để truy cập một cluster, bạn cần biết vị trí (location) của cluster và có thông tin
xác thực (credentials) để truy cập nó. Thông thường, những thứ này được thiết lập tự
động khi bạn làm theo một [hướng dẫn Bắt đầu (Getting started guide)](https://kubernetes.io/docs/setup/),
hoặc do một người khác đã dựng cluster và cung cấp cho bạn thông tin xác thực cùng
vị trí của cluster.

Kiểm tra vị trí và thông tin xác thực mà kubectl đang biết bằng lệnh sau:

```shell
kubectl config view
```

Nhiều [ví dụ](https://github.com/kubernetes/examples/tree/master/) cung cấp phần giới
thiệu về cách dùng kubectl. Tài liệu đầy đủ nằm trong
[kubectl manual](https://kubernetes.io/docs/reference/kubectl/).

### Truy cập trực tiếp REST API (Directly accessing the REST API)

kubectl đảm nhận việc định vị và xác thực tới API server. Nếu bạn muốn truy cập trực
tiếp REST API bằng một http client như `curl` hoặc `wget`, hoặc bằng trình duyệt, có
nhiều cách để bạn định vị và xác thực với API server:

1. Chạy kubectl ở chế độ proxy (khuyến nghị). Phương pháp này được khuyến nghị vì nó
   dùng vị trí API server đã được lưu sẵn và xác minh danh tính của API server bằng
   certificate tự ký (self-signed). Không thể xảy ra tấn công xen giữa
   (man-in-the-middle - MITM) khi dùng phương pháp này.
1. Cách khác là bạn có thể cung cấp vị trí và thông tin xác thực trực tiếp cho http
   client. Cách này phù hợp với mã client bị proxy gây nhầm lẫn. Để phòng chống tấn
   công xen giữa, bạn sẽ cần import một root certificate vào trình duyệt của mình.

Việc dùng thư viện client Go hoặc Python sẽ cho phép truy cập kubectl ở chế độ proxy.

#### Sử dụng kubectl proxy (Using kubectl proxy)

Lệnh sau chạy kubectl ở chế độ hoạt động như một reverse proxy. Nó đảm nhận việc định
vị API server và xác thực.

Chạy như sau:

```shell
kubectl proxy --port=8080 &
```

Xem [kubectl proxy](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#proxy)
để biết thêm chi tiết.

Sau đó, bạn có thể khám phá API bằng curl, wget hoặc trình duyệt, như sau:

```shell
curl http://localhost:8080/api/
```

Kết quả tương tự như sau:

```json
{
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "10.0.1.149:443"
    }
  ]
}
```

#### Không dùng kubectl proxy (Without kubectl proxy)

Bạn có thể tránh dùng kubectl proxy bằng cách truyền trực tiếp một token xác thực cho
API server, như sau:

Dùng cách tiếp cận với `grep/cut`:

```shell
# Kiểm tra tất cả các cluster có thể có, vì .KUBECONFIG của bạn có thể chứa nhiều context:
kubectl config view -o jsonpath='{"Cluster name\tServer\n"}{range .clusters[*]}{.name}{"\t"}{.cluster.server}{"\n"}{end}'

# Chọn tên cluster mà bạn muốn tương tác từ output ở trên:
export CLUSTER_NAME="some_server_name"

# Trỏ tới API server dựa theo tên cluster
APISERVER=$(kubectl config view -o jsonpath="{.clusters[?(@.name==\"$CLUSTER_NAME\")].cluster.server}")

# Tạo một secret để giữ token cho service account default
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: default-token
  annotations:
    kubernetes.io/service-account.name: default
type: kubernetes.io/service-account-token
EOF

# Chờ token controller điền token vào secret:
while ! kubectl describe secret default-token | grep -E '^token' >/dev/null; do
  echo "waiting for token..." >&2
  sleep 1
done

# Lấy giá trị token
TOKEN=$(kubectl get secret default-token -o jsonpath='{.data.token}' | base64 --decode)

# Khám phá API với TOKEN
curl -X GET $APISERVER/api --header "Authorization: Bearer $TOKEN" --insecure
```

Kết quả tương tự như sau:

```json
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "10.0.1.149:443"
    }
  ]
}
```

Ví dụ trên dùng cờ `--insecure`. Điều này khiến nó dễ bị tấn công MITM. Khi kubectl
truy cập cluster, nó dùng một root certificate đã được lưu sẵn cùng các client
certificate để truy cập server. (Chúng được cài trong thư mục `~/.kube`). Vì
certificate của cluster thường là tự ký, có thể cần cấu hình đặc biệt để http client
của bạn dùng được root certificate.

Trên một số cluster, API server không yêu cầu xác thực; nó có thể phục vụ trên
localhost, hoặc được bảo vệ bởi firewall. Không có một chuẩn nào cho việc này.
[Kiểm soát truy cập vào Kubernetes API](https://kubernetes.io/docs/concepts/security/controlling-access)
mô tả cách bạn — với vai trò quản trị viên cluster — có thể cấu hình điều này.

### Truy cập API bằng lập trình (Programmatic access to the API)

Kubernetes chính thức hỗ trợ các thư viện client cho [Go](#go-client),
[Python](#python-client), [Java](#java-client), [dotnet](#dotnet-client),
[JavaScript](#javascript-client) và [Haskell](#haskell-client). Còn có các thư viện
client khác do chính tác giả của chúng cung cấp và duy trì, không phải do đội ngũ
Kubernetes. Xem [thư viện client](https://kubernetes.io/docs/reference/using-api/client-libraries/)
để biết cách truy cập API từ các ngôn ngữ khác và cách chúng xác thực.

#### Go client

* Để lấy thư viện, chạy lệnh sau: `go get k8s.io/client-go@kubernetes-<kubernetes-version-number>`.
  Xem [https://github.com/kubernetes/client-go/releases](https://github.com/kubernetes/client-go/releases)
  để biết những phiên bản nào được hỗ trợ.
* Viết một ứng dụng dựa trên các client của client-go.

> **Ghi chú:**
> `client-go` định nghĩa các API object của riêng nó, vì vậy nếu cần, hãy import các
> định nghĩa API từ client-go thay vì từ repository chính. Ví dụ,
> `import "k8s.io/client-go/kubernetes"` là đúng.

Go client có thể dùng cùng một [file kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
như kubectl CLI để định vị và xác thực tới API server. Xem
[ví dụ](https://git.k8s.io/client-go/examples/out-of-cluster-client-configuration/main.go) này:

```golang
package main

import (
  "context"
  "fmt"
  "k8s.io/apimachinery/pkg/apis/meta/v1"
  "k8s.io/client-go/kubernetes"
  "k8s.io/client-go/tools/clientcmd"
)

func main() {
  // dùng context hiện tại trong kubeconfig
  // path-to-kubeconfig -- ví dụ, /root/.kube/config
  config, _ := clientcmd.BuildConfigFromFlags("", "<path-to-kubeconfig>")
  // tạo clientset
  clientset, _ := kubernetes.NewForConfig(config)
  // truy cập API để liệt kê các pod
  pods, _ := clientset.CoreV1().Pods("").List(context.TODO(), v1.ListOptions{})
  fmt.Printf("There are %d pods in the cluster\n", len(pods.Items))
}
```

Nếu ứng dụng được triển khai như một Pod trong cluster, xem
[Truy cập API từ bên trong một Pod](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/#accessing-the-api-from-a-pod).

#### Python client

Để dùng [Python client](https://github.com/kubernetes-client/python), chạy lệnh sau:
`pip install kubernetes`. Xem [trang Python Client Library](https://github.com/kubernetes-client/python)
để biết thêm các tùy chọn cài đặt.

Python client có thể dùng cùng một [file kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
như kubectl CLI để định vị và xác thực tới API server. Xem
[ví dụ](https://github.com/kubernetes-client/python/blob/master/examples/out_of_cluster_config.py) này:

```python
from kubernetes import client, config

config.load_kube_config()

v1=client.CoreV1Api()
print("Listing pods with their IPs:")
ret = v1.list_pod_for_all_namespaces(watch=False)
for i in ret.items:
    print("%s\t%s\t%s" % (i.status.pod_ip, i.metadata.namespace, i.metadata.name))
```

#### Java client

Để cài [Java client](https://github.com/kubernetes-client/java), chạy:

```shell
# Clone thư viện java
git clone --recursive https://github.com/kubernetes-client/java

# Cài đặt các artifact của project, POM v.v.:
cd java
mvn install
```

Xem [https://github.com/kubernetes-client/java/releases](https://github.com/kubernetes-client/java/releases)
để biết những phiên bản nào được hỗ trợ.

Java client có thể dùng cùng một [file kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
như kubectl CLI để định vị và xác thực tới API server. Xem
[ví dụ](https://github.com/kubernetes-client/java/blob/master/examples/examples-release-15/src/main/java/io/kubernetes/client/examples/KubeConfigFileClientExample.java) này:

```java
package io.kubernetes.client.examples;

import io.kubernetes.client.ApiClient;
import io.kubernetes.client.ApiException;
import io.kubernetes.client.Configuration;
import io.kubernetes.client.apis.CoreV1Api;
import io.kubernetes.client.models.V1Pod;
import io.kubernetes.client.models.V1PodList;
import io.kubernetes.client.util.ClientBuilder;
import io.kubernetes.client.util.KubeConfig;
import java.io.FileReader;
import java.io.IOException;

/**
 * Một ví dụ đơn giản về cách dùng Java API từ một ứng dụng bên ngoài cluster kubernetes
 *
 * <p>Cách dễ nhất để chạy: mvn exec:java
 * -Dexec.mainClass="io.kubernetes.client.examples.KubeConfigFileClientExample"
 *
 */
public class KubeConfigFileClientExample {
  public static void main(String[] args) throws IOException, ApiException {

    // đường dẫn tới file KubeConfig của bạn
    String kubeConfigPath = "~/.kube/config";

    // nạp cấu hình out-of-cluster, một kubeconfig từ hệ thống tập tin
    ApiClient client =
        ClientBuilder.kubeconfig(KubeConfig.loadKubeConfig(new FileReader(kubeConfigPath))).build();

    // đặt api-client mặc định toàn cục thành client in-cluster đã tạo ở trên
    Configuration.setDefaultApiClient(client);

    // CoreV1Api nạp api-client mặc định từ cấu hình toàn cục.
    CoreV1Api api = new CoreV1Api();

    // gọi client CoreV1Api
    V1PodList list = api.listPodForAllNamespaces(null, null, null, null, null, null, null, null, null);
    System.out.println("Listing all pods: ");
    for (V1Pod item : list.getItems()) {
      System.out.println(item.getMetadata().getName());
    }
  }
}
```

#### dotnet client

Để dùng [dotnet client](https://github.com/kubernetes-client/csharp), chạy lệnh sau:
`dotnet add package KubernetesClient --version 1.6.1`. Xem
[trang dotnet Client Library](https://github.com/kubernetes-client/csharp) để biết
thêm các tùy chọn cài đặt. Xem
[https://github.com/kubernetes-client/csharp/releases](https://github.com/kubernetes-client/csharp/releases)
để biết những phiên bản nào được hỗ trợ.

dotnet client có thể dùng cùng một [file kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
như kubectl CLI để định vị và xác thực tới API server. Xem
[ví dụ](https://github.com/kubernetes-client/csharp/blob/master/examples/simple/PodList.cs) này:

```csharp
using System;
using k8s;

namespace simple
{
    internal class PodList
    {
        private static void Main(string[] args)
        {
            var config = KubernetesClientConfiguration.BuildDefaultConfig();
            IKubernetes client = new Kubernetes(config);
            Console.WriteLine("Starting Request!");

            var list = client.ListNamespacedPod("default");
            foreach (var item in list.Items)
            {
                Console.WriteLine(item.Metadata.Name);
            }
            if (list.Items.Count == 0)
            {
                Console.WriteLine("Empty!");
            }
        }
    }
}
```

#### JavaScript client

Để cài [JavaScript client](https://github.com/kubernetes-client/javascript), chạy
lệnh sau: `npm install @kubernetes/client-node`. Xem
[https://github.com/kubernetes-client/javascript/releases](https://github.com/kubernetes-client/javascript/releases)
để biết những phiên bản nào được hỗ trợ.

JavaScript client có thể dùng cùng một [file kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
như kubectl CLI để định vị và xác thực tới API server. Xem
[ví dụ](https://github.com/kubernetes-client/javascript/blob/master/examples/example.js) này:

```javascript
const k8s = require('@kubernetes/client-node');

const kc = new k8s.KubeConfig();
kc.loadFromDefault();

const k8sApi = kc.makeApiClient(k8s.CoreV1Api);

k8sApi.listNamespacedPod({ namespace: 'default' }).then((res) => {
    console.log(res);
});
```

#### Haskell client

Xem [https://github.com/kubernetes-client/haskell/releases](https://github.com/kubernetes-client/haskell/releases)
để biết những phiên bản nào được hỗ trợ.

[Haskell client](https://github.com/kubernetes-client/haskell) có thể dùng cùng một
[file kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
như kubectl CLI để định vị và xác thực tới API server. Xem
[ví dụ](https://github.com/kubernetes-client/haskell/blob/master/kubernetes-client/example/App.hs) này:

```haskell
exampleWithKubeConfig :: IO ()
exampleWithKubeConfig = do
    oidcCache <- atomically $ newTVar $ Map.fromList []
    (mgr, kcfg) <- mkKubeClientConfig oidcCache $ KubeConfigFile "/path/to/kubeconfig"
    dispatchMime
            mgr
            kcfg
            (CoreV1.listPodForAllNamespaces (Accept MimeJSON))
        >>= print
```

## Tiếp theo (What's next)

* [Truy cập Kubernetes API từ một Pod](https://kubernetes.io/docs/tasks/run-application/access-api-from-pod/)
