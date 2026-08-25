# Chuẩn bảo mật Pod (Pod Security Standards)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/pod-security-standards/>
>
> Cái nhìn chi tiết về các cấp độ chính sách (policy) khác nhau được định nghĩa trong Chuẩn bảo mật Pod (Pod Security Standards).

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), bài 6/18 · Kiểm chứng ở [Lab 9b](labs/LAB-9B-POD-SECURITY-VA-HARDENING.md).

Bài dài hơn 600 dòng, nhưng khoảng ba phần tư là **hai bảng đặc tả trường** viết bằng HTML.
Đừng học thuộc bảng — đó là tài liệu tra cứu lúc viết manifest. Bài này chỉ **định nghĩa** ba
profile; việc **áp** chúng vào namespace là bài kế tiếp,
[116](116-pod-security-admission-vi.md). Đọc hai bài này liền nhau.

**Phải hiểu ở lần đọc này:**

- Ba profile có tính **tích lũy**, trải từ dễ dãi tới hạn chế: **Privileged** (không hạn chế
  gì, cho workload cấp hệ thống và hạ tầng do người dùng đặc quyền quản lý), **Baseline** (hạn
  chế tối thiểu, ngăn các cách leo thang đặc quyền đã biết, vẫn chạy được cấu hình Pod mặc
  định), **Restricted** (hardening theo thực hành tốt nhất, **đánh đổi bằng tính tương thích**).
- Ranh giới của Baseline, tức những thứ nó cấm: chia sẻ namespace của host
  (`hostNetwork`, `hostPID`, `hostIPC`), container `privileged`, volume `hostPath`, thêm
  capability ngoài danh sách cho phép, và đặt seccomp profile thành `Unconfined`.
- Restricted **thêm** gì trên Baseline: giới hạn danh sách loại volume được phép,
  `allowPrivilegeEscalation: false`, `runAsNonRoot: true` cùng `runAsUser` khác 0, seccomp
  profile phải được đặt **tường minh** (không đặt cũng bị cấm), và **drop `ALL` capability**,
  chỉ được thêm lại `NET_BIND_SERVICE`.
- Ghi chú về wildcard đứng trước cả hai bảng: `spec.containers[*]` nghĩa là **tất cả** container
  đã định nghĩa, và **chỉ cần một container không đạt là cả Pod trượt validation**.
- Phân biệt trong mục FAQ: **security context** là cấu hình trong manifest Pod/container, là
  tham số truyền cho container runtime; **security profile** là cơ chế **của control plane** để
  thực thi những thiết lập đó cùng các tham số liên quan nằm ngoài security context.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Toàn bộ danh sách trường trong hai bảng Baseline và Restricted | là bảng tra cứu lúc viết manifest, không phải nội dung học thuộc | giai đoạn 9, khi làm Lab 9b |
| Ý nghĩa của seccomp, AppArmor, SELinux, capability trong bảng | chưa học các ràng buộc bảo mật của kernel | bài [127](127-linux-kernel-security-vi.md) |
| `windowsOptions.hostProcess`, mục *Trường OS của Pod* và *Các kiểm soát chính sách đặc thù theo hệ điều hành* | cluster lab chỉ có node Linux | giai đoạn 15 |
| *Khởi tạo chính sách* — ba manifest Namespace kèm label | chính là nội dung bài kế tiếp | bài [116](116-pod-security-admission-vi.md) |
| *Các lựa chọn thay thế* — Kubewarden, Kyverno, OPA Gatekeeper | công cụ bên thứ ba ngoài lộ trình | không cần |
| FAQ *Không gian tên người dùng* và *Còn các Pod sandbox thì sao?* | cần user namespace và runtime sandbox | bài [55](55-user-namespaces-vi.md) và [43](43-runtime-class-vi.md) |

---

Chuẩn bảo mật Pod định nghĩa ba _chính sách_ (policy) khác nhau nhằm bao quát rộng
phổ bảo mật. Các chính sách này có tính _tích lũy_ (cumulative) và trải dài từ mức rất
dễ dãi (highly-permissive) đến mức rất hạn chế (highly-restrictive).
Hướng dẫn này trình bày các yêu cầu của từng chính sách.

| Profile | Mô tả |
| ------ | ----------- |
| <strong style="white-space: nowrap">Privileged</strong> | Chính sách không hạn chế, cung cấp mức quyền rộng nhất có thể. Chính sách này cho phép cả những cách leo thang đặc quyền (privilege escalation) đã biết. |
| <strong style="white-space: nowrap">Baseline</strong> | Chính sách hạn chế ở mức tối thiểu, ngăn chặn các cách leo thang đặc quyền đã biết. Cho phép cấu hình Pod mặc định (khai báo tối thiểu). |
| <strong style="white-space: nowrap">Restricted</strong> | Chính sách hạn chế nghiêm ngặt, tuân theo các thực hành tốt nhất hiện tại về gia cố (hardening) Pod. |

## Chi tiết các profile (Profile Details)

### Privileged

**Chính sách _Privileged_ được chủ đích để mở, và hoàn toàn không hạn chế.** Loại chính
sách này thường nhắm đến các workload cấp hệ thống và hạ tầng, được quản lý bởi những
người dùng đặc quyền, đáng tin cậy.

