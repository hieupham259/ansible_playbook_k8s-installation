# Dùng Service để truy cập một ứng dụng trong cluster (Use a Service to Access an Application in a Cluster)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/access-application-cluster/service-access-application-cluster/

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
