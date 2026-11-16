# Ceph Storage Management


## 1. Tổng quan mô hình Lab

Môi trường sử dụng Ceph 19.2.6 Squid, triển khai bằng Cephadm.

| Node | OSD | Device Class |
|---|---|---|
| ceph-node1 | osd.0 | ssd (logical) |
| ceph-node1 | osd.1, osd.2 | hdd |
| ceph-node2 | osd.3 | ssd (logical) |
| ceph-node2 | osd.4, osd.5 | hdd |
| ceph-node3 | osd.6 | ssd (logical) |
| ceph-node3 | osd.7, osd.8 | hdd |

- Tổng số OSD: 9.
- Dung lượng mỗi OSD: 100 GB.
- Tổng dung lượng raw: 900 GB.
- 6 OSD mang nhãn HDD.
- 3 OSD mang nhãn SSD.


## 2. Các thành phần quan trọng của Ceph

| Thành phần | Chức năng |
|---|---|
| OSD | Lưu trữ và quản lý dữ liệu |
| Device Class | Phân loại OSD như HDD, SSD, NVMe |
| CRUSH Map | Mô tả cấu trúc lưu trữ của Ceph |
| CRUSH Rule | Quy định nhóm OSD được phép lưu dữ liệu |
| Pool | Không gian lưu trữ logic chứa Object |
| PG | Nhóm Object phục vụ quản lý và phân phối dữ liệu |
| RBD | Cung cấp Block Storage trên Ceph |

### Nguyên lý hoạt động

```text
Application
    |
    v
RBD Image
    |
    v
Object
    |
    v
Placement Group
    |
    v
CRUSH Rule
    |
    v
OSD
```

### Replication

Ví dụ cấu hình:

- `size = 3`: mỗi Object có 3 bản sao.
- `min_size = 2`: ngưỡng replica tối thiểu để PG phục vụ ghi trong điều kiện phù hợp.

Với failure domain là `host`, Ceph cố gắng phân bố các replica trên các máy chủ khác nhau.

## 3. Device Class

Device Class là nhãn phân loại OSD được CRUSH sử dụng để lựa chọn thiết bị lưu trữ.

Các class phổ biến:

- `hdd`
- `ssd`
- `nvme`

Ngoài ra, quản trị viên có thể tạo class tùy chỉnh như `fast`, `backup`, `archive`, `abc`.

### Kiểm tra Device Class

```bash
ceph osd tree
ceph osd crush class ls
```

### Thay đổi Device Class

```bash
# Gỡ class cũ
ceph osd crush rm-device-class osd.0

# Gán class mới
ceph osd crush set-device-class ssd osd.0
```

Lệnh thay đổi ba OSD trong Lab:

```bash
ceph osd crush rm-device-class osd.0
ceph osd crush set-device-class ssd osd.0

ceph osd crush rm-device-class osd.3
ceph osd crush set-device-class ssd osd.3

ceph osd crush rm-device-class osd.6
ceph osd crush set-device-class ssd osd.6
```

**Lưu ý:** Device Class là nhãn logic. Đổi HDD sang SSD không biến ổ vật lý thành SSD.

## 4. CRUSH Rule

CRUSH Rule quyết định dữ liệu được phân bố trên nhóm OSD nào.

Trong Lab tạo hai Rule:

```bash
# Rule dành cho HDD
ceph osd crush rule create-replicated hdd-rule default host hdd

# Rule dành cho SSD
ceph osd crush rule create-replicated ssd-rule default host ssd
```

Ý nghĩa:

| Thành phần | Ý nghĩa |
|---|---|
| hdd-rule / ssd-rule | Tên Rule |
| default | Root bucket |
| host | Failure domain |
| hdd / ssd | Device Class |

Kiểm tra:

```bash
ceph osd crush rule ls
ceph osd crush rule dump hdd-rule
ceph osd crush rule dump ssd-rule
```

