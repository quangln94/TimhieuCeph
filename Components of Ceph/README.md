# Các Thành Phần Cốt Lõi Trong Kiến Trúc Ceph

## 1. Tổng quan

Trong kiến trúc Ceph, các thành phần được tổ chức để phối hợp quản lý cluster, lưu trữ dữ liệu, phân phối dữ liệu và cung cấp các giao diện lưu trữ cho ứng dụng.

Theo mô hình kiến trúc Ceph trong slide thuyết trình, bốn thành phần quan trọng cần tìm hiểu là:

- **MON (Monitor):** Duy trì trạng thái và bản đồ của cluster, quản lý quorum giữa các MON.
- **MGR (Manager):** Cung cấp chức năng quản lý, giám sát và các module mở rộng.
- **OSD (Object Storage Daemon):** Trực tiếp lưu trữ dữ liệu, xử lý đọc/ghi và tham gia cơ chế bảo vệ dữ liệu.
- **MDS (Metadata Server):** Quản lý metadata của CephFS.

MON, MGR và OSD là các thành phần quan trọng trong một Ceph cluster thông dụng. MDS chỉ cần khi triển khai CephFS, không bắt buộc đối với Ceph RBD hoặc Ceph RGW.

## 2. MON — Monitor

### 2.1. MON là gì?

MON (Monitor) là thành phần quản lý trạng thái điều khiển của Ceph cluster. MON duy trì các bản đồ và thông tin quan trọng để các daemon và client biết được cấu trúc, trạng thái hiện tại của cluster.

MON không trực tiếp lưu dữ liệu người dùng. Thay vào đó, nó cung cấp thông tin giúp các thành phần xác định trạng thái của cluster và cách phối hợp với nhau.

### 2.2. Chức năng của MON

Các chức năng chính của MON bao gồm:

- Duy trì các cluster map, chẳng hạn MON map, OSD map, PG map và CRUSH map.
- Tham gia cơ chế quorum để các MON thống nhất trạng thái điều khiển.
- Quản lý thông tin xác thực CephX và hỗ trợ xác thực các kết nối theo cơ chế của Ceph.
- Cung cấp cluster map cho client và các daemon cần sử dụng.
- Giúp các thành phần nhận biết thay đổi về trạng thái cluster, chẳng hạn khi OSD tham gia hoặc rời khỏi cluster.

Nhờ cluster map, client có thể tính toán vị trí dữ liệu cần truy cập thay vì phải gửi mọi yêu cầu đọc/ghi qua một máy chủ lưu trữ trung tâm.

### 2.3. Quorum trong Ceph

Quorum là cơ chế giúp các MON thống nhất trạng thái điều khiển của cluster. Ceph thường triển khai số lượng MON lẻ để có thể duy trì quorum khi một số MON gặp sự cố.

Ví dụ với 3 MON:

| Số MON hoạt động | Khả năng duy trì quorum |
|---|---|
| 3/3 | Có |
| 2/3 | Có thể duy trì quorum |
| 1/3 | Không đủ quorum |

Với 3 MON, cần ít nhất 2 MON tham gia quorum. Với 5 MON, cần ít nhất 3 MON.

Quorum không đồng nghĩa với replication. Quorum giúp duy trì sự thống nhất về trạng thái điều khiển, còn replication là cơ chế bảo vệ dữ liệu do các OSD thực hiện theo cấu hình pool.

Việc có nhiều MON cũng không tự động bảo đảm khả năng chịu lỗi nếu tất cả MON cùng phụ thuộc vào một máy chủ vật lý hoặc một miền lỗi duy nhất.

### 2.4. Vai trò của MON trong hoạt động lưu trữ

Khi một client cần truy cập dữ liệu:

1. Client lấy thông tin cluster map từ MON khi cần.
2. Client sử dụng thông tin về cluster và CRUSH để xác định vị trí dữ liệu.
3. Client kết nối đến các OSD phù hợp để thực hiện thao tác đọc/ghi.

