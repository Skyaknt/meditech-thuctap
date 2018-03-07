# Tài liệu lab Live Migrate trên KVM

## Tổng quan:

![Imgur](https://i.imgur.com/ytNdqJe.png)

Migration là quá trình di chuyển máy ảo từ host vật lí này sang một host vật lí khác. 
Migration được sinh ra để làm nhiệm vụ bảo trì nâng cấp hệ thống. Ngày nay tính năng này đã được phát triển để thực hiện nhiều tác vụ hơn:

	- Cân bằng tải: Di chuyển VMs tới các host khác kh phát hiện host đang chạy có dấu hiệu quá tải.

	- Bảo trì, nâng cấp hệ thống: Di chuyển các VMs ra khỏi host trước khi tắt nó đi.
	
	- Khôi phục lại máy ảo khi host gặp lỗi: Restart máy ảo trên một host khác.
	
	- Trong OpenStack, việc migrate được thực hiện giữa các node compute với nhau hoặc giữa các project trên cùng 1 node compute.


## 1. Mô hình lab : 2 máy ubuntu server 14.04 với cùng cấu hình, đã cài đặt sẵn kvm và linux bridge để máy ảo 
có thể đi ra ngoài qua card mạng thật tên kvm.

- Hướng dẫn cài đặt kvm + linux bridge trên Ubuntu: https://github.com/vuvandang1995/Meditech_2017/blob/master/Ghi%20chep%20KVM/Thuc%20hanh/Install-KVM.md

- kvm-server1 : máy host cài máy ảo lên và tiến hành migrate từ máy này sang máy khác.
- kvm-server2 : máy target nhận VM migrate từ kvm-server1

**Chú ý: 2 máy server đều phải tắt tường lửa để các kết nối ra ngoài không bị chặn: `sudo ufw disable`**

#### Bước 1: Tạo máy ảo trên kvm server1 :

Ở đây tôi cài máy ảo bằng virt-manager : https://github.com/thaonguyenvan/meditech-thuctap/blob/master/ThaoNV/KVM/install-kvm-CentOS.md

- Kiểm tra máy ảo đang chạy trên kvm-server1: 

```
root@kvm-server1:~# virsh list
 Id    Name                           State
----------------------------------------------------
 1     centos7                        running
```

#### Bước 2: Cấu hình hosts cho 2 server :

**vi /etc/hosts**: 

```
192.168.100.106 kvm-server2
192.168.100.105 kvm-server1
```

#### Bước 3: Cấu hình cho phép giao thức TCP trong dịch vụ libvirt 

- Sửa file `/etc/libvirt/libvirtd.conf`

```
listen_tls = 0  
listen_tcp = 1
tcp_port = "16509"
listen_addr = "0.0.0.0"
auth_tcp = "none"
mdns_adv = 0
```

Giải thích : 

	- Listen_tls : tắt tls, mặc định nó được mở.
	- listen_tcp: bật chức năng kiểm duyệt tcp
	- tcp_port: cấu hình cổng tcp, mặc định là 16509
	- auth_tcp: bật hoặc tắt việc kiểm duyệt bằng mật khẩu
	- mdns_adv: bật/tắt tính năng mdns multicast, mặc định là tắt.
	
- Khởi động lại dịch vụ libvirtd: `service libvirtd restart`

- Cấu hình iptables cho phép giao thông đi qua các cổng 16509 và 49152:

```
#ACCESS kvm
iptables -A INPUT -m state --state NEW -m tcp -p tcp --dport 5900:5909 -j ACCEPT
iptables -A INPUT -p tcp --dport 16509 -j ACCEPT
iptables -A INPUT -p tcp --dport 49152 -j ACCEPT
```

#### Bước 4: Cấu hình cho dịch vụ libvirt với option `-l` ( --listen) lắng nghe các kết nối bằng TCP:

`vi /etc/init/libvirt-bin.conf`

`exec /usr/sbin/libvirtd -d -l` 

- Khởi động lại dịch vụ trên host:

`service libvirt-bin restart`

- Đảm bảo rằng libvirt hoạt động với thiết lập listen :

` ps -ef | grep libvirt`

```
root@kvm-server1:~# ps -ef | grep libvirt
root      1180     1  0 14:55 ?        00:00:00 /usr/sbin/libvirtd -d -l
libvirt+  1355     1  0 14:55 ?        00:00:00 /usr/sbin/dnsmasq --conf-file=/var/lib/libvirt/dnsmasq/default.conf --leasefile-ro --dhcp-script=/usr/lib/libvirt libvirt_leaseshelper
root      1356  1355  0 14:55 ?        00:00:00 /usr/sbin/dnsmasq --conf-file=/var/lib/libvirt/dnsmasq/default.conf --leasefile-ro --dhcp-script=/usr/lib/libvirt libvirt_leaseshelper
root      1597  1468  0 15:38 pts/0    00:00:00 grep --color=auto libvirt
```

- Kiểm tra kết nối từ kvm-server1 đến kvm-server2:

`virsh -c qemu+tcp://root@192.168.100.106/system`

*root: người dùng root đăng nhập vào server2 ; 192.168.100.106 : ip của server2*

#### Bước 5: Tiến hành migrate máy ảo đang chạy từ kvm-server1 sang kvm-server2 :

- Trong bài lab này, tôi tạo 1 máy VM chạy CentOS7 với file cấu hình là `centos7.qcow2`, đang chạy trên kvm-server1.

- Đầu tiên, muốn migrate máy ảo sang kvm-server2 cần tạo một file y hệt với file cấu hình của máy ảo trên kvm-server1.

	- Trước khi tạo, bạn kiểm tra dung lượng file cấu hình máy ảo trên kvm-server1 có dung lượng là bao nhiêu bằng lệnh `ls -l`:
	
	```
	cd /var/lib/libvirt/image
	ls -l	
	```
	
	- Lệnh tạo : `qemu-img create -f qcow2 -o preallocation=metadata centos7.qcow2 9665380352`
	*centos7.qcow2 : tên file cấu hình của máy ảo; 966538052 : dung lượng máy ảo đúng với trên máy kvm-server1*
	
- Tiếp theo, tiến hành live-migrate khi máy ảo vẫn đang chạy:

	`virsh migrate --live --copy-storage-all --persistent centos7 qemu+tcp://root@192.168.100.106:16509/system`

	- Có thông báo: `Migration: [100 %]` là thành công.root@kvm-server1:~# virsh list

	
- Tiến hành kiểm tra xem máy ảo đã được migrate sang kvm-server2 chưa :

Trên kvm-server1 dùng lệnh `virsh list` kiểm tra, như ta thấy máy ảo trên kvm-server1 đã không còn tồn tại:

```
root@kvm-server1:~# virsh list
 Id    Name                           State
----------------------------------------------------
```

Trên kvm-server2 kiểm tra xem có máy ảo đang chạy không :

```
 Id    Name                           State
----------------------------------------------------
 1     centos7                        running

```

Tiến hành sử dụng máy ảo trên kvm-server2 bình thường.

## Tham khảo:

https://www.codeday.top/2017/01/09/8568.html

http://pineapplesoftware.blogspot.com.tr/2012/11/configuring-unsecure-remote-access-to.html
