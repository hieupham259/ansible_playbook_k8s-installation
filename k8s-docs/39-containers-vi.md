# Các Container (Containers)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/containers/>
>
> Công nghệ đóng gói một ứng dụng cùng với các dependency lúc chạy (runtime dependency) của nó.

Trang này sẽ thảo luận về container và container image, cũng như việc sử dụng chúng
trong vận hành (operations) và phát triển giải pháp (solution development).

Từ _container_ là một thuật ngữ mang nhiều nghĩa (overloaded term). Bất cứ khi nào bạn
dùng từ này, hãy kiểm tra xem người nghe có đang dùng cùng một định nghĩa hay không.

Mỗi container mà bạn chạy đều có thể lặp lại được (repeatable); việc chuẩn hóa nhờ đã
bao gồm sẵn các dependency đồng nghĩa với việc bạn nhận được cùng một hành vi ở bất kỳ
đâu bạn chạy nó.

Container tách rời (decouple) ứng dụng khỏi hạ tầng host bên dưới. Điều này giúp việc
triển khai dễ dàng hơn trên các môi trường cloud hoặc hệ điều hành khác nhau.

Mỗi node trong một cluster Kubernetes chạy các container tạo nên các
[Pod](https://kubernetes.io/docs/concepts/workloads/pods/) được gán cho node đó.
Các container trong một Pod được đặt cùng chỗ (co-located) và được lập lịch cùng nhau
(co-scheduled) để chạy trên cùng một node.

## Container image (Container images)

Một [container image](./40-images-vi.md) là một gói phần mềm sẵn sàng để chạy, chứa
mọi thứ cần thiết để chạy một ứng dụng: mã nguồn và mọi runtime mà nó cần, các thư viện
của ứng dụng và của hệ thống, cùng các giá trị mặc định cho mọi thiết lập thiết yếu.

Container được thiết kế để không lưu trạng thái (stateless) và
[bất biến (immutable)](https://glossary.cncf.io/immutable-infrastructure/):
bạn không nên thay đổi mã của một container đang chạy. Nếu bạn có một ứng dụng đã được
container hóa và muốn thực hiện thay đổi, quy trình đúng là build một image mới bao gồm
thay đổi đó, rồi tạo lại container để khởi động từ image đã được cập nhật.

## Container runtime (Container runtimes)

Một thành phần nền tảng giúp Kubernetes chạy các container một cách hiệu quả.
Nó chịu trách nhiệm quản lý việc thực thi và vòng đời (lifecycle) của các container
trong môi trường Kubernetes.

Kubernetes hỗ trợ các container runtime như containerd, CRI-O, và bất kỳ hiện thực
(implementation) nào khác của
[Kubernetes CRI (Container Runtime Interface)](https://github.com/kubernetes/community/blob/main/contributors/devel/sig-node/container-runtime-interface.md).

Thông thường, bạn có thể để cluster tự chọn container runtime mặc định cho một Pod.
Nếu bạn cần dùng nhiều hơn một container runtime trong cluster, bạn có thể chỉ định
[RuntimeClass](https://kubernetes.io/docs/concepts/containers/runtime-class/)
cho một Pod để bảo đảm rằng Kubernetes chạy các container đó bằng một container runtime
cụ thể.

Bạn cũng có thể dùng RuntimeClass để chạy các Pod khác nhau với cùng một container
runtime nhưng với các thiết lập khác nhau.
