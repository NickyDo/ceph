###1. Ceph objects

Một object thường bao gồm dữ liệu và metadata được gói lại với nhau và được cung cấp một định danh duy nhất, đảm bảo ko có object khác có cùng ID.

![CEPH_OBJECT](https://camo.githubusercontent.com/fdfc6d902d91aeeb997b3f1649bf84ecbeb78389/687474703a2f2f692e696d6775722e636f6d2f38636b4c3775302e706e67)

***Locating objects

Mỗi đơn vị data trong Ceph được lưu trữ dưới dạng object trong một pool. Một Ceph pool là một phân vùng lưu trữ.

Khi một cụm Ceph đc triển khai nó tạo ra một số pool mặc định như data, metadata, and RBD pools.

Sau khi MDS triển khai, nó tạo ra các objects trong metadata pool do yêu cầu của CephFS để giúp hoạt động chính xác.

###2. The CRUSH algorithm (Controlled Replication Under Scalable Hashing)

Thuật toán CRUSH là cốt lõi trong cơ chế lưu trữ của Ceph. Ceph sử dụng CRUSH để xác định nơi dữ liệu sẽ ghi hoặc đọc. Thay vì lưu trữ metadata, CRUSH tính toán metadata theo yêu cầu.

***The CRUSH lookup

Công việc tính toán metadata được coi như là CRUSH lookup và được thực hiện khi cần.

Khi đọc và ghi, data được chuyển thành các objects với object và pool names/ID. Object được băm với number of placement groups to generate a final placement group within the required Ceph pool. Các tính toán về placement group được chuyển tới CRUSH lookup để xác định vị trí OSD lưu trữ hoặc lấy dữ liệu. Các tính toán này đc thực hiện bên phía client nên ko ảnh hưởng tới cluster.

Vi du:

![crush](https://camo.githubusercontent.com/c025e52122375ce1f7b830f2b89a2e03c7e12fb5/687474703a2f2f692e696d6775722e636f6d2f545356383558452e706e67)

***The CRUSH hierarchy

CRUSH là một cơ sở hạ tầng đầy đủ, nó duy trì một hệ thống phân cấp lồng nhau cho các thành phần của cơ sở hạ tầng. Danh sách thiết bị bao gồm disk, node, rack, row, switch, power circuit, room, data center. Các thành phần này đc gọi là failure zones or CRUSH buckets. CRUSH map chứa danh sách các buckets có sẵn để tổng hợp vào physical locations. NÓ cũng bao gồm danh sách các rules giúp cho CRUSH biết làm như nào để replicate data giữa các Ceph pool.

###3. Placement groups

Khi Ceph cluster nhận các requests data, nó tách thành các sections placement group (PG). Một PG là một tập hợp logic của các objects được replicate trên các OSDs.

![pg](https://camo.githubusercontent.com/62e38d8db289e14f990e530cbfdbd0d097a751a0/687474703a2f2f692e696d6775722e636f6d2f463548415879762e706e67)

###4. Ceph pools

Ceph pool là một phân vùng lưu trữ objects. Ceph pool giữ một số PG và đc phân phối trên các cluster. Pool đảm bảo dữ liệu bằng cách tạo một số object copies, đó là các bản replicate hoặc erasure codes. Khi tạo pool, ta có thể tùy chọn replica size. Mặc định là 2.

Pool cung cấp cùng với tùy chọn:
<ul>
<li>
    Resilience: Có thể tùy chọn số lượng OSD cho phép lỗi mà ko làm mất data. Với replicated pools ta có tỉ lệ copies/replicas của một object. Với erasure coded pools thì tỉ lệ là khối mã hóa ví dụ m=2 in the erasure code profile.
    </li>
    <li>
Placement Groups: Bạn có thể tùy chọn số PG cho pool.
</li>
<li>
CRUSH Rules: Khi lưu data xuống pool CRUSH Rules ánh xạ tới các pool CRUSH để xác định các luật cho PG của object và bản replicate của nó. Có thể tạo các CRUSH rule cho pool.
 <li>
   Snapshots
  </li>
 <li>
    Set Ownership: Có thể cấu hình quyền sở hữu cho pool. 
</li>

</ul>
###5. Erasure Code

Erasure Code pool là một loại thay cho pool replica nhằm tiết kiệm không gian.
Ví dụ: 
```
ceph osd pool create ecpool 12 12 erasurepool 'ecpool' created
```

Nó giống với RAID5 và cần 3 hosts. Mặc định các erasure code profile duy trì cho 1 OSD và chỉ cần 1.5TB thay vì 2TB để lưu 1TB data. Các erasure code profile ko thể sửa đổi sau khi đc tạo.

![pg](https://camo.githubusercontent.com/4449ab648df6ae2a8b149562c1e100538a862fd7/687474703a2f2f692e696d6775722e636f6d2f68323539344c562e706e67)

Các thông số quan trọng k, m, ruleset-failure-domain nó xác định chi phí lưu trữ và độ bền dữ liệu.

ví dụ:
```
$ ceph osd erasure-code-profile set myprofile \
   k=3 \
   m=2 \
   ruleset-failure-domain=rack
```

Các object sẽ đc chia làm 3 và m xác định bao nhiêu OSD có thể hỏng cùng lúc mà ko mất dữ liệu. Số OSD dùng là m+k.

###6. Cache Tiering

Cache tier cung cấp I/O tốt hơn với việc tối ưu các lớp data đc lưu ở backing storage tier. Cache đc sắp xếp với nhau để tạo thành một pool với các storage device có tốc độ nhanh ví dụ SSD. Chúng đc cấu hình tạo thành cache tier và backing pool cho erasure-coded. Ceph objecter handles nơi chứa Objects và tiering agent xác định khi chuyển Objects từ cache sang backing storage tier, quá trình này trong suốt với người dùng.

![cache tiering](https://camo.githubusercontent.com/dbdb6b7569e6a0a9fb2fd5bb6f5947afca7dae16/687474703a2f2f692e696d6775722e636f6d2f347478526b62522e706e67)

Cache tiering agent tiến hành migration data giữa cache tier and the backing storage tier tự động.

2 chế độ migration
<ul>
<li>
    Writeback Mode: Ceph client ghi data lên cache tier và nhận ACK từ cache tier. Lúc này, data đang đc ghi vào cache tier migrates tới storage tier. Cache tier đc đặt trước backing storage tier. Khi Ceph client cần data ở storage tier, cache tiering agent migrates the data to the cache tier. Sau đó nó chuyển tới Ceph client với I/O tối ưu.
</li>
<li>
    Read-proxy Mode: Chế độ này sẽ dùng các Objects đang có trong cache tier, nếu Objects ko có trong cache thì sẽ chuyển yêu cầu xuống bên dưới.
</li>
</ul>

###7. Data Striping

Thiết bị lưu trữ có giới hạn về lưu lượng, nó tác động tới hiệu suất và khả năng tính toán. Vì vậy hệ thống hỗ trợ Striping trên nhiều thiết bị để tăng hiệu suất.

Ceph striping giống như chế độ RAID 0

Ví dụ:

![dt](https://camo.githubusercontent.com/e5bc94dd8deb964e39b22886b0a9d8875bda19ef/687474703a2f2f692e696d6775722e636f6d2f555070576778332e706e67)

Trên hình ví dụ có 1 đoạn dữ liệu 123456. 123 được chia cho Object set 1. 456 được chia cho Object set 2. Các Object set tiếp tục phân chia thành các Objects theo số OSD. Sau đấy Striping nhỏ ra strip unit lưu trữ lên OSD.

Với tính năng này thay vì xử lý lần lượt từ đầu đến cuối dữ liệu, ta xử lý các đoạn dữ liệu nhỏ cùng 1 lúc.