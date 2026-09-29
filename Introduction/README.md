# Tổng Quan Về Ceph: Nguồn Gốc, Lợi Ích, Kiến Trúc Và Cách Vận Hành

Tài liệu này tổng hợp toàn bộ bức tranh về Ceph: lý do ra đời, cách thức hoạt động bên dưới, các thành phần cấu tạo và lý do Ceph luôn là lựa chọn hàng đầu cho hạ tầng điện toán đám mây như OpenStack.

---

## 1. Nguồn Gốc Lịch Sử Và Triết Lý Của Ceph

### 1.1. Bối cảnh ra đời
- Tác giả: Sage Weil bắt đầu nghiên cứu đề tài này tại Đại học California (UCSC) vào giai đoạn 2004–2006.
- Động lực: Tìm kiếm một giải pháp lưu trữ dữ liệu khổng lồ nhưng không bị phụ thuộc vào các tủ đĩa đắt tiền của các hãng độc quyền.
- Phát triển: Dự án sau đó được thương mại hóa bởi công ty Inktank, được hãng Red Hat mua lại vào năm 2014, và hiện nay là dự án mã nguồn mở do Quỹ Linux (Ceph Foundation) duy trì.

### 1.2. Ý nghĩa tên gọi "Ceph"
Tên gọi Ceph bắt nguồn từ Cephalopod (loài động vật chân đầu như bạch tuộc, mực).
- Bạch tuộc có hệ thần kinh phân tán: phần lớn các tế bào thần kinh nằm ở các xúc tu. Mỗi xúc tu có thể tự xử lý tình huống linh hoạt mà không cần chờ não trung tâm ra lệnh từng thao tác.
- Ceph học tập triết lý sinh học này: một cụm gồm nhiều máy chủ lưu trữ tự phối hợp, tự kiểm tra sức khỏe của nhau, tự sao chép dữ liệu để phòng ngừa rủi ro mà không cần một máy chủ điều khiển trung tâm làm đầu não.

---

## 2. Vì Sao Cần Ceph? (So Sánh Các Mô Hình Lưu Trữ)

Trước khi có giải pháp lưu trữ điều khiển bằng phần mềm (Software-Defined Storage), quản trị viên hệ thống thường phải chọn giữa 2 phương án:

| Tiêu chí | Lưu trữ cục bộ (Local Storage) | Tủ đĩa chuyên dụng (SAN/NAS vật lý) | Ceph (Software-Defined Storage) |
|---|---|---|---|
| Phần cứng | Dùng ổ cứng cắm trực tiếp trên máy chủ | Tủ đĩa chuyên dụng từ hãng (Dell, HP, NetApp) | Máy chủ và ổ đĩa tiêu chuẩn thông thường |
| Chi phí | Rẻ ban đầu | Rất đắt tiền mua thiết bị và phí duy trì bản quyền | Tối ưu, tận dụng được phần cứng phổ thông |
| Rủi ro mất dữ liệu | Rất cao (máy chủ hỏng là mất toàn bộ dữ liệu đĩa) | Thấp (nhờ linh kiện dự phòng kép trong tủ đĩa) | Rất thấp (dữ liệu tự động nhân bản qua mạng) |
| Khả năng mở rộng | Giới hạn theo số khe cắm ổ của từng máy chủ | Phải mua thêm cả một tủ đĩa mới khi hết chỗ | Cắm thêm ổ cứng hoặc thêm máy mới vào là xong |
| Sự phụ thuộc | Phân tán, quản lý rời rạc từng máy | Bị trói buộc hoàn toàn vào một nhà cung cấp | Mã nguồn mở, quản trị tập trung qua một hệ thống |

### Các giá trị thực tế Ceph mang lại:
1. Gom 3 loại lưu trữ vào một hệ thống duy nhất: Cấp ổ cứng ảo cho máy ảo, tạo thư mục chia sẻ cho nhiều máy dùng chung, và làm kho chứa tệp ảnh/video qua mạng web (chuẩn S3).
2. Không bị thắt cổ chai khi mở rộng: Các hệ thống khác thường có một "máy chủ mục lục" ghi nhớ vị trí file. Càng nhiều file thì mục lục càng nặng và chạy chậm. Ceph loại bỏ hoàn toàn danh bạ này bằng cách dùng công thức tính toán vị trí tự động (CRUSH).
3. Khả năng tự phục hồi: Nếu một ổ đĩa hoặc cả một máy chủ bị cháy nguồn, cụm Ceph tự phát hiện và tự động lấy các bản sao còn lại trên các máy khác để chép bù sang ổ mới, đảm bảo luôn duy trì đủ số lượng bản sao an toàn.

---

## 3. Kiến Trúc Phân Tầng Chuẩn Của Ceph (Ceph Software Stack)

Kiến trúc phần mềm của Ceph được tổ chức từ tầng ứng dụng bên ngoài xuống tận lớp ổ đĩa vật lý theo bảng phân tầng dưới đây:

