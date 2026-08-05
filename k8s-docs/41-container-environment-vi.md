# Môi trường Container (Container Environment)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/containers/container-environment/>

Trang này mô tả các tài nguyên khả dụng cho các Container trong môi trường Container.

## Môi trường container (Container environment)

Môi trường Container của Kubernetes cung cấp một số tài nguyên quan trọng cho các Container:

* Một hệ thống tệp (filesystem), là sự kết hợp của một [image](./40-images-vi.md)
  và một hoặc nhiều [volume](https://kubernetes.io/docs/concepts/storage/volumes/).
* Thông tin về chính Container đó.
* Thông tin về các đối tượng (object) khác trong cluster.

### Thông tin về Container (Container information)

*Hostname* của một Container là tên của Pod mà Container đó đang chạy bên trong.
Nó có thể được lấy thông qua lệnh `hostname` hoặc lời gọi hàm
[`gethostname`](https://man7.org/linux/man-pages/man2/gethostname.2.html)
trong libc.

Tên Pod và namespace được cung cấp dưới dạng các biến môi trường (environment variable)
thông qua [downward API](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/).

Các biến môi trường do người dùng định nghĩa trong định nghĩa Pod cũng khả dụng
cho Container, cùng với mọi biến môi trường được chỉ định tĩnh trong container image.

### Thông tin về cluster (Cluster information)

Danh sách tất cả các service đang chạy tại thời điểm một Container được tạo sẽ khả dụng
cho Container đó dưới dạng các biến môi trường.
Danh sách này giới hạn trong các service thuộc cùng namespace với Pod của Container mới
và các service của control plane Kubernetes.

Với một service tên là *foo* expose một tập các Pod, mỗi Pod chạy một container tên là *bar*,
các biến sau được định nghĩa:

```shell
FOO_SERVICE_HOST=<the host the service is running on>
FOO_SERVICE_PORT=<the port the service is running on>
```

Các Service có địa chỉ IP riêng và khả dụng cho Container thông qua DNS,
nếu [DNS addon](https://releases.k8s.io/v1.36.0/cluster/addons/dns/) được bật.

## Tiếp theo (What's next)

* Tìm hiểu thêm về [các hook vòng đời Container (Container lifecycle hooks)](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/).
* Thực hành thực tế việc
  [gắn handler vào các sự kiện trong vòng đời Container](https://kubernetes.io/docs/tasks/configure-pod-container/attach-handler-lifecycle-event/).
