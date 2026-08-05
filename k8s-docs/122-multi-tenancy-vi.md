# Đa người thuê (Multi-tenancy)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/multi-tenancy/>

Trang này cung cấp cái nhìn tổng quan về các lựa chọn cấu hình sẵn có và các thực hành tốt nhất cho
mô hình đa người thuê (multi-tenancy) trên cluster.

Việc chia sẻ cluster giúp tiết kiệm chi phí và đơn giản hóa việc quản trị. Tuy nhiên, chia sẻ cluster cũng
đặt ra những thách thức như bảo mật, tính công bằng, và việc quản lý các _"hàng xóm ồn ào" (noisy neighbors)_.

Cluster có thể được chia sẻ theo nhiều cách. Trong một số trường hợp, các ứng dụng khác nhau có thể chạy trong cùng một
cluster. Trong các trường hợp khác, nhiều instance của cùng một ứng dụng có thể chạy trong cùng một cluster,
mỗi instance cho một người dùng cuối. Tất cả các kiểu chia sẻ này thường được mô tả bằng thuật ngữ bao trùm
_đa người thuê (multi-tenancy)_.

Mặc dù Kubernetes không có các khái niệm hạng nhất (first-class) về người dùng cuối hay người thuê (tenant), nó cung cấp nhiều
tính năng giúp quản lý các yêu cầu thuê (tenancy) khác nhau. Các tính năng này được thảo luận bên dưới.

## Các trường hợp sử dụng (Use cases)

Bước đầu tiên để xác định cách chia sẻ cluster của bạn là hiểu rõ trường hợp sử dụng của mình, để bạn có thể
đánh giá các mẫu hình (pattern) và công cụ sẵn có. Nhìn chung, đa người thuê trong các cluster Kubernetes rơi vào
hai nhóm lớn, mặc dù cũng có thể có nhiều biến thể và mô hình lai (hybrid).

### Nhiều team (Multiple teams)

Một hình thức đa người thuê phổ biến là chia sẻ cluster giữa nhiều team trong một
tổ chức, mỗi team có thể vận hành một hoặc nhiều workload. Các workload này thường cần
giao tiếp với nhau, và với các workload khác nằm trên cùng cluster hoặc trên các cluster khác.

Trong kịch bản này, các thành viên trong team thường có quyền truy cập trực tiếp vào các tài nguyên Kubernetes qua các công cụ
như `kubectl`, hoặc truy cập gián tiếp qua các controller GitOps hoặc các loại công cụ tự động hóa
phát hành (release automation) khác. Thường có một mức độ tin cậy nhất định giữa thành viên của các team khác nhau, nhưng
các chính sách Kubernetes như RBAC, hạn ngạch (quota) và network policy là thiết yếu để chia sẻ
cluster một cách an toàn và công bằng.

### Nhiều khách hàng (Multiple customers)

Hình thức đa người thuê lớn còn lại thường liên quan đến một nhà cung cấp Software-as-a-Service (SaaS)
chạy nhiều instance của một workload cho các khách hàng. Mô hình kinh doanh này gắn liền
với kiểu triển khai này đến mức nhiều người gọi nó là "SaaS tenancy". Tuy nhiên, một thuật ngữ
chính xác hơn có thể là "đa người thuê theo khách hàng (multi-customer tenancy)", vì các nhà cung cấp SaaS cũng có thể dùng các mô hình triển khai khác,
và mô hình triển khai này cũng có thể được dùng ngoài phạm vi SaaS.

Trong kịch bản này, khách hàng không có quyền truy cập vào cluster; Kubernetes vô hình
dưới góc nhìn của họ và chỉ được nhà cung cấp dùng để quản lý các workload. Tối ưu chi phí
thường là mối quan tâm then chốt, và các chính sách Kubernetes được dùng để đảm bảo các workload
được cách ly chặt chẽ với nhau.

## Thuật ngữ (Terminology)

### Tenant (Tenants)

Khi thảo luận về đa người thuê trong Kubernetes, không có định nghĩa duy nhất nào cho "tenant" (người thuê).
Thay vào đó, định nghĩa về tenant sẽ thay đổi tùy theo việc bạn đang nói về mô hình đa team hay
đa khách hàng.

Trong cách dùng đa team, một tenant thường là một team, trong đó mỗi team thường triển khai một số lượng nhỏ
workload và số lượng này tăng theo độ phức tạp của dịch vụ. Tuy nhiên, chính định nghĩa về
"team" cũng có thể mơ hồ, vì các team có thể được tổ chức thành các bộ phận cấp cao hơn hoặc chia nhỏ
thành các team nhỏ hơn.

Ngược lại, nếu mỗi team triển khai các workload chuyên biệt cho từng khách hàng mới, thì họ đang dùng
mô hình thuê theo khách hàng (multi-customer). Trong trường hợp này, một "tenant" đơn giản là một nhóm người dùng cùng chia sẻ
một workload duy nhất. Nhóm này có thể lớn bằng cả một công ty, hoặc nhỏ bằng một team duy nhất của
công ty đó.

