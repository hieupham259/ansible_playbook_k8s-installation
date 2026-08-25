# Sử dụng Port Forwarding để truy cập ứng dụng trong Cluster (Use Port Forwarding to Access Applications in a Cluster)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 5 — Mạng nền tảng](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng), dòng
**Thực hành**, bài 10/10 — **bài cuối của dòng Thực hành** · Kiểm chứng ở
[Lab 5a — Service, EndpointSlice và DNS](labs/LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md) phần B9.3.

Bài gốc dùng MongoDB làm ví dụ và yêu cầu cài MongoDB Shell (`mongosh`) — một công cụ client
**nằm ngoài baseline** của cluster lab. Lab 5a **không** thực hành phần MongoDB: ở phần B9.3 lab
chạy `port-forward` tới một Service HTTP đã dựng sẵn trong lab. Đọc phần MongoDB để hiểu bối
cảnh, còn thứ phải nắm là **cú pháp, giới hạn và ranh giới an toàn** của `kubectl port-forward`.

Đây là đường vào thứ ba và là đường cuối trong ba đường của cuối giai đoạn 5. Bài
[363](363-connecting-frontend-backend-vi.md) nối hai tầng bên trong cluster bằng Service, bài
[364](364-create-external-load-balancer-vi.md) là đường từ bên ngoài vào qua bộ cân bằng tải, còn
`port-forward` là **công cụ gỡ lỗi cho quản trị viên** — không phải đường vào cho người dùng cuối
và không dùng cho production.

**Phải hiểu ở lần đọc này:**

- Cú pháp `<port cục bộ>:<port trên Pod>` và **năm cách chỉ đích tương đương nhau** ở mục
  *Chuyển tiếp một port cục bộ tới một port trên Pod*: tên Pod trần, `pods/`, `deployment/`,
  `replicaset/`, `service/`. Bài nói rõ mọi cách đều dùng tên tài nguyên để **chọn một pod khớp**
  làm đích chuyển tiếp.
- Ghi chú ngay sau đó: lệnh **không tự kết thúc và không trả lại dấu nhắc lệnh** — muốn làm tiếp
  việc khác thì phải mở một terminal khác.
- Mục *Tùy chọn: để kubectl tự chọn port cục bộ*: bỏ trống vế trái (`deployment/mongo :27017`)
  thì `kubectl` tự tìm một port cục bộ chưa được sử dụng, nhờ đó bạn không phải quản lý xung đột
  port cục bộ.
- Mục *Thảo luận*: kết nối tới port cục bộ được chuyển tiếp tới port của Pod, nhờ đó **máy trạm
  cục bộ gỡ lỗi được thứ đang chạy trong Pod**; và `port-forward` **chỉ được hiện thực cho các
  port TCP**, UDP chưa hỗ trợ.
- Mục *Cân nhắc về ủy quyền và bảo mật*: việc ủy quyền do **API server** thực thi chứ không phải
  do client `kubectl`; và port-forwarding cho **truy cập mạng trực tiếp tới workload**, có thể
  **vượt qua các biện pháp kiểm soát ở tầng mạng** — vì vậy quản trị viên phải hạn chế quyền này
  một cách cẩn trọng.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Toàn bộ phần MongoDB — cài MongoDB Shell, `mongosh --port 28015`, `db.runCommand( { ping: 1 } )` | `mongosh` là công cụ nằm ngoài baseline của cluster lab; MongoDB chỉ là ví dụ, không phải nội dung của `port-forward` | [Lab 5a](labs/LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md) phần B9.3 kiểm chứng đúng cơ chế này bằng một Service HTTP đã có sẵn trong lab, không cần `mongosh` |
| Tên quyền cụ thể mà mục *Cân nhắc về ủy quyền và bảo mật* liệt kê | RBAC chưa học ở giai đoạn 5; ở đây bạn dùng kubeconfig quản trị dựng từ [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) nên quyền không phải vấn đề | [Giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), bài [119](119-controlling-access-vi.md) và [120](120-rbac-good-practices-vi.md); thực hành ở [Lab 9a](labs/LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md) |

---

Trang này hướng dẫn cách dùng `kubectl port-forward` để kết nối tới một MongoDB
server đang chạy trong một cluster Kubernetes. Kiểu kết nối này có thể hữu ích
cho việc gỡ lỗi (debug) cơ sở dữ liệu.

## Trước khi bạn bắt đầu (Before you begin)

* Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
  với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
  vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
  [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
  chơi (playground) Kubernetes sau:

  * [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
  * [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
  * [KodeKloud](https://kodekloud.com/public-playgrounds)

  Kubernetes server của bạn phải ở phiên bản v1.10 hoặc mới hơn.
  Để kiểm tra phiên bản, nhập `kubectl version`.
* Cài đặt [MongoDB Shell](https://www.mongodb.com/try/download/shell).

## Tạo MongoDB deployment và service (Creating MongoDB deployment and service)

1. Tạo một Deployment chạy MongoDB:

   ```shell
   kubectl apply -f https://k8s.io/examples/application/mongodb/mongo-deployment.yaml
   ```

   Đầu ra của lệnh chạy thành công xác nhận rằng deployment đã được tạo:

   ```
   deployment.apps/mongo created
   ```

   Xem trạng thái pod để kiểm tra rằng nó đã sẵn sàng:

   ```shell
   kubectl get pods
   ```

   Đầu ra hiển thị pod đã được tạo:

   ```
   NAME                     READY   STATUS    RESTARTS   AGE
   mongo-75f59d57f4-4nd6q   1/1     Running   0          2m4s
   ```

   Xem trạng thái của Deployment:

   ```shell
   kubectl get deployment
   ```

   Đầu ra hiển thị rằng Deployment đã được tạo:

   ```
   NAME    READY   UP-TO-DATE   AVAILABLE   AGE
   mongo   1/1     1            1           2m21s
   ```

   Deployment tự động quản lý một ReplicaSet.
   Xem trạng thái của ReplicaSet bằng lệnh:

   ```shell
   kubectl get replicaset
   ```

   Đầu ra hiển thị rằng ReplicaSet đã được tạo:

   ```
   NAME               DESIRED   CURRENT   READY   AGE
   mongo-75f59d57f4   1         1         1       3m12s
   ```

2. Tạo một Service để mở MongoDB ra mạng:

   ```shell
   kubectl apply -f https://k8s.io/examples/application/mongodb/mongo-service.yaml
   ```

   Đầu ra của lệnh chạy thành công xác nhận rằng Service đã được tạo:

   ```
   service/mongo created
   ```

   Kiểm tra Service vừa tạo:

   ```shell
   kubectl get service mongo
   ```

   Đầu ra hiển thị service đã được tạo:

   ```
   NAME    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
   mongo   ClusterIP   10.96.41.183   <none>        27017/TCP   11s
   ```

3. Xác minh rằng MongoDB server đang chạy trong Pod và lắng nghe trên port 27017:

   ```shell
   # Thay mongo-75f59d57f4-4nd6q bằng tên của Pod
   kubectl get pod mongo-75f59d57f4-4nd6q --template='{{(index (index .spec.containers 0).ports 0).containerPort}}{{"\n"}}'
   ```

   Đầu ra hiển thị port của MongoDB trong Pod đó:

   ```
   27017
   ```

   27017 là port TCP chính thức của MongoDB.

## Chuyển tiếp một port cục bộ tới một port trên Pod (Forward a local port to a port on the Pod) {#forward-a-local-port-to-a-port-on-the-pod}

1. `kubectl port-forward` cho phép dùng tên tài nguyên, chẳng hạn tên pod, để chọn một pod khớp
   làm đích chuyển tiếp port (port forward).

   ```shell
   # Thay mongo-75f59d57f4-4nd6q bằng tên của Pod
   kubectl port-forward mongo-75f59d57f4-4nd6q 28015:27017
   ```

   lệnh này tương đương với

   ```shell
   kubectl port-forward pods/mongo-75f59d57f4-4nd6q 28015:27017
   ```

   hoặc

   ```shell
   kubectl port-forward deployment/mongo 28015:27017
   ```

   hoặc

   ```shell
   kubectl port-forward replicaset/mongo-75f59d57f4 28015:27017
   ```

   hoặc

   ```shell
   kubectl port-forward service/mongo 28015:27017
   ```

   Bất kỳ lệnh nào ở trên đều dùng được. Đầu ra tương tự như sau:

   ```
   Forwarding from 127.0.0.1:28015 -> 27017
   Forwarding from [::1]:28015 -> 27017
   ```

   > **Ghi chú:**
   > `kubectl port-forward` không tự kết thúc và trả lại dấu nhắc lệnh. Để tiếp tục các bài thực hành, bạn sẽ cần mở một terminal khác.

2. Khởi động giao diện dòng lệnh của MongoDB:

   ```shell
   mongosh --port 28015
   ```

3. Tại dấu nhắc lệnh của MongoDB, nhập lệnh `ping`:

   ```
   db.runCommand( { ping: 1 } )
   ```

   Một yêu cầu ping thành công trả về:

   ```
   { ok: 1 }
   ```

### Tùy chọn: để _kubectl_ tự chọn port cục bộ (Optionally let _kubectl_ choose the local port) {#let-kubectl-choose-local-port}

Nếu bạn không cần một port cục bộ cụ thể, bạn có thể để `kubectl` tự chọn và cấp phát
port cục bộ, nhờ đó bạn không phải quản lý xung đột port cục bộ, với
cú pháp đơn giản hơn một chút:

```shell
kubectl port-forward deployment/mongo :27017
```

Công cụ `kubectl` tìm một số port cục bộ chưa được sử dụng (tránh các số port thấp,
vì chúng có thể đang được các ứng dụng khác dùng). Đầu ra tương tự như:

```
Forwarding from 127.0.0.1:63753 -> 27017
Forwarding from [::1]:63753 -> 27017
```

## Thảo luận (Discussion)

Các kết nối tới port cục bộ 28015 được chuyển tiếp đến port 27017 của Pod đang
chạy MongoDB server. Với kết nối này, bạn có thể dùng máy trạm cục bộ của mình
để gỡ lỗi cơ sở dữ liệu đang chạy trong Pod.

> **Ghi chú:**
> `kubectl port-forward` chỉ được hiện thực cho các port TCP.
> Việc hỗ trợ giao thức UDP đang được theo dõi tại
> [issue 47862](https://github.com/kubernetes/kubernetes/issues/47862).

## Cân nhắc về ủy quyền và bảo mật (Authorization and security considerations)

Quyền truy cập `kubectl port-forward` được kiểm soát bởi các cơ chế ủy quyền (authorization) của Kubernetes như Điều khiển truy cập dựa trên vai trò (Role-Based Access Control - RBAC). Việc ủy quyền được thực thi bởi Kubernetes API server, không phải bởi client `kubectl`.

Để dùng `kubectl port-forward`, người dùng phải có quyền truy cập tài nguyên đích (ví dụ, một Pod hoặc Service) và subresource `portforward`. Các quyền thường được yêu cầu bao gồm `get` trên `pods` và `create` trên `pods/portforward`.

Quản trị viên cluster nên hạn chế các quyền này một cách cẩn trọng, vì port-forwarding có thể cung cấp truy cập mạng trực tiếp tới các workload và có thể vượt qua (bypass) các biện pháp kiểm soát ở tầng mạng.

## Tiếp theo (What's next)

Tìm hiểu thêm về [kubectl port-forward](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#port-forward).

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 5.

1. Giả sử Deployment `mongo` có nhiều replica trải trên `lab-k8s-worker1` và `lab-k8s-worker2`.
   Bạn chạy `kubectl port-forward deployment/mongo 28015:27017` trên `lab-k8s-master`. Các kết
   nối tới `127.0.0.1:28015` chạm tới **bao nhiêu** Pod, và vì sao?
2. **Câu bẫy.** Trong danh sách lệnh tương đương có `kubectl port-forward service/mongo
   28015:27017`. Dạng này có nghĩa là lưu lượng đi **qua** Service và được cân bằng tải giữa các
   Pod phía sau không?
3. Lệnh đã in `Forwarding from 127.0.0.1:28015 -> 27017` và bạn muốn chạy tiếp `kubectl get
   pods`. Vì sao dấu nhắc lệnh không quay lại, và bài bảo làm gì?
4. Bạn cần forward nhưng không quan tâm port cục bộ nào được dùng. Viết lệnh thế nào, và cái lợi
   là gì?
5. Có nên mở `kubectl port-forward` làm đường cho người dùng cuối truy cập ứng dụng không? Ai
   thực thi việc kiểm soát ai được dùng nó, và rủi ro bài nêu ra là gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Đúng một Pod.** Bài nói `kubectl port-forward` "cho phép dùng tên tài nguyên, chẳng hạn tên
   pod, để chọn **một pod khớp** làm đích chuyển tiếp port". Tên Deployment ở đây chỉ là cách
   **tìm ra** một Pod, không phải cách gom nhiều Pod lại; mọi kết nối tới `127.0.0.1:28015` đều
   đi tới đúng Pod đó, trên đúng một worker.
2. **Không.** Đây là chỗ dễ nhầm nhất của bài: `service/mongo` được liệt kê **cùng danh sách** với
   `pods/…`, `deployment/…` và `replicaset/…`, và bài kết luận "bất kỳ lệnh nào ở trên đều dùng
   được" — tức năm dạng cho **cùng một kết quả**: chọn một pod khớp. Tên Service ở đây chỉ là
   **cách chỉ ra Pod**, không phải đường đi của lưu lượng; không có cân bằng tải nào ở đây.
3. Vì bài ghi rõ `kubectl port-forward` **không tự kết thúc và không trả lại dấu nhắc lệnh** — nó
   phải sống suốt thời gian đường chuyển tiếp còn mở. Bài bảo: **mở một terminal khác** để tiếp
   tục các bài thực hành.
4. `kubectl port-forward deployment/mongo :27017` — **bỏ trống vế port cục bộ**. `kubectl` tự tìm
   một số port cục bộ chưa được sử dụng (và tránh các số port thấp vì chúng có thể đang được ứng
   dụng khác dùng), nhờ đó **bạn không phải tự quản lý xung đột port cục bộ**.
5. **Không.** Bài định vị `port-forward` là kiểu kết nối **hữu ích cho việc gỡ lỗi** cơ sở dữ
   liệu chạy trong Pod, chứ không phải đường phục vụ người dùng. Việc ủy quyền do **Kubernetes
   API server thực thi, không phải client `kubectl`** — người dùng phải có quyền trên tài nguyên
   đích và trên subresource `portforward`. Rủi ro bài nêu: port-forwarding **cung cấp truy cập
   mạng trực tiếp tới các workload và có thể vượt qua các biện pháp kiểm soát ở tầng mạng**, nên
   quản trị viên cluster phải hạn chế các quyền này một cách cẩn trọng.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng. Đây là bài cuối của dòng **Thực hành**
ở [Giai đoạn 5 — Mạng nền tảng](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng): trả lời xong
cả năm câu thì mở
[Lab 5a — Service, EndpointSlice và DNS](labs/LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md).
