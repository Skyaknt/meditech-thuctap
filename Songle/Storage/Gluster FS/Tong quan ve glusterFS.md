# GlusteFS

## GlusteFS là gì?

GlusterFS là một open source. GlusterFS là tập hợp file hệ thống có thể được nhân rộng tới vài peta-byte và có thể xử lý hàng ngàn Client.

GlusterFS có thể linh hoạt kết hợp với các thiết bị lưu trữ vật lý, ảo, và tài nguyên điện toán đám mây để cung cấp 1 hệ thống lưu trữ có tính sẵn sàng cao với hiệu suất cao .

Chương trình có thể lưu trữ dữ liệu trên các mô hình, thiết bị khác nhau , nó kết nối với tất cả các nút cài đặt GlusterFS qua giao thức TCP hoặc RDMA tạo ra một nguồn tài nguyên lưu trữ duy nhất kết hợp tất cả các không gian lưu trữ có sẵn thành một khối lượng lưu trữ duy nhất (distributed mode) hoặc sử dụng tối đa không gian ổ cứng có sẵn trên tất cả các ghi chú để nhân bản dữ liệu của bạn (replicated mode).

### Các thành phần trong hệ thống lưu trữ GlusteFS

- **Node** : các máy chủ vật lý hoặc máy chủ ảo ( Virtual Private Server) được cài đặt GlusterFS.
- **Brick** : Trong tiếng Việt Brick có nghĩa là viên gạch, bác bạn có thể hiểu để xây nên một cái nhà, 
cần những viên gạch để ghép lại gắn kết bằng xi măng. Thì Brick trong GlusterFS cũng vậy, 
là một đơn vị lưu trữ cơ bản, bản chất nó là một thư mục được Export ra từ một máy chủ nào đó.

	- Ví dụ đây là một **Brick**: server1.vngeek.com:/folder/brick/
	Các bạn có thể hiểu server1.songle.com chính là máy chủ được cài đặt GlusterFS, 
	và thư mục lưu trữ là /folder/brick/. Sau này hệ thống GlusterFS sẽ 
	lưu trữ những tập tin tại thư mục này trên server này khi nó tham gia vào hệ thống

- **Volume** : Gộp các Brick lại, bằng một cách nào đó theo bạn muốn (Giống như Raid trên các ổ cứng). 
Volume đóng vai trò quản lý các Brick được thêm vào nó. Và tất nhiên sau này bạn sẽ làm việc với Volume, 
bạn có thể coi nó là thư mục đích để lưu trữ dữ liệu vào đó.

- **Client** : Client là máy khách, các máy tính kết nối với hệ thống GlusterFS và thực hiện lưu trữ hoặc truy cập đến dữ liệu của GlusterFS. 
Khi bạn mount một Volume nào đó và sử dụng nó để lưu trữ, thì bạn là một Client rồi đó.

### Ưu điểm của GlusteFS

- **Khả năng mở rộng** : GluserFS có khả năng tương thích linh hoạt với mức độ tăng trưởng của dữ liệu, 
có khả năng lưu trữ đến hàng nghìn Petabyte và lớn hơn.

- **Tính linh hoạt** :  GlusterFS có thể sử dụng nhiều định dạng lưu trữ hệ thống như ext4, xfs…

- **Đơn giản** :  Việc quản lý GlusterFS rất dễ dàng, được tách biệt khỏi kernel space và thực thi trên user space.

- **Chi phí** : GlusterFS có thể triển khai trên các hệ thống phần cứng thông thường mà không yêu cầu bất kỳ một thiết bị chuyên biệt nào.

- **Mã nguồn mở** : Hiện nay mã nguồn của GlusterFS vẫn được công khai và điều hành bởi Red Hat Inc.

## Các dạng Volume của GlusteFS

- Bạn có thể tạo 9 loại Volume khác nhau trông GlusteFS.

### 1. Distributed GluserFS Volume ( Dạng phân tán )