| Tầng | Nhóm thành phần | Tên thành phần | Chức năng cụ thể |
|---|---|---|---|
| **Tầng 1** | **Ứng dụng tiêu thụ (Clients)** | OpenStack, Kubernetes, Apps | Nơi phát sinh nhu cầu đọc/ghi dữ liệu (VM, Container, App S3). |
| **Tầng 2** | **Giao diện dịch vụ (Interfaces)** | **RBD** (Block Storage) | Cung cấp ổ đĩa ảo để gắn trực tiếp vào máy ảo (OpenStack dùng). |
| | | **RGW** (Object Storage) | Cổng tương thích chuẩn AWS S3 / Swift để lưu file qua HTTP REST. |
| | | **CephFS** (File System) | Thư mục chia sẻ qua mạng chuẩn POSIX (cần thêm dịch vụ MDS). |
| **Tầng 3** | **Thư viện nền tảng (Library)** | **Librados** | Lớp thư viện lập trình (C++, Python...) để giao tiếp trực tiếp với lõi RADOS. |
| **Tầng 4** | **Lõi lưu trữ (RADOS Core)** | **Ceph MON** (Monitor) | Giữ bản đồ trạng thái toàn cụm (máy nào sống, đĩa nào chết). |
| | | **Ceph MGR** (Manager) | Cung cấp giao diện Dashboard, thu thập thông số hiệu năng cụm. |
| | | **Ceph OSD** (OSD Daemon) | Tiến trình trực tiếp điều khiển từng ổ cứng, xử lý ghi/nhân bản. |
| | | **Thuật toán CRUSH** | Công thức toán học tự động tính toán vị trí dữ liệu rơi vào ổ đĩa nào. |
| **Tầng 5** | **Cơ chế ghi đĩa (Engine)** | **BlueStore** | Cho phép Ceph ghi thẳng vào đĩa thô, tích hợp sẵn RocksDB lưu chỉ mục. |
| **Tầng 6** | **Hạ tầng vật lý & Mạng** | **Ổ cứng (Disks)** | Ổ đĩa NVMe, SSD, HDD thông thường. |
| | | **Mạng Public** | Dải mạng ngoài dành cho OpenStack gửi lệnh đọc/ghi vào Ceph. |
| | | **Mạng Cluster** | Dải mạng nội bộ riêng cho các OSD tự copy dữ liệu nhân bản sang nhau. |

---

## 4. Các Thành Phần Trọng Yếu Trong Cụm Ceph

### 4.1. Lõi lưu trữ RADOS
RADOS là nền tảng cốt lõi của Ceph. Mọi dữ liệu đưa vào hệ thống (dù là file văn bản hay ổ đĩa máy ảo 100GB) đều bị cắt nhỏ thành các mẩu dữ liệu chuẩn gọi là Object (kích thước mặc định là 4MB) để phân tán đi khắp cụm.

Các tiến trình cốt lõi chạy trên các máy chủ:
- OSD (Tiến trình quản lý đĩa): 
  - Mỗi ổ cứng vật lý gắn vào máy sẽ chạy tương ứng một tiến trình OSD.
  - Chịu trách nhiệm ghi dữ liệu thực tế vào đĩa, tự động nhân bản dữ liệu sang các máy khác, và quét kiểm tra lỗi dữ liệu định kỳ.
- MON (Tiến trình giám sát):
  - Nắm giữ bản đồ trạng thái của toàn bộ cụm: danh sách các máy chủ, trạng thái các ổ đĩa và quy tắc phân chia dữ liệu.
  - Luôn cần số lượng máy lẻ (3 hoặc 5 máy) để khi có tranh chấp, hệ thống dựa vào số đông biểu quyết để đưa ra quyết định chính xác.
- MGR (Tiến trình quản lý phụ trợ):
  - Hỗ trợ cho MON để giảm tải công việc giám sát, cung cấp giao diện Web trực quan (Ceph Dashboard) và xuất các biểu đồ theo dõi hiệu năng.
- MDS (Tiến trình quản lý thư mục):
  - Chỉ cần cài đặt khi bạn dùng tính năng chia sẻ thư mục dùng chung (CephFS). Tiến trình này chuyên lưu tên file, đường dẫn thư mục và quyền truy cập vào bộ nhớ RAM để tìm kiếm file nhanh chóng.

### 4.2. Cơ chế ghi đĩa hiện đại (BlueStore)
- Trước đây (FileStore): Ceph phải nhờ hệ điều hành format đĩa (dạng XFS/ext4) rồi mới ghi file vào. Cách này làm chậm hệ thống vì dữ liệu bị ghi lặp lại 2 lần do cơ chế ghi nhật ký của hệ điều hành.
- Hiện nay (BlueStore): Ceph chiếm quyền điều khiển trực tiếp toàn bộ bề mặt ổ đĩa thô. Ceph tự quản lý việc chia vùng và tự kiểm tra lỗi hỏng hóc vật lý của từng khối nhớ, mang lại tốc độ ghi nhanh hơn và độ an toàn cao hơn.

