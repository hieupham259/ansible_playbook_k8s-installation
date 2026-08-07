# Các Controller (Controllers)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/architecture/controllers/>
>
> Trong Kubernetes, controller là các vòng lặp điều khiển theo dõi trạng thái cluster của bạn,
> rồi thực hiện hoặc yêu cầu các thay đổi khi cần thiết.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 1 → nhóm [1a](LO-TRINH-ADMIN.md#1a-kiến-trúc-và-mô-hình-điều-khiển),
bài 8/8 · Kiểm chứng ở [Lab 1a](labs/LAB-1A-KIEN-TRUC-VA-MO-HINH-DIEU-KHIEN.md) phần B8.

Đây là **ý tưởng cốt lõi của toàn bộ Kubernetes**, và là bài quan trọng nhất nhóm 1a. Bài ngắn
nhưng đừng đọc nhanh. Toàn bộ ví dụ trong bài dùng Job và Pod — hai thứ bạn chưa học; hãy đọc
chúng như minh họa cho **hình dạng của vòng lặp**, đừng cố hiểu Job.

**Phải hiểu ở lần đọc này:**

- Vòng lặp điều khiển là gì, qua ví dụ bộ điều nhiệt: có trạng thái mong muốn, có trạng thái
  hiện tại, và một vòng lặp không kết thúc kéo cái sau về cái trước.
- Controller **thường không tự làm việc** — nó gửi request tới API server để tạo ra hiệu ứng.
  Job controller không chạy Pod nào cả; nó yêu cầu API server tạo Pod.
- Một số controller tác động ra ngoài cluster (*Điều khiển trực tiếp*), nhưng vẫn lấy trạng
  thái mong muốn từ API server và báo cáo trạng thái hiện tại về đó.
- Cluster **không cần đạt trạng thái ổn định**. Miễn các controller còn chạy và còn làm được
  việc hữu ích thì hệ thống vẫn khỏe. Đây là điểm phản trực giác nhất của bài.
- Thiết kế nhiều controller nhỏ, mỗi controller lo một khía cạnh; các controller có sẵn chạy
  chung trong `kube-controller-manager`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Chi tiết Job, Pod, Deployment trong các ví dụ | chưa học các workload này | giai đoạn 3 và 4 |
| Ghi chú về việc dùng label để phân biệt Pod của Deployment và của Job | chưa học label | nhóm 1b |
| Tự viết controller, `sample-controller` | dành cho người phát triển operator | giai đoạn 14 |

---

Trong lĩnh vực robot và tự động hóa, _vòng lặp điều khiển_ (control loop) là
một vòng lặp không kết thúc, có nhiệm vụ điều chỉnh trạng thái của một hệ thống.

Đây là một ví dụ về vòng lặp điều khiển: bộ điều nhiệt (thermostat) trong một căn phòng.

Khi bạn đặt nhiệt độ, tức là bạn đang cho bộ điều nhiệt biết
*trạng thái mong muốn* (desired state) của bạn. Nhiệt độ thực tế trong phòng là
*trạng thái hiện tại* (current state). Bộ điều nhiệt hành động để đưa trạng thái hiện tại
tiến gần hơn tới trạng thái mong muốn, bằng cách bật hoặc tắt thiết bị.

Trong Kubernetes, controller là các vòng lặp điều khiển theo dõi trạng thái
cluster của bạn, rồi thực hiện hoặc yêu cầu các thay đổi khi cần thiết.
Mỗi controller cố gắng đưa trạng thái hiện tại của cluster tiến gần hơn tới trạng thái mong muốn.

## Mẫu controller (Controller pattern)

Một controller theo dõi ít nhất một loại tài nguyên (resource type) của Kubernetes.
Các đối tượng (object) này có trường spec đại diện cho trạng thái mong muốn. (Các)
controller của tài nguyên đó chịu trách nhiệm làm cho trạng thái hiện tại
tiến gần hơn tới trạng thái mong muốn ấy.

Controller có thể tự mình thực hiện hành động; nhưng phổ biến hơn trong Kubernetes,
controller sẽ gửi các thông điệp tới API server để tạo ra
những hiệu ứng phụ (side effect) hữu ích. Bạn sẽ thấy các ví dụ về điều này bên dưới.

### Điều khiển thông qua API server (Control via API server)

Controller của Job là một ví dụ về controller có sẵn (built-in) của
Kubernetes. Các controller có sẵn quản lý trạng thái bằng cách
tương tác với API server của cluster.

Job là một tài nguyên Kubernetes chạy một
Pod, hoặc có thể là vài Pod, để thực hiện
một tác vụ rồi dừng lại.

(Một khi đã được [lập lịch](https://kubernetes.io/docs/concepts/scheduling-eviction/) (schedule), các đối tượng Pod trở thành
một phần trạng thái mong muốn đối với một kubelet).

Khi controller của Job thấy một tác vụ mới, nó bảo đảm rằng, ở đâu đó
trong cluster của bạn, các kubelet trên một tập các Node đang chạy đúng
số lượng Pod cần thiết để hoàn thành công việc.
Bản thân controller của Job không chạy bất kỳ Pod hay container nào.
Thay vào đó, controller của Job yêu cầu API server tạo hoặc xóa các Pod.
Các thành phần khác trong control plane
hành động dựa trên thông tin mới đó (có các Pod mới cần lập lịch và chạy),
và cuối cùng công việc được hoàn thành.

Sau khi bạn tạo một Job mới, trạng thái mong muốn là Job đó được hoàn thành.
Controller của Job làm cho trạng thái hiện tại của Job đó tiến gần hơn tới trạng thái
mong muốn của bạn: tạo các Pod thực hiện công việc bạn muốn cho Job đó, để
Job tiến gần hơn tới việc hoàn thành.

Các controller cũng cập nhật chính những đối tượng cấu hình chúng.
Ví dụ: khi công việc của một Job đã xong, controller của Job
cập nhật đối tượng Job đó để đánh dấu nó là `Finished`.

(Điều này hơi giống cách một số bộ điều nhiệt tắt một bóng đèn để
báo hiệu rằng căn phòng của bạn đã đạt tới nhiệt độ bạn đặt).

### Điều khiển trực tiếp (Direct control)

Khác với Job, một số controller cần thay đổi
những thứ bên ngoài cluster của bạn.

Ví dụ, nếu bạn dùng một vòng lặp điều khiển để bảo đảm có
đủ Node trong cluster, thì controller đó cần một thứ gì đó bên ngoài
cluster hiện tại để thiết lập các Node mới khi cần.

Các controller tương tác với trạng thái bên ngoài sẽ lấy trạng thái mong muốn từ
API server, sau đó giao tiếp trực tiếp với một hệ thống bên ngoài để đưa
trạng thái hiện tại tiến gần lại trạng thái mong muốn.

(Thực tế có một [controller](https://github.com/kubernetes/autoscaler/)
mở rộng theo chiều ngang (horizontally scale) số node trong cluster của bạn.)

Điểm quan trọng ở đây là controller thực hiện một số thay đổi để đạt được
trạng thái mong muốn của bạn, sau đó báo cáo trạng thái hiện tại về API server của cluster.
Các vòng lặp điều khiển khác có thể quan sát dữ liệu được báo cáo đó và tự thực hiện các hành động của riêng chúng.

Trong ví dụ về bộ điều nhiệt, nếu căn phòng quá lạnh thì một controller khác
cũng có thể bật máy sưởi chống đóng băng (frost protection heater). Với các cluster Kubernetes, control
plane làm việc gián tiếp với các công cụ quản lý địa chỉ IP, các dịch vụ lưu trữ,
API của nhà cung cấp đám mây (cloud provider) và các dịch vụ khác bằng cách
[mở rộng Kubernetes](https://kubernetes.io/docs/concepts/extend-kubernetes/) để hiện thực điều đó.

## Trạng thái mong muốn so với trạng thái hiện tại (Desired versus current state) {#desired-vs-current}

Kubernetes nhìn hệ thống theo góc nhìn cloud-native, và có khả năng xử lý
sự thay đổi liên tục.

Cluster của bạn có thể thay đổi tại bất kỳ thời điểm nào khi công việc diễn ra và
các vòng lặp điều khiển tự động sửa chữa các hỏng hóc. Điều này có nghĩa là,
rất có thể, cluster của bạn không bao giờ đạt tới một trạng thái ổn định.

Miễn là các controller của cluster vẫn đang chạy và có thể thực hiện
những thay đổi hữu ích, thì việc trạng thái tổng thể có ổn định hay không không quan trọng.

## Thiết kế (Design)

Như một nguyên tắc trong thiết kế của mình, Kubernetes sử dụng nhiều controller, mỗi controller quản lý
một khía cạnh cụ thể của trạng thái cluster. Phổ biến nhất, một vòng lặp điều khiển
(controller) cụ thể dùng một loại tài nguyên làm trạng thái mong muốn của nó, và có một loại
tài nguyên khác mà nó quản lý để hiện thực hóa trạng thái mong muốn đó. Ví dụ,
controller cho các Job theo dõi các đối tượng Job (để phát hiện công việc mới) và các đối tượng Pod
(để chạy các Job, rồi sau đó biết khi nào công việc hoàn thành). Trong trường hợp này,
một thành phần khác tạo ra các Job, còn controller của Job thì tạo ra các Pod.

Sẽ hữu ích hơn khi có các controller đơn giản, thay vì một tập vòng lặp điều khiển
nguyên khối (monolithic) liên kết chằng chịt với nhau. Controller có thể gặp lỗi, vì vậy Kubernetes được
thiết kế để chấp nhận điều đó.

> **Ghi chú:**
> Có thể có nhiều controller cùng tạo hoặc cập nhật cùng một loại đối tượng.
> Ở phía sau, các controller của Kubernetes bảo đảm rằng chúng chỉ chú ý
> tới những tài nguyên liên kết với tài nguyên điều khiển (controlling resource) của chúng.
>
> Ví dụ, bạn có thể có các Deployment và các Job; cả hai đều tạo Pod.
> Controller của Job không xóa các Pod mà Deployment của bạn đã tạo,
> vì có thông tin (các label)
> mà các controller có thể dùng để phân biệt những Pod đó.

## Các cách chạy controller (Ways of running controllers) {#running-controllers}

Kubernetes đi kèm một tập các controller có sẵn chạy bên trong
kube-controller-manager. Các
controller có sẵn này cung cấp những hành vi cốt lõi quan trọng.

Controller của Deployment và controller của Job là ví dụ về các controller
đi kèm như một phần của chính Kubernetes (các controller "có sẵn" — built-in).
Kubernetes cho phép bạn chạy một control plane có khả năng chống chịu lỗi (resilient), để nếu bất kỳ controller
có sẵn nào gặp sự cố, một phần khác của control plane sẽ tiếp quản công việc.

Bạn có thể tìm thấy các controller chạy bên ngoài control plane để mở rộng Kubernetes.
Hoặc, nếu muốn, bạn có thể tự viết một controller mới.
Bạn có thể chạy controller của riêng mình dưới dạng một tập các Pod,
hoặc chạy bên ngoài Kubernetes. Cách nào phù hợp nhất sẽ tùy thuộc vào chức năng của
controller cụ thể đó.

## Tiếp theo (What's next)

* Đọc về [control plane của Kubernetes](https://kubernetes.io/docs/concepts/architecture/#control-plane-components)
* Khám phá một số [đối tượng Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/) cơ bản
* Tìm hiểu thêm về [Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/)
* Nếu bạn muốn tự viết controller của riêng mình, hãy xem
  [các mẫu mở rộng Kubernetes](https://kubernetes.io/docs/concepts/extend-kubernetes/#extension-patterns)
  và repository [sample-controller](https://github.com/kubernetes/sample-controller).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 1:

1. Kể bốn bước của một vòng lặp điều khiển, dùng ví dụ bộ điều nhiệt.
2. Job controller có tự chạy container không? Nếu không thì nó làm gì để công việc được chạy?
3. Bạn xóa ServiceAccount `default` trong một namespace đang `Active`. Chuyện gì xảy ra, và
   thành phần nào làm việc đó?
4. Có người nói "cluster của tôi không bao giờ ở trạng thái ổn định, chắc đang hỏng". Nhận
   định đó đúng hay sai theo bài này?
5. Controller mở rộng số node của cluster phải gọi API của nhà cung cấp cloud. Vậy nó còn dùng
   Kubernetes API để làm gì?

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng. Đây là bài cuối của nhóm 1a — khi
trả lời được hết, bạn sẵn sàng vào [Lab 1a](labs/LAB-1A-KIEN-TRUC-VA-MO-HINH-DIEU-KHIEN.md).
