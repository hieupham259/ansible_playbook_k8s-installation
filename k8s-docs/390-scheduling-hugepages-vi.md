# Quản lý HugePages (Manage HugePages)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/manage-hugepages/scheduling-hugepages/>
>
> Cấu hình và quản lý huge page như một tài nguyên có thể lập lịch (schedulable resource) trong cluster.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 29 — DaemonSet, Job nâng cao và thiết bị chuyên dụng](00-ALO-TRINH-ADMIN.md#giai-đoạn-29--daemonset-job-nâng-cao-và-thiết-bị-chuyên-dụng),
bài 7/8 · Kiểm chứng **một phần** trên cluster lab, chỉ bằng lệnh đọc — bài này thuộc nhóm
[7b. Chính sách giới hạn tài nguyên](00-ALO-TRINH-ADMIN.md#7b-chính-sách-giới-hạn-tài-nguyên) và nối
tiếp bài [110](110-manage-resources-containers-vi.md).

**Ranh giới thực hành, nói thẳng.** Huge page phải được **cấp phát trước ở tầng kernel của node**
rồi kubelet mới báo cáo được — ba VM của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) không cấu hình
điều đó. Nên chia làm hai phần:

- **Đo được ngay, không đụng vào baseline:** `kubectl describe node lab-k8s-worker1` để xem phần
  `Capacity` và `Allocatable` có dòng `hugepages-` nào và giá trị bằng bao nhiêu; và trên chính
  node, `grep -i huge /proc/meminfo` để đối chiếu với con số kernel đang báo. Hai phép đọc này cho
  bạn thấy **trạng thái "chưa cấp phát trước"** trông ra sao — đó cũng là một kết quả.
- **Không đo được nếu không đổi máy:** chạy thật một Pod tiêu thụ huge page. Muốn thế phải sửa
  `GRUB_CMDLINE_LINUX`, khởi động lại node và khởi động lại kubelet — tức thay đổi baseline đã khóa
  của Lab 00. Việc này **không nằm trong Checkpoint giai đoạn 29**; nếu vẫn muốn thử, chỉ làm trên
  `lab-k8s-worker2` (node duy nhất được phép gây lỗi) và khôi phục lại sau.

Vì vậy đọc bài này ở mức **hiểu cơ chế và hiểu cú pháp**, không cần chạy được.

**Phải hiểu ở lần đọc này:**

- Điều kiện tiên quyết ở mục *Trước khi bạn bắt đầu*: node phải **cấp phát trước** huge page (ví dụ
  bằng `hugepagesz`/`hugepages` trong `GRUB_CMDLINE_LINUX`) thì mới báo cáo được sức chứa. Node tự
  phát hiện và phơi bày chúng thành **tài nguyên có thể lập lịch**, hiện ra ở `Capacity` và
  `Allocatable` dưới tên `hugepages-1Gi`, `hugepages-2Mi`.
- Ghi chú ngay sau đó: với trang được cấp phát **động** (sau khi máy đã khởi động), phải **khởi
  động lại kubelet** thì phần cấp phát mới được phản ánh.
- Ba ràng buộc ở mục *API*: tên tài nguyên là `hugepages-<size>` với `<size>` là ký hiệu nhị phân
  gọn nhất mà node hỗ trợ; huge page **không hỗ trợ overcommit**, khác hẳn CPU và bộ nhớ; và khi
  xin huge page bạn **bắt buộc phải xin kèm** tài nguyên memory hoặc CPU.
- Hai dạng volume trong hai manifest ví dụ: Pod xin **nhiều kích thước** thì mọi volume mount phải
  dùng `medium: HugePages-<hugepagesize>`; chỉ được dùng `medium: HugePages` trơn khi Pod xin
  **đúng một** kích thước.
- Danh sách ràng buộc ở cuối bài: `requests` phải **bằng** `limits` (khai limits mà bỏ trống
  requests thì mặc định đã bằng nhau); huge page cô lập ở **phạm vi container** — mỗi container có
  limit riêng trên cgroup sandbox của nó; volume `emptyDir` hậu thuẫn bởi huge page không được tiêu
  thụ quá mức Pod đã xin; và ResourceQuota kiểm soát được mức dùng trong namespace qua token
  `hugepages-<size>`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Cách cấp phát trước huge page ở tầng kernel — dòng `GRUB_CMDLINE_LINUX` và tài liệu `hugetlbpage` của kernel | Sửa kernel cmdline rồi khởi động lại node là thay đổi baseline node | Không có trong Checkpoint giai đoạn 29 — ba VM lab để mặc định, nên bạn chỉ quan sát được trạng thái chưa cấp phát |
| Ràng buộc `shmget()` với `SHM_HUGETLB` và supplemental group khớp `proc/sys/vm/hugetlb_shm_group` | Đây là ràng buộc của **ứng dụng** dùng shared memory, không phải của người vận hành cluster | Chỉ có nghĩa khi bạn vận hành một workload thật thuộc loại đó; cluster lab không có workload nào như vậy |
| Cơ chế ResourceQuota nói chung, ở gạch đầu dòng cuối bài | Ở đây chỉ cần biết `hugepages-<size>` là một token quota hợp lệ | Bài [134](134-resource-quotas-vi.md) ở [giai đoạn 7b](00-ALO-TRINH-ADMIN.md#7b-chính-sách-giới-hạn-tài-nguyên) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.14 [stable]`

Kubernetes hỗ trợ việc cấp phát và tiêu thụ các huge page đã được cấp phát trước
(pre-allocated) bởi các ứng dụng chạy trong Pod. Trang này mô tả cách người dùng
sử dụng huge page.

## Trước khi bạn bắt đầu (Before you begin)

Các node Kubernetes phải
[cấp phát trước huge page](https://www.kernel.org/doc/html/latest/admin-guide/mm/hugetlbpage.html)
thì node mới báo cáo được sức chứa (capacity) huge page của nó.

Một node có thể cấp phát trước huge page với nhiều kích thước khác nhau. Ví dụ,
dòng sau trong `/etc/default/grub` cấp phát `2*1GiB` gồm các trang 1 GiB và
`512*2 MiB` gồm các trang 2 MiB:

```
GRUB_CMDLINE_LINUX="hugepagesz=1G hugepages=2 hugepagesz=2M hugepages=512"
```

Các node sẽ tự động phát hiện và báo cáo toàn bộ tài nguyên huge page dưới dạng
tài nguyên có thể lập lịch (schedulable resource).

Khi bạn mô tả (describe) Node, bạn sẽ thấy nội dung tương tự như sau ở phần
`Capacity` và `Allocatable`:

```
Capacity:
  cpu:                ...
  ephemeral-storage:  ...
  hugepages-1Gi:      2Gi
  hugepages-2Mi:      1Gi
  memory:             ...
  pods:               ...
Allocatable:
  cpu:                ...
  ephemeral-storage:  ...
  hugepages-1Gi:      2Gi
  hugepages-2Mi:      1Gi
  memory:             ...
  pods:               ...
```

> **Ghi chú:** Với các trang được cấp phát động (sau khi khởi động máy), bạn cần khởi động lại
> kubelet thì các phần cấp phát mới mới được phản ánh.

## API

Huge page có thể được tiêu thụ thông qua khai báo tài nguyên ở mức container, dùng
tên tài nguyên `hugepages-<size>`, trong đó `<size>` là ký hiệu nhị phân gọn nhất
sử dụng các giá trị nguyên được node đó hỗ trợ. Ví dụ, nếu một node hỗ trợ page size
2048KiB và 1048576KiB, node đó sẽ phơi bày hai tài nguyên có thể lập lịch là
`hugepages-2Mi` và `hugepages-1Gi`. Khác với CPU hay bộ nhớ, huge page không hỗ trợ
overcommit. Lưu ý rằng khi yêu cầu tài nguyên hugepage, bạn cũng phải yêu cầu kèm
tài nguyên bộ nhớ hoặc CPU.

Một pod có thể tiêu thụ nhiều kích thước huge page khác nhau trong cùng một pod spec.
Trong trường hợp này, pod phải dùng ký hiệu `medium: HugePages-<hugepagesize>` cho
tất cả các volume mount.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: huge-pages-example
spec:
  containers:
  - name: example
    image: fedora:latest
    command:
    - sleep
    - inf
    volumeMounts:
    - mountPath: /hugepages-2Mi
      name: hugepage-2mi
    - mountPath: /hugepages-1Gi
      name: hugepage-1gi
    resources:
      limits:
        hugepages-2Mi: 100Mi
        hugepages-1Gi: 2Gi
        memory: 100Mi
      requests:
        memory: 100Mi
  volumes:
  - name: hugepage-2mi
    emptyDir:
      medium: HugePages-2Mi
  - name: hugepage-1gi
    emptyDir:
      medium: HugePages-1Gi
```

Một pod chỉ có thể dùng `medium: HugePages` khi nó yêu cầu huge page của duy nhất
một kích thước.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: huge-pages-example
spec:
  containers:
  - name: example
    image: fedora:latest
    command:
    - sleep
    - inf
    volumeMounts:
    - mountPath: /hugepages
      name: hugepage
    resources:
      limits:
        hugepages-2Mi: 100Mi
        memory: 100Mi
      requests:
        memory: 100Mi
  volumes:
  - name: hugepage
    emptyDir:
      medium: HugePages
```

- Giá trị requests của huge page phải bằng giá trị limits. Đây là mặc định khi bạn
  chỉ định limits mà không chỉ định requests.
- Huge page được cô lập ở phạm vi container, nên mỗi container có limit riêng trên
  cgroup sandbox của nó theo đúng khai báo trong container spec.
- Các volume `emptyDir` được hậu thuẫn bởi huge page không được tiêu thụ nhiều bộ nhớ
  huge page hơn mức pod yêu cầu.
- Các ứng dụng tiêu thụ huge page qua `shmget()` với `SHM_HUGETLB` phải chạy với một
  supplemental group khớp với `proc/sys/vm/hugetlb_shm_group`.
- Mức sử dụng huge page trong một namespace có thể được kiểm soát bằng ResourceQuota,
  tương tự các tài nguyên tính toán khác như `cpu` hay `memory`, thông qua token
  `hugepages-<size>`.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 29:

1. Bạn chạy `kubectl describe node lab-k8s-worker1` trên cluster lab. Bạn kỳ vọng thấy gì ở phần
   `Capacity` và `Allocatable` liên quan tới huge page, và điều gì phải xảy ra **trước** trên node
   thì con số ở đó mới khác 0?
2. **Câu bẫy.** Bạn viết một Pod chỉ khai `limits: {hugepages-2Mi: 100Mi}`, không khai `memory`,
   không khai `requests`. Hai chỗ sai ở đây là gì? Và vì sao không thể trông chờ vào overcommit như
   với bộ nhớ thường?
3. Một Pod cần cả trang 2 MiB lẫn trang 1 GiB. Nó được dùng `medium: HugePages` trơn cho volume
   không? Nếu không thì phải viết thế nào?
4. Bạn cấp phát thêm huge page trên node **sau khi máy đã khởi động**, nhưng `describe node` vẫn
   báo con số cũ. Bài chỉ ra thiếu bước nào?
5. Một Pod có hai container, mỗi container khai `hugepages-2Mi: 100Mi`. Giới hạn 100Mi đó được áp ở
   phạm vi nào — cả Pod hay từng container?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Kỳ vọng thấy **không có huge page dùng được** — hoặc không có dòng `hugepages-` nào, hoặc có
   nhưng giá trị bằng `0`. Điều phải xảy ra trước là **node cấp phát trước huge page ở tầng kernel**
   (ví dụ qua `hugepagesz`/`hugepages` trong `GRUB_CMDLINE_LINUX`); chỉ khi đó node mới **tự phát
   hiện và báo cáo** sức chứa huge page như một tài nguyên có thể lập lịch. Kubernetes không tự tạo
   ra huge page — **nó chỉ chia phần thứ mà kernel đã dành sẵn**.
2. Hai chỗ sai: **thiếu yêu cầu memory hoặc CPU đi kèm** — bài nói rõ khi yêu cầu hugepage bạn phải
   yêu cầu kèm bộ nhớ hoặc CPU; và về `requests`, thật ra khai limits mà bỏ trống requests **không
   sai**, vì mặc định requests bằng limits — điều bị cấm là đặt requests **khác** limits. Không trông
   chờ được vào overcommit vì bài nói thẳng: **khác với CPU và bộ nhớ, huge page không hỗ trợ
   overcommit**. Chỗ dễ nhầm là mang trực giác từ `memory` sang: với bộ nhớ thường bạn quen đặt
   requests thấp hơn limits và dựa vào việc không phải ai cũng dùng hết; với huge page thì trang đã
   được kernel dành cứng từ lúc khởi động, không có phần dôi ra để chia.
3. **Không.** `medium: HugePages` trơn chỉ dùng được khi Pod xin huge page của **duy nhất một kích
   thước**. Pod cần cả hai kích thước thì mọi volume mount phải viết dạng có hậu tố —
   `medium: HugePages-2Mi` và `medium: HugePages-1Gi` — đúng như manifest đầu tiên của bài.
4. Thiếu **khởi động lại kubelet**. Ghi chú trong bài nói rõ: với các trang được cấp phát động, sau
   khi khởi động máy, phần cấp phát mới chỉ được phản ánh sau khi kubelet khởi động lại.
5. **Từng container** — huge page được cô lập ở **phạm vi container**, mỗi container có limit riêng
   trên cgroup sandbox của nó theo đúng khai báo trong container spec. Nên Pod đó chiếm tổng cộng
   200Mi huge page 2 MiB, chứ không phải 100Mi dùng chung.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