Chính sách Privileged được định nghĩa bằng việc không có bất kỳ hạn chế nào. Nếu bạn định
nghĩa một Pod mà chính sách bảo mật Privileged được áp dụng, Pod đó có thể vượt qua các
cơ chế cô lập container thông thường. Ví dụ, bạn có thể định nghĩa một Pod có quyền truy
cập vào mạng host của node.

### Baseline

**Chính sách _Baseline_ hướng đến việc dễ dàng áp dụng cho các workload container hóa
thông thường, đồng thời ngăn chặn các cách leo thang đặc quyền đã biết.** Chính sách này
nhắm đến những người vận hành ứng dụng và các nhà phát triển của những ứng dụng không
trọng yếu. Các biện pháp kiểm soát (control) liệt kê dưới đây cần được thực thi/cấm:

> **Ghi chú:** Trong bảng này, ký tự đại diện (wildcard, `*`) biểu thị tất cả các phần tử
> trong một danh sách. Ví dụ, `spec.containers[*].securityContext` chỉ đối tượng Security
> Context của _tất cả các container đã được định nghĩa_. Nếu bất kỳ container nào trong
> danh sách không đáp ứng các yêu cầu, toàn bộ Pod sẽ không vượt qua được bước kiểm tra
> hợp lệ (validation).

