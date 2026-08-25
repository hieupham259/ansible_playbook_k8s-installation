# Tự động dọn dẹp các Job đã hoàn thành (Automatic Cleanup for Finished Jobs)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/>
>
> Cơ chế time-to-live (TTL) để dọn dẹp các Job cũ đã chạy xong.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 4](00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller), bài 7/14 ·
Kiểm chứng ở [Lab 4b](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md).

Bài ngắn nhất giai đoạn, và nó trả lời trực tiếp câu hỏi mà bài [67](67-job-vi.md) để mở:
Job xong rồi thì ai dọn? Đọc mất năm phút. Toàn bộ giá trị nằm ở hai chỗ dễ hiểu sai — mốc
tính giờ và hành vi khi TTL đã hết hạn.

**Phải hiểu ở lần đọc này:**

- Controller TTL-after-finished **chỉ hỗ trợ Job**, và được kích hoạt bằng đúng một trường:
  `.spec.ttlSecondsAfterFinished`.
- **Đồng hồ bắt đầu chạy khi status condition của Job chuyển sang `Complete` hoặc `Failed`**,
  không phải từ lúc tạo Job.
- Khi dọn, controller xóa Job **theo tầng** — các object phụ thuộc, tức các Pod, bị xóa cùng
  — và các bảo đảm vòng đời như [finalizer](29-finalizers-vi.md) vẫn được tôn trọng.
- Bạn có thể đặt trường này bất cứ lúc nào: trong manifest, đặt tay cho Job đã chạy xong, hay
  để một mutating admission webhook đặt tự động — cách cuối là cách quản trị viên **áp một
  chính sách TTL cho toàn cluster**.
- Hai điểm lưu ý phải nhớ: kéo dài TTL **sau khi** TTL cũ đã hết hạn thì Kubernetes **không
  bảo đảm** giữ Job lại, dù API trả về thành công; và cơ chế này **nhạy với lệch thời gian**
  giữa các máy vì nó dựa trên timestamp lưu trong Job.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Hai gạch đầu dòng về *mutating admission webhook* | chưa học admission controller | giai đoạn 9 |