Do đó, MON giữ vai trò cung cấp thông tin điều khiển, nhưng không phải trung gian bắt buộc chuyển tiếp mọi gói dữ liệu giữa client và OSD.


## 3. MGR — Manager

### 3.1. MGR là gì?

MGR (Manager) là thành phần cung cấp các chức năng quản lý và giám sát bổ sung cho Ceph cluster.

Trong kiến trúc Ceph, MGR làm việc cùng MON và các daemon khác để thu thập thông tin vận hành, cung cấp số liệu hiệu năng, giao diện quản trị và các chức năng mở rộng thông qua module.

MGR không thay thế MON và không trực tiếp đảm nhiệm việc lưu dữ liệu người dùng như OSD.

### 3.2. Chức năng của MGR

Các chức năng chính của MGR bao gồm:

- Thu thập và cung cấp thông tin về trạng thái, dung lượng và hiệu năng cluster.
- Cung cấp Ceph Dashboard để quản trị viên theo dõi cluster qua giao diện web.
- Cung cấp các module phục vụ monitoring và tích hợp với hệ thống giám sát.
- Hỗ trợ thống kê, quản trị và tự động hóa thông qua các module.
- Cung cấp cơ chế active/standby để hỗ trợ tính sẵn sàng của dịch vụ Manager.

MGR giúp quản trị viên dễ dàng theo dõi và vận hành cluster, đặc biệt khi hệ thống có nhiều node và nhiều OSD.

### 3.3. Active MGR và Standby MGR

Ceph có thể triển khai nhiều MGR daemon. Thông thường, một daemon giữ vai trò active và các daemon khác ở trạng thái standby.

- **Active MGR:** Đảm nhiệm vai trò quản lý đang hoạt động và cung cấp các module tương ứng.
- **Standby MGR:** Chờ tiếp quản vai trò active khi daemon hiện tại gặp sự cố.

Mô hình minh họa:

```text
Ceph Cluster
     |
     +-- MGR 1: Active
     |
     +-- MGR 2: Standby
     |
     +-- MGR 3: Standby
```

Khi MGR active gặp sự cố, một MGR standby có thể được chọn để tiếp quản. Điều này giúp duy trì các chức năng quản lý, tuy nhiên một số kết nối hoặc chức năng giám sát vẫn có thể bị gián đoạn trong quá trình chuyển đổi.

### 3.4. Ceph Dashboard và MGR module

Ceph Dashboard là giao diện web phục vụ giám sát và quản trị cluster. Dashboard cho phép theo dõi:

- Trạng thái tổng thể của cluster.
- MON, MGR và OSD.
- Các pool, PG và tình trạng sử dụng dung lượng.
- Cảnh báo và một số chỉ số hiệu năng.
- Những thao tác quản trị được hỗ trợ.


## 4. OSD — Object Storage Daemon

### 4.1. OSD là gì?

OSD (Object Storage Daemon) là thành phần trực tiếp quản lý việc lưu trữ dữ liệu trong Ceph.

Mỗi OSD có một định danh riêng và thường được triển khai để quản lý một thiết bị lưu trữ hoặc cấu hình thiết bị tương ứng.

OSD chịu trách nhiệm thực hiện các thao tác đọc/ghi dữ liệu, đồng thời tham gia các cơ chế replication, recovery và phân phối lại dữ liệu của cluster.

### 4.2. Chức năng của OSD

Các chức năng chính của OSD bao gồm:

- Lưu trữ dữ liệu dưới dạng object.
- Xử lý các yêu cầu đọc/ghi.
- Phối hợp với các OSD khác để thực hiện replication hoặc erasure coding theo cấu hình pool.
- Phát hiện và xử lý các thay đổi về trạng thái dữ liệu.
- Tham gia quá trình recovery khi dữ liệu cần được phục hồi.
- Thực hiện backfill khi dữ liệu cần được chuyển sang OSD khác.
- Báo cáo trạng thái hoạt động cho các thành phần liên quan.

