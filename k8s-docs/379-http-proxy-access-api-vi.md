# Dùng HTTP Proxy để truy cập Kubernetes API (Use an HTTP Proxy to Access the Kubernetes API)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/extend-kubernetes/http-proxy-access-api/>
>
> Trang này hướng dẫn cách dùng một HTTP proxy để truy cập Kubernetes API.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 28 — Mở rộng Kubernetes](00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes), bài 8/11 ·
Phần II không có lab riêng: bạn thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) rồi tự chấm bằng **Checkpoint** của
[giai đoạn 28](00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes).
Bài nối tiếp [164](164-proxies-vi.md) ở
[giai đoạn 5](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng).

**Đây là bài đầu của một cụm ba bài rất dễ lẫn vào nhau.** Ba bài 8/11, 9/11 và 10/11 đều nói về
việc cho đường mạng tới API đi qua một trung gian, nhưng ba trung gian đó khác nhau cả về **nơi
đặt** lẫn **chiều đi**. Bài này là loại thứ nhất: một tiến trình chạy **trên máy của bạn**, phơi
ra một endpoint HTTP cục bộ để các công cụ HTTP thường gọi vào. Giữ mốc đó trong đầu, rồi so với
[SOCKS5](382-socks5-proxy-access-api-vi.md) ở bài 9/11 và
[Konnectivity](381-setup-konnectivity-vi.md) ở bài 10/11.

**Phải hiểu ở lần đọc này:**

- `kubectl proxy --port=8080` ở mục *Dùng kubectl để khởi động một proxy server* là một tiến
  trình **chạy trên máy bạn gõ lệnh**, mở một proxy tới Kubernetes API server. Nó không tạo,
  không sửa và không xóa object nào trong cluster.
- Khi proxy đang chạy, bài nói rõ ở mục *Khám phá Kubernetes API*: bạn khám phá API bằng `curl`,
  `wget` **hoặc trình duyệt**, gọi vào `http://localhost:<port>`. Đó là điểm lợi của cách này —
  công cụ HTTP thường dùng được ngay.
- Hình dạng đường dẫn REST mà proxy phơi ra, đọc từ hai ví dụ của bài: `/api/` trả về một object
  `kind: APIVersions`, còn `/api/v1/namespaces/default/pods` trả về `kind: PodList`. Từ đó suy ra
  khuôn `/api/v1/namespaces/<namespace>/<resource>`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Bước chuẩn bị `kubectl create deployment hello-app --image=gcr.io/google-samples/hello-app:2.0 --port=8080` | chỉ là mồi để trong namespace có object mà đọc; image nằm ngoài [baseline của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) và phần còn lại của bài không dùng ứng dụng đó vào việc gì | Deployment đã học ở bài [63](63-deployment-vi.md), [giai đoạn 4](00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller); ở bài này `curl` vào một namespace đang có sẵn Pod là đủ |
| Từng trường trong hai khối JSON kết quả — `serverAddressByClientCIDRs`, `resourceVersion`, `generateName`, `uid` | bài in chúng ra để cho thấy **hình dạng** phản hồi, và không giải thích trường nào | gọi thẳng REST API của cluster rồi đọc phản hồi là bài thực hành [190](190-access-cluster-api-vi.md) ở [giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy) |

---

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

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 28:

1. `kubectl proxy --port=8080` tạo ra tiến trình chạy ở đâu, và nó thêm hay sửa object nào trong
   cluster? Khi nó đang chạy, bài cho phép bạn dùng những công cụ nào để khám phá API?
2. **Câu bẫy.** Bài dùng con số `8080` hai lần: một lần ở `--port=8080` khi tạo Deployment
   `hello-app`, một lần ở `kubectl proxy --port=8080`. Hai con số đó có liên quan gì với nhau
   không? `curl http://localhost:8080/api/` đang gọi tới cái nào trong hai cái?
3. Bạn chạy `kubectl proxy --port=8080` trên `lab-k8s-master`. Viết URL để lấy danh sách phiên
   bản API, và URL để lấy danh sách Pod của namespace `kube-system`. Mỗi URL trả về object có
   `kind` là gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Tiến trình chạy **trên chính máy bạn gõ lệnh**, không phải trong cluster. Nó **không thêm và
   không sửa object nào** — nó chỉ khởi động một proxy tới Kubernetes API server, nên cluster
   nhìn từ bên ngoài vẫn y nguyên. Khi proxy đang chạy, bài nói rõ bạn có thể khám phá API bằng
   **`curl`, `wget` hoặc trình duyệt**.
2. **Không liên quan gì với nhau.** `--port=8080` của `kubectl create deployment` là port mà
   **ứng dụng `hello-app`** lắng nghe bên trong cluster; `--port=8080` của `kubectl proxy` là
   port mà **proxy trên máy bạn** lắng nghe. Trùng số chỉ là tình cờ của tài liệu. Và
   `curl http://localhost:8080/api/` gọi tới **proxy**, không tới `hello-app`: bằng chứng là nó
   trả về `kind: APIVersions` chứ không trả về trang của ứng dụng.
3. Danh sách phiên bản API: `curl http://localhost:8080/api/` — trả về object
   **`kind: APIVersions`**. Danh sách Pod của `kube-system`:
   `curl http://localhost:8080/api/v1/namespaces/kube-system/pods` — trả về object
   **`kind: PodList`**. Khuôn đường dẫn lấy thẳng từ ví dụ của bài, chỉ thay `default` bằng
   `kube-system`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
