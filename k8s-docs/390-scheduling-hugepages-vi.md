# Quản lý HugePages (Manage HugePages)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/manage-hugepages/scheduling-hugepages/>
>
> Cấu hình và quản lý huge page như một tài nguyên có thể lập lịch (schedulable resource) trong cluster.

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
