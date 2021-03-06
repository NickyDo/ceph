Librados là thư viện C cho phép ứng dụng làm việc trực tiếp với RADOS, bypass qua các lớp khác để tương tác với Ceph Cluste.librados là thư viện cho RADOS, cung cấp các hàm API, giúp ứng dụng tương tác trực tiếp và truy xuất song song vào cluster.Ứng dụng có thể mở rộng các giao thức của nó để truy cập vào RADOS bằng cách sử dụng librados. Các thư viện tương tự cũng sẵn sàng cho C++, Java, Python, Ruby, PHP. librados là nền tảng cho các service giao diện khác chạy bên trên, gồm Ceph block device, Ceph filesystem, Ceph RADOS Gateway. librados cung cấp rất nhiều API, các phương thức lưu trữ key/value trong object. API hỗ trợ atomic-single-object bằng cách update dữ liệu, key và các thuộc tính.
##1. Ceph Block Storage Device

![image of ceph bsd](https://camo.githubusercontent.com/f624926bd03312b983437ba119c944d1d52a761a/687474703a2f2f692e696d6775722e636f6d2f4b6b45736775682e706e67)

Ceph Block Device có tên là RADOS block device (RBD); cung cấp block storage cho hypervisor và máy ảo. Ceph RBD driver được tích hợp với Linux kernel (từ bản 2.6.39) và hỗ trợ QEMU/KVM.

Khi Ceph block device được map vào máy chủ Linux, nó có thể được sử dụng như một phân vùng RAW hoặc cố thể định dạng theo các loại filesystem phổ biến.

Ceph đã được tích hợp chặt chẽ với các nền tảng Cloud như OpenStack. Ceph cung cấp backend là block device để lưu trữ volume máy ảo và OS image choCinder và Glance. Các volume và image này là thin provisioned, có nghĩa chỉ lưu trữ các dữ liệu object bị thay đổi, giúp tiết kiệm tài nguyên lưu trữ.

Tính năng copy-on-write và cloning của Ceph giúp OpenStack tạo hàng trăm máy ảo trong thời gian ngắn. RBD cũng hỗ trợ snapshot, giúp lưu giữ trạng thái máy ảo, dùng để khôi phục máy ảo lại môt thời điểm hoặc để tạo ra một máy ảo khác tương tự. Ceph cũng là backend cho máy ảo, giúp di chuyển máy ảo giữa các node Compute bởi tất cả dữ liệu máy ảo đều nằm trên Ceph. Các hypervisor như QEMY, KVM, XEN có thể boot máy ảo từ volume nằm trên Ceph.

RBD sử dụng thư viện librbd để tận dụng các tiện ích của RADOS và cung cấp tính tin cậy, phân tán và khả năng lưu trữ dựa trên object.RBD hỗ trợ update vào các object. Client có thể ghi, nối thêm, cắt xén vào các object đang có. Do đó RBD là giải pháp lưu trữ tối ưu cho volume máy ảo.

##1.1. Snapshot

Ceph hỗ trợ snapshot layering, nó cho phép clone image nhanh chóng và dễ dàng. Ceph hỗ trợ block device snapshots bằng rbd command và interfaces.

Để sử dụng RBD snapshots, ta cần chạy Ceph cluster. Khuyến cáo nên dừng I/O trước khi snapshot. We recommend to stop I/O before taking a snapshot of an image. If the image contains a filesystem, the filesystem must be in a consistent state before taking a snapshot. To stop I/O you can use fsfreeze command.

![snapshop](https://camo.githubusercontent.com/753e9b6f4e7197857e48ec79c5b75118d9e3d3a5/687474703a2f2f692e696d6775722e636f6d2f6b684b30324c792e706e67)

##1.2. Layering

Ceph hỗ trợ khả năng copy-on-write(COW) clone of a block device snapshot.

![layering](https://camo.githubusercontent.com/cfb0bcab9374055212bc281215a09ab6e89fe742/687474703a2f2f692e696d6775722e636f6d2f6546446947446c2e706e67)

Mỗi cloned image(child) tham chiếu tới parent image, nó cho phép mở parent snapshot và đọc snapshot.

Note: Ceph chỉ hỗ trợ cloning định dạng images 2 ( rbd create --image-format 2)

Các bản cloned image có tham chiếu tới parent shapshot, bao gồm pool ID, image ID, snapshot ID. Có thể clone snapshot từ pool này sang pool khác.
<ul>
<li>
Image Template: Sử dụng block device layering để tạo master image và snapshot mẫu phục vụ cho việc clone.
</li>
<li>
Extended Template: Cung cấp khả năng mở rộng cho image mẫu để cung cấp nhiều thông tin hơn base image.
</li>
<li>
Tempate Pool: Sử dụng block device layering để tạo pool chứa master images và snapshot mẫu.
</li>
<li>
Image Migration/Recovery: Sử dụng block device layering để migrate or recover data từ pool này sang pool khác.
</li>

</ul>

##1.3. RBD Mirroring

RBD images có thể mirrored trên 2 cụm Ceph cluster thông qua RBD journaling image. Mirroring đc cấu hình trên các pool trong peer cluster và có thể cấu hình tự động mirror tất cả các images trong 1 pool hoặc 1 nhóm images.

##1.4. Cache Settings

Ceph block device ko thể sử dụng page cache của linux vì vậy nó có 1 thành phần là RBD caching hoạt động giống như là ổ đĩa caching.

##2. Ceph Object Gateway

![COG](https://camo.githubusercontent.com/c8a15ce4526bb69c2e7c9dd9080f2feda5caf811/687474703a2f2f692e696d6775722e636f6d2f324d4b383373792e706e67)

Ceph Object Gateway, hay RADOS Gateway, là một proxy chuyển các request HTTP thành các RADOS request và ngược lại, cung cấp RESTful object storage, tương thích với S3 và Swift. Ceph Object Storage sử dụng Ceph Object Gateway Daemon (radosgw) để tương tác với librgw và Ceph cluster, librados. Nó sử dụng một module FastCGI là libfcgi, và có thể sử dụng với bất cứ Server web tương thích với FASTCGI nào. Ceph Object Store hỗ trợ 3 giao diện sau:
<ul>
<li>
S3: Cung cấp Amazon S3 RESTful API.
</li>
<li>
Swift: Cung cấp OpenStack Swift API. Ceph Object Gateway có thể thay thê Swift.
</li>
<li>
Admin: Hỗ trợ quản trị Ceph Cluster thông qua HTTP RESTful API.
</li>
</ul>

##3. CephFS

Ceph Filesystem (CephFS) là một POSIX-compliant filesystem sử dụng Ceph Storage Cluster để lưu data. CephFS kế thừa các tính năng từ RADOS.

Để sử dụng CephFS cần một Ceph MDS hay Metadata Server là daemon cho Ceph filesystem (CephFS). MDS là thành phần duy nhất trong ceph chưa production, hiện chỉ 1 ceph MDS daemon hoạt động tại 1 thời điểm. MDS không lưu dữ liệu local. Nếu 1 MDS daemon lỗi, ta có thể tạo lại trên bất cứ hệ thống nào mà cluster có thể truy cập. Các metadata server daemon được cấu hình là active-passive. Primary MDS là acive, còn các node khác chạy standby.

![CEPHFS](https://camo.githubusercontent.com/cdaec535a1be9259d3b706bde72020dc3248a7a8/687474703a2f2f692e696d6775722e636f6d2f665050663569562e706e67)

Thư viện libcephfs hỗ trợ Linux kernel driver, người dùng có thể dùng phương thức mounting filesystem qua lệnh mount. Nó hỗ trợ CIFS và SMB. CephFS hỗ trợ filesystem in userspace(FUSE) dùng module cephfuse. Nó cũng hỗ trợ ứng dụng tương tác trực tiếpvới RADOS cluster dùng thư viện libcephfs.