# Khoảng giới hạn tài nguyên (Limit Ranges)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/policy/limit-range/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 7 → nhóm [7b](00-ALO-TRINH-ADMIN.md#7b-chính-sách-giới-hạn-tài-nguyên),
bài 2/6 · Kiểm chứng ở [Lab 7b](labs/LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md).

Bài này và bài [134](134-resource-quotas-vi.md) ngay sau tạo thành một cặp rất dễ lẫn. Giữ
chặt một câu phân biệt ngay từ đầu: **LimitRange đặt trần cho từng đối tượng, ResourceQuota
đặt trần cho tổng cả namespace.** Bài mở đầu bằng đúng ý đó — quota giới hạn cả namespace, còn
LimitRange có mặt để "một đối tượng đơn lẻ không thể chiếm dụng toàn bộ tài nguyên khả dụng
trong một namespace". Bạn đã có `requests`/`limits` từ bài
[110](110-manage-resources-containers-vi.md) ở giai đoạn 3, nên bài này đọc nhanh được.

**Phải hiểu ở lần đọc này:**

- LimitRange là chính sách **cấp từng đối tượng** trong một namespace: ép min/max tài nguyên
  tính toán cho mỗi Pod hoặc Container, ép min/max storage request cho mỗi PersistentVolumeClaim,
  ép tỷ lệ request/limit, và **đặt request/limit mặc định** rồi tự chèn vào Container lúc chạy.
- Đúng thứ tự hai bước trong mục *Ràng buộc đối với limit và request tài nguyên*: admission
  controller LimitRange **trước hết** điền giá trị mặc định cho Pod chưa khai tài nguyên tính
  toán, **sau đó** mới soi mức sử dụng so với min/max/tỷ lệ. Vi phạm thì yêu cầu gửi tới API
  server thất bại với `403 Forbidden` kèm thông báo nói rõ ràng buộc nào bị vi phạm.
- Ranh giới thời điểm: việc kiểm tra **chỉ diễn ra ở giai đoạn admission của Pod**. Thêm hay
  sửa LimitRange không đụng tới các Pod đã tồn tại — chúng chạy tiếp không đổi.
- Cái bẫy ở mục *LimitRange và kiểm tra admission cho Pod*: LimitRange **không** kiểm tra tính
  nhất quán của giá trị mặc định nó áp. `default` limit có thể nhỏ hơn `requests` mà client
  gửi lên, và Pod cuối cùng không xếp lịch được với lỗi `must be less than or equal to cpu limit`.
- Hai hệ quả vận hành: namespace đã có LimitRange cho `cpu`/`memory` thì bạn **phải** chỉ định
  request hoặc limit, nếu không hệ thống có thể từ chối tạo Pod; và nếu có **từ hai LimitRange
  trở lên** trong cùng namespace thì giá trị mặc định nào được áp là **không xác định**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ràng buộc tỷ lệ request/limit, và min/max storage request cho PersistentVolumeClaim | bài chỉ liệt kê một dòng, không có ví dụ; trọng tâm nhóm 7b là `cpu` và `memory` | nhóm task giai đoạn 25 *Quản trị tài nguyên theo namespace* ở cuối lộ trình |
| Câu "tổng limit của namespace nhỏ hơn tổng các limit của các Pod/Container" ở cuối mục *Ví dụ về ràng buộc tài nguyên* | đây đã là chuyện trần tổng, tức địa hạt của ResourceQuota | bài [134](134-resource-quotas-vi.md), ngay sau bài này |
| Toàn bộ mục *Tiếp theo* | là loạt trang task hướng dẫn từng bước, không phải khái niệm | nhóm task giai đoạn 25 ở cuối lộ trình |

---

Theo mặc định, các container chạy với [tài nguyên tính toán](./110-manage-resources-containers-vi.md)
không bị giới hạn trên một cluster Kubernetes.
Bằng cách sử dụng [hạn ngạch tài nguyên (resource quota)](./134-resource-quotas-vi.md) của Kubernetes,
quản trị viên (còn gọi là _người vận hành cluster_ — cluster operator) có thể hạn chế mức tiêu thụ và việc tạo mới
tài nguyên của cluster (chẳng hạn thời gian CPU, bộ nhớ và lưu trữ bền vững — persistent storage) trong một
namespace được chỉ định.
Bên trong một namespace, một Pod có thể tiêu thụ lượng CPU và bộ nhớ tối đa mà các ResourceQuota áp dụng cho namespace đó cho phép.
Với vai trò người vận hành cluster, hoặc quản trị viên cấp namespace, bạn có thể cũng muốn đảm bảo
rằng một đối tượng đơn lẻ không thể chiếm dụng toàn bộ tài nguyên khả dụng trong một namespace.

