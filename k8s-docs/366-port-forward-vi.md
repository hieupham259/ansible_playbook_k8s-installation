# Sử dụng Port Forwarding để truy cập ứng dụng trong Cluster (Use Port Forwarding to Access Applications in a Cluster)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/

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