| "Tự viết controller của riêng bạn để quản lý TTL" | cần biết viết controller | giai đoạn 14 |
| Link tới KEP ở mục *Tiếp theo* | là tài liệu thiết kế của tính năng | không cần |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.23 [stable]`

Khi Job của bạn đã chạy xong, việc giữ lại Job đó trong API (thay vì xóa Job ngay lập tức)
là rất hữu ích, để bạn có thể biết được Job đã thành công hay thất bại.

Controller TTL-after-finished của Kubernetes cung cấp một cơ chế TTL (time to live —
thời gian tồn tại) để giới hạn vòng đời của các object Job đã chạy xong.

## Dọn dẹp các Job đã hoàn thành (Cleanup for finished Jobs)

Controller TTL-after-finished chỉ được hỗ trợ cho Job. Bạn có thể dùng cơ chế này để tự động
dọn dẹp các Job đã hoàn thành (ở trạng thái `Complete` hoặc `Failed`) bằng cách chỉ định
trường `.spec.ttlSecondsAfterFinished` của Job, như trong
[ví dụ này](67-job-vi.md#clean-up-finished-jobs-automatically).

Controller TTL-after-finished coi một Job là đủ điều kiện để được dọn dẹp sau khi Job đó
kết thúc được TTL giây. Bộ đếm thời gian bắt đầu chạy khi status condition của Job thay đổi
để cho thấy Job đang ở trạng thái `Complete` hoặc `Failed`; khi TTL hết hạn, Job đó trở nên
đủ điều kiện để bị xóa [theo tầng](./36-garbage-collection-vi.md#cascading-deletion)
(cascading). Khi controller TTL-after-finished dọn dẹp một Job, nó sẽ xóa Job theo tầng,
nghĩa là nó sẽ xóa cả các object phụ thuộc (dependent object) cùng với Job đó.

Kubernetes tôn trọng các bảo đảm về vòng đời object trên Job, chẳng hạn như chờ các
[finalizer](./29-finalizers-vi.md).

Bạn có thể đặt giá trị TTL (tính bằng giây) vào bất kỳ lúc nào. Dưới đây là một số ví dụ
về cách đặt trường `.spec.ttlSecondsAfterFinished` của một Job:

* Chỉ định trường này ngay trong manifest của Job, để Job có thể được tự động dọn dẹp một
  khoảng thời gian sau khi nó chạy xong.
* Đặt trường này theo cách thủ công cho các Job đã tồn tại và đã chạy xong, để chúng trở
  nên đủ điều kiện được dọn dẹp.
* Dùng một
  [mutating admission webhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook)
  để đặt trường này một cách động (dynamic) tại thời điểm tạo Job. Quản trị viên cluster có
  thể dùng cách này để áp đặt một chính sách TTL cho các Job đã hoàn thành.
* Dùng một
  [mutating admission webhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook)
  để đặt trường này một cách động sau khi Job đã chạy xong, và chọn các giá trị TTL khác
  nhau dựa trên trạng thái (status), các label của Job. Với trường hợp này, webhook cần
  phát hiện các thay đổi trong `.status` của Job và chỉ đặt TTL khi Job đang được đánh dấu
  là đã hoàn thành.
* Tự viết controller của riêng bạn để quản lý TTL dọn dẹp cho các Job khớp với một
  selector cụ thể.

## Những điểm cần lưu ý (Caveats)

### Cập nhật TTL cho các Job đã hoàn thành (Updating TTL for finished Jobs)

Bạn có thể sửa đổi khoảng thời gian TTL, ví dụ trường `.spec.ttlSecondsAfterFinished` của
Job, sau khi Job đã được tạo hoặc đã chạy xong. Nếu bạn kéo dài khoảng TTL sau khi khoảng
`ttlSecondsAfterFinished` hiện có đã hết hạn, Kubernetes không bảo đảm sẽ giữ lại Job đó,
ngay cả khi yêu cầu cập nhật để kéo dài TTL trả về phản hồi API thành công.

### Lệch thời gian (Time skew)

Vì controller TTL-after-finished dùng các timestamp được lưu trong các Job của Kubernetes
để xác định TTL đã hết hạn hay chưa, tính năng này nhạy cảm với hiện tượng lệch thời gian
(time skew) trong cluster của bạn, điều này có thể khiến control plane dọn dẹp các object
Job vào sai thời điểm.

Đồng hồ không phải lúc nào cũng chính xác, nhưng chênh lệch thường rất nhỏ. Hãy lưu ý rủi
ro này khi đặt một giá trị TTL khác không.

## Tiếp theo (What's next)

* Đọc [Tự động dọn dẹp Job](67-job-vi.md#clean-up-finished-jobs-automatically)

* Tham khảo [Kubernetes Enhancement Proposal](https://github.com/kubernetes/enhancements/blob/master/keps/sig-apps/592-ttl-after-finish/README.md)
  (KEP) về việc bổ sung cơ chế này.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 4:

1. Bạn đặt `ttlSecondsAfterFinished: 100` cho một Job chạy mất 10 phút. Job đủ điều kiện bị
   xóa vào thời điểm nào tính từ lúc bạn `kubectl apply`?
2. **Câu bẫy.** TTL của một Job đã hết hạn nhưng bạn thấy Job vẫn còn, nên `kubectl patch`
   nâng `ttlSecondsAfterFinished` lên và API trả về thành công. Job có chắc còn không?
3. Cluster lab của bạn gồm ba VM `lab-k8s-master`, `lab-k8s-worker1`, `lab-k8s-worker2`. Vì sao bài cảnh
   báo về lệch thời gian, và hậu quả cụ thể là gì?
4. Khi controller dọn một Job hết TTL, các Pod của Job đó ra sao?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Khoảng 10 phút 100 giây** sau khi apply — tức **100 giây sau khi Job kết thúc**, không
   phải 100 giây sau khi tạo. Bài nói rõ: "Bộ đếm thời gian bắt đầu chạy khi status condition
   của Job thay đổi để cho thấy Job đang ở trạng thái `Complete` hoặc `Failed`". Nếu đặt
   trường này bằng `0` thì Job đủ điều kiện bị xóa **ngay sau khi hoàn tất**; nếu không đặt
   thì controller **không bao giờ** dọn Job đó.
2. **Không.** Bài nói thẳng: nếu bạn kéo dài khoảng TTL **sau khi** khoảng
   `ttlSecondsAfterFinished` hiện có đã hết hạn, "Kubernetes **không bảo đảm** sẽ giữ lại Job
   đó, **ngay cả khi yêu cầu cập nhật để kéo dài TTL trả về phản hồi API thành công**". Trực
   giác sai ở chỗ coi `200 OK` là bằng chứng Job sẽ sống — Job đã vào diện đủ điều kiện bị
   xóa từ trước, và việc controller chưa kịp xóa chỉ là vấn đề thời điểm.
3. Vì controller TTL-after-finished **dùng các timestamp lưu trong chính object Job** để xác
   định TTL đã hết hạn hay chưa. Đồng hồ ba máy lệch nhau thì hậu quả là **control plane dọn
   các object Job vào sai thời điểm** — xóa sớm hơn hoặc muộn hơn mong đợi. Bài lưu ý chênh
   lệch thường rất nhỏ, nhưng đây là rủi ro cần cân nhắc mỗi khi đặt một giá trị TTL khác
   không, đặc biệt với TTL ngắn.
4. **Chúng bị xóa cùng.** Controller xóa Job **theo tầng** (cascading), nghĩa là nó xóa cả các
   object phụ thuộc của Job. Việc xóa vẫn tôn trọng các bảo đảm về vòng đời object, chẳng hạn
   chờ các [finalizer](29-finalizers-vi.md) được gỡ.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