Rule HDD lọc OSD thuộc class HDD; Rule SSD lọc OSD thuộc class SSD.

## 5. Storage Pool

Pool là không gian lưu trữ logic trong Ceph. Mỗi Pool có thể sử dụng một CRUSH Rule khác nhau để kiểm soát vị trí dữ liệu.

### Tạo HDD Pool

```bash
ceph osd pool create hdd-pool 32 32 replicated hdd-rule
ceph osd pool set hdd-pool size 3
ceph osd pool set hdd-pool min_size 2
ceph osd pool application enable hdd-pool rbd
rbd pool init hdd-pool
```

### Tạo SSD Pool

```bash
ceph osd pool create ssd-pool 32 32 replicated ssd-rule
ceph osd pool set ssd-pool size 3
ceph osd pool set ssd-pool min_size 2
ceph osd pool application enable ssd-pool rbd
rbd pool init ssd-pool
```


Kiểm tra:

```bash
ceph osd pool ls detail
ceph osd pool get lab-vm-pool crush_rule
ceph osd pool get lab-backup-pool crush_rule
```

Lưu ý:
- Không chạy lại lệnh tạo Pool nếu Pool đã tồn tại.
- Thông số PG `32` được sử dụng trong Lab. Môi trường mới cần xem xét PG Autoscaler và khuyến nghị của phiên bản Ceph.
- Tạo Pool backup không đồng nghĩa Cinder Backup đã sử dụng Pool đó. Cần cấu hình riêng Cinder Backup Service.

## 6. Truy vết dữ liệu đến OSD

Luồng lưu trữ:

```text
VM
 |
 v
Cinder Volume
 |
 v
RBD Image
 |
 v
RBD Object
 |
 v
Placement Group
 |
 v
OSD
```

### Liệt kê RBD Image

```bash
rbd ls lab-vm-pool
```

### Xem thông tin Image

```bash
rbd info lab-vm-pool/<image-name>
```

### Liệt kê Object

```bash
rados -p lab-vm-pool ls
```

### Xác định PG và OSD

```bash
ceph osd map lab-vm-pool <object-name>
```

Ví dụ:

```text
pool 'lab-vm-pool'
object 'rbd_data.xxxxx'
pg 11.63af0bc2
up    [1,7,4]
acting [1,7,4]
```

Ý nghĩa: Object thuộc PG có các replica trên osd.1, osd.7 và osd.4.

Một Object không đại diện cho toàn bộ Volume. Các Object khác nhau có thể nằm trên PG và tổ hợp OSD khác nhau.


## 7. Các lệnh kiểm tra tổng hợp

```bash
# Ceph cluster
ceph -s
ceph df

# OSD và Device Class
ceph osd tree
ceph osd crush class ls

# CRUSH Rule
ceph osd crush rule ls
ceph osd crush rule dump hdd-rule
ceph osd crush rule dump ssd-rule

# Pool
ceph osd pool ls detail
ceph osd pool get <POOL_NAME> crush_rule

# OpenStack Volume Type
openstack volume type list
openstack volume type show <TYPE_NAME>

# Cinder Backend
openstack volume service list

# Volume
openstack volume list
openstack volume show <VOLUME_ID>
```

## 8. Tài liệu tham khảo

- [Ceph - CRUSH Map](https://docs.ceph.com/en/squid/rados/operations/crush-map/)
- [Ceph - CRUSH Map Editing](https://docs.ceph.com/en/squid/rados/operations/crush-map-edits/)
- [Ceph - Storage Pools](https://docs.ceph.com/en/squid/rados/operations/pools/)
- [OpenStack Cinder - Multiple Backends](https://docs.openstack.org/cinder/latest/admin/blockstorage-multi-backend.html)
- [OpenStack Cinder - Volume Types](https://docs.openstack.org/cinder/latest/admin/volume-types.html)
- [OpenStack Cinder - Volume Backups](https://docs.openstack.org/cinder/latest/admin/volume-backups.html)