### 4.3. Mô hình mạng 2 lớp an toàn (Public Network & Cluster Network)
Một cụm Ceph chuẩn luôn được khuyến nghị chạy trên 2 dải mạng riêng:
- Dải mạng dịch vụ (Public Network): Dành cho các máy ngoài (như cụm OpenStack) gửi lệnh đọc/ghi vào Ceph.
- Dải mạng nội bộ (Cluster Network): Dành riêng cho các máy Ceph tự trao đổi với nhau. Khi một ổ cứng bị hỏng và cụm phải copy hàng trăm GB sang ổ khác để bù đắp, toàn bộ lưu lượng copy này chạy trên mạng nội bộ, không làm nghẽn đường truyền của các máy ảo đang chạy.

---

## 5. Cách Dữ Liệu Được Phân Bổ (Công Thức CRUSH Và Nhóm Dữ Liệu)

### 5.1. Công thức CRUSH hoạt động thế nào?
CRUSH là nét độc đáo nhất của Ceph:
- Khi một máy ảo cần đọc một khối dữ liệu, máy ảo không cần gửi câu hỏi tới một máy chủ trung tâm để xin địa chỉ.
- Máy ảo nhận bản đồ cụm từ MON một lần, sau đó tự chạy công thức CRUSH trên máy của mình: kết hợp tên của mẩu dữ liệu với sơ đồ cụm để ra ngay kết quả ổ đĩa đích đang giữ dữ liệu đó.
- Máy ảo kết nối thẳng tới đúng máy chủ và ổ cứng đó để lấy dữ liệu. Nhờ vậy, cụm dù có mở rộng lên hàng nghìn ổ cứng thì tốc độ tìm kiếm dữ liệu vẫn nhanh như ban đầu.

### 5.2. Đường đi của một gói dữ liệu
Dữ liệu đi qua 3 cấp độ phân chia trước khi nằm trên ổ cứng:

| Cấp độ | Tên gọi | Bản chất & Chức năng |
|---|---|---|
| **Cấp 1** | **Object** | Mẩu dữ liệu nhỏ (4MB) được cắt ra từ file gốc và đánh mã ID duy nhất. |
| **Cấp 2** | **Placement Group (PG)** | Chiếc "thùng chứa logic" gom hàng triệu Object lại thành vài trăm nhóm để hệ thống dễ quản lý. |
| **Cấp 3** | **OSD (Ổ cứng)** | Ổ đĩa vật lý thực tế nhận các thùng PG theo công thức phân bổ để đảm bảo tính an toàn. |

---

## 6. Mô Hình Tích Hợp Ceph Với OpenStack

Khi ghép nối cụm OpenStack với cụm Ceph, toàn bộ gánh nặng về lưu trữ được chuyển giao hoàn toàn cho Ceph:

| Dịch vụ OpenStack | Vai trò | Kho lưu trữ Ceph (Pool) | Giao thức kết nối |
|---|---|---|---|
| **Glance** | Quản lý file Image cài hệ điều hành | Pool `images` | Giao diện ổ đĩa ảo (RBD) |
| **Cinder** | Cấp ổ đĩa gắn thêm cho máy ảo | Pool `volumes` | Giao diện ổ đĩa ảo (RBD) |
| **Nova** | Chạy máy ảo và quản lý ổ đĩa root của máy ảo | Pool `vms` | Thư viện ảo hóa `libvirt / qemu-rbd` |

### Lợi ích thực tế khi kết hợp:
1. Tạo máy ảo cực nhanh (Cơ chế Copy-on-Write): Khi tạo một máy ảo mới từ file hệ điều hành (Image), Ceph không tốn thời gian copy hàng chục GB sang máy ảo đó. Ceph chỉ tạo một "lối tắt" trỏ vào file gốc. Quá trình bật máy ảo diễn ra trong vài giây, và dung lượng chỉ tăng lên khi máy ảo bắt đầu ghi thêm dữ liệu mới.
2. Di chuyển máy ảo không gián đoạn (Live Migration): Do ổ đĩa của máy ảo thực chất nằm tập trung trên cụm Ceph chứ không nằm ở máy chủ chạy máy ảo, nên khi máy chủ Compute bị quá tải hoặc cần bảo trì, bạn có thể chuyển máy ảo sang máy chủ khác mà dịch vụ của người dùng không bị mất kết nối.
3. Độ tin cậy cho hạ tầng ảo hóa: Nếu một máy chủ chạy máy ảo bị cháy nguồn đột ngột, toàn bộ ổ cứng của máy ảo vẫn an toàn 100% trên cụm Ceph. OpenStack chỉ việc khởi động lại máy ảo đó trên một máy chủ khác ngay lập tức.
