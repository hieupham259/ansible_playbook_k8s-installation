# Mở rộng kubectl bằng plugin (Extend kubectl with plugins)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/
>
> Mở rộng kubectl bằng cách tạo và cài đặt các kubectl plugin.

Hướng dẫn này trình bày cách cài đặt và viết các phần mở rộng cho [kubectl](https://kubernetes.io/docs/reference/kubectl/kubectl/).
Nếu xem các lệnh `kubectl` cốt lõi như những khối xây dựng thiết yếu để tương tác với một cluster
Kubernetes, thì quản trị viên cluster có thể xem plugin như một cách tận dụng những khối xây dựng
đó để tạo ra các hành vi phức tạp hơn.
Plugin mở rộng `kubectl` bằng các sub-command mới, cho phép có thêm những tính năng mới và tùy
chỉnh vốn không nằm trong bản phân phối chính của `kubectl`.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một binary `kubectl` đã được cài đặt và hoạt động được.

## Cài đặt kubectl plugin (Installing kubectl plugins)

Một plugin là một file thực thi độc lập (standalone executable), có tên bắt đầu bằng `kubectl-`.
Để cài đặt một plugin, hãy di chuyển file thực thi của nó tới bất kỳ vị trí nào nằm trong `PATH`
của bạn.

Bạn cũng có thể khám phá và cài đặt các kubectl plugin có sẵn trong cộng đồng mã nguồn mở bằng
[Krew](https://krew.dev/). Krew là một trình quản lý plugin (plugin manager) được duy trì bởi
cộng đồng Kubernetes SIG CLI.

> **Thận trọng:** Các kubectl plugin có sẵn thông qua [plugin index](https://krew.sigs.k8s.io/plugins/)
> của Krew không được kiểm định (audit) về mặt bảo mật. Bạn tự chịu rủi ro khi cài đặt và chạy các
> plugin của bên thứ ba, vì chúng là những chương trình tùy ý đang chạy trên máy của bạn.

### Khám phá plugin (Discovering plugins)

`kubectl` cung cấp lệnh `kubectl plugin list` để tìm kiếm trong `PATH` của bạn những file thực thi
plugin hợp lệ. Việc chạy lệnh này khiến toàn bộ file trong `PATH` của bạn được duyệt qua. Bất kỳ
file nào có quyền thực thi và có tên bắt đầu bằng `kubectl-` đều sẽ xuất hiện trong output của lệnh
này *theo đúng thứ tự chúng hiện diện trong `PATH` của bạn*.
Một cảnh báo sẽ được kèm theo với mọi file có tên bắt đầu bằng `kubectl-` nhưng *không* có quyền
thực thi.
Một cảnh báo cũng sẽ được kèm theo với mọi file plugin hợp lệ có tên trùng lặp với nhau.

Bạn có thể dùng [Krew](https://krew.dev/) để khám phá và cài đặt các plugin `kubectl` từ một
[plugin index](https://krew.sigs.k8s.io/plugins/) do cộng đồng tuyển chọn.

#### Plugin cho lệnh create (Create plugins)

`kubectl` cho phép plugin bổ sung các lệnh create tùy chỉnh có dạng `kubectl create something`
bằng cách đặt một binary `kubectl-create-something` trong `PATH`.

#### Giới hạn (Limitations)

Hiện tại chưa thể tạo các plugin ghi đè lên những lệnh `kubectl` đã tồn tại, hoặc mở rộng các lệnh
khác ngoài `create`.
Ví dụ, việc tạo một plugin `kubectl-version` sẽ khiến plugin đó không bao giờ được thực thi, vì
lệnh `kubectl version` sẵn có luôn được ưu tiên hơn nó.
Do giới hạn này, bạn cũng *không* thể dùng plugin để thêm các subcommand mới vào những lệnh
`kubectl` đã tồn tại.
Ví dụ, việc thêm subcommand `kubectl attach vm` bằng cách đặt tên plugin của bạn là
`kubectl-attach-vm` sẽ khiến plugin đó bị bỏ qua.

`kubectl plugin list` sẽ hiển thị cảnh báo cho mọi plugin hợp lệ có ý định làm điều này.

## Viết kubectl plugin (Writing kubectl plugins)

Bạn có thể viết plugin bằng bất kỳ ngôn ngữ lập trình hoặc ngôn ngữ script nào cho phép bạn viết
các lệnh dòng lệnh (command-line).

Không cần cài đặt hay nạp trước (pre-loading) plugin. Các file thực thi của plugin nhận môi trường
được kế thừa từ binary `kubectl`.
Một plugin xác định đường dẫn lệnh (command path) mà nó muốn hiện thực dựa trên tên của chính nó.
Ví dụ, một plugin tên `kubectl-foo` cung cấp lệnh `kubectl foo`. Bạn phải cài đặt file thực thi
của plugin vào một nơi nào đó trong `PATH` của bạn.

### Ví dụ về plugin (Example plugin)

```bash
#!/bin/bash

# xử lý tham số tùy chọn
if [[ "$1" == "version" ]]
then
    echo "1.0.0"
    exit 0
fi

# xử lý tham số tùy chọn
if [[ "$1" == "config" ]]
then
    echo "$KUBECONFIG"
    exit 0
fi

echo "I am a plugin named kubectl-foo"
```

### Sử dụng một plugin (Using a plugin)

Để dùng một plugin, hãy cấp quyền thực thi cho plugin đó:

```shell
sudo chmod +x ./kubectl-foo
```

và đặt nó vào bất kỳ vị trí nào trong `PATH` của bạn:

```shell
sudo mv ./kubectl-foo /usr/local/bin
```

Bây giờ bạn có thể gọi plugin của mình như một lệnh `kubectl`:

```shell
kubectl foo
```

```
I am a plugin named kubectl-foo
```

Mọi tham số (args) và cờ (flags) đều được truyền nguyên vẹn tới file thực thi:

```shell
kubectl foo version
```

```
1.0.0
```

Mọi biến môi trường cũng được truyền nguyên vẹn tới file thực thi:

```bash
export KUBECONFIG=~/.kube/config
kubectl foo config
```

```
/home/<user>/.kube/config
```

```shell
KUBECONFIG=/etc/kube/config kubectl foo config
```

```
/etc/kube/config
```

Ngoài ra, tham số đầu tiên được truyền cho một plugin sẽ luôn là đường dẫn đầy đủ tới vị trí mà nó
được gọi (trong ví dụ ở trên, `$0` sẽ bằng `/usr/local/bin/kubectl-foo`).

### Đặt tên cho plugin (Naming a plugin)

Như đã thấy trong ví dụ ở trên, một plugin xác định đường dẫn lệnh mà nó sẽ hiện thực dựa trên tên
file của nó. Mỗi sub-command trong đường dẫn lệnh mà plugin nhắm tới được phân tách bằng dấu gạch
ngang (`-`).
Ví dụ, một plugin muốn được gọi mỗi khi người dùng gọi lệnh `kubectl foo bar baz` thì phải có tên
file là `kubectl-foo-bar-baz`.

#### Xử lý cờ và tham số (Flags and argument handling)

> **Ghi chú:** Cơ chế plugin _không_ tạo ra bất kỳ giá trị hay biến môi trường tùy chỉnh, riêng
> cho plugin nào cho tiến trình plugin.
>
> Một cơ chế kubectl plugin cũ hơn từng cung cấp các biến môi trường như
> `KUBECTL_PLUGINS_CURRENT_NAMESPACE`; điều đó không còn nữa.

Các kubectl plugin phải tự phân tích (parse) và kiểm tra hợp lệ (validate) toàn bộ tham số được
truyền tới chúng.
Xem [sử dụng gói command line runtime](#using-the-command-line-runtime-package) để biết chi tiết
về một thư viện Go dành cho tác giả plugin.

Dưới đây là một số trường hợp bổ sung khi người dùng gọi plugin của bạn kèm theo các cờ và tham số
bổ sung. Phần này dựa trên plugin `kubectl-foo-bar-baz` từ kịch bản ở trên.

Nếu bạn chạy `kubectl foo bar baz arg1 --flag=value arg2`, cơ chế plugin của kubectl trước tiên sẽ
cố tìm plugin có tên dài nhất có thể, trong trường hợp này là `kubectl-foo-bar-baz-arg1`. Khi không
tìm thấy plugin đó, kubectl sẽ coi giá trị cuối cùng được phân tách bằng dấu gạch ngang là một tham
số (`arg1` trong trường hợp này), rồi thử tìm tên dài nhất kế tiếp, `kubectl-foo-bar-baz`.
Khi đã tìm thấy plugin có tên này, kubectl sẽ gọi plugin đó, truyền toàn bộ các tham số và cờ đứng
sau tên plugin làm tham số cho tiến trình plugin.

Ví dụ:

```bash
# tạo một plugin
echo -e '#!/bin/bash\n\necho "My first command-line argument was $1"' > kubectl-foo-bar-baz
sudo chmod +x ./kubectl-foo-bar-baz

# "cài đặt" plugin của bạn bằng cách chuyển nó vào một thư mục nằm trong $PATH
sudo mv ./kubectl-foo-bar-baz /usr/local/bin

# kiểm tra xem kubectl có nhận diện được plugin của bạn không
kubectl plugin list
```

```
The following kubectl-compatible plugins are available:

/usr/local/bin/kubectl-foo-bar-baz
```

```
# kiểm tra rằng việc gọi plugin của bạn qua một lệnh "kubectl" vẫn hoạt động
# ngay cả khi người dùng truyền thêm tham số và cờ vào
# file thực thi plugin của bạn.
kubectl foo bar baz arg1 --meaningless-flag=true
```

```
My first command-line argument was arg1
```

Như bạn thấy, plugin của bạn đã được tìm thấy dựa trên lệnh `kubectl` mà người dùng chỉ định, và
toàn bộ các tham số cùng cờ bổ sung đã được truyền nguyên vẹn tới file thực thi plugin sau khi nó
được tìm thấy.

#### Tên chứa dấu gạch ngang và dấu gạch dưới (Names with dashes and underscores)

Mặc dù cơ chế plugin của `kubectl` dùng dấu gạch ngang (`-`) trong tên file plugin để phân tách
chuỗi các sub-command mà plugin xử lý, bạn vẫn có thể tạo một lệnh plugin chứa dấu gạch ngang trong
cách gọi trên dòng lệnh bằng cách dùng dấu gạch dưới (`_`) trong tên file của nó.

Ví dụ:

```bash
# tạo một plugin có dấu gạch dưới trong tên file
echo -e '#!/bin/bash\n\necho "I am a plugin with a dash in my name"' > ./kubectl-foo_bar
sudo chmod +x ./kubectl-foo_bar

# chuyển plugin vào $PATH của bạn
sudo mv ./kubectl-foo_bar /usr/local/bin

# Bây giờ bạn có thể gọi plugin của mình qua kubectl:
kubectl foo-bar
```

```
I am a plugin with a dash in my name
```

Lưu ý rằng việc đưa dấu gạch dưới vào tên file plugin không ngăn bạn có các lệnh như
`kubectl foo_bar`.
Lệnh trong ví dụ trên có thể được gọi bằng dấu gạch ngang (`-`) hoặc dấu gạch dưới (`_`):

```bash
# Bạn có thể gọi lệnh tùy chỉnh của mình bằng dấu gạch ngang
kubectl foo-bar
```

```
I am a plugin with a dash in my name
```

```bash
# Bạn cũng có thể gọi lệnh tùy chỉnh của mình bằng dấu gạch dưới
kubectl foo_bar
```

```
I am a plugin with a dash in my name
```

#### Xung đột tên và che khuất (Name conflicts and overshadowing)

Bạn hoàn toàn có thể có nhiều plugin cùng tên file ở những vị trí khác nhau trong `PATH` của mình.
Ví dụ, với một `PATH` có giá trị như sau: `PATH=/usr/local/bin/plugins:/usr/local/bin/moreplugins`,
một bản sao của plugin `kubectl-foo` có thể tồn tại đồng thời trong `/usr/local/bin/plugins` và
`/usr/local/bin/moreplugins`, khiến output của lệnh `kubectl plugin list` là:

```bash
PATH=/usr/local/bin/plugins:/usr/local/bin/moreplugins kubectl plugin list
```

```
The following kubectl-compatible plugins are available:

/usr/local/bin/plugins/kubectl-foo
/usr/local/bin/moreplugins/kubectl-foo
  - warning: /usr/local/bin/moreplugins/kubectl-foo is overshadowed by a similarly named plugin: /usr/local/bin/plugins/kubectl-foo

error: one plugin warning was found
```

Trong kịch bản trên, cảnh báo nằm dưới `/usr/local/bin/moreplugins/kubectl-foo` cho bạn biết rằng
plugin này sẽ không bao giờ được thực thi. Thay vào đó, file thực thi xuất hiện trước tiên trong
`PATH` của bạn, tức `/usr/local/bin/plugins/kubectl-foo`, sẽ luôn được cơ chế plugin của `kubectl`
tìm thấy và thực thi trước.

Một cách để giải quyết vấn đề này là bảo đảm vị trí của plugin mà bạn muốn dùng với `kubectl` luôn
đứng trước trong `PATH` của bạn. Ví dụ, nếu bạn luôn muốn dùng
`/usr/local/bin/moreplugins/kubectl-foo` mỗi khi lệnh `kubectl foo` của `kubectl` được người dùng gọi, hãy đổi
giá trị `PATH` của bạn thành `/usr/local/bin/moreplugins:/usr/local/bin/plugins`.

#### Gọi tên file thực thi dài nhất (Invocation of the longest executable filename)

Có một kiểu che khuất khác cũng có thể xảy ra với tên file plugin. Với hai plugin cùng hiện diện
trong `PATH` của người dùng: `kubectl-foo-bar` và `kubectl-foo-bar-baz`, cơ chế plugin của
`kubectl` sẽ luôn chọn tên plugin dài nhất có thể ứng với một lệnh người dùng nhất định. Một vài
ví dụ dưới đây làm rõ thêm điều này:

```bash
# với một lệnh kubectl cho trước, plugin có tên file dài nhất có thể sẽ luôn được ưu tiên
kubectl foo bar baz
```

```
Plugin kubectl-foo-bar-baz is executed
```

```bash
kubectl foo bar
```

```
Plugin kubectl-foo-bar is executed
```

```bash
kubectl foo bar baz buz
```

```
Plugin kubectl-foo-bar-baz is executed, with "buz" as its first argument
```

```bash
kubectl foo bar buz
```

```
Plugin kubectl-foo-bar is executed, with "buz" as its first argument
```

Lựa chọn thiết kế này bảo đảm rằng các sub-command của plugin có thể được hiện thực trên nhiều file
khác nhau nếu cần, và rằng những sub-command này có thể được lồng dưới một lệnh plugin "cha":

```bash
ls ./plugin_command_tree
```

```
kubectl-parent
kubectl-parent-subcommand
kubectl-parent-subcommand-subsubcommand
```

### Kiểm tra cảnh báo của plugin (Checking for plugin warnings)

Bạn có thể dùng lệnh `kubectl plugin list` đã nhắc tới ở trên để bảo đảm rằng plugin của bạn được
`kubectl` nhìn thấy, và để xác nhận rằng không có cảnh báo nào ngăn nó được gọi như một lệnh
`kubectl`.

```bash
kubectl plugin list
```

```
The following kubectl-compatible plugins are available:

test/fixtures/pkg/kubectl/plugins/kubectl-foo
/usr/local/bin/kubectl-foo
  - warning: /usr/local/bin/kubectl-foo is overshadowed by a similarly named plugin: test/fixtures/pkg/kubectl/plugins/kubectl-foo
plugins/kubectl-invalid
  - warning: plugins/kubectl-invalid identified as a kubectl plugin, but it is not executable

error: 2 plugin warnings were found
```

### Sử dụng gói command line runtime (Using the command line runtime package) {#using-the-command-line-runtime-package}

Nếu bạn đang viết một plugin cho kubectl và bạn dùng Go, bạn có thể tận dụng các thư viện tiện ích
[cli-runtime](https://github.com/kubernetes/cli-runtime).

Những thư viện này cung cấp các hàm hỗ trợ để phân tích hoặc cập nhật file
[kubeconfig](111-kubeconfig-vi.md)
của người dùng, để thực hiện các request kiểu REST tới API server, hoặc để gắn (bind) các cờ liên
quan tới cấu hình và việc in ấn (printing).

Xem [Sample CLI Plugin](https://github.com/kubernetes/sample-cli-plugin) để có một ví dụ về cách
sử dụng các công cụ được cung cấp trong repo CLI Runtime.

## Phân phối kubectl plugin (Distributing kubectl plugins)

Nếu bạn đã phát triển một plugin để người khác dùng, bạn nên cân nhắc cách đóng gói, phân phối và
đưa các bản cập nhật tới người dùng của mình.

### Krew {#distributing-krew}

[Krew](https://krew.dev/) cung cấp một cách đóng gói và phân phối plugin của bạn theo kiểu đa nền
tảng (cross-platform). Theo cách này, bạn dùng một định dạng đóng gói duy nhất cho tất cả các nền
tảng mục tiêu (Linux, Windows, macOS, v.v.) và đưa các bản cập nhật tới người dùng của mình.
Krew cũng duy trì một [plugin index](https://krew.sigs.k8s.io/plugins/) để người khác có thể khám
phá plugin của bạn và cài đặt nó.

### Quản lý gói gốc / theo từng nền tảng (Native / platform specific package management) {#distributing-native}

Ngoài ra, bạn có thể dùng các trình quản lý gói truyền thống như `apt` hoặc `yum` trên Linux,
Chocolatey trên Windows, và Homebrew trên macOS. Bất kỳ trình quản lý gói nào cũng phù hợp nếu nó
có thể đặt các file thực thi mới vào một nơi nào đó trong `PATH` của người dùng.
Là tác giả plugin, nếu bạn chọn phương án này thì bạn cũng phải gánh thêm việc cập nhật gói phân
phối kubectl plugin của mình trên nhiều nền tảng cho mỗi lần phát hành (release).

### Mã nguồn (Source code) {#distributing-source-code}

Bạn có thể công bố mã nguồn; ví dụ, dưới dạng một Git repository. Nếu bạn chọn phương án này, người
muốn dùng plugin đó phải lấy mã nguồn về, thiết lập môi trường build (nếu cần biên dịch), và triển
khai plugin. Nếu bạn cũng cung cấp sẵn các gói đã biên dịch, hoặc dùng Krew, thì việc cài đặt sẽ dễ
dàng hơn.

## Tiếp theo (What's next)

* Xem repository Sample CLI Plugin để có một
  [ví dụ chi tiết](https://github.com/kubernetes/sample-cli-plugin) về một plugin viết bằng Go.
  Nếu có bất kỳ câu hỏi nào, hãy thoải mái liên hệ
  [nhóm SIG CLI](https://github.com/kubernetes/community/tree/main/sig-cli).
* Đọc về [Krew](https://krew.dev/), một trình quản lý gói dành cho các kubectl plugin.