OSD thường sử dụng BlueStore làm backend lưu trữ, cho phép Ceph quản lý dữ liệu trực tiếp trên thiết bị block.

### 4.3. Mối quan hệ giữa OSD và ổ đĩa

Trong một mô hình triển khai phổ biến, mỗi ổ đĩa dữ liệu được sử dụng để tạo một OSD riêng.

Ví dụ:

```text
Ceph Node 1
  |
  +-- Disk 1 -> OSD.0
  +-- Disk 2 -> OSD.1
  +-- Disk 3 -> OSD.2

Ceph Node 2
  |
  +-- Disk 1 -> OSD.3
  +-- Disk 2 -> OSD.4
  +-- Disk 3 -> OSD.5
```

Đây là mô hình minh họa. Số lượng OSD thực tế phụ thuộc vào thiết kế phần cứng và cách triển khai.

Cần phân biệt OSD với ổ đĩa vật lý: ổ đĩa là thiết bị lưu trữ, còn OSD là daemon Ceph quản lý dữ liệu trên thiết bị được cấu hình.

### 4.4. Replication và bảo vệ dữ liệu

Replication là cơ chế lưu nhiều bản sao dữ liệu trên các OSD khác nhau nhằm giảm nguy cơ mất dữ liệu khi thiết bị gặp sự cố.

Ví dụ, pool có `size = 3` thường được cấu hình để duy trì ba bản sao của dữ liệu trên các OSD theo quy tắc phân phối của cluster.

Mô hình khái quát:

```text
             Yêu cầu ghi
                  |
                  v
             Primary OSD
              /       \
             v         v
       Secondary 1  Secondary 2
```

Sơ đồ trên chỉ minh họa ý tưởng. Quá trình ghi thực tế phụ thuộc vào PG, acting set, trạng thái OSD và cấu hình pool.

Để tăng khả năng chịu lỗi, các bản sao nên được phân bố trên những miền lỗi khác nhau, chẳng hạn các máy chủ vật lý khác nhau. Nếu tất cả bản sao cùng nằm trên một máy chủ, sự cố của máy chủ đó vẫn có thể ảnh hưởng đến toàn bộ bản sao.

### 4.5. Recovery, backfill và rebalance

Khi một OSD gặp sự cố hoặc cluster thay đổi, Ceph có thể cần phục hồi hoặc phân phối lại dữ liệu.

- **Recovery:** Khôi phục dữ liệu hoặc bản sao cần thiết để đáp ứng chính sách bảo vệ dữ liệu.
- **Backfill:** Chuyển dữ liệu sang OSD cần nhận dữ liệu theo trạng thái và quy tắc phân phối mới.
- **Rebalance:** Cân bằng lại dữ liệu giữa các OSD khi cách phân phối dữ liệu thay đổi.

Những hoạt động này có thể tạo ra lưu lượng mạng và tải đọc/ghi đáng kể. Quản trị viên cần theo dõi trạng thái cluster và hiệu năng trong quá trình phục hồi.

### 4.6. Kiểm tra OSD

```bash
ceph osd ls
ceph osd tree
ceph osd stat
ceph osd df
```

Các lệnh trên lần lượt giúp liệt kê OSD, xem cây phân cấp, kiểm tra trạng thái hoạt động và xem mức sử dụng dung lượng theo OSD.

## 5. MDS — Metadata Server

### 5.1. MDS là gì?

MDS (Metadata Server) là thành phần quản lý metadata của CephFS.

Metadata bao gồm những thông tin như:

- Tên file và tên thư mục.
- Cấu trúc cây thư mục.
- Quan hệ giữa file và thư mục.
- Các thuộc tính và thông tin quyền truy cập.
- Những thông tin cần thiết để xử lý các thao tác trên namespace của filesystem.

