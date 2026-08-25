# Scale một StatefulSet (Scale a StatefulSet)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/run-application/scale-stateful-set/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 4 — Workload controller](00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller)
→ [4b. StatefulSet, DaemonSet, Job và autoscaling](00-ALO-TRINH-ADMIN.md#4b-statefulset-daemonset-job-và-autoscaling),
bài 2/7 của dòng **Thực hành** · Kiểm chứng ở
[Lab 4b — StatefulSet, DaemonSet và Job](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md),
**phần B2**.

Bài rất ngắn, và phần đáng giá nhất nằm ở mục *Xử lý sự cố* chứ không ở mấy lệnh scale. Đọc kỹ
mục đó: nó giải thích vì sao StatefulSet không chịu thu nhỏ khi có Pod đang hỏng.

**Phải hiểu ở lần đọc này:**

- Scale một StatefulSet là tăng hoặc giảm số `replicas`, và đường mệnh lệnh là
  `kubectl scale statefulsets <tên> --replicas=<số mới>` (mục *Dùng kubectl để scale StatefulSet*).
- Ba đường cập nhật tại chỗ tương đương: sửa `.spec.replicas` trong manifest rồi `kubectl apply`
  nếu StatefulSet vốn được tạo bằng `kubectl apply`, hoặc `kubectl edit statefulsets`, hoặc
  `kubectl patch statefulsets` (mục *Thực hiện cập nhật tại chỗ trên StatefulSet của bạn*).
- Điều kiện bài đặt ra trước mọi thao tác: **chỉ scale khi bạn tin chắc cluster ứng dụng stateful
  hoàn toàn khỏe mạnh**, và không phải ứng dụng stateful nào cũng scale tốt
  (mục *Trước khi bạn bắt đầu*).
- **Không scale down được** khi bất kỳ Pod stateful nào mà StatefulSet quản lý đang không khỏe
  mạnh; việc thu nhỏ chỉ diễn ra **sau khi** các Pod đó trở về running và ready
  (mục *Scale down không hoạt động đúng*).
- Vì sao Kubernetes không tự phán: với `spec.replicas > 1` nó **không xác định được** Pod hỏng vì
  lỗi vĩnh viễn hay lỗi tạm thời. Lỗi vĩnh viễn mà cứ scale thì số thành viên có thể **tụt dưới
  mức tối thiểu** cần để hoạt động đúng và StatefulSet thành không khả dụng; lỗi tạm thời thì
  chính nó cản trở thao tác scale, vì một số cơ sở dữ liệu phân tán gặp vấn đề khi node vào và ra
  cùng lúc. Kết luận của bài: suy xét thao tác scale ở **cấp độ ứng dụng**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ghi chú StatefulSet chỉ khả dụng từ Kubernetes 1.5 trở lên, và lệnh `kubectl version` để kiểm tra | phiên bản khóa của cluster lab mới hơn rất nhiều nên điều kiện luôn thỏa | [bảng A1.3 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) |
| Hai link trỏ thẳng ra kubernetes.io: *hướng dẫn StatefulSet* và *cập nhật tại chỗ* | là tài liệu ngoài lộ trình dịch, không cần cho lần đọc này | thao tác scale hai chiều làm trực tiếp ở [Lab 4b](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md) phần B2 |
| Link *xóa một StatefulSet* (bài [340](340-delete-stateful-set-vi.md)) ở mục *Tiếp theo* | lộ trình xếp bài đó ở một giai đoạn sau, không thuộc nhóm 4b | dòng **Thực hành** của [giai đoạn 5 — Mạng nền tảng](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng); riêng thao tác xóa theo tầng và `--cascade=orphan` đã có ở [Lab 4b](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md) phần B3.3 |

---

Tác vụ này hướng dẫn cách scale một StatefulSet. Scale một StatefulSet nghĩa là
tăng hoặc giảm số lượng replicas.

## Trước khi bạn bắt đầu (Before you begin)

- StatefulSet chỉ khả dụng từ Kubernetes phiên bản 1.5 trở lên.
  Để kiểm tra phiên bản Kubernetes của bạn, hãy chạy `kubectl version`.

- Không phải ứng dụng stateful nào cũng scale tốt. Nếu bạn không chắc có nên
  scale StatefulSet của mình hay không, hãy xem [khái niệm StatefulSet](65-statefulset-vi.md)
  hoặc [hướng dẫn StatefulSet](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/) để biết thêm thông tin.

- Bạn chỉ nên thực hiện scale khi bạn tin chắc rằng cluster ứng dụng stateful
  của mình hoàn toàn khỏe mạnh.

## Scale các StatefulSet (Scaling StatefulSets)

### Dùng kubectl để scale StatefulSet (Use kubectl to scale StatefulSets)

Trước tiên, tìm StatefulSet mà bạn muốn scale.

```shell
kubectl get statefulsets <stateful-set-name>
```

Thay đổi số replicas của StatefulSet:

```shell
kubectl scale statefulsets <stateful-set-name> --replicas=<new-replicas>
```

### Thực hiện cập nhật tại chỗ trên StatefulSet của bạn (Make in-place updates on your StatefulSets)

Ngoài ra, bạn có thể thực hiện
[cập nhật tại chỗ (in-place updates)](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/#in-place-updates-of-resources)
trên các StatefulSet của mình.

Nếu StatefulSet của bạn ban đầu được tạo bằng `kubectl apply`,
hãy cập nhật `.spec.replicas` trong manifest của StatefulSet, sau đó chạy `kubectl apply`:

```shell
kubectl apply -f <stateful-set-file-updated>
```

Nếu không, hãy chỉnh sửa trường đó bằng `kubectl edit`:

```shell
kubectl edit statefulsets <stateful-set-name>
```

Hoặc dùng `kubectl patch`:

```shell
kubectl patch statefulsets <stateful-set-name> -p '{"spec":{"replicas":<new-replicas>}}'
```

## Xử lý sự cố (Troubleshooting)

### Scale down không hoạt động đúng (Scaling down does not work right)

Bạn không thể scale down một StatefulSet khi bất kỳ Pod stateful nào mà nó quản lý
đang không khỏe mạnh (unhealthy). Việc scale down chỉ diễn ra sau khi các Pod stateful
đó trở về trạng thái running và ready.

Nếu spec.replicas > 1, Kubernetes không thể xác định nguyên nhân khiến một Pod không
khỏe mạnh. Đó có thể là kết quả của một lỗi vĩnh viễn (permanent fault) hoặc một lỗi
tạm thời (transient fault). Lỗi tạm thời có thể do một lần khởi động lại cần thiết
trong quá trình nâng cấp hoặc bảo trì.

Nếu Pod không khỏe mạnh do lỗi vĩnh viễn, việc scale mà không sửa lỗi đó
có thể dẫn tới trạng thái mà số thành viên của StatefulSet
tụt xuống dưới một số lượng replicas tối thiểu cần thiết để hoạt động
đúng. Điều này có thể khiến StatefulSet của bạn trở nên không khả dụng.

Nếu Pod không khỏe mạnh do lỗi tạm thời và Pod có thể khả dụng trở lại,
lỗi tạm thời đó có thể gây trở ngại cho thao tác scale up hoặc scale down của bạn. Một số
cơ sở dữ liệu phân tán gặp vấn đề khi các node tham gia và rời đi cùng lúc. Trong những
trường hợp này, tốt hơn là suy xét các thao tác scale ở cấp độ ứng dụng, và
chỉ thực hiện scale khi bạn chắc chắn rằng cluster ứng dụng stateful của mình
hoàn toàn khỏe mạnh.

## Tiếp theo (What's next)

- Tìm hiểu thêm về [xóa một StatefulSet](340-delete-stateful-set-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở nhóm 4b:

1. StatefulSet 3 replica trên cluster lab, Pod `web-1` đang hỏng và không Ready. Bạn chạy
   `kubectl scale statefulsets web --replicas=2` trên `lab-k8s-master`. Việc thu nhỏ diễn ra ngay
   không, và điều kiện nào phải đạt trước?
2. **Câu bẫy.** Bài đưa ra `kubectl scale`, `kubectl apply`, `kubectl edit` và `kubectl patch`.
   Bốn cách đó cho ra bốn hành vi khác nhau à? Bài lấy gì làm căn cứ để chọn giữa chúng?
3. Vì sao bài nói Kubernetes không xác định được nguyên nhân khiến một Pod không khỏe mạnh khi
   `spec.replicas > 1`, và hai loại lỗi đó gây ra hai rắc rối khác nhau nào cho thao tác scale?
4. Bài đặt điều kiện gì trước mọi thao tác scale, và vì sao nó khuyên suy xét scale ở **cấp độ
   ứng dụng** thay vì cứ đổi số replica?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không diễn ra ngay.** Bài nói rõ: bạn **không thể scale down một StatefulSet khi bất kỳ Pod
   stateful nào mà nó quản lý đang không khỏe mạnh**. Việc thu nhỏ **chỉ diễn ra sau khi** các Pod
   đó trở về trạng thái running và ready. Nghĩa là muốn hạ xuống 2 thì phải sửa cho `web-1` khỏe
   trở lại trước, dù chính `web-1` không phải Pod sẽ bị bỏ đi.
2. **Không** — cả bốn đều chỉ làm một việc: **đổi số `replicas` của StatefulSet**. Đây là chỗ dễ
   tưởng mỗi lệnh có một ngữ nghĩa riêng. Bài trình bày chúng như các lựa chọn thay thế, và căn cứ
   chọn mà bài nêu là **cách StatefulSet đó được tạo ra**: nếu nó vốn được tạo bằng `kubectl apply`
   thì sửa `.spec.replicas` trong manifest rồi apply lại; **nếu không**, dùng `kubectl edit` hoặc
   `kubectl patch`.
3. Vì khi `spec.replicas > 1`, Kubernetes **không có cách nào biết** Pod hỏng do **lỗi vĩnh viễn**
   hay **lỗi tạm thời** — lỗi tạm thời có thể chỉ là một lần khởi động lại trong lúc nâng cấp hoặc
   bảo trì. Hai rắc rối khác nhau: với **lỗi vĩnh viễn**, scale mà không sửa lỗi có thể khiến **số
   thành viên tụt xuống dưới số replica tối thiểu cần thiết**, làm StatefulSet không khả dụng; với
   **lỗi tạm thời**, chính lỗi đó **cản trở thao tác scale up hoặc scale down**, và một số cơ sở
   dữ liệu phân tán gặp vấn đề khi node tham gia và rời đi cùng lúc.
4. Điều kiện: **chỉ scale khi bạn tin chắc cluster ứng dụng stateful của mình hoàn toàn khỏe
   mạnh** — bài nêu điều này ở cả mục *Trước khi bạn bắt đầu* lẫn mục *Xử lý sự cố*. Lý do khuyên
   suy xét ở **cấp độ ứng dụng**: Kubernetes chỉ thấy Pod khỏe hay không, còn **ứng dụng mới biết**
   mất một thành viên có phá vỡ quorum hay không, và **không phải ứng dụng stateful nào cũng scale
   tốt**.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