Với concept này, các files (data) sẽ được phân tán, lưu trữ rời rạc (distributed) trong các bricks khác nhau . 
Ví dụ, bạn có 100 files: file1, file2, file3…, file100. Thì file1, file2 lưu ở brick1, file3,4lưu ở brick2, etc. 
Việc phân bố các files trên brick dựa vào thuật toán hash, kiểu concept này tương tự JBOD

![Imgur](https://i.imgur.com/qo1J4UG.png)

**Ưu điểm:** Mở rộng được dung lượng lưu trữ nhanh chóng và dễ dàng, tổng dung lượng lưu trữ của volume bằng tổng dung lượng của các brick.

**Nhược điểm:** Khi một trong các brick bị ngắt kết nối, hoặc bị lỗi thì dữ liệu sẽ bị mất hoặc không truy vấn được.

**Câu lệnh để tạo**: 

`gluster volume create NEW-VOLNAME [transport [tcp | rdma | tcp,rdma]] NEW-BRICK...`

Ví dụ, tạo một distributed volume với 4 server lưu trữ sử dụng TCP:

```
gluster volume create test-volume server1:/exp1 server2:/exp2 server3:/exp3 server4:/exp4
Creation of test-volume has been successful
Please start the volume to access data
```

Hiển thị thông tin volume:

```
#gluster volume info
Volume Name: test-volume
Type: Distribute
Status: Created
Number of Bricks: 4
Transport-type: tcp
Bricks:
Brick1: server1:/exp1
Brick2: server2:/exp2
Brick3: server3:/exp3
Brick4: server4:/exp4
```

### 2. Replicated GluserFS Volume

Với concept này, dữ liệu sẽ được copy sang các bricks trong cùng một volume. 
Nhìn vào concept này chúng ta thấy rõ ưu điểm đó là dữ liệu sẽ có tính sẵn sàng 
cao và luôn trong trạng thái dự phòng tương tự Raid 1.

![Imgur](https://i.imgur.com/0YiZAgK.png)

Tổng dung lượng của volume sẽ chỉ bằng một dung lượng của một brick trong volume.

**Tạo Replicated Volume**

`gluster volume create NEW-VOLNAME [replica COUNT] [transport [tcp | rdma | tcp,rdma]] NEW-BRICK...`

Ví dụ, tạo replicated volume với 2 servers lưu trữ:

```
# gluster volume create test-volume replica 2 transport tcp server1:/exp1 server2:/exp2
Creation of test-volume has been successful
Please start the volume to access data
```

### 3. Striped Glusterfs Volume

Nhìn vào hình ảnh trên bạn có thể thấy, Tập tin cần lưu trữ được chia nhỏ thành nhiều phần, và mỗi phần nhỏ được lưu lần lượt trên các server tham gia vào hệ thống. 
Không có sự trùng lặp nào. Hầu hết được sử dụng trong trường hợp lưu trữ một tập tin lớn và cần sự truy xuất dữ liệu nhanh.

![Imgur](https://i.imgur.com/ZVJbtUP.png)

**Ưu điểm:** Phù hợp với việc lưu trữ mà dữ liệu cần truy xuất với hiệu năng cao, 
đặc biệt là truy cập vào những tệp tin lớn.

**Nhược điểm:** Khi một trong những brick trong volume bị lỗi, thì volume đó không thể hoạt động được

**Tạo Striped Volume**:

`gluster volume create NEW-VOLNAME [stripe COUNT] [transport [tcp | dma | tcp,rdma]] NEW-BRICK...`

Ví dụ, tạo striped volume với 2 servers lưu trữ:

```
# gluster volume create test-volume stripe 2 transport tcp server1:/exp1 server2:/exp2
Creation of test-volume has been successful
Please start the volume to access data
```

### 4. Distributed Striped Volume

![Imgur](https://i.imgur.com/91v9DZO.png)


Nhìn vào hình trên bạn có thể thấy File 1 được chia nhỏ (Strip) và lưu trữ lần lượt trên mỗi Brick thuộc Server 1. Tương tự File 2 được chia nhỏ và lưu trữ lần lượt trên mỗi Brick thuộc server 2. 
File 1 và File 2 được lưu trữ phân tán trên 2 server. Phương pháp sử dụng cho các tập tin lớn được lưu trữ đồng thời trong hệ thống. Tận dụng được tiềm lực lưu trữ tối đa của mỗi máy chủ tham gia. 
Nhưng với mỗi Brick hỏng nó sẽ ảnh hưởng đến chính tập tin đã được Striped lưu trữ trên chính server đó.

- Lệnh cấu hình: 
**# gluster volume create NEW-VOLNAME [stripe COUNT] [transport tcp | rdma | tcp,rdma] NEW-BRICK...**

- Ví dụ với 4 servers:

```
# gluster volume create test-volume stripe 2 transport tcp server1:/exp1 server1:/exp2 server2:/exp3 server2:/exp4
```
### 5. Distributed Replicated Volumes

![Imgur](https://i.imgur.com/oJCasHz.png)

Các tập tin được lưu trữ phân tán, trên mỗi Volume Replicated chính bản thân tập tin lại được sao lưu trên các Brick 
của Volume Replicated. Mô hình trên sử dụng cho các tập tin quan trọng và cần hệ thống để có thể mở rộng lưu trữ dễ dàng, 
giúp truy xuất nhanh chóng đến các tập tin.

**Tạo distributed replicated volume**:

`# gluster volume create NEW-VOLNAME [replica COUNT] [transport [tcp | rdma | tcp,rdma]] NEW-BRICK...`

Ví dụ, 4 node distributed (replicated) volume với 2 giải mirror:

```
# gluster volume create test-volume replica 2 transport tcp server1:/exp1 server2:/exp2 server3:/exp3 server4:/exp4
Creation of test-volume has been successful
Please start the volume to access data
```

### 6. Striped Replicated Volumes

![Imgur](https://i.imgur.com/DNFQ4IW.png)

Nếu không chú ý kỹ, bạn có thể nhầm lẫn giữa các loại Volume trong GlusterFS. Đối với Striped Replicated Volumes. 
Tập tin lưu trữ thường rất lớn,  nó vừa được chia nhỏ rải rác trên các Brick, đồng thời được sao lưu với mỗi phần được chia nhỏ. 
Điều này đảm bảo an toàn để lưu trữ tập tin có khối lượng lớn cũng như tốc độ truy xuất đến tập tin.

**# gluster volume create NEW-VOLNAME [stripe COUNT] [replica COUNT] [transport tcp | rdma | tcp,rdma] NEW-BRICK...**

ví dụ: 

- Trên 4 servers 
```
gluster volume create test-volume stripe 2 replica 2 transport tcp server1:/exp1 server2:/exp3 server3:/exp2 server4:/exp4
```
- Trên 6 servers 
```
 gluster volume create test-volume stripe 3 replica 2 transport tcp server1:/exp1 server2:/exp2 server3:/exp3 server4:/exp4 
 server5:/exp5 server6:/exp6
 ```
 
### 7. Distributed Replicated Striped Volume 

Hình thức glusterfs này vẫn đang trong quá trình thử nghiệm và chưa hỗ trợ hoàn toàn 
các nền tảng của Red Hat.

![Imgur](https://i.imgur.com/AhacLXE.png)

- Cấu hình với 4 server:

**gluster volume create NEW-VOLNAME [stripe COUNT] [replica COUNT] [transport tcp | rdma | tcp,rdma] NEW-BRICK...**

- Ví dụ:

```
gluster volume create test-volume stripe 2 replica 2 transport tcp server1:/exp1 server1:/exp2 server2:/exp3 server2:/exp4 
server3:/exp5 server3:/exp6 server4:/exp7 server4:/exp8
```

### 8. Dispered GlusterFS Volume

![Imgur](https://i.imgur.com/vWmRzZM.png)

Dispersed Volumes dựa trên erasure coding (EC). EC là một phương pháp bảo vệ dữ liệu, trong đó dữ liệu được chia 
thành các mẩu nhỏ, có thể mở rộng và mã hóa với các mẩu *redundant data* và lưu trữ trên nhiều vị trí khác nhau. Việc 
này cho phép phục hồi lại dữ liệu lưu trên một hoặc nhiều brick trong trường hợp chúng bị hỏng. Số lượng brick có thể 
hỏng mà không bị mật dữ liệu được cấu hình bởi *redundancy count*. 

Dispersed Volume yêu cầu ít không gian lưu trữ hơn so với replicated volume. Nó tương đương với một replicated pool với kích 
thước là 2TB nhưng chỉ yêu cầu 1.5TB thay vì 2TB để lưu 1TB dữ liệu khi cấp độ dự phòng được đặt là 2. Trong một dispersed volume, mỗi brick lưu một vài phần của dữ liệu và **parity** (*redundancy data*). Dispersed volume đảm bảo việc *chống thất thoát* dữ liệu dựa trên redundancy level.

Việc bảo vệ dữ liệu bằng erasure coding có thể hiểu đơn giản bằng phương trình sau : `n = k + m` . 
Trong đó, `n` là tổng số brick, `k` số brick lưu dữ liệu , `m` brick dùng để phục hồi dữ liệu.
Như vậy, hệ thống 6 brick với redundancy level 2 thì có 4 brick lưu data, 2 brick lưu parity hoặc redundancy data, có thể phục hồi được 2 brick trong số 4 brick lưu dữ liệu. 

- Hệ thống 11 bricks redundancy level 3 = 8 + 3
- Hệ thống 12 bricks redundancy level 4 = 8 + 4

Cấu hình Dispersed GlusterFS : `The syntax is # gluster volume create NEW-VOLNAME [disperse-data COUNT] [redundancy COUNT] [transport tcp | rdma | tcp,rdma] NEW-BRICK...`

Ví dụ với 6 storage servers:

```
# gluster volume create test-volume disperse-data 4 redundancy 2 transport tcp server1:/exp1/brick server2:/exp2/brick server3:/exp3/brick server4:/exp4/brick server5:/exp5/brick server6:/exp6/brick
Creation of test-volume has been successful
Please start the volume to access data.
```

Tham khảo : https://access.redhat.com/documentation/en-us/red_hat_gluster_storage/3.1/html-single/administration_guide/index#chap-Red_Hat_Storage_Volumes-Creating_Dispersed_Volumes_1

### 9. Distributed dispersed GlusterFS Volume 

![Imgur](https://i.imgur.com/fdocrqK.png)

Distributed dispersed volumes hỗ trợ cùng cấu hình của erasure coding giống dispersed volumes. Số lượng bricks trong distributed 
dispersed volume phải là cấp số nhân của (K+M). Với phiên bản này, các cấu hình sau được hỗ trợ:

- Hệ thống 6 bricks sử dụng redundancy level 2
- Hệ thống 11 bricks sử dụng redundancy level 3
- Hệ thống 12 bricks sử dụng redundancy level 4 

Cấu hình Distributed Dispersed GlusterFS : `# gluster volume create NEW-VOLNAME disperse-data COUNT [redundancy COUNT] [transport tcp | rdma | tcp,rdma] NEW-BRICK...`

Ví dụ với 6 storage servers:

```
# gluster volume create test-volume disperse-data 4 redundancy 2 transport tcp server1:/exp1/brick1 server2:/exp2/brick2 server3:/exp3/brick3 server4:/exp4/brick4 server5:/exp5/brick5 server6:/exp6/brick6 server1:/exp7/brick7 server2:/exp8/brick8 server3:/exp9/brick9 server4:/exp10/brick10 server5:/exp11/brick11 server6:/exp12/brick12
Creation of test-volume has been successful
Please start the volume to access data.
```

## Tham khảo: 
1. https://access.redhat.com/documentation/en-US/Red_Hat_Storage/2.1/html/Administration_Guide/chap-User_Guide-Storage-pool.html
2. http://searchstorage.techtarget.com/definition/erasure-coding

