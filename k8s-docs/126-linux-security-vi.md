# Bảo mật cho node Linux (Security For Linux Nodes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/linux-security/>

Trang này mô tả các cân nhắc về bảo mật và các thực hành tốt dành riêng cho hệ điều hành Linux.

## Bảo vệ dữ liệu Secret trên node (Protection for Secret data on nodes)

Trên các node Linux, những volume lưu trong bộ nhớ (memory-backed volume) — chẳng hạn như
volume mount kiểu [`secret`](./109-secret-vi.md), hoặc [`emptyDir`](./91-volumes-vi.md#emptydir)
với `medium: Memory` — được hiện thực bằng filesystem `tmpfs`.

Nếu bạn có cấu hình swap và đang dùng một Linux kernel cũ (hoặc kernel hiện tại nhưng với một
cấu hình Kubernetes không được hỗ trợ), các volume lưu trong **bộ nhớ** vẫn có thể bị ghi
dữ liệu xuống bộ lưu trữ bền vững (persistent storage).

Linux kernel hỗ trợ chính thức tùy chọn `noswap` từ phiên bản 6.3, do đó nếu swap được bật
trên node, khuyến nghị dùng kernel phiên bản 6.3 trở lên, hoặc kernel có hỗ trợ tùy chọn
`noswap` thông qua backport.

Đọc [quản lý bộ nhớ swap (swap memory management)](https://kubernetes.io/docs/concepts/cluster-administration/swap-memory-management/#memory-backed-volumes)
để biết thêm thông tin.