Trong nhiều trường hợp, cùng một tổ chức có thể dùng cả hai định nghĩa về "tenant" trong các ngữ cảnh khác nhau.
Ví dụ, một team nền tảng (platform team) có thể cung cấp các dịch vụ dùng chung như công cụ bảo mật và cơ sở dữ liệu cho
nhiều "khách hàng" nội bộ, và một nhà cung cấp SaaS cũng có thể có nhiều team cùng chia sẻ một cluster
phát triển. Cuối cùng, các kiến trúc lai cũng khả thi, chẳng hạn một nhà cung cấp SaaS dùng
kết hợp giữa các workload riêng cho từng khách hàng đối với dữ liệu nhạy cảm, cùng với các dịch vụ
dùng chung đa người thuê.

![Một cluster với các mô hình thuê (tenancy) cùng tồn tại](https://kubernetes.io/images/docs/multi-tenancy.png)

*Hình. Một cluster minh họa các mô hình thuê (tenancy) cùng tồn tại*

### Cách ly (Isolation)

Có nhiều cách để thiết kế và xây dựng các giải pháp đa người thuê với Kubernetes. Mỗi phương pháp
đi kèm với tập các đánh đổi riêng, ảnh hưởng đến mức độ cách ly, công sức hiện thực,
độ phức tạp vận hành, và chi phí dịch vụ.

Một cluster Kubernetes bao gồm một control plane chạy phần mềm Kubernetes, và một data plane
gồm các worker node nơi các workload của tenant được thực thi dưới dạng pod. Việc cách ly tenant có thể
được áp dụng ở cả control plane lẫn data plane tùy theo yêu cầu của tổ chức.

Mức độ cách ly được cung cấp đôi khi được mô tả bằng các thuật ngữ như đa người thuê "cứng" (hard multi-tenancy),
ngụ ý cách ly mạnh, và đa người thuê "mềm" (soft multi-tenancy), ngụ ý cách ly yếu hơn. Cụ thể,
"hard" multi-tenancy thường được dùng để mô tả các trường hợp mà các tenant không tin tưởng lẫn nhau,
thường xét từ góc độ bảo mật và chia sẻ tài nguyên (ví dụ: phòng chống các cuộc tấn công như đánh cắp dữ liệu
(data exfiltration) hay DoS). Vì data plane thường có bề mặt tấn công (attack surface) lớn hơn nhiều, "hard"
multi-tenancy thường đòi hỏi sự chú ý đặc biệt đến việc cách ly data plane, mặc dù việc cách ly
control plane cũng vẫn rất quan trọng.

Tuy nhiên, các thuật ngữ "hard" và "soft" thường dễ gây nhầm lẫn, vì không có một định nghĩa duy nhất nào
phù hợp với mọi người dùng. Thay vào đó, độ "cứng" hay "mềm" nên được hiểu như một phổ (spectrum) rộng,
với nhiều kỹ thuật khác nhau có thể được dùng để duy trì các kiểu cách ly khác nhau
trong cluster của bạn, dựa trên các yêu cầu của bạn.

Trong các trường hợp cực đoan hơn, có thể sẽ dễ dàng hơn hoặc cần thiết phải từ bỏ hoàn toàn mọi hình thức chia sẻ ở cấp cluster và
gán cho mỗi tenant một cluster chuyên dụng, thậm chí có thể chạy trên phần cứng chuyên dụng nếu máy ảo (VM)
không được xem là một ranh giới bảo mật đủ tốt. Việc này có thể dễ hơn với các cluster Kubernetes được quản lý (managed),
nơi chi phí tạo và vận hành cluster ít nhất được nhà cung cấp cloud gánh vác một phần.
Lợi ích của việc cách ly tenant mạnh hơn phải được cân nhắc so với chi phí và
độ phức tạp của việc quản lý nhiều cluster. [Multi-cluster SIG](https://git.k8s.io/community/sig-multicluster/README.md)
chịu trách nhiệm giải quyết các loại trường hợp sử dụng này.

Phần còn lại của trang này tập trung vào các kỹ thuật cách ly dùng cho các cluster Kubernetes dùng chung.
Tuy nhiên, ngay cả khi bạn đang cân nhắc dùng các cluster chuyên dụng, việc xem qua các khuyến nghị này
vẫn có giá trị, vì nó sẽ cho bạn sự linh hoạt để chuyển sang các cluster dùng chung trong tương lai nếu
nhu cầu hoặc năng lực của bạn thay đổi.

## Cách ly control plane (Control plane isolation)

Cách ly control plane đảm bảo rằng các tenant khác nhau không thể truy cập hoặc tác động đến
các tài nguyên Kubernetes API của nhau.

### Namespace (Namespaces)

Trong Kubernetes, một Namespace cung cấp một
cơ chế để cách ly các nhóm tài nguyên API trong cùng một cluster. Sự cách ly này có hai
khía cạnh then chốt:

1. Tên các đối tượng trong một namespace có thể trùng với tên trong các namespace khác, tương tự như các file trong
   các thư mục. Điều này cho phép các tenant đặt tên tài nguyên của mình mà không cần quan tâm
   các tenant khác đang làm gì.

2. Nhiều chính sách bảo mật của Kubernetes có phạm vi theo namespace. Ví dụ, RBAC Role và Network
   Policy là các tài nguyên có phạm vi namespace. Sử dụng RBAC, User và Service Account có thể
   bị giới hạn trong một namespace.

Trong môi trường đa người thuê, một Namespace giúp phân đoạn workload của tenant thành một đơn vị quản lý
logic và tách biệt. Trên thực tế, một thực hành phổ biến là cách ly mỗi workload trong namespace riêng
của nó, ngay cả khi nhiều workload được vận hành bởi cùng một tenant. Điều này đảm bảo mỗi
workload có danh tính (identity) riêng và có thể được cấu hình với một chính sách bảo mật phù hợp.

Mô hình cách ly bằng namespace đòi hỏi phải cấu hình thêm một số tài nguyên Kubernetes khác,
các networking plugin, và tuân thủ các thực hành bảo mật tốt nhất để cách ly các workload của tenant một cách đúng đắn.
Các cân nhắc này được thảo luận bên dưới.

### Kiểm soát truy cập (Access controls)

Loại cách ly quan trọng nhất đối với control plane là phân quyền (authorization). Nếu các team hoặc các
workload của họ có thể truy cập hoặc sửa đổi các tài nguyên API của nhau, họ có thể thay đổi hoặc vô hiệu hóa tất cả các
loại chính sách khác, qua đó vô hiệu hóa mọi sự bảo vệ mà các chính sách đó có thể mang lại. Do đó, điều
tối quan trọng là đảm bảo mỗi tenant chỉ có quyền truy cập phù hợp vào những namespace họ cần,
và không hơn. Đây được gọi là "Nguyên tắc đặc quyền tối thiểu (Principle of Least Privilege)".

Kiểm soát truy cập dựa trên vai trò (RBAC) thường được dùng để thực thi phân quyền trong control plane
của Kubernetes, cho cả người dùng lẫn workload (service account).
[Role](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole) và
[RoleBinding](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#rolebinding-and-clusterrolebinding) là
các đối tượng Kubernetes được dùng ở cấp namespace để thực thi kiểm soát truy cập trong
ứng dụng của bạn; các đối tượng tương tự tồn tại cho việc phân quyền truy cập các đối tượng cấp cluster, tuy nhiên chúng
ít hữu ích hơn đối với các cluster đa người thuê.

Trong môi trường đa team, RBAC phải được dùng để giới hạn quyền truy cập của các tenant vào các
namespace phù hợp, và đảm bảo rằng các tài nguyên phạm vi toàn cluster chỉ có thể được truy cập hoặc sửa đổi bởi những người dùng
đặc quyền như quản trị viên cluster.

Nếu một chính sách rốt cuộc cấp cho người dùng nhiều quyền hơn mức họ cần, đây có thể là một tín hiệu cho thấy
namespace chứa các tài nguyên bị ảnh hưởng nên được tái cấu trúc thành các namespace
chi tiết hơn (finer-grained). Các công cụ quản lý namespace có thể đơn giản hóa việc quản lý các namespace
chi tiết này bằng cách áp dụng các chính sách RBAC chung cho các namespace khác nhau, trong khi vẫn cho phép
các chính sách chi tiết ở những nơi cần thiết.

### Hạn ngạch (Quotas)

Các workload Kubernetes tiêu thụ tài nguyên của node, như CPU và bộ nhớ. Trong môi trường đa người thuê,
bạn có thể dùng [Resource Quota](https://kubernetes.io/docs/concepts/policy/resource-quotas/) để quản lý mức sử dụng tài nguyên
của các workload của tenant. Với trường hợp sử dụng nhiều team, khi các tenant có quyền truy cập Kubernetes
API, bạn có thể dùng resource quota để giới hạn số lượng tài nguyên API (ví dụ: số lượng
Pod, hay số lượng ConfigMap) mà một tenant có thể tạo. Các giới hạn về số lượng đối tượng đảm bảo
tính công bằng và nhằm tránh các vấn đề _hàng xóm ồn ào (noisy neighbor)_ ảnh hưởng đến các tenant khác cùng chia sẻ
một control plane.

Resource quota là các đối tượng thuộc phạm vi namespace. Bằng cách ánh xạ các tenant vào các namespace, quản trị viên cluster có thể dùng
quota để đảm bảo một tenant không thể độc chiếm tài nguyên của cluster hay làm quá tải control plane
của nó. Các công cụ quản lý namespace đơn giản hóa việc quản trị quota. Ngoài ra, trong khi
quota của Kubernetes chỉ áp dụng bên trong một namespace duy nhất, một số công cụ quản lý namespace cho phép
các nhóm namespace chia sẻ quota, mang lại cho quản trị viên sự linh hoạt lớn hơn nhiều với ít công sức hơn
so với quota tích hợp sẵn.

Quota ngăn một tenant đơn lẻ tiêu thụ vượt phần tài nguyên được cấp phát cho họ,
qua đó giảm thiểu vấn đề "hàng xóm ồn ào", khi một tenant gây ảnh hưởng tiêu cực đến hiệu năng
workload của các tenant khác.

Khi bạn áp dụng một quota lên namespace, Kubernetes yêu cầu bạn cũng phải chỉ định request và
limit tài nguyên cho mỗi container. Limit là giới hạn trên cho lượng tài nguyên mà một container
có thể tiêu thụ. Các container cố tiêu thụ tài nguyên vượt quá limit đã cấu hình sẽ
bị hạn chế (throttle) hoặc bị kill, tùy theo loại tài nguyên. Khi resource request được đặt thấp hơn
limit, mỗi container được đảm bảo lượng đã yêu cầu, nhưng vẫn có thể còn một số
khả năng gây ảnh hưởng chéo giữa các workload.

Quota không thể bảo vệ chống lại mọi loại chia sẻ tài nguyên, chẳng hạn như lưu lượng mạng.
Cách ly node (được mô tả bên dưới) có thể là một giải pháp tốt hơn cho vấn đề này.

## Cách ly data plane (Data Plane Isolation) {#data-plane-isolation}

Cách ly data plane đảm bảo rằng các pod và workload của các tenant khác nhau được cách ly
một cách đầy đủ.

### Cách ly mạng (Network isolation)

Theo mặc định, tất cả các pod trong một cluster Kubernetes được phép giao tiếp với nhau, và mọi
lưu lượng mạng đều không được mã hóa. Điều này có thể dẫn đến các lỗ hổng bảo mật khi lưu lượng bị
gửi nhầm hoặc bị gửi có chủ đích xấu đến một đích không mong muốn, hoặc bị chặn bắt bởi một workload trên
một node đã bị xâm nhập.

Giao tiếp pod-với-pod có thể được kiểm soát bằng [Network Policy](./84-network-policies-vi.md),
cơ chế giới hạn giao tiếp giữa các pod bằng các label của namespace hoặc các dải địa chỉ IP.
Trong môi trường đa người thuê đòi hỏi cách ly mạng nghiêm ngặt giữa các tenant, nên bắt đầu
với một chính sách mặc định từ chối giao tiếp giữa các pod, kèm theo một quy tắc khác
cho phép tất cả các pod truy vấn DNS server để phân giải tên. Với một chính sách mặc định như vậy,
bạn có thể bắt đầu thêm các quy tắc cho phép giao tiếp bên trong một namespace.
Cũng khuyến nghị không dùng label selector rỗng '{}' cho trường namespaceSelector trong định nghĩa network policy,
trong trường hợp cần cho phép lưu lượng giữa các namespace.
Mô hình này có thể được tinh chỉnh thêm khi cần. Lưu ý rằng điều này chỉ áp dụng cho các pod trong cùng một
control plane; các pod thuộc các control plane ảo khác nhau không thể nói chuyện với nhau qua
mạng của Kubernetes.

Các công cụ quản lý namespace có thể đơn giản hóa việc tạo các network policy mặc định hoặc dùng chung.
Ngoài ra, một số công cụ này cho phép bạn thực thi một tập nhất quán các label của namespace trên toàn
cluster, đảm bảo chúng là một cơ sở đáng tin cậy cho các chính sách của bạn.

> **Cảnh báo:**
>
> Network policy yêu cầu một [CNI plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/#cni)
> hỗ trợ việc hiện thực network policy. Nếu không, các tài nguyên NetworkPolicy sẽ bị bỏ qua.

Mức cách ly mạng cao cấp hơn có thể được cung cấp bởi các service mesh, vốn cung cấp các chính sách
Layer 7 theo mô hình OSI dựa trên danh tính workload (workload identity), bên cạnh namespace. Các chính sách cấp cao hơn này có thể
giúp việc quản lý đa người thuê dựa trên namespace dễ dàng hơn, đặc biệt khi nhiều namespace
được dành riêng cho một tenant duy nhất. Chúng thường cũng cung cấp mã hóa bằng mutual TLS, bảo vệ
dữ liệu của bạn ngay cả khi có một node bị xâm nhập, và hoạt động trên các cluster chuyên dụng hoặc cluster ảo.
Tuy nhiên, chúng có thể phức tạp hơn đáng kể trong việc quản lý và có thể không phù hợp với mọi người dùng.

### Cách ly lưu trữ (Storage isolation)

Kubernetes cung cấp nhiều loại volume có thể được dùng làm lưu trữ bền vững (persistent storage) cho các workload.
Vì lý do bảo mật và cách ly dữ liệu, [cấp phát volume động (dynamic volume provisioning)](./98-dynamic-provisioning-vi.md)
được khuyến nghị, và nên tránh các loại volume sử dụng tài nguyên của node.

[StorageClass](./96-storage-classes-vi.md) cho phép bạn mô tả các "lớp" (class) lưu trữ tùy chỉnh
do cluster của bạn cung cấp, dựa trên các mức chất lượng dịch vụ (quality-of-service), chính sách sao lưu, hoặc các chính sách
tùy chỉnh do quản trị viên cluster quyết định.

Các Pod có thể yêu cầu lưu trữ bằng một [PersistentVolumeClaim](./92-persistent-volumes-vi.md).
PersistentVolumeClaim là một tài nguyên thuộc phạm vi namespace, cho phép cách ly các phần của hệ thống
lưu trữ và dành riêng chúng cho các tenant bên trong cluster Kubernetes dùng chung.
Tuy nhiên, điều quan trọng cần lưu ý là PersistentVolume là một tài nguyên phạm vi toàn cluster và có
vòng đời độc lập với các workload và namespace.

Ví dụ, bạn có thể cấu hình một StorageClass riêng cho mỗi tenant và dùng nó để tăng cường sự cách ly.
Nếu một StorageClass được dùng chung, bạn nên đặt [chính sách thu hồi (reclaim policy) là `Delete`](https://kubernetes.io/docs/concepts/storage/storage-classes/#reclaim-policy)
để đảm bảo một PersistentVolume không thể bị tái sử dụng giữa các namespace khác nhau.

### Sandbox cho container (Sandboxing containers)

Các pod Kubernetes bao gồm một hoặc nhiều container thực thi trên các worker node.
Container tận dụng ảo hóa cấp hệ điều hành (OS-level virtualization), do đó cung cấp một ranh giới cách ly yếu hơn so với
các máy ảo sử dụng ảo hóa dựa trên phần cứng.

Trong môi trường dùng chung, các lỗ hổng chưa được vá ở tầng ứng dụng và tầng hệ thống có thể bị
kẻ tấn công khai thác để thoát khỏi container (container breakout) và thực thi mã từ xa cho phép truy cập vào tài nguyên
của host. Trong một số ứng dụng, như Hệ quản trị nội dung (Content Management System - CMS), khách hàng có thể được phép
tải lên và thực thi các script hoặc mã không đáng tin cậy. Trong cả hai trường hợp, các cơ chế để cách ly
và bảo vệ workload hơn nữa bằng sự cách ly mạnh là điều đáng mong muốn.

Sandbox (hộp cát) cung cấp một cách để cách ly các workload chạy trong một cluster dùng chung. Nó thường bao gồm
việc chạy mỗi pod trong một môi trường thực thi riêng biệt như một máy ảo hoặc một
kernel không gian người dùng (userspace kernel). Sandbox thường được khuyến nghị khi bạn chạy mã không đáng tin cậy, khi các workload
được giả định là độc hại. Một phần lý do khiến kiểu cách ly này cần thiết là vì
các container là các tiến trình chạy trên một kernel dùng chung; chúng mount các hệ thống file như `/sys` và `/proc`
từ host bên dưới, khiến chúng kém an toàn hơn một ứng dụng chạy trên một máy ảo
có kernel riêng. Mặc dù các cơ chế kiểm soát như seccomp, AppArmor và SELinux có thể được
dùng để tăng cường bảo mật cho container, rất khó để áp dụng một tập quy tắc chung cho tất cả
các workload chạy trong một cluster dùng chung. Chạy workload trong môi trường sandbox giúp
cách ly host khỏi các cuộc thoát container (container escape), khi kẻ tấn công khai thác một lỗ hổng để giành quyền
truy cập vào hệ thống host và tất cả các tiến trình/file chạy trên host đó.

Máy ảo và kernel không gian người dùng là hai cách tiếp cận sandbox phổ biến.

### Cách ly node (Node Isolation)

Cách ly node là một kỹ thuật khác mà bạn có thể dùng để cách ly các workload của các tenant với nhau.
Với cách ly node, một tập các node được dành riêng để chạy các pod của một tenant cụ thể, và việc
trộn lẫn pod của các tenant bị cấm. Cấu hình này giảm vấn đề tenant ồn ào, vì
tất cả các pod chạy trên một node sẽ thuộc về một tenant duy nhất. Rủi ro lộ lọt thông tin
thấp hơn một chút với cách ly node, vì kẻ tấn công thoát được khỏi một container
sẽ chỉ có quyền truy cập vào các container và các volume được mount trên node đó.

Mặc dù workload của các tenant khác nhau chạy trên các node khác nhau, điều quan trọng cần
lưu ý là kubelet và (trừ khi dùng control plane ảo) API service vẫn là các dịch vụ
dùng chung. Một kẻ tấn công có kỹ năng có thể lợi dụng các quyền được gán cho kubelet hoặc các pod khác
chạy trên node để di chuyển ngang (lateral movement) trong cluster và giành quyền truy cập vào workload của các tenant
chạy trên các node khác. Nếu đây là mối lo lớn, hãy cân nhắc triển khai các biện pháp kiểm soát bù đắp
như seccomp, AppArmor hoặc SELinux, hoặc tìm hiểu việc sử dụng container sandbox hay tạo cluster
riêng cho mỗi tenant.

Cách ly node dễ tính toán hơn một chút về mặt hóa đơn (billing) so với sandbox cho container,
vì bạn có thể tính phí theo node thay vì theo pod. Nó cũng có ít vấn đề tương thích
và hiệu năng hơn, và có thể dễ hiện thực hơn so với sandbox cho container.
Ví dụ, các node của mỗi tenant có thể được cấu hình với các taint sao cho chỉ những pod có
toleration tương ứng mới chạy được trên đó. Sau đó, một mutating webhook có thể được dùng để tự động
thêm các toleration và node affinity vào các pod được triển khai vào các namespace của tenant, để chúng chạy trên
một tập node cụ thể được chỉ định cho tenant đó.

Cách ly node có thể được hiện thực bằng [pod node selector](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/).

## Các cân nhắc bổ sung (Additional Considerations)

Phần này thảo luận các cấu trúc và mẫu hình Kubernetes khác có liên quan đến đa người thuê.

### Độ ưu tiên và công bằng của API (API Priority and Fairness)

[Độ ưu tiên và công bằng của API (API priority and fairness)](https://kubernetes.io/docs/concepts/cluster-administration/flow-control/) là một tính năng của Kubernetes
cho phép bạn gán độ ưu tiên cho một số pod nhất định chạy trong cluster.
Khi một ứng dụng gọi Kubernetes API, API server sẽ đánh giá độ ưu tiên được gán cho pod đó.
Các lời gọi từ những pod có độ ưu tiên cao hơn được xử lý trước các lời gọi có độ ưu tiên thấp hơn.
Khi mức tranh chấp (contention) cao, các lời gọi có độ ưu tiên thấp hơn có thể được xếp hàng đợi cho đến khi server bớt bận, hoặc bạn
có thể từ chối các request.

Việc dùng API priority and fairness sẽ không phổ biến lắm trong các môi trường SaaS, trừ khi bạn
cho phép khách hàng chạy các ứng dụng tương tác với Kubernetes API, ví dụ như
một controller.

### Chất lượng dịch vụ (Quality-of-Service, QoS) {#qos}

Khi vận hành một ứng dụng SaaS, bạn có thể muốn khả năng cung cấp các bậc
Chất lượng dịch vụ (Quality-of-Service - QoS) khác nhau cho các tenant khác nhau. Ví dụ, bạn có thể có dịch vụ
freemium đi kèm ít đảm bảo hiệu năng và tính năng hơn, và một bậc dịch vụ trả phí với
các đảm bảo hiệu năng cụ thể. May mắn là có một số cấu trúc Kubernetes có thể
giúp bạn thực hiện điều này trong một cluster dùng chung, bao gồm QoS mạng, storage class, và độ ưu tiên
cùng cơ chế chiếm chỗ (preemption) của pod. Ý tưởng của mỗi cấu trúc này là cung cấp cho các tenant chất lượng
dịch vụ tương xứng với những gì họ đã trả tiền. Hãy bắt đầu với QoS mạng.

Thông thường, tất cả các pod trên một node dùng chung một network interface. Không có QoS mạng, một số pod có thể
tiêu thụ một phần băng thông sẵn có một cách bất công, gây thiệt hại cho các pod khác.
[Bandwidth plugin](https://www.cni.dev/plugins/current/meta/bandwidth/) của Kubernetes tạo một
[tài nguyên mở rộng (extended resource)](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#extended-resources)
cho mạng, cho phép bạn dùng các cấu trúc tài nguyên của Kubernetes, tức là request/limit, để
áp dụng giới hạn tốc độ (rate limit) cho các pod bằng cách sử dụng hàng đợi tc của Linux.
Lưu ý rằng plugin này được xem là thử nghiệm (experimental) theo tài liệu
[Network Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/#support-traffic-shaping)
và nên được kiểm thử kỹ lưỡng trước khi dùng trong môi trường production.

Với QoS lưu trữ, bạn có thể sẽ muốn tạo các storage class hoặc profile khác nhau với
các đặc tính hiệu năng khác nhau. Mỗi profile lưu trữ có thể được gắn với một
bậc dịch vụ khác nhau, được tối ưu cho các workload khác nhau như IO, độ dư thừa (redundancy), hay thông lượng (throughput).
Có thể cần thêm logic bổ sung để cho phép tenant gắn profile lưu trữ
phù hợp với workload của họ.

Cuối cùng, có [độ ưu tiên và chiếm chỗ của pod (pod priority and preemption)](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/),
nơi bạn có thể gán các giá trị ưu tiên cho pod. Khi lập lịch pod, scheduler sẽ thử
trục xuất (evict) các pod có độ ưu tiên thấp hơn khi không đủ tài nguyên để lập lịch cho các pod được
gán độ ưu tiên cao hơn. Nếu bạn có trường hợp sử dụng trong đó các tenant có các bậc dịch vụ khác nhau trong một
cluster dùng chung, ví dụ miễn phí và trả phí, bạn có thể muốn cấp độ ưu tiên cao hơn cho một số bậc nhất định
bằng tính năng này.

### DNS

Các cluster Kubernetes bao gồm một dịch vụ Hệ thống tên miền (Domain Name System - DNS) để cung cấp việc chuyển đổi từ tên
sang địa chỉ IP, cho tất cả các Service và Pod. Theo mặc định, dịch vụ DNS của Kubernetes cho phép tra cứu (lookup)
trên tất cả các namespace trong cluster.

Trong các môi trường đa người thuê nơi các tenant có thể truy cập pod và các tài nguyên Kubernetes khác, hoặc nơi
cần sự cách ly mạnh hơn, có thể cần ngăn các pod tra cứu các service ở các
Namespace khác.
Bạn có thể hạn chế tra cứu DNS liên namespace bằng cách cấu hình các quy tắc bảo mật cho dịch vụ DNS.
Ví dụ, CoreDNS (dịch vụ DNS mặc định của Kubernetes) có thể tận dụng metadata của Kubernetes
để giới hạn các truy vấn chỉ trong phạm vi các Pod và Service trong một namespace. Để biết thêm thông tin, đọc một
[ví dụ](https://github.com/coredns/policy#kubernetes-metadata-multi-tenancy-policy) về cách
cấu hình điều này trong tài liệu CoreDNS.

Khi mô hình [Control plane ảo cho mỗi tenant](#virtual-control-plane-per-tenant) được sử dụng, một dịch vụ DNS
phải được cấu hình cho mỗi tenant, hoặc phải dùng một dịch vụ DNS đa người thuê.
Đây là một ví dụ về một [phiên bản tùy biến của CoreDNS](https://github.com/kubernetes-sigs/cluster-api-provider-nested/blob/main/virtualcluster/doc/tenant-dns.md)
hỗ trợ nhiều tenant.

### Operator (Operators)

[Operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) là các controller của Kubernetes quản lý
các ứng dụng. Operator có thể đơn giản hóa việc quản lý nhiều instance của một ứng dụng, như
một dịch vụ cơ sở dữ liệu, điều này khiến chúng trở thành một khối xây dựng phổ biến trong trường hợp sử dụng đa người thuê
kiểu đa khách hàng (SaaS).

Các Operator được dùng trong môi trường đa người thuê nên tuân theo một tập hướng dẫn nghiêm ngặt hơn.
Cụ thể, Operator nên:

* Hỗ trợ tạo tài nguyên trong các namespace của các tenant khác nhau, thay vì chỉ trong namespace
  mà Operator được triển khai.
* Đảm bảo các Pod được cấu hình với request và limit tài nguyên, để đảm bảo việc lập lịch và tính công bằng.
* Hỗ trợ cấu hình Pod cho các kỹ thuật cách ly data plane như cách ly node và
  container sandbox.

## Các cách hiện thực (Implementations)

Có hai cách chính để chia sẻ một cluster Kubernetes cho đa người thuê: dùng Namespace
(tức là một Namespace cho mỗi tenant) hoặc ảo hóa control plane (tức là control plane
ảo cho mỗi tenant).

Trong cả hai trường hợp, việc cách ly data plane và quản lý các cân nhắc bổ sung như API
Priority and Fairness cũng được khuyến nghị.

Cách ly bằng namespace được Kubernetes hỗ trợ tốt, có chi phí tài nguyên không đáng kể, và cung cấp
các cơ chế cho phép các tenant tương tác với nhau một cách phù hợp, chẳng hạn cho phép giao tiếp
giữa các service. Tuy nhiên, nó có thể khó cấu hình, và không áp dụng được cho các tài nguyên Kubernetes
không thể thuộc namespace, chẳng hạn như Custom Resource Definition, Storage Class và Webhook.

Ảo hóa control plane cho phép cách ly các tài nguyên không thuộc namespace với cái giá là
mức sử dụng tài nguyên cao hơn phần nào và việc chia sẻ giữa các tenant khó khăn hơn. Đây là một lựa chọn tốt khi
cách ly bằng namespace là không đủ nhưng cluster chuyên dụng lại không mong muốn, do chi phí cao
của việc duy trì chúng (đặc biệt là on-prem) hoặc do overhead cao hơn và thiếu khả năng chia sẻ
tài nguyên của chúng. Tuy nhiên, ngay cả bên trong một control plane ảo hóa, bạn nhiều khả năng vẫn sẽ thấy lợi ích khi dùng
thêm các namespace.

Hai lựa chọn này được thảo luận chi tiết hơn trong các phần sau.

### Namespace cho mỗi tenant (Namespace per tenant)

Như đã đề cập trước đó, bạn nên cân nhắc cách ly mỗi workload trong namespace riêng của nó, ngay cả khi
bạn đang dùng các cluster chuyên dụng hoặc control plane ảo hóa. Điều này đảm bảo mỗi workload
chỉ có quyền truy cập vào các tài nguyên của chính nó, chẳng hạn ConfigMap và Secret, và cho phép bạn điều chỉnh
các chính sách bảo mật chuyên biệt cho từng workload. Ngoài ra, một thực hành tốt nhất là đặt cho mỗi
namespace những cái tên duy nhất trên toàn bộ đội hình (fleet) của bạn (tức là, ngay cả khi chúng ở các cluster
tách biệt), vì điều này cho bạn sự linh hoạt để chuyển đổi giữa cluster chuyên dụng và cluster dùng chung trong
tương lai, hoặc để dùng các công cụ đa cluster như service mesh.

Ngược lại, việc gán namespace ở cấp tenant, chứ không chỉ ở cấp workload, cũng có những lợi thế,
vì thường có các chính sách áp dụng cho tất cả các workload thuộc sở hữu của một
tenant duy nhất. Tuy nhiên, cách này lại đặt ra các vấn đề riêng của nó. Thứ nhất, nó khiến việc tùy chỉnh
chính sách cho từng workload riêng lẻ trở nên khó hoặc bất khả thi, và thứ hai, có thể sẽ khó xác định được một
mức "tenancy" duy nhất nên được cấp một namespace. Ví dụ, một tổ chức có thể có
các bộ phận, team và team con - vậy cấp nào nên được gán một namespace?

Một cách tiếp cận khả dĩ là tổ chức các namespace của bạn thành các cấu trúc phân cấp, và chia sẻ một số chính sách và
tài nguyên nhất định giữa chúng. Điều này có thể bao gồm việc quản lý label của namespace, vòng đời namespace,
quyền truy cập được ủy quyền, và hạn ngạch tài nguyên dùng chung giữa các namespace có liên quan. Các khả năng này có thể
hữu ích trong cả kịch bản đa team lẫn đa khách hàng.

### Control plane ảo cho mỗi tenant (Virtual control plane per tenant) {#virtual-control-plane-per-tenant}

Một hình thức cách ly control plane khác là sử dụng các phần mở rộng (extension) của Kubernetes để cung cấp cho mỗi tenant một
control plane ảo, cho phép phân đoạn các tài nguyên API phạm vi toàn cluster.
Các kỹ thuật [cách ly data plane](#data-plane-isolation) có thể được dùng cùng mô hình này để quản lý
các worker node một cách an toàn giữa các tenant.

Mô hình đa người thuê dựa trên control plane ảo mở rộng mô hình đa người thuê dựa trên namespace bằng cách
cung cấp cho mỗi tenant các thành phần control plane chuyên dụng, và do đó có toàn quyền kiểm soát
các tài nguyên phạm vi cluster và các dịch vụ bổ trợ (add-on). Các worker node được chia sẻ giữa tất cả các tenant, và được
quản lý bởi một cluster Kubernetes mà các tenant thông thường không thể truy cập.
Cluster này thường được gọi là _super-cluster_ (hoặc đôi khi là _host-cluster_).
Vì control plane của một tenant không gắn trực tiếp với các tài nguyên tính toán bên dưới, nó
được gọi là _control plane ảo (virtual control plane)_.

Một control plane ảo thường bao gồm Kubernetes API server, controller manager,
và kho dữ liệu etcd. Nó tương tác với super-cluster qua một controller đồng bộ metadata,
controller này điều phối các thay đổi giữa các control plane của tenant và control plane của
super-cluster.

Bằng cách sử dụng các control plane chuyên dụng cho từng tenant, hầu hết các vấn đề cách ly do việc chia sẻ một
API server giữa tất cả các tenant được giải quyết. Các ví dụ bao gồm hàng xóm ồn ào trong control plane,
phạm vi ảnh hưởng không kiểm soát được của các cấu hình chính sách sai, và xung đột giữa các đối tượng phạm vi cluster
như webhook và CRD. Do đó, mô hình control plane ảo đặc biệt
phù hợp cho các trường hợp mỗi tenant cần quyền truy cập vào một Kubernetes API server và mong đợi
khả năng quản lý cluster đầy đủ.

Sự cách ly được cải thiện đi kèm với cái giá là phải vận hành và duy trì một control plane ảo
riêng cho mỗi tenant. Ngoài ra, control plane theo từng tenant không giải quyết các vấn đề cách ly ở
data plane, chẳng hạn hàng xóm ồn ào ở cấp node hay các mối đe dọa bảo mật. Những vấn đề này vẫn phải được xử lý
riêng.
