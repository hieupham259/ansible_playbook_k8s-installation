# EndpointSlices

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/>
>
> EndpointSlice API là cơ chế mà Kubernetes dùng để cho phép Service của bạn
> mở rộng quy mô nhằm xử lý số lượng lớn backend, đồng thời cho phép cluster
> cập nhật danh sách các backend khỏe mạnh của nó một cách hiệu quả.

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.21 [stable]`

EndpointSlice theo dõi địa chỉ IP của các endpoint backend.
EndpointSlice thường được gắn với một Service, và các endpoint backend
thường đại diện cho các Pod.

## EndpointSlice API {#endpointslice-resource}

Trong Kubernetes, một EndpointSlice chứa các tham chiếu tới một tập các
endpoint mạng (network endpoint). Control plane tự động tạo các EndpointSlice
cho bất kỳ Service nào của Kubernetes có chỉ định selector. Các EndpointSlice này bao gồm
tham chiếu tới tất cả các Pod khớp với selector của Service. EndpointSlice nhóm
các endpoint mạng lại với nhau theo tổ hợp duy nhất của họ địa chỉ IP (IP family), giao thức,
số port và tên Service.
Tên của một đối tượng EndpointSlice phải là một
[tên DNS subdomain](./17-names-vi.md) hợp lệ.

Ví dụ, dưới đây là một đối tượng EndpointSlice mẫu, thuộc sở hữu của Service
Kubernetes tên `example`.

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: example-abc
  labels:
    kubernetes.io/service-name: example
addressType: IPv4
ports:
  - name: http
    protocol: TCP
    port: 80
endpoints:
  - addresses:
      - "10.1.2.3"
    conditions:
      ready: true
    hostname: pod-1
    nodeName: node-1
    zone: us-west2-a
```

Theo mặc định, control plane tạo và quản lý các EndpointSlice sao cho mỗi
EndpointSlice có không quá 100 endpoint. Bạn có thể cấu hình điều này bằng flag
`--max-endpoints-per-slice`
của kube-controller-manager,
với giá trị tối đa lên tới 1000.

EndpointSlice đóng vai trò là nguồn dữ liệu tin cậy (source of truth) cho
kube-proxy trong việc
định tuyến lưu lượng nội bộ như thế nào.

### Các loại địa chỉ (Address types)

EndpointSlice hỗ trợ hai loại địa chỉ:

* IPv4
* IPv6

Mỗi đối tượng `EndpointSlice` đại diện cho một loại địa chỉ IP cụ thể. Nếu bạn có
một Service khả dụng qua cả IPv4 lẫn IPv6, sẽ có ít nhất hai
đối tượng `EndpointSlice` (một cho IPv4, và một cho IPv6).

### Các condition (Conditions)

EndpointSlice API lưu trữ các condition về endpoint mà có thể hữu ích cho bên tiêu thụ (consumer).
Ba condition đó là `serving`, `terminating` và `ready`.

#### Serving

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.26 [stable]`

Condition `serving` cho biết endpoint hiện đang phục vụ các phản hồi, và
do đó nó nên được dùng làm đích cho lưu lượng của Service. Với các endpoint được hậu thuẫn bởi một Pod,
condition này ánh xạ tới condition `Ready` của Pod đó.

#### Terminating

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.26 [stable]`

Condition `terminating` cho biết endpoint đang trong quá trình
kết thúc (terminating). Với các endpoint được hậu thuẫn bởi một Pod, condition này được đặt khi
Pod lần đầu bị xóa (tức là khi nó nhận được timestamp
xóa, nhưng nhiều khả năng là trước khi các container của Pod thoát).

Các service proxy thông thường sẽ bỏ qua các endpoint đang `terminating`,
nhưng chúng có thể định tuyến lưu lượng tới các endpoint vừa `serving` vừa
`terminating` nếu tất cả các endpoint khả dụng đều đang `terminating`. (Điều này
giúp đảm bảo không có lưu lượng nào của Service bị mất trong quá trình cập nhật cuốn chiếu (rolling update)
các Pod bên dưới.)

