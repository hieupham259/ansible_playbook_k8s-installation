# Bảo đảm lập lịch cho các Pod add-on quan trọng (Guaranteed Scheduling For Critical Add-On Pods)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/>

Các thành phần cốt lõi của Kubernetes như API server, scheduler và controller-manager chạy trên
node control plane. Tuy nhiên, các add-on lại phải chạy trên node thông thường của cluster.
Một số add-on trong đó là thiết yếu để cluster hoạt động đầy đủ chức năng, chẳng hạn như
metrics-server, DNS và UI. Cluster có thể ngừng hoạt động bình thường nếu một add-on quan trọng
bị trục xuất (evict) — dù là thủ công hay như một tác dụng phụ của thao tác khác, ví dụ nâng cấp —
và rơi vào trạng thái pending (ví dụ khi cluster đang được sử dụng ở mức cao và hoặc là có các pod
pending khác được lập lịch vào chỗ trống mà pod add-on quan trọng vừa bị trục xuất để lại, hoặc là
lượng tài nguyên khả dụng trên node thay đổi vì một lý do nào đó khác).

Lưu ý rằng việc đánh dấu một pod là quan trọng (critical) không nhằm ngăn chặn hoàn toàn việc
trục xuất; nó chỉ ngăn pod đó trở nên không khả dụng vĩnh viễn. Một static pod được đánh dấu là
quan trọng thì không thể bị trục xuất. Tuy nhiên, các pod không phải static được đánh dấu là
quan trọng sẽ luôn được lập lịch lại.

### Đánh dấu pod là quan trọng (Marking pod as critical) {#marking-pod-as-critical}

Để đánh dấu một Pod là quan trọng, hãy đặt priorityClassName cho Pod đó thành
`system-cluster-critical` hoặc `system-node-critical`. `system-node-critical` là mức ưu tiên
(priority) cao nhất hiện có, thậm chí còn cao hơn cả `system-cluster-critical`.
