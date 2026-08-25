# Dùng Service để truy cập một ứng dụng trong cluster (Use a Service to Access an Application in a Cluster)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/access-application-cluster/service-access-application-cluster/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 30 — Truy cập ứng dụng trong cluster](00-ALO-TRINH-ADMIN.md#giai-đoạn-30--truy-cập-ứng-dụng-trong-cluster),
bài 3/4 · Phần II không có lab: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) và tự chấm bằng **Checkpoint** ở cuối
[mục giai đoạn 30](00-ALO-TRINH-ADMIN.md#giai-đoạn-30--truy-cập-ứng-dụng-trong-cluster). Cơ chế
của bài đã được kiểm chứng ở [Lab 5a](labs/LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md) — phần **B2**
(selector, `port`/`targetPort`, cluster IP) và phần **B6** (`NodePort` mở cùng một port trên mọi
node, `LoadBalancer` treo trên cluster bare metal).

Bài nối tiếp [82 — Service](82-service-vi.md) và là **đường vào thứ nhất** trong ba đường mà
Checkpoint giai đoạn 30 bắt phân biệt: đường **Service**, đường dành cho người dùng thật của ứng
dụng. Đường thứ hai — `kubectl port-forward` — bạn đã đọc ở bài
[366](366-port-forward-vi.md) của giai đoạn 5 và nó là công cụ gỡ lỗi cho một máy trạm. Đường thứ
ba — apiserver proxy — là bài [369](369-access-cluster-services-vi.md) ngay sau đây. Đọc bài này để
chốt phần "vào bằng Service", chưa cần so ba đường vội.

**Phải hiểu ở lần đọc này:**

- Bước 4 của mục *Tạo một Service cho ứng dụng chạy trong hai pod*:
  `kubectl expose deployment hello-world --type=NodePort --name=example-service` tạo Service **từ
  Deployment**, nên selector của Service lấy đúng label của Deployment — trong output bước 5 là
  `Selector: run=load-balancer-example`.
- Output `kubectl describe services` ở bước 5 có ba con số phải tách bạch: `Port` (port của
  Service), `TargetPort` (port trong container) và `NodePort` (port mở trên node). Cùng output đó
  còn có dòng `IP:` — Service `NodePort` **vẫn được cấp cluster IP** — và dòng `Endpoints:` liệt kê
  đúng các Pod IP kèm port đang đứng sau Service.
- Bước 6 và 7: hai Pod của Deployment nằm trên hai node khác nhau, và để gọi được ứng dụng bạn cần
  **địa chỉ của node** cộng với giá trị `NodePort` — không phải cluster IP, cũng không phải Pod IP.
  Bài lấy địa chỉ node theo cách phụ thuộc môi trường, không có một lệnh chung.
- Bước 8: phải có **quy tắc tường lửa cho phép TCP đi vào node port**, và mỗi nhà cung cấp cấu hình
  khác nhau. Kubernetes cấp port và mở listener trên node; việc gói tin có tới được port đó hay
  không là chuyện của tầng mạng bên ngoài cluster.
- Mục *Dùng file cấu hình Service* và mục *Dọn dẹp*: `kubectl expose` chỉ là lối tắt — Service viết
  bằng [file cấu hình](82-service-vi.md) cho kết quả tương đương; và dọn dẹp phải xóa **hai** thứ
  riêng biệt là Service và Deployment, vì chúng là hai object độc lập.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Gợi ý dựng cluster bằng minikube hoặc các playground ở mục *Trước khi bạn bắt đầu* | lộ trình không dùng minikube, kind hay cluster dùng chung | **không áp dụng**: bạn đã có cluster ba VM từ [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md), thỏa luôn yêu cầu "ít nhất hai node không phải control plane" của bài |
| Cụm "địa chỉ IP công khai (public IP)" và cách lấy nó bằng `kubectl cluster-info` của Minikube hay `gcloud compute instances list` | ba VM lab không có IP công khai và không thuộc nhà cung cấp cloud nào | **không áp dụng cho cluster lab**: địa chỉ cần dùng là IP nội bộ của node, đọc bằng `kubectl get nodes -o wide` đúng như [Lab 5a](labs/LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md) phần B6.1 đã làm |
| Bước 8 — cách tạo quy tắc tường lửa của từng nhà cung cấp cloud | không có security group của cloud để cấu hình | **không áp dụng cho cluster lab**: ở [Lab 5a](labs/LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md) phần B6.1 bạn đã `curl` thẳng vào node port của cả ba node mà không phải khai báo luật nào của cloud |
| Các giá trị ví dụ `IP: 10.32.0.16`, `Endpoints: 10.200.1.4:8080,10.200.2.5:8080`, `NodePort 31496` | đó là dải địa chỉ của cluster viết tài liệu, không phải của bạn | dải thật của cluster lab: [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) phần B1 (Pod CIDR và Service CIDR) cùng bài [88](88-cluster-ip-allocation-vi.md) |
| Link *Kết nối ứng dụng với Service* ở mục *Tiếp theo* | là tutorial trên kubernetes.io, không nằm trong lộ trình | bài [363](363-connecting-frontend-backend-vi.md) đã đọc ở [giai đoạn 5](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng) đã phủ đúng nội dung đó |

---

Trang này hướng dẫn cách tạo một đối tượng Service của Kubernetes để các client bên ngoài có thể
dùng nó truy cập một ứng dụng đang chạy trong cluster. Service này cung cấp khả năng cân bằng tải
(load balancing) cho một ứng dụng đang chạy hai instance.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Mục tiêu (Objectives)

- Chạy hai instance của một ứng dụng Hello World.
- Tạo một đối tượng Service expose một node port.
- Dùng đối tượng Service để truy cập ứng dụng đang chạy.

## Tạo một Service cho ứng dụng chạy trong hai pod (Creating a service for an application running in two pods)

Đây là file cấu hình cho Deployment của ứng dụng:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  selector:
    matchLabels:
      run: load-balancer-example
  replicas: 2
  template:
    metadata:
      labels:
        run: load-balancer-example
    spec:
      containers:
        - name: hello-world
          image: us-docker.pkg.dev/google-samples/containers/gke/hello-app:2.0
          ports:
            - containerPort: 8080
              protocol: TCP
```

1. Chạy một ứng dụng Hello World trong cluster của bạn:
   Tạo Deployment của ứng dụng bằng file ở trên:

   ```shell
   kubectl apply -f https://k8s.io/examples/service/access/hello-application.yaml
   ```

   Lệnh trên tạo ra một Deployment và một ReplicaSet đi kèm. ReplicaSet này có hai Pod,
   mỗi Pod chạy ứng dụng Hello World.

2. Hiển thị thông tin về Deployment:

   ```shell
   kubectl get deployments hello-world
   kubectl describe deployments hello-world
   ```

3. Hiển thị thông tin về các đối tượng ReplicaSet của bạn:

   ```shell
   kubectl get replicasets
   kubectl describe replicasets
   ```

4. Tạo một đối tượng Service để expose Deployment:

   ```shell
   kubectl expose deployment hello-world --type=NodePort --name=example-service
   ```

5. Hiển thị thông tin về Service:

   ```shell
   kubectl describe services example-service
   ```

   Kết quả sẽ tương tự như sau:

   ```none
   Name:                   example-service
   Namespace:              default
   Labels:                 run=load-balancer-example
   Annotations:            <none>
   Selector:               run=load-balancer-example
   Type:                   NodePort
   IP:                     10.32.0.16
   Port:                   <unset> 8080/TCP
   TargetPort:             8080/TCP
   NodePort:               <unset> 31496/TCP
   Endpoints:              10.200.1.4:8080,10.200.2.5:8080
   Session Affinity:       None
   Events:                 <none>
   ```

   Hãy ghi lại giá trị NodePort của Service. Ví dụ, trong kết quả ở trên, giá trị NodePort
   là 31496.

6. Liệt kê các pod đang chạy ứng dụng Hello World:

   ```shell
   kubectl get pods --selector="run=load-balancer-example" --output=wide
   ```

   Kết quả sẽ tương tự như sau:

   ```none
   NAME                           READY   STATUS    ...  IP           NODE
   hello-world-2895499144-bsbk5   1/1     Running   ...  10.200.1.4   worker1
   hello-world-2895499144-m1pwt   1/1     Running   ...  10.200.2.5   worker2
   ```

7. Lấy địa chỉ IP công khai (public IP) của một trong các node đang chạy pod Hello World.
   Cách lấy địa chỉ này phụ thuộc vào cách bạn dựng cluster. Ví dụ, nếu bạn dùng Minikube,
   bạn có thể xem địa chỉ node bằng lệnh `kubectl cluster-info`. Nếu bạn dùng các instance
   Google Compute Engine, bạn có thể dùng lệnh `gcloud compute instances list` để xem địa chỉ
   công khai của các node.

8. Trên node bạn đã chọn, hãy tạo một quy tắc tường lửa (firewall rule) cho phép lưu lượng TCP
   đi vào node port của bạn. Ví dụ, nếu Service của bạn có giá trị NodePort là 31568, hãy tạo
   một quy tắc tường lửa cho phép lưu lượng TCP trên port 31568. Mỗi nhà cung cấp cloud có cách
   cấu hình quy tắc tường lửa khác nhau.

9. Dùng địa chỉ node và node port để truy cập ứng dụng Hello World:

   ```shell
   curl http://<public-node-ip>:<node-port>
   ```

   trong đó `<public-node-ip>` là địa chỉ IP công khai của node, còn `<node-port>` là giá trị
   NodePort của Service. Phản hồi cho một yêu cầu thành công là một thông điệp chào hỏi:

   ```none
   Hello, world!
   Version: 2.0.0
   Hostname: hello-world-cdd4458f4-m47c8
   ```

## Dùng file cấu hình Service (Using a service configuration file)

Thay vì dùng `kubectl expose`, bạn có thể dùng một
[file cấu hình Service](82-service-vi.md) để tạo Service.

## Dọn dẹp (Cleaning up)

Để xóa Service, hãy chạy lệnh này:

    kubectl delete services example-service

Để xóa Deployment, ReplicaSet và các Pod đang chạy ứng dụng Hello World, hãy chạy lệnh này:

    kubectl delete deployment hello-world

## Tiếp theo (What's next)

Hãy làm tiếp hướng dẫn
[Kết nối ứng dụng với Service (Connecting Applications with Services)](https://kubernetes.io/docs/tutorials/services/connect-applications-service/).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 30:

1. `kubectl expose deployment hello-world --type=NodePort --name=example-service` tạo ra Service với
   selector nào, và selector đó ở đâu ra? Ngoài node port, Service vừa tạo còn được cấp thứ gì?
2. **Câu bẫy.** Output `describe` có `Port: 8080`, `TargetPort: 8080` và `NodePort: 31496`. Khi
   `curl` từ máy ngoài cluster bạn gõ số nào, và hai số 8080 kia đang mô tả chặng nào của đường đi?
3. Trên cluster lab, hai Pod `hello-world` nằm trên `lab-k8s-worker1` và `lab-k8s-worker2`. Bạn
   `curl http://192.168.100.221:<nodePort>` — tức vào `lab-k8s-master`, node **không** chạy Pod nào
   của Deployment. Kết quả ra sao, và vì sao?
4. Bước 8 bắt tạo quy tắc tường lửa cho TCP đi vào node port. Việc đó là phần của Kubernetes hay
   phần của môi trường, và trên ba VM lab nó tương ứng với chuyện gì?
5. Trong ba đường vào cluster mà giai đoạn 30 nói tới, vì sao đường của bài này — Service — là
   đường bạn giao cho người dùng thật, còn `kubectl port-forward` của bài
   [366](366-port-forward-vi.md) thì không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Selector là **`run=load-balancer-example`**, và nó **lấy từ label của Deployment** — vì bạn
   `expose` chính Deployment đó chứ không viết selector bằng tay. Ngoài `NodePort`, Service vẫn
   được cấp **một cluster IP** (dòng `IP:` trong output `describe`) và vẫn có `Port`/`TargetPort`:
   `NodePort` **cộng thêm** một lối vào chứ không thay thế lối vào cũ.
2. Bạn gõ **`31496`**, tức giá trị `NodePort` — đó là port duy nhất mở ra bên ngoài node. `Port`
   là port của **Service** (dùng khi gọi qua cluster IP hoặc tên DNS **từ bên trong** cluster), còn
   `TargetPort` là port **container đang lắng nghe**, chặng cuối cùng của đường đi. Ba số nằm ở ba
   chặng khác nhau; ở đây chúng tình cờ trùng nhau ở hai chặng sau nên rất dễ gõ nhầm.
3. **Vẫn ra `Hello, world!`.** `NodePort` mở **cùng một số port trên mọi node**, kể cả node không
   chạy backend nào, rồi chuyển tiếp tới một endpoint sẵn sàng — đúng điều bạn đã đo ở
   [Lab 5a](labs/LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md) phần B6.1 khi cả ba node cùng trả `200`.
   Bài gốc bảo lấy IP của "một trong các node đang chạy pod" là cách nói cho chắc, không phải ràng
   buộc của cơ chế.
4. **Phần của môi trường.** Kubernetes chỉ cấp số port và làm mọi node lắng nghe trên port đó; còn
   gói tin từ máy ngoài có tới được `<ip node>:<nodePort>` hay không là do tầng mạng bên ngoài
   quyết định. Trên ba VM lab không có security group của cloud nào để khai báo — máy host và ba VM
   nằm chung mạng lab, nên bước này quy về việc firewall trên chính OS của node không chặn port đó.
5. Vì Service là lối vào **thường trực** của cluster: nó tồn tại như một object, mọi node đều lắng
   nghe, và ai chạm được tới địa chỉ node cũng dùng được — không cần kubeconfig, không cần quyền
   trên API. `kubectl port-forward` thì ngược lại: nó là **một tiến trình bạn đang chạy**, chỉ phục
   vụ chính máy chạy lệnh, và cần quyền trên API server — bài [366](366-port-forward-vi.md) xếp nó
   là công cụ **gỡ lỗi** cho quản trị viên, không phải đường vào cho người dùng cuối.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
