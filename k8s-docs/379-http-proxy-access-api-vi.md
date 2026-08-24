# Dùng HTTP Proxy để truy cập Kubernetes API (Use an HTTP Proxy to Access the Kubernetes API)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/extend-kubernetes/http-proxy-access-api/>
>
> Trang này hướng dẫn cách dùng một HTTP proxy để truy cập Kubernetes API.

Trang này hướng dẫn cách dùng một HTTP proxy để truy cập Kubernetes API.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, nhập `kubectl version`.

Nếu bạn chưa có ứng dụng nào đang chạy trong cluster, hãy khởi động một ứng dụng Hello world
bằng cách chạy lệnh sau:

```shell
kubectl create deployment hello-app --image=gcr.io/google-samples/hello-app:2.0 --port=8080
```

## Dùng kubectl để khởi động một proxy server (Using kubectl to start a proxy server)

Lệnh sau khởi động một proxy tới Kubernetes API server:

```shell
kubectl proxy --port=8080
```

## Khám phá Kubernetes API (Exploring the Kubernetes API)

Khi proxy server đang chạy, bạn có thể khám phá API bằng `curl`, `wget` hoặc trình duyệt.

Lấy danh sách các phiên bản API:

```shell
curl http://localhost:8080/api/
```

Kết quả trả về sẽ trông tương tự như sau:

```
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "10.0.2.15:8443"
    }
  ]
}
```

Lấy danh sách các pod:

```shell
curl http://localhost:8080/api/v1/namespaces/default/pods
```

Kết quả trả về sẽ trông tương tự như sau:

```
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "33074"
  },
  "items": [
    {
      "metadata": {
        "name": "kubernetes-bootcamp-2321272333-ix8pt",
        "generateName": "kubernetes-bootcamp-2321272333-",
        "namespace": "default",
        "uid": "ba21457c-6b1d-11e6-85f7-1ef9f1dab92b",
        "resourceVersion": "33003",
        "creationTimestamp": "2016-08-25T23:43:30Z",
        "labels": {
          "pod-template-hash": "2321272333",
          "run": "kubernetes-bootcamp"
        },
        ...
}
```

## Tiếp theo (What's next)

Tìm hiểu thêm về [kubectl proxy](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#proxy).