LimitRange là một chính sách nhằm ràng buộc việc cấp phát tài nguyên (limit và request) mà bạn có thể chỉ định
cho từng loại đối tượng áp dụng được (chẳng hạn Pod hoặc PersistentVolumeClaim) trong một namespace.

Một _LimitRange_ cung cấp các ràng buộc có thể:

- Ép buộc mức sử dụng tài nguyên tính toán tối thiểu và tối đa cho mỗi Pod hoặc Container trong một namespace.
- Ép buộc mức yêu cầu lưu trữ (storage request) tối thiểu và tối đa cho mỗi
  PersistentVolumeClaim trong một namespace.
- Ép buộc tỷ lệ giữa request và limit cho một loại tài nguyên trong một namespace.
- Đặt request/limit mặc định cho tài nguyên tính toán trong một namespace và tự động
  chèn chúng vào các Container lúc chạy (runtime).

Kubernetes ràng buộc việc cấp phát tài nguyên cho các Pod trong một namespace cụ thể
mỗi khi có ít nhất một đối tượng LimitRange trong namespace đó.

Tên của một đối tượng LimitRange phải là một
[tên miền con DNS hợp lệ](./17-names-vi.md#dns-subdomain-names).

## Ràng buộc đối với limit và request tài nguyên (Constraints on resource limits and requests) {#constraints-on-resource-limits-and-requests}

- Quản trị viên tạo một LimitRange trong một namespace.
- Người dùng tạo (hoặc cố gắng tạo) các đối tượng trong namespace đó, chẳng hạn Pod hoặc
  PersistentVolumeClaim.
- Trước hết, admission controller LimitRange áp dụng các giá trị request và limit mặc định
  cho tất cả các Pod (và các container của chúng) chưa đặt yêu cầu tài nguyên tính toán.
- Sau đó, LimitRange theo dõi mức sử dụng để đảm bảo nó không vượt quá mức tối thiểu,
  tối đa và tỷ lệ tài nguyên được định nghĩa trong bất kỳ LimitRange nào có mặt trong namespace.
- Nếu bạn cố gắng tạo hoặc cập nhật một đối tượng (Pod hoặc PersistentVolumeClaim) vi phạm
  một ràng buộc của LimitRange, yêu cầu của bạn gửi tới API server sẽ thất bại với mã trạng thái HTTP
  `403 Forbidden` kèm theo một thông báo giải thích ràng buộc đã bị vi phạm.
- Nếu bạn thêm một LimitRange trong một namespace áp dụng cho các tài nguyên liên quan đến tính toán
  như `cpu` và `memory`, bạn phải chỉ định request hoặc limit cho các giá trị đó.
  Nếu không, hệ thống có thể từ chối việc tạo Pod.
- Việc kiểm tra LimitRange chỉ diễn ra ở giai đoạn admission của Pod, không áp dụng trên các Pod đang chạy.
  Nếu bạn thêm hoặc sửa đổi một LimitRange, các Pod đã tồn tại từ trước trong namespace đó
  vẫn tiếp tục chạy không thay đổi.
- Nếu có hai hoặc nhiều đối tượng LimitRange tồn tại trong namespace, việc giá trị mặc định nào
  sẽ được áp dụng là không xác định (not deterministic).

## LimitRange và kiểm tra admission cho Pod (LimitRange and admission checks for Pods)

Một LimitRange **không** kiểm tra tính nhất quán của các giá trị mặc định mà nó áp dụng.
Điều này có nghĩa là giá trị mặc định cho _limit_ do LimitRange đặt có thể
nhỏ hơn giá trị _request_ được chỉ định cho container trong spec mà client
gửi tới API server. Nếu điều đó xảy ra, Pod cuối cùng sẽ không thể được xếp lịch (schedule).

Ví dụ, bạn định nghĩa một LimitRange với manifest dưới đây:

> **Ghi chú:** Các ví dụ sau đây hoạt động trong namespace default của cluster của bạn, vì tham số
> namespace không được định nghĩa và phạm vi của LimitRange giới hạn ở cấp namespace.
> Điều này ngụ ý rằng mọi tham chiếu hoặc thao tác trong các ví dụ này sẽ tương tác
> với các thành phần trong namespace default của cluster của bạn. Bạn có thể ghi đè
> namespace hoạt động bằng cách cấu hình namespace trong trường `metadata.namespace`.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-resource-constraint
spec:
  limits:
  - default: # phần này định nghĩa các limit mặc định
      cpu: 500m
    defaultRequest: # phần này định nghĩa các request mặc định
      cpu: 500m
    max: # max và min định nghĩa khoảng giới hạn
      cpu: "1"
    min:
      cpu: 100m
    type: Container
```

cùng với một Pod khai báo CPU resource request là `700m`, nhưng không khai báo limit:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-conflict-with-limitrange-cpu
spec:
  containers:
  - name: demo
    image: registry.k8s.io/pause:3.8
    resources:
      requests:
        cpu: 700m
```

khi đó Pod này sẽ không được xếp lịch, thất bại với lỗi tương tự như:

```
Pod "example-conflict-with-limitrange-cpu" is invalid: spec.containers[0].resources.requests: Invalid value: "700m": must be less than or equal to cpu limit
```

Nếu bạn đặt cả `request` lẫn `limit`, thì Pod mới đó sẽ được xếp lịch thành công
ngay cả khi vẫn có cùng LimitRange đó:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-no-conflict-with-limitrange-cpu
spec:
  containers:
  - name: demo
    image: registry.k8s.io/pause:3.8
    resources:
      requests:
        cpu: 700m
      limits:
        cpu: 700m
```

## Ví dụ về ràng buộc tài nguyên (Example resource constraints)

Một số ví dụ về chính sách có thể được tạo bằng LimitRange là:

- Trong một cluster 2 node với dung lượng 8 GiB RAM và 16 core, ràng buộc các Pod trong một
  namespace request 100m CPU với limit tối đa 500m cho CPU, và request 200Mi
  cho bộ nhớ với limit tối đa 600Mi cho bộ nhớ.
- Định nghĩa limit và request CPU mặc định là 150m và request bộ nhớ mặc định là 300Mi cho
  các Container được khởi chạy mà không có request cpu và memory trong spec của chúng.

Trong trường hợp tổng limit của namespace nhỏ hơn tổng các limit
của các Pod/Container, có thể xảy ra tranh chấp (contention) tài nguyên. Trong trường hợp này,
các Container hoặc Pod sẽ không được tạo.

Cả tranh chấp lẫn các thay đổi đối với một LimitRange đều không ảnh hưởng đến các tài nguyên đã được tạo trước đó.

## Tiếp theo (What's next)

Để xem các ví dụ về việc sử dụng limit, hãy xem:

- [cách cấu hình ràng buộc CPU tối thiểu và tối đa cho mỗi namespace](229-cpu-constraint-namespace-vi.md).
- [cách cấu hình ràng buộc bộ nhớ tối thiểu và tối đa cho mỗi namespace](231-memory-constraint-namespace-vi.md).
- [cách cấu hình CPU Request và Limit mặc định cho mỗi namespace](230-cpu-default-namespace-vi.md).
- [cách cấu hình Request và Limit bộ nhớ mặc định cho mỗi namespace](232-memory-default-namespace-vi.md).
- [cách cấu hình mức tiêu thụ lưu trữ tối thiểu và tối đa cho mỗi namespace](227-limit-storage-consumption-vi.md#limitrange-to-limit-requests-for-storage).
- một [ví dụ chi tiết về cấu hình hạn ngạch cho mỗi namespace](233-quota-memory-cpu-namespace-vi.md).

Tham khảo [tài liệu thiết kế LimitRanger](https://git.k8s.io/design-proposals-archive/resource-management/admission_control_limit_range.md)
để biết bối cảnh và thông tin lịch sử.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 7:

1. Namespace có đúng một LimitRange đặt `defaultRequest.cpu` và `default.cpu` cho
   `type: Container`. Bạn tạo một Pod không khai `requests` lẫn `limits`. Pod có bị từ chối
   không? Ai điền giá trị vào, và ở bước nào của vòng đời yêu cầu?
2. Bài đưa ra một LimitRange hoàn toàn hợp lệ nhưng vẫn khiến một Pod không chạy được. Kể lại
   tình huống đó, giải thích cơ chế, và nêu cách sửa **từ phía Pod** mà không đụng LimitRange.
3. Bạn hạ `max` của một LimitRange xuống thấp hơn mức các Pod đang chạy trong namespace đó
   đang dùng. Những Pod đó bị từ chối, bị sửa lại, hay bị khởi động lại?
4. Cluster lab của bạn có hai worker, mỗi máy 2 vCPU / 6 GB RAM. Bạn đặt cho namespace một
   LimitRange với `max.cpu: "1"`. Điều đó có ngăn được namespace ấy tạo 4 Pod cùng lúc và
   chiếm hết 4 vCPU của cả cluster không?
5. Có hai đối tượng LimitRange trong cùng một namespace, mỗi cái đặt `defaultRequest.cpu` một
   giá trị khác nhau. Pod không khai `requests` sẽ nhận giá trị nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Pod được tạo bình thường, và admission controller LimitRange là thứ điền giá trị vào.**
   Bài mô tả rõ trình tự: *trước hết*, admission controller LimitRange áp các giá trị request
   và limit mặc định cho tất cả Pod (và container của chúng) chưa đặt yêu cầu tài nguyên tính
   toán; *sau đó* mới theo dõi mức sử dụng để đảm bảo không vượt min, max và tỷ lệ. Nghĩa là
   việc điền mặc định xảy ra **ở giai đoạn admission, trước khi Pod được lưu**, và Pod cuối
   cùng chạy với giá trị đã bị chèn vào chứ không phải với ô trống. Lưu ý mặt trái: nếu
   LimitRange đó chỉ ràng buộc `cpu`/`memory` mà không cung cấp mặc định, bài cảnh báo bạn
   **phải** tự chỉ định request hoặc limit, nếu không hệ thống có thể từ chối tạo Pod.
2. **Tình huống: `default.cpu: 500m` gặp một Pod khai `requests.cpu: 700m` mà không khai limit.**
   Cơ chế: LimitRange **không kiểm tra tính nhất quán của các giá trị mặc định mà nó áp**, nên
   nó cứ chèn limit `500m` vào cạnh request `700m` do client gửi. Kết quả là một Pod tự mâu
   thuẫn — request lớn hơn limit — và nó không xếp lịch được, lỗi
   `must be less than or equal to cpu limit`. Cách sửa từ phía Pod: **khai cả `requests` lẫn
   `limits`**. Bài chỉ rõ Pod đặt cả hai bằng `700m` sẽ được xếp lịch thành công ngay cả khi
   vẫn còn nguyên LimitRange đó, vì lúc này không còn chỗ trống nào để giá trị mặc định chen vào.
3. **Không cái nào cả — chúng chạy tiếp không thay đổi.** Việc kiểm tra LimitRange **chỉ diễn ra
   ở giai đoạn admission của Pod**, không áp dụng trên Pod đang chạy. Bài nói thẳng: thêm hoặc
   sửa một LimitRange thì các Pod đã tồn tại từ trước trong namespace vẫn tiếp tục chạy không
   thay đổi, và cả tranh chấp lẫn thay đổi LimitRange đều không ảnh hưởng đến tài nguyên đã tạo
   trước đó. Ràng buộc mới chỉ có hiệu lực với Pod tạo hoặc cập nhật sau đó.
4. **Không.** `max.cpu: "1"` là trần **cho từng Container/Pod**, không phải trần tổng. Bốn Pod
   mỗi Pod 1 CPU đều hợp lệ với LimitRange này. Đây đúng là ranh giới bài vạch ra ngay đoạn mở
   đầu: LimitRange tồn tại để đảm bảo **một đối tượng đơn lẻ** không chiếm dụng toàn bộ tài
   nguyên khả dụng trong namespace, còn việc hạn chế **mức tiêu thụ và việc tạo mới tài nguyên
   của cả namespace** là việc của ResourceQuota. Trực giác "đặt max là chặn được namespace" sai
   vì nó nhầm trần đơn vị với trần tổng. Trên cluster hai worker 2 vCPU của bạn, muốn chặn
   namespace ăn hết cluster thì phải thêm ResourceQuota ở bài [134](134-resource-quotas-vi.md).
5. **Không xác định được** — bài ghi rõ: nếu có hai hoặc nhiều đối tượng LimitRange tồn tại
   trong namespace, việc giá trị mặc định nào sẽ được áp là **không xác định (not deterministic)**.
   Đây không phải "cái nào chặt hơn thắng" hay "cái tạo sau thắng"; đơn giản là đừng đặt hai
   LimitRange chồng nhau trong một namespace.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