<table>
	<caption style="display:none">Đặc tả chính sách Baseline</caption>
	<tbody>
		<tr>
			<th>Kiểm soát (Control)</th>
			<th>Chính sách (Policy)</th>
		</tr>
		<tr>
			<td style="white-space: nowrap">HostProcess</td>
			<td>
				<p>Các Pod Windows cho phép chạy <a href="https://kubernetes.io/docs/tasks/configure-pod-container/create-hostprocess-pod">container HostProcess</a>, cho phép truy cập đặc quyền vào máy chủ Windows. Truy cập đặc quyền vào host bị cấm trong chính sách Baseline. <strong>TRẠNG THÁI TÍNH NĂNG:</strong> <code>Kubernetes v1.26 [stable]</code></p>
				<p><strong>Các trường bị hạn chế (Restricted Fields)</strong></p>
				<ul>
					<li><code>spec.securityContext.windowsOptions.hostProcess</code></li>
					<li><code>spec.containers[*].securityContext.windowsOptions.hostProcess</code></li>
					<li><code>spec.initContainers[*].securityContext.windowsOptions.hostProcess</code></li>
					<li><code>spec.ephemeralContainers[*].securityContext.windowsOptions.hostProcess</code></li>
				</ul>
				<p><strong>Các giá trị được phép (Allowed Values)</strong></p>
				<ul>
					<li>Không định nghĩa/nil</li>
					<li><code>false</code></li>
				</ul>
			</td>
		</tr>
		<tr>
			<td style="white-space: nowrap">Host Namespaces</td>
			<td>
				<p>Việc chia sẻ các namespace của host phải bị cấm.</p>
				<p><strong>Các trường bị hạn chế (Restricted Fields)</strong></p>
				<ul>
					<li><code>spec.hostNetwork</code></li>
					<li><code>spec.hostPID</code></li>
					<li><code>spec.hostIPC</code></li>
				</ul>
				<p><strong>Các giá trị được phép (Allowed Values)</strong></p>
				<ul>
					<li>Không định nghĩa/nil</li>
					<li><code>false</code></li>
				</ul>
			</td>
		</tr>
		<tr>
			<td style="white-space: nowrap">Privileged Containers</td>
			<td>
				<p>Các Pod đặc quyền (privileged) vô hiệu hóa hầu hết các cơ chế bảo mật và phải bị cấm.</p>
				<p><strong>Các trường bị hạn chế (Restricted Fields)</strong></p>
				<ul>
					<li><code>spec.containers[*].securityContext.privileged</code></li>
					<li><code>spec.initContainers[*].securityContext.privileged</code></li>
					<li><code>spec.ephemeralContainers[*].securityContext.privileged</code></li>
				</ul>
				<p><strong>Các giá trị được phép (Allowed Values)</strong></p>
				<ul>
					<li>Không định nghĩa/nil</li>
					<li><code>false</code></li>
				</ul>
			</td>
		</tr>
		<tr>
			<td style="white-space: nowrap">Capabilities</td>
			<td>
				<p>Việc thêm các capability khác ngoài những capability được liệt kê dưới đây phải bị cấm.</p>
				<p><strong>Các trường bị hạn chế (Restricted Fields)</strong></p>
				<ul>
					<li><code>spec.containers[*].securityContext.capabilities.add</code></li>
					<li><code>spec.initContainers[*].securityContext.capabilities.add</code></li>
					<li><code>spec.ephemeralContainers[*].securityContext.capabilities.add</code></li>
				</ul>
				<p><strong>Các giá trị được phép (Allowed Values)</strong></p>
				<ul>
					<li>Không định nghĩa/nil</li>
					<li><code>AUDIT_WRITE</code></li>
					<li><code>CHOWN</code></li>
					<li><code>DAC_OVERRIDE</code></li>
					<li><code>FOWNER</code></li>
					<li><code>FSETID</code></li>
					<li><code>KILL</code></li>
					<li><code>MKNOD</code></li>
					<li><code>NET_BIND_SERVICE</code></li>
					<li><code>SETFCAP</code></li>
					<li><code>SETGID</code></li>
					<li><code>SETPCAP</code></li>
					<li><code>SETUID</code></li>
					<li><code>SYS_CHROOT</code></li>
				</ul>
			</td>
		</tr>
		<tr>
			<td style="white-space: nowrap">HostPath Volumes</td>
			<td>
				<p>Volume kiểu HostPath phải bị cấm.</p>
				<p><strong>Các trường bị hạn chế (Restricted Fields)</strong></p>
				<ul>
					<li><code>spec.volumes[*].hostPath</code></li>
				</ul>
				<p><strong>Các giá trị được phép (Allowed Values)</strong></p>
				<ul>
					<li>Không định nghĩa/nil</li>
				</ul>
			</td>
		</tr>
		<tr>
			<td style="white-space: nowrap">Host Ports</td>
			<td>
				<p>HostPort nên bị cấm hoàn toàn (khuyến nghị) hoặc bị giới hạn trong một danh sách đã biết</p>
				<p><strong>Các trường bị hạn chế (Restricted Fields)</strong></p>
				<ul>
					<li><code>spec.containers[*].ports[*].hostPort</code></li>
					<li><code>spec.initContainers[*].ports[*].hostPort</code></li>
					<li><code>spec.ephemeralContainers[*].ports[*].hostPort</code></li>
				</ul>
				<p><strong>Các giá trị được phép (Allowed Values)</strong></p>
				<ul>
					<li>Không định nghĩa/nil</li>
					<li>Danh sách đã biết (không được hỗ trợ bởi <a href="https://kubernetes.io/docs/concepts/security/pod-security-admission/">bộ điều khiển Pod Security Admission</a> tích hợp sẵn)</li>
					<li><code>0</code></li>
				</ul>
			</td>
		</tr>
		<tr>
			<td>Host Probes / Lifecycle Hooks (v1.34+)</td>
			<td>
				<p>Trường Host trong các probe và lifecycle hook phải bị cấm.</p>
				<p><strong>Các trường bị hạn chế (Restricted Fields)</strong></p>
				<ul>
					<li><code>spec.containers[*].livenessProbe.httpGet.host</code></li>
					<li><code>spec.containers[*].readinessProbe.httpGet.host</code></li>
					<li><code>spec.containers[*].startupProbe.httpGet.host</code></li>
					<li><code>spec.containers[*].livenessProbe.tcpSocket.host</code></li>
					<li><code>spec.containers[*].readinessProbe.tcpSocket.host</code></li>
					<li><code>spec.containers[*].startupProbe.tcpSocket.host</code></li>
					<li><code>spec.containers[*].lifecycle.postStart.tcpSocket.host</code>
					<li><code>spec.containers[*].lifecycle.preStop.tcpSocket.host</code>
					<li><code>spec.containers[*].lifecycle.postStart.httpGet.host</code></li>
					<li><code>spec.containers[*].lifecycle.preStop.httpGet.host</code></li>
					<li><code>spec.initContainers[*].livenessProbe.httpGet.host</code></li>
					<li><code>spec.initContainers[*].readinessProbe.httpGet.host</code></li>
					<li><code>spec.initContainers[*].startupProbe.httpGet.host</code></li>
					<li><code>spec.initContainers[*].livenessProbe.tcpSocket.host</code></li>
					<li><code>spec.initContainers[*].readinessProbe.tcpSocket.host</code></li>
					<li><code>spec.initContainers[*].startupProbe.tcpSocket.host</code></li>
					<li><code>spec.initContainers[*].lifecycle.postStart.tcpSocket.host</code>
					<li><code>spec.initContainers[*].lifecycle.preStop.tcpSocket.host</code>
					<li><code>spec.initContainers[*].lifecycle.postStart.httpGet.host</code></li>
					<li><code>spec.initContainers[*].lifecycle.preStop.httpGet.host</code></li>
				</ul>
				<p><strong>Các giá trị được phép (Allowed Values)</strong></p>
				<ul>
					<li>Không định nghĩa/nil</li>
					<li>""</li>
				</ul>
			</td>
		</tr>
		<tr>
			<td style="white-space: nowrap">AppArmor</td>
			<td>
				<p>Trên các host được hỗ trợ, profile AppArmor <code>RuntimeDefault</code> được áp dụng theo mặc định. Chính sách baseline nên ngăn việc ghi đè hoặc vô hiệu hóa profile AppArmor mặc định, hoặc giới hạn việc ghi đè trong một tập các profile được cho phép.</p>
				<p><strong>Các trường bị hạn chế (Restricted Fields)</strong></p>
				<ul>
					<li><code>spec.securityContext.appArmorProfile.type</code></li>
					<li><code>spec.containers[*].securityContext.appArmorProfile.type</code></li>
					<li><code>spec.initContainers[*].securityContext.appArmorProfile.type</code></li>
					<li><code>spec.ephemeralContainers[*].securityContext.appArmorProfile.type</code></li>
				</ul>
				<p><strong>Các giá trị được phép (Allowed Values)</strong></p>
				<ul>
					<li>Không định nghĩa/nil</li>
					<li><code>RuntimeDefault</code></li>
					<li><code>Localhost</code></li>
				</ul>
				<hr />
				<ul>
					<li><code>metadata.annotations["container.apparmor.security.beta.kubernetes.io/*"]</code></li>
				</ul>
				<p><strong>Các giá trị được phép (Allowed Values)</strong></p>
				<ul>
					<li>Không định nghĩa/nil</li>
					<li><code>runtime/default</code></li>
					<li><code>localhost/*</code></li>
				</ul>
			</td>
		</tr>
		<tr>
			<td style="white-space: nowrap">SELinux</td>
			<td>
				<p>Việc đặt kiểu (type) SELinux bị hạn chế, và việc đặt tùy chọn user hoặc role SELinux tùy chỉnh bị cấm.</p>
				<p><strong>Các trường bị hạn chế (Restricted Fields)</strong></p>
				<ul>
					<li><code>spec.securityContext.seLinuxOptions.type</code></li>
					<li><code>spec.containers[*].securityContext.seLinuxOptions.type</code></li>
					<li><code>spec.initContainers[*].securityContext.seLinuxOptions.type</code></li>
					<li><code>spec.ephemeralContainers[*].securityContext.seLinuxOptions.type</code></li>
				</ul>
				<p><strong>Các giá trị được phép (Allowed Values)</strong></p>
				<ul>
					<li>Không định nghĩa/""</li>
					<li><code>container_t</code></li>
					<li><code>container_init_t</code></li>
					<li><code>container_kvm_t</code></li>
					<li><code>container_engine_t</code> (từ Kubernetes 1.31)</li>
				</ul>
				<hr />
				<p><strong>Các trường bị hạn chế (Restricted Fields)</strong></p>
				<ul>
					<li><code>spec.securityContext.seLinuxOptions.user</code></li>
					<li><code>spec.containers[*].securityContext.seLinuxOptions.user</code></li>
					<li><code>spec.initContainers[*].securityContext.seLinuxOptions.user</code></li>
					<li><code>spec.ephemeralContainers[*].securityContext.seLinuxOptions.user</code></li>
					<li><code>spec.securityContext.seLinuxOptions.role</code></li>
					<li><code>spec.containers[*].securityContext.seLinuxOptions.role</code></li>
					<li><code>spec.initContainers[*].securityContext.seLinuxOptions.role</code></li>
					<li><code>spec.ephemeralContainers[*].securityContext.seLinuxOptions.role</code></li>
				</ul>
				<p><strong>Các giá trị được phép (Allowed Values)</strong></p>
				<ul>
					<li>Không định nghĩa/""</li>
				</ul>
			</td>
		</tr>
		<tr>
			<td style="white-space: nowrap">Kiểu mount <code>/proc</code></td>
			<td>
				<p>Các mask <code>/proc</code> mặc định được thiết lập nhằm giảm bề mặt tấn công, và nên là bắt buộc.</p>
				<p><strong>Các trường bị hạn chế (Restricted Fields)</strong></p>
				<ul>
					<li><code>spec.containers[*].securityContext.procMount</code></li>
					<li><code>spec.initContainers[*].securityContext.procMount</code></li>
					<li><code>spec.ephemeralContainers[*].securityContext.procMount</code></li>
				</ul>
				<p><strong>Các giá trị được phép (Allowed Values)</strong></p>
				<ul>
					<li>Không định nghĩa/nil</li>
					<li><code>Default</code></li>
				</ul>
			</td>
		</tr>
		<tr>
  			<td>Seccomp</td>
  			<td>
  				<p>Profile seccomp không được đặt một cách tường minh thành <code>Unconfined</code>.</p>
  				<p><strong>Các trường bị hạn chế (Restricted Fields)</strong></p>
				<ul>
					<li><code>spec.securityContext.seccompProfile.type</code></li>
					<li><code>spec.containers[*].securityContext.seccompProfile.type</code></li>
					<li><code>spec.initContainers[*].securityContext.seccompProfile.type</code></li>
					<li><code>spec.ephemeralContainers[*].securityContext.seccompProfile.type</code></li>
				</ul>
				<p><strong>Các giá trị được phép (Allowed Values)</strong></p>
				<ul>
					<li>Không định nghĩa/nil</li>
					<li><code>RuntimeDefault</code></li>
					<li><code>Localhost</code></li>
				</ul>
  			</td>
  		</tr>
		<tr>
			<td style="white-space: nowrap">Sysctls</td>
			<td>
				<p>Sysctl có thể vô hiệu hóa các cơ chế bảo mật hoặc ảnh hưởng đến tất cả container trên một host, vì vậy nên bị cấm, ngoại trừ một tập con "an toàn" được cho phép. Một sysctl được coi là an toàn nếu nó nằm trong namespace của container hoặc Pod, và được cô lập khỏi các Pod hay tiến trình khác trên cùng Node.</p>
				<p><strong>Các trường bị hạn chế (Restricted Fields)</strong></p>
				<ul>
					<li><code>spec.securityContext.sysctls[*].name</code></li>
				</ul>
				<p><strong>Các giá trị được phép (Allowed Values)</strong></p>
				<ul>
					<li>Không định nghĩa/nil</li>
					<li><code>kernel.shm_rmid_forced</code></li>
					<li><code>net.ipv4.ip_local_port_range</code></li>
					<li><code>net.ipv4.ip_unprivileged_port_start</code></li>
					<li><code>net.ipv4.tcp_syncookies</code></li>
					<li><code>net.ipv4.ping_group_range</code></li>
					<li><code>net.ipv4.ip_local_reserved_ports</code> (từ Kubernetes 1.27)</li>
					<li><code>net.ipv4.tcp_keepalive_time</code> (từ Kubernetes 1.29)</li>
					<li><code>net.ipv4.tcp_fin_timeout</code> (từ Kubernetes 1.29)</li>
					<li><code>net.ipv4.tcp_keepalive_intvl</code> (từ Kubernetes 1.29)</li>
					<li><code>net.ipv4.tcp_keepalive_probes</code> (từ Kubernetes 1.29)</li>
				</ul>
			</td>
		</tr>
	</tbody>
</table>

### Restricted

**Chính sách _Restricted_ hướng đến việc thực thi các thực hành tốt nhất hiện tại về gia
cố (hardening) Pod, đánh đổi bằng một phần tính tương thích.** Chính sách này nhắm đến
những người vận hành và các nhà phát triển của các ứng dụng trọng yếu về bảo mật, cũng
như những người dùng có mức tin cậy thấp hơn. Các biện pháp kiểm soát liệt kê dưới đây
cần được thực thi/cấm:

> **Ghi chú:** Trong bảng này, ký tự đại diện (wildcard, `*`) biểu thị tất cả các phần tử
> trong một danh sách. Ví dụ, `spec.containers[*].securityContext` chỉ đối tượng Security
> Context của _tất cả các container đã được định nghĩa_. Nếu bất kỳ container nào trong
> danh sách không đáp ứng các yêu cầu, toàn bộ Pod sẽ không vượt qua được bước kiểm tra
> hợp lệ (validation).

<table>
	<caption style="display:none">Đặc tả chính sách Restricted</caption>
	<tbody>
		<tr>
			<td><strong>Kiểm soát (Control)</strong></td>
			<td><strong>Chính sách (Policy)</strong></td>
		</tr>
		<tr>
			<td colspan="2"><em>Tất cả mọi thứ từ chính sách Baseline</em></td>
		</tr>
		<tr>
			<td style="white-space: nowrap">Volume Types</td>
			<td>
				<p>Chính sách Restricted chỉ cho phép các loại volume sau.</p>
				<p><strong>Các trường bị hạn chế (Restricted Fields)</strong></p>
				<ul>
					<li><code>spec.volumes[*]</code></li>
				</ul>
				<p><strong>Các giá trị được phép (Allowed Values)</strong></p>
				Mỗi phần tử trong danh sách <code>spec.volumes[*]</code> phải đặt một trong các trường sau thành giá trị khác null:
				<ul>
					<li><code>spec.volumes[*].configMap</code></li>
					<li><code>spec.volumes[*].csi</code></li>
					<li><code>spec.volumes[*].downwardAPI</code></li>
					<li><code>spec.volumes[*].emptyDir</code></li>
					<li><code>spec.volumes[*].ephemeral</code></li>
					<li><code>spec.volumes[*].persistentVolumeClaim</code></li>
					<li><code>spec.volumes[*].projected</code></li>
					<li><code>spec.volumes[*].secret</code></li>
				</ul>
			</td>
		</tr>
		<tr>
			<td style="white-space: nowrap">Privilege Escalation (v1.8+)</td>
			<td>
				<p>Không được cho phép leo thang đặc quyền (chẳng hạn thông qua chế độ file set-user-ID hoặc set-group-ID). <em><a href="#os-specific-policy-controls">Đây là chính sách chỉ dành cho Linux</a> trong v1.25+ <code>(spec.os.name != windows)</code></em></p>
				<p><strong>Các trường bị hạn chế (Restricted Fields)</strong></p>
				<ul>
					<li><code>spec.containers[*].securityContext.allowPrivilegeEscalation</code></li>
					<li><code>spec.initContainers[*].securityContext.allowPrivilegeEscalation</code></li>
					<li><code>spec.ephemeralContainers[*].securityContext.allowPrivilegeEscalation</code></li>
				</ul>
				<p><strong>Các giá trị được phép (Allowed Values)</strong></p>
				<ul>
					<li><code>false</code></li>
				</ul>
			</td>
		</tr>
		<tr>
			<td style="white-space: nowrap">Running as Non-root</td>
			<td>
				<p>Container phải bị bắt buộc chạy dưới người dùng không phải root (non-root).</p>
				<p><strong>Các trường bị hạn chế (Restricted Fields)</strong></p>
				<ul>
					<li><code>spec.securityContext.runAsNonRoot</code></li>
					<li><code>spec.containers[*].securityContext.runAsNonRoot</code></li>
					<li><code>spec.initContainers[*].securityContext.runAsNonRoot</code></li>
					<li><code>spec.ephemeralContainers[*].securityContext.runAsNonRoot</code></li>
				</ul>
				<p><strong>Các giá trị được phép (Allowed Values)</strong></p>
				<ul>
					<li><code>true</code></li>
				</ul>
				<small>
					Các trường ở cấp container có thể là không định nghĩa/<code>nil</code> nếu trường
					<code>spec.securityContext.runAsNonRoot</code> ở cấp Pod được đặt thành <code>true</code>.
				</small>
			</td>
		</tr>
		<tr>
			<td style="white-space: nowrap">Running as Non-root user (v1.23+)</td>
			<td>
				<p>Container không được đặt <tt>runAsUser</tt> thành 0</p>
				<p><strong>Các trường bị hạn chế (Restricted Fields)</strong></p>
				<ul>
					<li><code>spec.securityContext.runAsUser</code></li>
				    <li><code>spec.containers[*].securityContext.runAsUser</code></li>
					<li><code>spec.initContainers[*].securityContext.runAsUser</code></li>
					<li><code>spec.ephemeralContainers[*].securityContext.runAsUser</code></li>
				</ul>
				<p><strong>Các giá trị được phép (Allowed Values)</strong></p>
				<ul>
					<li>bất kỳ giá trị nào khác 0</li>
					<li><code>undefined/null</code></li>
				</ul>
			</td>
		</tr>
		<tr>
  			<td style="white-space: nowrap">Seccomp (v1.19+)</td>
  			<td>
  				<p>Profile seccomp phải được đặt một cách tường minh thành một trong các giá trị được phép. Cả profile <code>Unconfined</code> lẫn việc <em>không đặt</em> profile đều bị cấm. <em><a href="#os-specific-policy-controls">Đây là chính sách chỉ dành cho Linux</a> trong v1.25+ <code>(spec.os.name != windows)</code></em></p>
  				<p><strong>Các trường bị hạn chế (Restricted Fields)</strong></p>
				<ul>
					<li><code>spec.securityContext.seccompProfile.type</code></li>
					<li><code>spec.containers[*].securityContext.seccompProfile.type</code></li>
					<li><code>spec.initContainers[*].securityContext.seccompProfile.type</code></li>
					<li><code>spec.ephemeralContainers[*].securityContext.seccompProfile.type</code></li>
				</ul>
				<p><strong>Các giá trị được phép (Allowed Values)</strong></p>
				<ul>
					<li><code>RuntimeDefault</code></li>
					<li><code>Localhost</code></li>
				</ul>
				<small>
					Các trường ở cấp container có thể là không định nghĩa/<code>nil</code> nếu trường
					<code>spec.securityContext.seccompProfile.type</code> ở cấp Pod được đặt một cách
					phù hợp. Ngược lại, trường ở cấp Pod có thể là không định nghĩa/<code>nil</code>
					nếu <em>tất cả</em> các trường ở cấp container đều được đặt.
				</small>
  			</td>
  		</tr>
		  <tr>
			<td style="white-space: nowrap">Capabilities (v1.22+)</td>
			<td>
				<p>
					Container phải drop (loại bỏ) <code>ALL</code> capability, và chỉ được phép thêm lại
 					capability <code>NET_BIND_SERVICE</code>. <em><a href="#os-specific-policy-controls">Đây là chính sách chỉ dành cho Linux</a> trong v1.25+ <code>(.spec.os.name != "windows")</code></em>
				</p>
				<p><strong>Các trường bị hạn chế (Restricted Fields)</strong></p>
				<ul>
					<li><code>spec.containers[*].securityContext.capabilities.drop</code></li>
					<li><code>spec.initContainers[*].securityContext.capabilities.drop</code></li>
					<li><code>spec.ephemeralContainers[*].securityContext.capabilities.drop</code></li>
				</ul>
				<p><strong>Các giá trị được phép (Allowed Values)</strong></p>
				<ul>
					<li>Bất kỳ danh sách capability nào có chứa <code>ALL</code></li>
				</ul>
				<hr />
				<p><strong>Các trường bị hạn chế (Restricted Fields)</strong></p>
				<ul>
					<li><code>spec.containers[*].securityContext.capabilities.add</code></li>
					<li><code>spec.initContainers[*].securityContext.capabilities.add</code></li>
					<li><code>spec.ephemeralContainers[*].securityContext.capabilities.add</code></li>
				</ul>
				<p><strong>Các giá trị được phép (Allowed Values)</strong></p>
				<ul>
					<li>Không định nghĩa/nil</li>
					<li><code>NET_BIND_SERVICE</code></li>
				</ul>
			</td>
		</tr>
	</tbody>
</table>

## Khởi tạo chính sách (Policy Instantiation)

Việc tách rời phần định nghĩa chính sách khỏi phần khởi tạo (instantiation) chính sách
cho phép có một cách hiểu chung và một ngôn ngữ nhất quán về các chính sách giữa các
cluster, độc lập với cơ chế thực thi (enforcement) bên dưới.

Khi các cơ chế trưởng thành hơn, chúng sẽ được định nghĩa bên dưới theo từng chính sách.
Phương thức thực thi của từng chính sách riêng lẻ không được định nghĩa ở đây.

[**Bộ điều khiển Pod Security Admission (Pod Security Admission Controller)**](116-pod-security-admission-vi.md)

- Namespace mức Privileged ([`security/podsecurity-privileged.yaml`](https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/security/podsecurity-privileged.yaml)):

  ```yaml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: my-privileged-namespace
    labels:
      pod-security.kubernetes.io/enforce: privileged
      pod-security.kubernetes.io/enforce-version: latest
  ```

- Namespace mức Baseline ([`security/podsecurity-baseline.yaml`](https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/security/podsecurity-baseline.yaml)):

  ```yaml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: my-baseline-namespace
    labels:
      pod-security.kubernetes.io/enforce: baseline
      pod-security.kubernetes.io/enforce-version: latest
      pod-security.kubernetes.io/warn: baseline
      pod-security.kubernetes.io/warn-version: latest
  ```

- Namespace mức Restricted ([`security/podsecurity-restricted.yaml`](https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/security/podsecurity-restricted.yaml)):

  ```yaml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: my-restricted-namespace
    labels:
      pod-security.kubernetes.io/enforce: restricted
      pod-security.kubernetes.io/enforce-version: latest
      pod-security.kubernetes.io/warn: restricted
      pod-security.kubernetes.io/warn-version: latest
  ```

### Các lựa chọn thay thế (Alternatives)

> **Ghi chú:** Mục này liên kết đến các dự án bên thứ ba cung cấp chức năng mà Kubernetes
> cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án này, vốn
> được liệt kê theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc
> [hướng dẫn về nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content)
> trước khi gửi thay đổi.

Các lựa chọn thay thế khác để thực thi chính sách đang được phát triển trong hệ sinh thái
Kubernetes, chẳng hạn:

- [Kubewarden](https://github.com/kubewarden)
- [Kyverno](https://kyverno.io/policies/pod-security/)
- [OPA Gatekeeper](https://github.com/open-policy-agent/gatekeeper)

## Trường OS của Pod (Pod OS field)

Kubernetes cho phép bạn dùng các node chạy Linux hoặc Windows. Bạn có thể trộn lẫn cả hai
loại node trong cùng một cluster.
Windows trong Kubernetes có một số hạn chế và điểm khác biệt so với các workload dựa trên
Linux. Cụ thể, nhiều trường `securityContext` của Pod
[không có tác dụng trên Windows](175-windows-intro-vi.md#compatibility-v1-pod-spec-containers-securitycontext).

> **Ghi chú:** Các kubelet trước v1.24 không thực thi trường OS của Pod, và nếu cluster có
> các node chạy phiên bản cũ hơn v1.24 thì các chính sách Restricted nên được ghim (pin)
> vào một phiên bản trước v1.25.

### Các thay đổi của chuẩn bảo mật Pod Restricted (Restricted Pod Security Standard changes)

Một thay đổi quan trọng khác, được thực hiện trong Kubernetes v1.25, là chính sách
_Restricted_ đã được cập nhật để sử dụng trường `pod.spec.os.name`. Dựa trên tên hệ điều
hành, một số chính sách đặc thù cho một hệ điều hành cụ thể có thể được nới lỏng đối với
hệ điều hành còn lại.

#### Các kiểm soát chính sách đặc thù theo hệ điều hành (OS-specific policy controls) {#os-specific-policy-controls}

Các hạn chế đối với những kiểm soát sau chỉ bắt buộc nếu `.spec.os.name` không phải là
`windows`:

- Leo thang đặc quyền (Privilege Escalation)
- Seccomp
- Linux Capabilities

## Không gian tên người dùng (User namespaces)

User namespace là một tính năng chỉ có trên Linux, dùng để chạy các workload với mức cô
lập cao hơn. Cách chúng hoạt động cùng với Chuẩn bảo mật Pod được mô tả trong
[tài liệu](./55-user-namespaces-vi.md) về các Pod sử dụng user namespace.

## Câu hỏi thường gặp (FAQ)

### Tại sao không có profile nằm giữa Privileged và Baseline? (Why isn't there a profile between Privileged and Baseline?)

Ba profile được định nghĩa ở đây có một tiến trình tuyến tính rõ ràng, từ an toàn nhất
(Restricted) đến kém an toàn nhất (Privileged), và bao quát một tập hợp rộng các workload.
Những đặc quyền vượt trên chính sách Baseline thường mang tính rất đặc thù theo từng ứng
dụng, vì vậy chúng tôi không đưa ra một profile chuẩn cho khoảng trống này. Điều đó không
có nghĩa là trong trường hợp này lúc nào cũng nên dùng profile Privileged, mà là các chính
sách trong vùng này cần được định nghĩa theo từng trường hợp cụ thể.

SIG Auth có thể xem xét lại quan điểm này trong tương lai, nếu xuất hiện nhu cầu rõ ràng
cho các profile khác.

### Sự khác biệt giữa security profile và security context là gì? (What's the difference between a security profile and a security context?)

[Security Context](291-security-context-vi.md)
cấu hình các Pod và Container tại thời điểm chạy (runtime). Security context được định
nghĩa như một phần của đặc tả Pod và container trong manifest của Pod, và đại diện cho
các tham số truyền cho container runtime.

Security profile là các cơ chế của control plane để thực thi những thiết lập cụ thể trong
Security Context, cũng như các tham số liên quan khác bên ngoài Security Context. Kể từ
tháng 7 năm 2021, [Pod Security Policy](117-pod-security-policy-vi.md)
đã bị loại bỏ dần (deprecated) để nhường chỗ cho
[Bộ điều khiển Pod Security Admission](116-pod-security-admission-vi.md)
tích hợp sẵn.

### Còn các Pod sandbox thì sao? (What about sandboxed Pods?)

Hiện tại chưa có chuẩn API nào kiểm soát việc một Pod có được coi là sandbox hay không.
Các Pod sandbox có thể được nhận biết thông qua việc sử dụng một runtime sandbox (chẳng
hạn gVisor hoặc Kata Containers), nhưng không có định nghĩa chuẩn nào về thế nào là một
runtime sandbox.

Các cơ chế bảo vệ cần thiết cho workload sandbox có thể khác với các workload còn lại. Ví
dụ, nhu cầu hạn chế các quyền đặc quyền sẽ giảm đi khi workload được cô lập khỏi kernel
bên dưới. Điều này cho phép những workload đòi hỏi quyền cao hơn vẫn có thể được cô lập.

Ngoài ra, việc bảo vệ các workload sandbox phụ thuộc rất nhiều vào phương thức sandbox.
Do đó, không có một profile khuyến nghị duy nhất nào cho tất cả các workload sandbox.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 9:

1. Kể ba profile theo thứ tự từ dễ dãi tới hạn chế, và nói mỗi profile nhắm tới loại workload
   hoặc loại người dùng nào.
2. Một Pod **không đặt** `seccompProfile` gì cả. Nó qua được Baseline không? Qua được Restricted
   không? Vì sao hai câu trả lời khác nhau?
3. Trên cluster lab, bạn muốn chạy một DaemonSet giám sát cần `hostNetwork: true` và mount
   `hostPath` `/var/log` trên `lab-k8s-worker1` và `lab-k8s-worker2`. Profile thấp nhất cho phép Pod
   đó chạy là gì, và vì sao Baseline không đủ?
4. Restricted thêm gì so với Baseline về **capability** và về **quyền chạy dưới root**?
5. "Security profile" và "security context" khác nhau ở đâu — cái nào nằm trong manifest của
   bạn, cái nào là cơ chế của control plane?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Privileged → Baseline → Restricted.**
   - **Privileged**: chính sách không hạn chế, chủ đích để mở; nhắm tới **workload cấp hệ thống
     và hạ tầng**, do **người dùng đặc quyền, đáng tin cậy** quản lý.
   - **Baseline**: hạn chế ở mức tối thiểu, ngăn các cách leo thang đặc quyền đã biết, vẫn chạy
     được cấu hình Pod mặc định; nhắm tới **người vận hành ứng dụng và nhà phát triển của các
     ứng dụng không trọng yếu**.
   - **Restricted**: hardening theo thực hành tốt nhất hiện tại, **đánh đổi một phần tính tương
     thích**; nhắm tới **ứng dụng trọng yếu về bảo mật và những người dùng có mức tin cậy thấp**.
2. **Qua được Baseline, trượt Restricted.** Trực giác "không khai báo thì không vi phạm" đúng ở
   Baseline nhưng sai ở Restricted. Baseline chỉ yêu cầu profile seccomp **không được đặt tường
   minh thành `Unconfined`**, nên "không định nghĩa/nil" nằm trong danh sách giá trị được phép.
   Restricted thì yêu cầu profile seccomp **phải được đặt tường minh** thành `RuntimeDefault`
   hoặc `Localhost`; bài ghi rõ **cả `Unconfined` lẫn việc *không đặt* profile đều bị cấm**.
3. Chỉ **Privileged**. Baseline không đủ vì nó cấm **cả hai** thứ Pod này cần: mục *Host
   Namespaces* cấm `spec.hostNetwork` khác `false`, và mục *HostPath Volumes* cấm hoàn toàn
   `spec.volumes[*].hostPath` — giá trị được phép duy nhất là không định nghĩa. Restricted càng
   không, vì nó chỉ cho một danh sách loại volume không có `hostPath`.
4. **Capability**: Baseline chỉ **giới hạn danh sách capability được thêm**; Restricted bắt
   container **drop `ALL`** và chỉ cho phép thêm lại duy nhất **`NET_BIND_SERVICE`**.
   **Quyền root**: Baseline không nói gì về việc chạy dưới root; Restricted bắt
   **`runAsNonRoot: true`** và **`runAsUser` khác 0**, đồng thời cấm leo thang đặc quyền bằng
   `allowPrivilegeEscalation: false`.
5. **Security context nằm trong manifest của bạn** — nó được định nghĩa như một phần của đặc tả
   Pod và container, cấu hình Pod/Container tại thời điểm chạy, và đại diện cho **các tham số
   truyền cho container runtime**. **Security profile là cơ chế của control plane** để thực thi
   những thiết lập cụ thể trong security context, cùng các tham số liên quan **nằm ngoài**
   security context.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
