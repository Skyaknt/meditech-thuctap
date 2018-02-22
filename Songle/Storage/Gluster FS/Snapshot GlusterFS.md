# GlusterFS Snapshot

GlusterFS volume snapshot dựa trên thinly provisioned LVM snapshot. ( Dùng bao nhiêu nở ra bấy nhiêu )
Để sử dụng chức năng snapshot GlusterFS volume cần đảm bảo được 4 yêu cầu sau:

- Mỗi brick nên được cấu hình independent thinly provisioned LVM.
- Brick LVM không nên chứa dữ liệu khác ngoài dữ liệu mà nó được cấu hình sẽ lưu.
- Không có brick nào là thick LVM.
- Gluster version nên từ 3.6 trở lên.

Cách tạo thin volume:  https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Logical_Volume_Manager_Administration/LV.html#thinly_provisioned_volume_creation

### Một số đặc điểm của snapshot:

#### Crash consistency (sự nhất quán về thời điểm tạo snapshot).

Khi một snapshot được tạo tại một thời điểm và khi nó được nạp lại thì dữ liệu được chụp lại phải đúng 
là dữ liệu ở trạng thái mà máy được tạo snapshot.

#### Online Snapshot

Khi một snapshot được tạo, toàn bộ filesystem và dữ liệu liên quan sẽ vẫn luôn khả dụng cho 
client.

#### Quorum Based 

Đặc điểm quorum đảm bảo rằng volume ở trong trạng thái tốt khi các bricks bị down.
Quorum sẽ không thể nguyên vẹn nếu bất cứ brick nào down trong n replication khi mà n <=2. 
Quorum nguyên vẹn khi m bricks hoạt động bình thường, khi m >= (n/2 + 1) khi n là số lẻ, và m >= n/2 với 
brick đầu tiên hoạt động bình thường khi n là số chẵn. snapshot sẽ không tạo được nếu quorum không khớp. 

#### Barrier 

Trong lúc tạo snapshot, một số fops bị khóa để đảm bảo crash consistency. Sẽ có 1 khoảng
thời gian mặc định time-out là 2 phút, nếu việc tạo snapshot hoàn thành trong khoảng thời gian đó, thì 
fops không bị unbarried ( cản trở). Nếu unbarried xảy ra trước khi snapshot được tạo xong thì việc tạo 
snapshot sẽ lỗi. Việc này đảm bảo rằng snapshot sẽ ở trạng thái đồng nhất.

## Quản lí Snapshot

#### Tạo snapshot

**Câu lệnh :** `gluster snapshot create <snapname> <volname> [no-timestamp] [description <description>] [force]`

- Người dùng có thể cung cấp snapname và mô tả cho snap, mô tả dưới **1024** kí tự.
- Snapshot sẽ được tạo bằng cách thêm timestamp với tên snap do người dùng cung cấp. Người dùng có thể ghi đè 
lên đặc tính này bằng việc cung cấp no-timestamp flag.
	
- Để có thể tạo được snapshot , volume phải đang ở trạng thái started state và đang hoạt động.

#### Snapshot clone

**Câu lệnh:** `gluster snapshot clone <clonename> <snapname>`

- Sau khi lệnh trên thành công, một glusterFS volume mới sẽ được tạo ra từ snapshot. 
Snapshot và clone cùng chia sẻ backend disk.
	
-  Để có thể tạo được snapshot , volume phải đang ở trạng thái started state và đang hoạt động.

#### Khôi phục snaps 

**Câu lệnh:** `gluster snapshot restore <snapname>`

- Snapshot restore là hoạt động offline do vậy nếu volume đang hoạt động thì chức năng này sẽ không thực hiện được.
- Khi một snapshot được restore lại thì nó sẽ không còn tồn tại trong danh sách các snapshots.
	
	
#### Xóa snaps

**Câu lệnh:** ` gluster snapshot delete (all | <snapname> | volume <volname>)`

-  Nếu volume name được chỉ rõ thì snapshots của volume đó sẽ bị xóa hết. Nếu keyword `all` được dùng thì tất 
cả snapshots trong hệ thống sẽ bị xóa.
	
	
#### Liệt kê danh sách các snaps 