#### Ready

Condition `ready` về bản chất là một cách viết tắt cho việc kiểm tra
"`serving` và không `terminating`" (tuy nhiên nó cũng sẽ luôn là
`true` đối với các Service có `spec.publishNotReadyAddresses` được đặt là
`true`).

### Thông tin topology (Topology information) {#topology}

Mỗi endpoint trong một EndpointSlice có thể chứa thông tin topology liên quan.
Thông tin topology bao gồm vị trí của endpoint và thông tin
về Node cùng zone tương ứng. Những thông tin này có trong các
trường sau của từng endpoint trên EndpointSlice:

* `nodeName` - Tên của Node mà endpoint này nằm trên đó.
* `zone` - Zone mà endpoint này thuộc về.

### Quản lý (Management)

Thông thường nhất, control plane (cụ thể là controller quản lý endpoint slice)
tạo và quản lý các đối tượng EndpointSlice. Có nhiều trường hợp sử dụng khác cho
EndpointSlice, chẳng hạn các hiện thực service mesh, có thể dẫn đến việc các
thực thể hoặc controller khác quản lý thêm những tập EndpointSlice khác.

Để đảm bảo nhiều thực thể có thể quản lý EndpointSlice mà không can thiệp
lẫn nhau, Kubernetes định nghĩa label
`endpointslice.kubernetes.io/managed-by`, cho biết thực thể đang quản lý
một EndpointSlice.
Endpoint slice controller đặt `endpointslice-controller.k8s.io` làm giá trị
cho label này trên tất cả các EndpointSlice mà nó quản lý. Các thực thể khác quản lý
EndpointSlice cũng nên đặt một giá trị duy nhất cho label này.

### Quyền sở hữu (Ownership)

Trong hầu hết các trường hợp sử dụng, EndpointSlice thuộc sở hữu của Service mà đối tượng
endpoint slice đó theo dõi các endpoint thay cho nó. Quyền sở hữu này được biểu thị bằng một
owner reference trên mỗi EndpointSlice, cùng với label `kubernetes.io/service-name`
cho phép tra cứu đơn giản tất cả các EndpointSlice thuộc về một Service.

### Phân bổ các EndpointSlice (Distribution of EndpointSlices)

Mỗi EndpointSlice có một tập port áp dụng cho tất cả các endpoint trong
tài nguyên đó. Khi dùng port có tên (named port) cho một Service, các Pod có thể có
số port đích khác nhau cho cùng một named port, đòi hỏi những
EndpointSlice khác nhau.

Control plane cố gắng lấp đầy các EndpointSlice nhiều nhất có thể, nhưng không
chủ động tái cân bằng chúng. Logic khá đơn giản:

1. Lặp qua các EndpointSlice hiện có, loại bỏ những endpoint không còn
   cần thiết và cập nhật những endpoint khớp đã có thay đổi.
2. Lặp qua các EndpointSlice đã bị sửa đổi ở bước thứ nhất và
   lấp đầy chúng bằng bất kỳ endpoint mới nào cần thêm.
3. Nếu vẫn còn endpoint mới cần thêm, thử đưa chúng vào một slice
   chưa bị thay đổi trước đó và/hoặc tạo các slice mới.

Điều quan trọng là, bước thứ ba ưu tiên việc hạn chế số lần cập nhật EndpointSlice hơn là
một sự phân bổ EndpointSlice được lấp đầy hoàn hảo. Ví dụ, nếu có 10
endpoint mới cần thêm và 2 EndpointSlice mỗi cái còn chỗ cho 5 endpoint nữa,
cách tiếp cận này sẽ tạo một EndpointSlice mới thay vì lấp đầy 2
EndpointSlice hiện có. Nói cách khác, việc tạo một EndpointSlice duy nhất
được ưu tiên hơn so với việc cập nhật nhiều EndpointSlice.

