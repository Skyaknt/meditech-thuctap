# Tìm hiểu KVM và QEMU

KVM kernel module không thể tự nó tạo nên một VM. Để làm điều này, nó phải sử dụng QEMU - một tiến trình trên 
userspace.

QEMU sở dĩ là một trình gỉa lập phần cứng. Nó cung cấp phương thức giống như một OSS (Open source software) 
để giả lập các chuẩn x86 máy tính cá nhân và các kiến trúc khác. Nó được tạo ra trước cả KVM và có thể
 hoạt động mà không cần KVM.
 
QEMU là một phần mềm giả lập, nó thông dịch và thực hiện các instructions của CPU 
chỉ một lần trên phần mềm tại một thời điểm, có nghĩa là hiệu suất của nó có hạn. Tuy 
nhiên ta vẫn có thể cải thiện hiệu suất QEMU đồng thời cũng thực hiện được chức năng của VM 
với ba điều kiện sau: 

	1. Một instruction cụ thể có thể được thực thi trực tiếp bởi CPU.
	
	2. Instruction đó có thể gửi tới CPU mà không bị sửa đổi để phục vụ cho việc thực thi trực tiếp trong chế độ người dùng thường VMX.
	
	3. Một instruction cụ thể mà không được thực thi một cách trực tiếp có thể được xác định và gửi tới QEMU phục vụ 
	cho tiến trình giả lập.
	
- Sự phát triển của KVM dựa trên ý  tưởng đó. Nó cho phép tạo các VM trong khi vẫn tận 
dụng được hiệu suất tối đa của tài nguyên OSS hiện có với ít sự thay đổi nhất.  

Các bước thực hiện QEMU/KVM được mô tả trong ảnh dưới.

![Imgur](https://i.imgur.com/SvPjDHb.png)

	- Đầu tiên, một file có tên `/dev/kvm` được tạo bởi KVM kernel module (bước 0).
	File đó cho phép QEMU nắm bắt được các yêu cầu đến KVM kernel module để thực hiện 
	các chức năng giám sát.
	
	- Khi QEMU khởi động để thực thi hệ thống máy khách, nó sẽ lặp lại các `ioctl() system call`.
	
	- Khi đến thời điểm để khởi chạy guest system, QEMU lại gọi đến `ioctl()` để hướng 
	dẫn cho KVM kernel module khởi động guest system (bước 1).
	
	- Đến lượt kernel module, nó thực hiện một truy cập đến VM gọi là `VM entry` (bước 2) và 
	bắt đầu thực hiện guest system,
	
	- Sau đó, khi guest system đã được hướng dẫn để thực thi các instruction, một tiến trình 
	`VM exit` được thực hiện (bước 3) và KVM xác định nguyên nhân cho việc thoát.
	Nếu sự tham gia của QEMU là cần thiết để thực hiện một tác vụ I/O hoặc các task khác, tác vụ 
	kiểm soát sẽ được gửi tới QEMU và nó sẽ thực hiện nhiệm vụ đó. Khi việc thực thi hoàn thành, QEMU 
	tiếp tục thực hiện một `ioctl() system call` và yêu cầu KVM tiếp tục tiến trình trên máy khách.
	(các bước thực hiện lại trở về từ bước 1).
	
	
Các bước tiến hành giữa QEMU/KVM về cơ bản luôn được lặp lại trong quá trình giả lập VM.

- QEMU/KVM có một cấu trúc khá tương đồng:

	1. Việc thực thi của KVM kernel module chuyển hóa từ Linux kernel sang hypervisor.
	
	2. Chỉ có một tiến trình QEMU cho mỗi guest system. Khi nhiều guest systems cùng chạy, số lượng QEMU process sẽ tương ứng với số guest.
	
	3. QEMU là một chương trình đa luồng, một CPU ảo (VCPU) của một guest system sẽ 
	tương tác với một luồng QEMU. Bước 1-4 trong ảnh trên được thực hiện với nhiều đơn vị và nhiều luồng.
	
	4. QEMU threads được xử lý như những tiến trình người dùng bình thường từ cái nhìn của 
	Linux kernel. Việc lên lịch cho thread tương tác với VCPU của guest system được kiểm soát 
	bởi Linux system scheduler.
	
	
### Hoạt động của Virtual machine

- Trong hệ thống phần cứng thật, HĐH sẽ dịch các chương trình thành các instructions để thực thi bởi CPU.
Đối với máy ảo cũng như vậy, tuy nhiên điểm khác biệt là VCPU là thành phần được ảo hóa bởi hypervisor. Vì vậy,
 phần mềm hypervisor cần phải dịch các instructions cho VCPU và convert chúng sao cho CPU thật hiểu được để thực thi. 
Tuy nhiên việc dịch đó gặp một vấn đề lớn về hiệu suất.

Để giảm thiếu tối đa việc quá tải hệ thống, các bộ xử lý hiện đại hỗ trợ các chức năng nâng cao mở rộng cho việc ảo hóa.
Intel hỗ trợ công nghệ goi là `VT-x` và AMD là `AMD-V`. Sử dụng các công nghệ trên, một đơn vị CPU vật lý có thể 
được ánh xạ trực tiếp đến VCPU. Do đó các instruction cho VCPU có thể được thực thi một cách trực tiếp bởi 1 đơn vị CPU vật lý đó.


	