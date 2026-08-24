# Quản trị Cloud Controller Manager (Cloud Controller Manager Administration)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.11 [beta]`

Vì các nhà cung cấp dịch vụ đám mây (cloud provider) phát triển và phát hành theo nhịp độ khác
so với dự án Kubernetes, việc trừu tượng hóa phần mã đặc thù của từng nhà cung cấp vào binary
`cloud-controller-manager` cho phép các nhà cung cấp cloud phát triển độc lập với mã lõi
(core) của Kubernetes.

`cloud-controller-manager` có thể được liên kết với bất kỳ cloud provider nào thỏa mãn
[cloudprovider.Interface](https://github.com/kubernetes/cloud-provider/blob/master/cloud.go).
Để tương thích ngược, thành phần
[cloud-controller-manager](https://github.com/kubernetes/kubernetes/tree/master/cmd/cloud-controller-manager)
được cung cấp trong dự án Kubernetes lõi sử dụng cùng các thư viện cloud với
`kube-controller-manager`. Các cloud provider đã được hỗ trợ sẵn trong Kubernetes lõi được kỳ
vọng sẽ dùng cloud-controller-manager dạng in-tree để chuyển dần ra khỏi phần lõi của Kubernetes.

## Quản trị (Administration)

### Yêu cầu (Requirements)

Mỗi nền tảng cloud có bộ yêu cầu riêng để chạy phần tích hợp cloud provider của họ, và các yêu
cầu này không quá khác biệt so với khi chạy `kube-controller-manager`. Theo nguyên tắc chung,
bạn sẽ cần:

* Xác thực/ủy quyền (authentication/authorization) với cloud: nền tảng cloud của bạn có thể yêu
  cầu token hoặc các quy tắc IAM để cho phép truy cập vào API của họ
* Xác thực/ủy quyền với Kubernetes: cloud-controller-manager có thể cần các quy tắc RBAC được
  thiết lập để giao tiếp với apiserver của Kubernetes
* Tính sẵn sàng cao (high availability): giống như kube-controller-manager, bạn có thể muốn một
  thiết lập có tính sẵn sàng cao cho cloud controller manager bằng cơ chế bầu chọn leader
  (leader election, được bật theo mặc định).

### Chạy cloud-controller-manager (Running cloud-controller-manager)

Để chạy cloud-controller-manager thành công, cần một số thay đổi trong cấu hình cluster của bạn.

* `kubelet` và `kube-controller-manager` phải được thiết lập tùy theo việc người dùng có sử dụng
  CCM bên ngoài (external CCM) hay không. Nếu người dùng có một CCM bên ngoài (không phải các
  vòng lặp cloud controller nội bộ trong Kubernetes Controller Manager), thì phải chỉ định
  `--cloud-provider=external`. Ngược lại thì không nên chỉ định.

Hãy lưu ý rằng việc thiết lập cluster của bạn để dùng cloud controller manager sẽ thay đổi hành
vi của cluster theo một vài cách:

* Các thành phần chỉ định `--cloud-provider=external` sẽ thêm một taint
  `node.cloudprovider.kubernetes.io/uninitialized` với effect `NoSchedule` trong quá trình khởi
  tạo. Điều này đánh dấu node là cần một lần khởi tạo thứ hai từ một controller bên ngoài trước
  khi có thể được lập lịch công việc. Lưu ý rằng trong trường hợp cloud controller manager không
  khả dụng, các node mới trong cluster sẽ bị bỏ lại ở trạng thái không thể lập lịch
  (unschedulable). Taint này quan trọng vì scheduler có thể cần các thông tin đặc thù cloud về
  node, chẳng hạn như region hoặc loại node (CPU cao, GPU, bộ nhớ lớn, spot instance, v.v.).
* Thông tin cloud về các node trong cluster sẽ không còn được truy xuất bằng metadata cục bộ
  nữa; thay vào đó, mọi lời gọi API để truy xuất thông tin node sẽ đi qua cloud controller
  manager. Điều này có nghĩa là bạn có thể hạn chế quyền truy cập cloud API trên các kubelet để
  tăng cường bảo mật. Với các cluster lớn, bạn nên cân nhắc liệu cloud controller manager có
  chạm ngưỡng giới hạn tần suất (rate limit) hay không, vì giờ đây nó chịu trách nhiệm cho gần
  như toàn bộ lời gọi API tới cloud của bạn từ bên trong cluster.

Cloud controller manager có thể triển khai:

* Node controller — chịu trách nhiệm cập nhật các node Kubernetes bằng cloud API và xóa các node
  Kubernetes tương ứng với những node đã bị xóa trên cloud của bạn.
* Service controller — chịu trách nhiệm về các bộ cân bằng tải (load balancer) trên cloud của
  bạn cho các Service loại LoadBalancer.
* Route controller — chịu trách nhiệm thiết lập các network route trên cloud của bạn
* bất kỳ tính năng nào khác mà bạn muốn triển khai nếu bạn đang chạy một provider dạng
  out-of-tree.

## Ví dụ (Examples)

Nếu bạn đang dùng một nền tảng cloud hiện được hỗ trợ trong Kubernetes lõi và muốn áp dụng cloud
controller manager, hãy xem
[cloud controller manager trong Kubernetes lõi](https://github.com/kubernetes/kubernetes/tree/master/cmd/cloud-controller-manager).

Với các cloud controller manager không nằm trong Kubernetes lõi, bạn có thể tìm các dự án tương
ứng trong các repository do các nhà cung cấp cloud hoặc các SIG duy trì.

Với các provider đã có trong Kubernetes lõi, bạn có thể chạy cloud controller manager dạng
in-tree như một DaemonSet trong cluster của mình; hãy dùng nội dung sau như một hướng dẫn tham
khảo:

```yaml
# Đây là ví dụ về cách thiết lập cloud-controller-manager như một DaemonSet trong cluster của bạn.
# Nó giả định rằng các master của bạn có thể chạy Pod và có role node-role.kubernetes.io/master
# Lưu ý rằng DaemonSet này sẽ không chạy được ngay lập tức trên cloud của bạn,
# nó chỉ được dùng như một hướng dẫn tham khảo.

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cloud-controller-manager
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:cloud-controller-manager
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: cloud-controller-manager
  namespace: kube-system
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    k8s-app: cloud-controller-manager
  name: cloud-controller-manager
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: cloud-controller-manager
  template:
    metadata:
      labels:
        k8s-app: cloud-controller-manager
    spec:
      serviceAccountName: cloud-controller-manager
      containers:
      - name: cloud-controller-manager
        # với các provider dạng in-tree, chúng ta dùng registry.k8s.io/cloud-controller-manager
        # image này có thể được thay bằng bất kỳ image nào khác cho các provider dạng out-of-tree
        image: registry.k8s.io/cloud-controller-manager:v1.8.0
        command:
        - /usr/local/bin/cloud-controller-manager
        - --cloud-provider=[YOUR_CLOUD_PROVIDER]  # Thêm cloud provider của bạn vào đây!
        - --leader-elect=true
        - --use-service-account-credentials
        # các flag này sẽ khác nhau tùy từng cloud provider
        - --allocate-node-cidrs=true
        - --configure-cloud-routes=true
        - --cluster-cidr=172.17.0.0/16
      tolerations:
      # toleration này là bắt buộc để CCM có thể tự khởi động (bootstrap)
      - key: node.cloudprovider.kubernetes.io/uninitialized
        value: "true"
        effect: NoSchedule
      # các toleration này để DaemonSet có thể chạy được trên các node control plane
      # hãy xóa chúng nếu các node control plane của bạn không nên chạy Pod
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      # phần này để giới hạn CCM chỉ chạy trên các node master
      # node selector có thể khác nhau tùy theo thiết lập cluster của bạn
      nodeSelector:
        node-role.kubernetes.io/master: ""
```

## Hạn chế (Limitations)

Việc chạy cloud controller manager đi kèm một vài hạn chế có thể gặp phải. Mặc dù các hạn chế
này đang được giải quyết trong các bản phát hành sắp tới, điều quan trọng là bạn cần nhận thức
được chúng đối với các workload trong môi trường sản xuất (production).

### Hỗ trợ Volume (Support for Volumes)

Cloud controller manager không triển khai bất kỳ volume controller nào có trong
`kube-controller-manager`, vì việc tích hợp volume cũng đòi hỏi sự phối hợp với các kubelet. Khi
CSI (container storage interface — giao diện lưu trữ container) tiếp tục phát triển và hỗ trợ
mạnh hơn cho các flex volume plugin được bổ sung, phần hỗ trợ cần thiết sẽ được thêm vào cloud
controller manager để các nền tảng cloud có thể tích hợp đầy đủ với volume. Tìm hiểu thêm về các
CSI volume plugin dạng out-of-tree
[tại đây](https://github.com/kubernetes/features/issues/178).

### Khả năng mở rộng (Scalability)

cloud-controller-manager truy vấn các API của cloud provider để truy xuất thông tin cho tất cả
các node. Với các cluster rất lớn, hãy cân nhắc các điểm nghẽn (bottleneck) có thể xảy ra, chẳng
hạn như yêu cầu tài nguyên và giới hạn tần suất gọi API (API rate limiting).

### Vấn đề con gà và quả trứng (Chicken and Egg)

Mục tiêu của dự án cloud controller manager là tách rời việc phát triển các tính năng cloud khỏi
dự án Kubernetes lõi. Đáng tiếc là nhiều khía cạnh của dự án Kubernetes vốn giả định rằng các
tính năng của cloud provider được tích hợp chặt chẽ vào dự án. Hệ quả là việc áp dụng kiến trúc
mới này có thể tạo ra một số tình huống trong đó một yêu cầu (request) đang cần thông tin từ
cloud provider, nhưng cloud controller manager lại không thể trả về thông tin đó nếu yêu cầu ban
đầu chưa hoàn tất.

Một ví dụ điển hình là tính năng TLS bootstrapping trong kubelet. TLS bootstrapping giả định
rằng kubelet có khả năng hỏi cloud provider (hoặc một dịch vụ metadata cục bộ) về tất cả các
loại địa chỉ của nó (private, public, v.v.), nhưng cloud controller manager không thể thiết lập
các loại địa chỉ của một node nếu bản thân nó chưa được khởi tạo, mà việc khởi tạo này lại đòi
hỏi kubelet phải có certificate TLS để giao tiếp với apiserver.

Khi sáng kiến này tiếp tục phát triển, các thay đổi sẽ được thực hiện để giải quyết những vấn đề
này trong các bản phát hành sắp tới.

## Tiếp theo (What's next)

Để xây dựng và phát triển cloud controller manager của riêng bạn, hãy đọc
[Phát triển Cloud Controller Manager](203-developing-cloud-controller-manager-vi.md).