MDS không trực tiếp lưu nội dung file như OSD. Nội dung file vẫn được lưu trên Ceph thông qua lớp lưu trữ dữ liệu của cluster.

### 5.2. Vì sao CephFS cần MDS?

CephFS là filesystem phân tán, cho phép client truy cập dữ liệu theo mô hình file và thư mục.

Khi người dùng thực hiện các thao tác như tạo thư mục, đổi tên file, liệt kê nội dung thư mục hoặc tra cứu thuộc tính file, hệ thống cần xử lý metadata tương ứng.

MDS đảm nhiệm phần quản lý metadata này, giúp CephFS cung cấp cách truy cập theo filesystem thay vì trực tiếp theo block hoặc object.

### 5.3. MDS và dữ liệu file

Cần phân biệt hai loại dữ liệu:

| Loại dữ liệu | Ví dụ | Thành phần liên quan |
|---|---|---|
| Metadata | Tên file, thư mục, thuộc tính | MDS |
| Nội dung file | Dữ liệu thực tế bên trong file | OSD thông qua lớp lưu trữ Ceph |

MDS quản lý namespace và metadata, không phải nơi chứa toàn bộ nội dung file.

### 5.4. Active MDS và Standby MDS

CephFS có thể triển khai nhiều MDS daemon để hỗ trợ tính sẵn sàng và phân phối tải metadata.

- **Active MDS:** Đang phục vụ các hoạt động metadata.
- **Standby MDS:** Chờ tiếp quản khi một MDS active gặp sự cố.
- **Standby-replay:** Có thể theo dõi nhật ký của một MDS active để hỗ trợ quá trình chuyển đổi, tùy cấu hình.

Việc có MDS standby giúp tăng khả năng sẵn sàng của dịch vụ metadata, nhưng không thay thế cơ chế bảo vệ dữ liệu do OSD thực hiện.

### 5.5. Khi nào cần MDS?

| Giao diện lưu trữ | Có cần MDS không? | Lý do |
|---|---|---|
| RBD | Không | Cung cấp block storage |
| CephFS | Có | Cần dịch vụ quản lý metadata |
| RGW | Không | Cung cấp object storage qua API |

Vì vậy, trong lab OpenStack đang sử dụng Ceph RBD cho Glance, Cinder và dự kiến Nova, bạn chưa cần triển khai MDS nếu chưa có nhu cầu sử dụng CephFS.

### 5.6. Kiểm tra MDS

```bash
ceph fs status
ceph mds stat
ceph fs dump
```

Nếu cluster chưa được cấu hình CephFS, các lệnh có thể cho thấy chưa có filesystem hoặc MDS đang hoạt động. Điều đó không tự động đồng nghĩa với việc Ceph RBD đang gặp lỗi.

## 6. Mối quan hệ giữa MON, MGR, OSD và MDS

Bốn thành phần phối hợp để tạo nên hệ thống Ceph hoàn chỉnh:

| Thành phần | Vai trò trong kiến trúc |
|---|---|
| MON | Duy trì trạng thái điều khiển và cung cấp cluster map |
| MGR | Hỗ trợ quản lý, giám sát và các module mở rộng |
| OSD | Lưu trữ dữ liệu và xử lý đọc/ghi |
| MDS | Quản lý metadata khi sử dụng CephFS |

Có thể hình dung luồng hoạt động khái quát như sau:

1. Client kết nối đến Ceph và lấy thông tin cluster cần thiết.
2. MON cung cấp thông tin trạng thái và cluster map.
3. Client sử dụng thông tin phân phối dữ liệu để xác định OSD phù hợp.
4. OSD xử lý dữ liệu và phối hợp với các OSD khác theo cấu hình pool.
5. MGR cung cấp thông tin giám sát và chức năng quản trị cho quản trị viên.
6. Nếu sử dụng CephFS, MDS xử lý các thao tác metadata của filesystem.

