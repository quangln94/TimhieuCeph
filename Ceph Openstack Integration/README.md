# Tích hợp Ceph với OpenStack

## 1. Tổng quan

Ceph là hệ thống lưu trữ phân tán, cung cấp khả năng mở rộng và lưu trữ dữ liệu cho các dịch vụ OpenStack.

Ceph có thể tích hợp với nhiều thành phần của OpenStack:

- **Glance:** Lưu trữ image của máy ảo.
- **Cinder:** Cung cấp block storage, cho phép tạo và quản lý volume.
- **Nova:** Sử dụng Ceph RBD để lưu trữ disk của instance.
- **Manila:** Có thể sử dụng CephFS làm backend chia sẻ file.
- **RGW:** Cung cấp object storage tương thích với API S3.

Trong phạm vi lab này, trọng tâm là tích hợp Ceph RBD với Glance, Cinder và Nova.

## 2. Kiến trúc lab

### 2.1. Các node OpenStack

| Node | Địa chỉ IP | Vai trò |
|---|---|---|
| Controller | `10.168.36.11` | Các dịch vụ điều khiển OpenStack, Glance API, Cinder API |
| Compute1 | `10.168.36.12` | Nova Compute, Cinder Volume |
| Compute2 | `10.168.36.13` | Nova Compute, Cinder Volume |

### 2.2. Các node Ceph

| Node | Địa chỉ IP |
|---|---|
| ceph-node1 | `10.168.36.32` |
| ceph-node2 | `10.168.36.33` |
| ceph-node3 | `10.168.36.34` |

Thông tin cluster tại thời điểm kiểm tra:

- Phiên bản Ceph: `19.2.6 Squid`
- Số lượng MON: 3
- Số lượng OSD: 9
- Trạng thái cluster: `HEALTH_OK`

### 2.3. Các pool Ceph

| Pool | Mục đích sử dụng |
|---|---|
| `images` | Lưu trữ image của Glance |
| `cinder-volumes` | Lưu trữ volume của Cinder |
| `vms` | Dự kiến lưu trữ disk của instance thông qua Nova/libvirt |

Việc phân chia pool giúp tách biệt dữ liệu theo mục đích sử dụng. Các pool vẫn sử dụng chung tài nguyên của Ceph cluster và có thể được cấu hình với chính sách lưu trữ khác nhau.

## 3. Nguyên lý tích hợp Ceph RBD

RBD (RADOS Block Device) là giao diện block storage được xây dựng trên nền tảng RADOS của Ceph.

RBD image có thể được sử dụng như một thiết bị block ảo. Các dịch vụ OpenStack truy cập Ceph thông qua thư viện RADOS/RBD, file cấu hình Ceph và thông tin xác thực CephX.

Các thành phần quan trọng:

- **MON:** Quản lý bản đồ và trạng thái cluster, đồng thời hỗ trợ xác thực.
- **OSD:** Lưu trữ dữ liệu và xử lý các thao tác đọc/ghi.
- **RADOS:** Lớp lưu trữ đối tượng phân tán nền tảng của Ceph.
- **RBD:** Cung cấp giao diện block storage trên RADOS.
- **CephX:** Cơ chế xác thực và phân quyền truy cập Ceph.

Mỗi dịch vụ OpenStack cần có cấu hình kết nối, thông tin xác thực và quyền truy cập phù hợp với pool mà dịch vụ sử dụng.

## 4. Tích hợp Glance với Ceph RBD

### 4.1. Mục đích

Glance là dịch vụ quản lý image trong OpenStack.

Khi sử dụng Ceph RBD làm backend, image có thể được lưu trong pool `images` thay vì chỉ lưu trên filesystem của Controller.

Lợi ích:

- Tách dữ liệu image khỏi filesystem của Controller.
- Tận dụng khả năng lưu trữ phân tán và mở rộng của Ceph.
- Cung cấp nguồn image cho quá trình khởi tạo instance của Nova.

### 4.2. Cấu hình backend

File cấu hình:

`/etc/glance/glance-api.conf`

Các tham số liên quan đến backend Ceph:

```ini
[DEFAULT]
enabled_backends = file:file,ceph:rbd

[ceph]
rbd_store_pool = images
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
```

Ý nghĩa các tham số:

- `enabled_backends`: Khai báo các backend lưu trữ mà Glance có thể sử dụng.
- `rbd_store_pool`: Chỉ định pool dùng để lưu image.
- `rbd_store_user`: Chỉ định tài khoản CephX được sử dụng.
- `rbd_store_ceph_conf`: Đường dẫn đến file cấu hình Ceph.

Cấu hình thực tế cần bảo đảm backend mặc định được chọn đúng và các tùy chọn phù hợp với phiên bản Glance đang sử dụng.

### 4.3. Xác thực CephX

Glance sử dụng tài khoản `client.glance` để truy cập pool `images`.

Các nguyên tắc bảo mật:

- Chỉ cấp những quyền cần thiết cho Glance.
- Không sử dụng `client.admin` cho hoạt động thông thường của Glance.
- Giới hạn quyền đọc file keyring.
- Không đưa keyring hoặc secret lên GitHub.


## 5. Tích hợp Cinder với Ceph RBD

### 5.1. Mục đích

Cinder là dịch vụ Block Storage của OpenStack, chịu trách nhiệm quản lý vòng đời của volume, bao gồm tạo, xóa, attach và detach.

Khi sử dụng Ceph RBD làm backend, dữ liệu volume được lưu trong pool `cinder-volumes`.

### 5.2. Pool và tài khoản CephX

Pool được sử dụng:

`cinder-volumes`

Tài khoản CephX:

`client.cinder`

Tài khoản này cần có quyền phù hợp với các pool và thao tác mà backend yêu cầu. Trong môi trường production, nên giới hạn quyền theo đúng nhu cầu của từng dịch vụ.

### 5.3. Cấu hình backend Cinder

File cấu hình trên các node chạy Cinder Volume:

`/etc/cinder/cinder.conf`

Ví dụ về các tham số của backend RBD:

```ini
[ceph]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
volume_backend_name = ceph
rbd_pool = cinder-volumes
rbd_user = cinder
rbd_ceph_conf = /etc/ceph/ceph.conf
```

Đây là ví dụ tham khảo. Tên section và các tham số phải khớp với cấu hình `enabled_backends` hiện có. Không ghi đè toàn bộ file cấu hình chỉ để bổ sung backend.


## 6. Tích hợp Nova/libvirt với Ceph RBD

### 6.1. Mục đích

Nova quản lý vòng đời của instance. Trên compute node, Nova phối hợp với libvirt và hypervisor để tạo và vận hành máy ảo.

Khi cấu hình RBD cho disk instance, dữ liệu disk có thể được lưu trong pool `vms` thay vì chỉ nằm trên local storage của compute node.

### 6.2. Các thành phần cần thiết

Trên Compute1 và Compute2 cần có:

- File `/etc/ceph/ceph.conf`.
- Keyring của tài khoản CephX được cấu hình cho Nova/libvirt.
- Libvirt secret tương ứng với thông tin xác thực Ceph.
- Quyền CephX phù hợp với pool RBD.
- Cấu hình RBD trong `/etc/nova/nova.conf`.