Vì kube-proxy chạy trên mỗi Node và theo dõi (watch) các EndpointSlice, mỗi thay đổi
đối với một EndpointSlice trở nên tương đối tốn kém do nó sẽ được truyền tới
mọi Node trong cluster. Cách tiếp cận này nhằm hạn chế số lượng
thay đổi cần gửi tới mọi Node, ngay cả khi nó có thể dẫn đến nhiều
EndpointSlice không được lấp đầy.

Trên thực tế, sự phân bổ chưa lý tưởng này hiếm khi xảy ra. Hầu hết các thay đổi
được EndpointSlice controller xử lý sẽ đủ nhỏ để vừa với một
EndpointSlice hiện có, và nếu không, thì đằng nào một EndpointSlice mới nhiều khả năng cũng sẽ
sớm cần đến. Cập nhật cuốn chiếu (rolling update) các Deployment cũng mang lại một quá trình
đóng gói lại tự nhiên cho các EndpointSlice khi tất cả các Pod cùng những endpoint tương ứng
của chúng được thay thế.

### Các endpoint trùng lặp (Duplicate endpoints)

Do bản chất của các thay đổi EndpointSlice, các endpoint có thể được biểu diễn trong
nhiều hơn một EndpointSlice cùng một lúc. Điều này xảy ra một cách tự nhiên vì các thay đổi đối với
những đối tượng EndpointSlice khác nhau có thể đến watch / cache của client Kubernetes
vào những thời điểm khác nhau.

> **Ghi chú:**
> Các client của EndpointSlice API phải lặp qua tất cả các EndpointSlice hiện có
> gắn với một Service và xây dựng một danh sách đầy đủ các endpoint mạng duy nhất. Điều
> quan trọng cần nhắc đến là các endpoint có thể bị trùng lặp trong những EndpointSlice khác nhau.
>
> Bạn có thể tìm thấy một hiện thực tham khảo về cách thực hiện việc gộp
> và khử trùng lặp endpoint này trong phần mã `EndpointSliceCache` bên trong `kube-proxy`.

### Phản chiếu EndpointSlice (EndpointSlice mirroring)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.33 [deprecated]`

EndpointSlice API là sự thay thế cho Endpoints API cũ hơn. Để
giữ tương thích với các controller cũ và các workload của người dùng vốn
kỳ vọng kube-proxy
định tuyến lưu lượng dựa trên các tài nguyên Endpoints, control plane
của cluster phản chiếu (mirror) hầu hết các tài nguyên Endpoints do người dùng tạo sang các
EndpointSlice tương ứng.

(Tuy nhiên, tính năng này, giống như phần còn lại của Endpoints API, đã
bị loại bỏ dần (deprecated). Người dùng muốn tự tay chỉ định endpoint cho các Service
không có selector nên làm điều đó bằng cách tạo trực tiếp các tài nguyên EndpointSlice,
thay vì tạo các tài nguyên Endpoints rồi để chúng được
phản chiếu.)

Control plane phản chiếu các tài nguyên Endpoints trừ khi:

* tài nguyên Endpoints có label `endpointslice.kubernetes.io/skip-mirror`
  được đặt là `true`.
* tài nguyên Endpoints có annotation `control-plane.alpha.kubernetes.io/leader`.
* tài nguyên Service tương ứng không tồn tại.
* tài nguyên Service tương ứng có selector khác nil.

Từng tài nguyên Endpoints riêng lẻ có thể được chuyển thành nhiều EndpointSlice. Điều này
sẽ xảy ra nếu một tài nguyên Endpoints có nhiều subset hoặc bao gồm các endpoint
với nhiều họ IP (IPv4 và IPv6). Tối đa 1000 địa chỉ mỗi
subset sẽ được phản chiếu sang các EndpointSlice.

## Tiếp theo (What's next)

* Làm theo hướng dẫn thực hành [Kết nối ứng dụng với Service (Connecting Applications with Services)](https://kubernetes.io/docs/tutorials/services/connect-applications-service/)
* Đọc [tài liệu tham khảo API](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/endpoint-slice-v1/) cho EndpointSlice API
* Đọc [tài liệu tham khảo API](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/endpoints-v1/) cho Endpoints API
