#1 Ceph BlueStore

Từ jewel trở đi và đặc biệt là vào phiên bản mới nhất hiện tại là luminous thì bluestore thật sự ổn định và sẵn sàng để sử dụng

ở các phiên bản trước đó thì ceph sử dụng 1 format khác cho ODS của nó, đó là các kho chứa dữ liệu, được gọi là Filestore. Filestore được sử dụng tựa như tên của nó, các tập tin nhị phân trên đỉnh của một hệ thống tập tin(thường là XFS) để lưu trữ các đối tượng. Ngay cả khi concept đằng sau Filestore không có gì sai, thì layout này đơn giản là không hiệu quả khi chạy một kho lưu trữ đối tượng. Các nhà phát triển ceph nghĩ rằng có quá nhiều layers, và vì lí do này, họ phát triển Bluestore

![GitHub Logo](/images/bluestore-01.png)


Như bạn có thể thấy từ sơ đồ này, lớp hệ thống tập tin đã được loại bỏ, và bây giờ các đối tượng Ceph được ghi trực tiếp vào môi trường lưu trữ cơ bản, là một HDD hoặc một SSD (hoặc khác). Điều này loại bỏ rõ ràng một số phức tạp, và hiệu suất được cho là tăng lên khi có một lớp trừu tượng ít hơn. Nhưng nó không chỉ là hiệu suất: Bluestore có đầy đủ dữ liệu checksumming và được xây dựng trong nén.


Nói chung, BlueStore nhanh gấp 2 lần FileStore, và hiệu suất phù hợp hơn với độ trễ thấp hơn. 
Ngoài ra, không giống như FileStore, BlueStore là copy-on-write: hiệu suất với khối lượng RBD hoặc tệp CephFS gần đây đã được chụp nhanh sẽ tốt hơn nhiều.

##1.1 How does BlueStore work?

Vì BlueStore tiêu thụ raw block devices, ta sẽ nhận thấy thư mục dữ liệu giờ đây là một phân vùng nhỏ (100MB) chỉ với một vài tệp trong đó và phần còn lại của device trông giống như một phân vùng lớn không được sử dụng, với một  block symlink trong thư mục dữ liệu trỏ đến nó. Đây là nơi BlueStore đặt tất cả dữ liệu của nó và nó đang thực hiện IO trực tiếp tới raw block devices (sử dụng cơ sở hạ tầng libaio không đồng bộ của Linux) từ quá trình ceph-osd. (ta vẫn có thể thấy việc sử dụng trên OSD thông qua lệnh ceph osd df tiêu chuẩn). Ngoài ra, ta không còn có thể xem các tệp đối tượng cơ bản như ta đã từng sử dụng.


BlueStore có thể chạy chống lại sự kết hợp giữa các thiết bị chậm và nhanh giống như FileStore, nhưng nó được thiết kế tốt hơn để tận dụng các đặc tính của thiết bị nhanh. Khuyến nghị chung là lấy nhiều không gian SSD như bạn có sẵn cho OSD và sử dụng nó cho thiết bị block.db. Theo mặc định, một phân vùng sẽ được tạo trên thiết bị sdc bằng 1% kích thước thiết bị chính.

Một khía cạnh khác là bộ nhớ. FileStore là một giải pháp dựa trên tập tin để nó sử dụng một hệ thống tập tin Linux bình thường, có nghĩa là kernel  chịu trách nhiệm quản lý bộ nhớ để lưu trữ dữ liệu và siêu dữ liệu. Bởi vì BlueStore được thực hiện trong không gian người dùng như là một phần của OSD, Ceph bây giờ đã quản lý bộ nhớ cache của chính nó; với BlueStore có một tùy chọn cấu hình bluestore_cache_size kiểm soát bộ nhớ mà mỗi OSD sẽ sử dụng cho bộ đệm BlueStore. Theo mặc định, đây là 1 GB dành cho OSD có HDD và 3 GB cho OSD được SSD lưu trữ, nhưng bạn có thể đặt nó thành bất cứ điều gì phù hợp với môi trường của bạn. Đây là một giá trị lớn hơn trước, vì vậy chúng ta có thể mong đợi các cụm Ceph sẽ trở nên đói hơn một chút so với trước đây.

##1.2 Checksums and Compression

Như ta đã tìm hiểu, BlueStore không chỉ là về hiệu suất. Nó có 1 điểm quan trọng là checksumming, để lưu trữ dữ liệu ngay bây giờ thậm chí còn đáng tin cậy hơn. Ceph với Bluestore giờ đây tính toán, lưu trữ và kiểm tra tổng kiểm tra tất cả dữ liệu và siêu dữ liệu mà nó lưu trữ. Bất kỳ dữ liệu thời gian nào được đọc ra khỏi đĩa, một kiểm tra được sử dụng để xác minh dữ liệu là chính xác trước khi nó được tiếp xúc.
Ngoài ra, BlueStore có thể nén dữ liệu một cách minh bạch bằng zlib, snappy hoặc lz4. Điều này bị vô hiệu hóa theo mặc định, nhưng nó có thể được kích hoạt trên toàn cầu, cho các nhóm cụ thể, hoặc được chọn lọc khi khách hàng RADOS gợi ý rằng dữ liệu được nén.

##1.3 Converting existing clusters to use BlueStore

Rõ ràng, bằng cách sử dụng Bluestore trong một cụm mới được tạo ra là dễ dàng, nhưng đối với các cụm hiện có mà có thể muốn nâng cấp thì làm như thế nào? Nhóm Ceph đã thiết kế kịch bản này một cách thông minh: một cụm Ceph 12.2 có thể có cùng lúc với cả hai hệ điều hành Filestore và Bluestore OSD, và thực sự, sự lựa chọn kiểu phụ trợ được thực hiện cho mỗi OSD ngay cả đối với những cái mới: chứa một số OSD của FileStore và một số OSD của BlueStore. Một cụm được nâng cấp sẽ tiếp tục hoạt động như trước đây, ngoại trừ các OSD mới (theo mặc định) sẽ được triển khai với BlueStore.