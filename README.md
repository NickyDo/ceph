#I RADOS
RADOS (Reliable Autonomic Distributed Object Store) là trái tim của hệ thống lưu trữ CEPH. RADOS cung cấp tất cả các tính năng của Ceph, gồm lưu trữ object phân tán, sẵn sàng cao, tin cậy, không có SPOF, tự sửa lỗi, tự quản lý,... lớp RADOS giữ vai trò đặc biệt quan trọng trong kiến trúc Ceph. Các phương thức truy xuất Ceph, như RBD, CephFS, RADOSGW và librados, đều hoạt động trên lớp RADOS. Khi Ceph cluster nhận một yêu cầu ghi từ người dùng, thuật toán CRUSH tính toán vị trí và thiết bị mà dữ liệu sẽ được ghi vào. Các thông tin này được đưa lên lớp RADOS để xử lý. Dựa vào quy tắc của CRUSH, RADOS phân tán dữ liệu lên tất cả các node dưới dạng object, các object này được lưu tại các OSD.

RADOS, khi cấu hình với số nhân bản nhiều hơn hai, sẽ chịu trách nhiệm về độ tin cậy của dữ liệu. Nó sao chép object, tạo các bản sao và lưu trữ tại các zone khác nhau, do đó các bản ghi giống nhau không nằm trên cùng 1 zone. RADOS đảm bảo có nhiều hơn một bản copy của object trong RADOS cluster. RADOS cũng đảm bảo object luôn nhất quán. Trong trường hợp object không nhất quán, tiến trình khôi phục sẽ chạy. Tiến trình này chạy tự động và trong suốt với người dùng, do đó mang lại khả năng tự sửa lỗi và tự quẩn lý cho Ceph. RADOS có 2 phần: phần thấp không tương tác trực tiếp với giao diện người dùng, và phần cao hơn có tất cả giao diện người dùng.

RADOS có 2 thành phần chính là OSD (object storage devices) và Monitor

##1. Ceph OSD

Một Ceph cluster bao gồm nhiều OSD. Ceph lưu trữ dữ liệu dưới dạng object trên các ổ đĩa vật lý.
Object
https://camo.githubusercontent.com/df9a6a0e6dbe4519057fa8f50c1eb9b526219f14/687474703a2f2f692e696d6775722e636f6d2f5a4a795873467a2e706e67

Với các tác vụ đọc hoặc ghi, client gửi yêu cầu tới node monitor để lấy cluster map sau đó tương tác trực tiếp với OSD ko cần sự can thiệp của monitor.

Object được phân tán lưu trên nhiều OSD, mỗi OSD là primary OSD cho một số object và là secondary OSD cho object khác để tăng tính sẵn sàng và khả năng chống chịu lỗi. Khi primary OSD bị lỗi thì secondary OSD được đẩy lên làm primary OSD. Quá trình này trong suốt với ngưới dùng.\

Với các tác vụ đọc hoặc ghi, client gửi yêu cầu tới node monitor để lấy cluster map sau đó tương tác trực tiếp với OSD ko cần sự can thiệp của monitor.

Object được phân tán lưu trên nhiều OSD, mỗi OSD là primary OSD cho một số object và là secondary OSD cho object khác để tăng tính sẵn sàng và khả năng chống chịu lỗi. Khi primary OSD bị lỗi thì secondary OSD được đẩy lên làm primary OSD. Quá trình này trong suốt với ngưới dùng.

###1.1.Ceph OSD File System
https://camo.githubusercontent.com/6a606cbf180b5cee22507ff3f8b57503343576fd/687474703a2f2f692e696d6775722e636f6d2f4a364e49656f6f2e706e67

Ceph OSD gồm ổ cứng vật lý, Linux filesystem trên nó và Ceph OSD Service. Linux filesystem của Ceph cần hỗ trợ extended attribute (XATTRs). Các thuộc tính của filesystem này cung cấp các thông tin về trạng thái object, metadata, snapshot và ACL cho Ceph OSD daemon, hỗ trợ việc quản lý dữ liêu. LinuxFileSystem có thể là Btrfs, XFS hay Ext4. Sự khác nhau giữa các filesystem này như sau:
<ul>
<li>
Btrfs: filesystem này cung cấp hiệu năng tốt nhất khi so với XFS hay ext4. Các ưu thế của Btrfs là hỗ trợ copy-on-write và writable snapshot, rất thuận tiện khi cung cấp VM và clone. Nó cũng hỗ trợ nén và checksum, và khả năng quản lý nhiều thiết bị trên cùng môt filesystem. Btrfs cũng hỗ trợ XATTRs, cung cấp khả năng quản lý volume hợp nhất gồm cả SSD, bổ xung tính năng fsck online. Tuy nhiên, btrfs vẫn chưa sẵn sàng để production.
</li>

<li>
XFS: Là filesystem đã hoàn thiện và rất ổn định, và được khuyến nghị làm filesystem cho Ceph khi production. Tuy nhiên, XFS không thế so sánh về mặt tính năng với Btrfs. XFS có vấn đề về hiệu năng khi mở rộng metadata, và XFS là một journaling filesystem, có nghĩa, mỗi khi client gửi dữ liệu tới Ceph cluster, nó sẽ được ghi vào journal trước rồi sau đó mới tới XFS filesystem. Nó làm tăng khả năng overhead khi dữ liệu được ghi 2 lần, và làm XFS chậm hơn so với Btrfs, filesystem không dùng journal.
</li>

<li>
Ext4: Là một filesystem dạng journaling và cũng có thể sử dụng cho Ceph khi production; tuy nhiên, nó không phôt biến bằng XFS. Ceph OSD sử dụng extended attribute của filesystem cho các thông tin của object và metadata. XATTRs cho phép lưu các thông tin liên quan tới object dưới dạng xattr_name và xattr_value, do vậy cho phép tagging object với nhiều thông tin metadata hơn. ext4 file system không cung cấp đủ dung lượng cho XATTRs do giới hạn về dung lượng bytes cho XATTRs. XFS có kích thước XATTRs lớn hơn.4
</li>
</ul>