**Câu lệnh**: `gluster snapshot list [volname]`

#### Thông tin của các snap trong danh sách 

**Câu lệnh**: ` gluster snapshot info [(snapname | volume <volname>)]`

-  Lệnh này gọi ra tên snapshot, snapshot UUID, thời gian snapshot được tạo, số lượng 
snapshots được tạo và số lượng snapshots khả dụng cho một volume cụ thể và trạng thái của snapshot.
	
	
#### Trạng thái của snapshots 

**Câu lệnh**: `gluster snapshot status [(snapname | volume <volname>)]`

- Thông báo thông tin chi tiết về snapshots như đường dẫn brick, volume group, trạng thái của 
snapshot brick, PID của bricks, trạng thái dữ liệu ...
- Nếu snapname được mô tả thì thông tin của snapshot sẽ được hiển thị, còn volname được mô tả thì 
trạng thái của toàn bộ snapshots của volume đó sẽ được hiển thị. Nếu không có tên của volume hay 
snapshot được mô tả thì thông tin tất cả snapshots sẽ được hiển thị.
	
#### Cấu hình các đặc tính của snapshot

**Câu lệnh**: ` snapshot config [volname] ([snap-max-hard-limit \] \[snap-max-soft-limit ]) | ([auto-delete <enable|disable>]) | ([activate-on-create <enable|disable>])`

- snap-max-soft-limit và auto-delete là những tùy chọn toàn bộ, nó sẽ được đặt trên tất cả các volumes trong 
hệ thống và không thể đặt cho một volume riêng rẽ nào cả.
	
- Khi tính năng auto-delete được kích hoạt, khi đạt đến giới hạn mềm (soft-limit), với mỗi lần tạo snapshot thành công,
snapshots cũ nhất sẽ bị xóa.
	 
- Khi tính năng auto-delete bị tắt, sau khi đạt đến soft-limit, người dùng sẽ nhận được một cảnh báo với mọi snapshot creation thành công.
	
- Khi đạt đến hard-limit, hệ thống không cho phép tạo snapshot nữa. 
	
- Activation-on-create bị vô hiệu hóa theo mặc định, nếu bạn kích hoạt nó, những snapshot sau đó sẽ sẽ được kích hoạt trong 
suốt thời gian tạo snapshot.
	
##### Kích hoạt một snapshot

**Câu lệnh:** `gluster snapshot activate <snapname>`
	
-  Theo mặc định, snapshot sẽ không được kích hoạt trong quá trình tạo snapshot.
	
##### Vô hiệu hóa snapshot 

**Câu lệnh:** ` gluster snapshot deactivate <snapname>`

##### Truy cập snapshot

Snapshots có thể được truy cập bằng 3 cách:

1. Mount snapshot 

Snapshot có thể được truy cập thông qua FUSE mount. Để làm được điều này, nó phải được mount trước tiên :
`mount -t glusterfs :/snaps//`

ví dụ : `mount -t glusterfs host1:/snaps/my-snap/vol /mnt/snapshot`

2. Khả năng sử dụng của người dùng

Ngoài phương pháp trên để mount snapshot, bạn có thể xem danh sách snapshot và nội dung của chúng từ bất 
cứ mount points nào truy cập đến glusterfs volume ( qua FUSE, NFS hay SMB). Để có được snapshot phục vụ 
người dùng, nó phải được kích hoạt cho volume đầu tiên. Bạn có thể kích hoạt tính khả dụng của người dùng cho một 
ổ đĩa bằng lệnh sau:

`gluster volume set features.uss enable`

- Tên của thư mục ẩn (hoặc điểm truy cập tới snapshot) có thể được thay đổi bằng cách sử dụng lệnh dưới đây:

`gluster volume set snapshot-directory`

3. Truy cập từ windows : 

Glusterfs volumes có thể truy cập từ windows qua giao thức samba.


## Tham khảo :

1. http://docs.gluster.org/en/latest/Administrator%20Guide/Managing%20Snapshots/
