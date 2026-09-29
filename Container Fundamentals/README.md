# Container, Docker, Podman và Cephadm

## 1. Container là gì?

**Container** là một phương pháp đóng gói và chạy ứng dụng trong một môi trường được cô lập tương đối với hệ điều hành host.

Container chứa ứng dụng cùng các thành phần cần thiết để ứng dụng có thể chạy, chẳng hạn như:

- Application
- Libraries
- Dependencies
- Configuration

Mô hình đơn giản:

```text
Ubuntu Host
    │
    └── Container
          ├── Application
          ├── Libraries
          ├── Dependencies
          └── Configuration
```

Điểm quan trọng:

> Container không phải là một Virtual Machine.

---

# 2. Container khác Virtual Machine như thế nào?

## Virtual Machine

VM chạy thông qua Hypervisor và mỗi VM thường có một Guest OS riêng.

```text
Physical Server
      │
  Hypervisor
      │
 ┌────┴────┐
 │         │
VM1       VM2
 │         │
OS        OS
 │         │
App       App
```

Ví dụ:

```text
VM1 → Ubuntu
VM2 → Ubuntu
VM3 → Windows
```

Mỗi VM có hệ điều hành riêng.

---

## Container

Container dùng chung kernel của hệ điều hành host.

```text
Physical Server
      │
    Linux
      │
Container Runtime
      │
 ┌────┼────┐
 │    │    │
 C1   C2   C3
 │    │    │
App  App  App
```

Do không cần một Guest OS đầy đủ cho mỗi container nên container thường nhẹ hơn VM và khởi động nhanh hơn.

### Có thể nhớ:

```text
VM
→ Virtualize một máy tính

Container
→ Isolate một ứng dụng/process
```

---

# 3. Container Image là gì?

**Container Image** là một template dùng để tạo container.

Image có thể chứa:

```text
Image
 ├── Application
 ├── Libraries
 ├── Dependencies
 └── Configuration
```

Từ một image có thể tạo nhiều container:

```text
             Image
               │
       ┌───────┼───────┐
       ▼       ▼       ▼
 Container  Container  Container
    1          2          3
```

Ví dụ với Nginx:

```bash
docker pull nginx
```

Lệnh trên tải Nginx image.

Sau đó:

```bash
docker run nginx
```

Docker tạo và chạy một container dựa trên image đó.

### Quan hệ:

```text
IMAGE
  │
  │ create / run
  ▼
CONTAINER
```

---

# 4. Docker là gì?

**Docker không phải là Container.**

Container là công nghệ/khái niệm để đóng gói và chạy ứng dụng.

**Docker là một platform/tooling ecosystem dùng để build, distribute và run container.**

Có thể hình dung:

```text
Container
    ↑
Khái niệm / công nghệ

Docker
    ↑
Platform / tooling để làm việc với container
```

Docker cung cấp các công cụ để:

- Build image
- Pull image
- Push image
- Create container
- Run container
- Stop container
- Remove container
- Manage container network
- Manage container storage

Ví dụ:

```bash
docker run nginx
```

Docker sẽ thực hiện các công việc cần thiết để tạo và chạy container từ image Nginx.

---

# 5. Docker Image và Docker Container

Có thể hình dung:

```text
Docker Image
     │
     │ docker run
     ▼
Docker Container
```

Ví dụ:

```text
nginx image
     │
     ├── Container 1
     ├── Container 2
     └── Container 3
```

Một image có thể được sử dụng để tạo nhiều container.

---

# 6. Docker Engine là gì?

Docker Engine là thành phần thực hiện việc quản lý và chạy container.

Mô hình đơn giản:

```text
docker CLI
    │
    ▼
Docker Engine
    │
    ├── Images
    ├── Containers
    ├── Network
    └── Storage
```

Khi chạy:

```bash
docker run nginx
```

Có thể hình dung:

```text
docker CLI
    │
    │ request
    ▼
Docker Engine
    │
    ├── Pull image nếu cần
    ├── Create container
    ├── Configure network
    └── Start container
```

---

# 7. Podman là gì?

**Podman cũng là một container engine/tooling**, được sử dụng để tạo và chạy container.

Podman có thể thực hiện các thao tác tương tự Docker:

```text
Pull image
Run container
Stop container
Remove container
Inspect container
```

Ví dụ Docker:

```bash
docker run nginx
```

Tương ứng với Podman:

```bash
podman run nginx
```

Một số lệnh rất tương đồng:

| Docker | Podman |
|---|---|
| `docker ps` | `podman ps` |
| `docker images` | `podman images` |
| `docker pull` | `podman pull` |
| `docker run` | `podman run` |
| `docker stop` | `podman stop` |
| `docker rm` | `podman rm` |
| `docker inspect` | `podman inspect` |

---

# 8. Docker và Podman khác nhau như thế nào?

Một điểm khác biệt quan trọng là kiến trúc.

## Docker

Có thể hình dung:

```text
docker CLI
     │
     ▼
Docker daemon
     │
     ▼
Containers
```

Docker sử dụng một daemon trung tâm để quản lý container.

---

## Podman

Podman được thiết kế theo hướng **daemonless**:

```text
podman CLI
     │
     ▼
Containers
```

Không cần một daemon trung tâm tương tự Docker daemon để thực hiện các thao tác container thông thường.

---

# 9. Tại sao Ceph lại liên quan đến Container?

Ceph có rất nhiều daemon:

```text
MON
MGR
OSD
MDS
RGW
...
```

Với phương thức triển khai bằng `cephadm`, các Ceph daemon được triển khai dưới dạng container.

Mô hình:

```text
Ubuntu Host
     │
     └── Container Runtime
            │
            ├── MON container
            ├── MGR container
            ├── OSD container
            └── ...
```

Container runtime có thể là:

```text
Podman
hoặc
Docker
```

---

# 10. Cephadm là gì?

**cephadm** là công cụ được Ceph sử dụng để triển khai và quản lý vòng đời của Ceph cluster.

Cephadm không phải là Ceph storage engine.

Nó chịu trách nhiệm cho những công việc như:

- Bootstrap Ceph cluster
- Deploy MON
- Deploy MGR
- Deploy OSD
- Add host
- Remove host
- Start/stop/redeploy daemon
- Quản lý lifecycle của Ceph daemon
- Điều phối các service trong cluster

Có thể hiểu:

```text
Ceph
 │
 ├── Storage system
 │
 └── Ceph daemons

cephadm
 │
 └── Deploy / Manage Ceph daemons
```

---

# 11. Cephadm có phải Container Runtime không?

**Không.**

Đây là điểm cần phân biệt rõ:

```text
Container
    ↑
Khái niệm / công nghệ

Docker / Podman
    ↑
Container engine / runtime

cephadm
    ↑
Ceph deployment & lifecycle management

MON / MGR / OSD
    ↑
Ceph daemons
```

---

# 12. Quan hệ giữa cephadm và Podman

Đây là mô hình quan trọng nhất trong lab:

```text
                    cephadm
                       │
                       │ deploy/manage
                       ▼
                Podman / Docker
                       │
              ┌────────┼────────┐
              ▼        ▼        ▼
             MON      MGR      OSD
              │        │        │
              └────────┼────────┘
                       ▼
                  Ceph Cluster
```

Ví dụ khi chạy:

```bash
cephadm bootstrap --mon-ip 10.168.36.32
```

cephadm sẽ thực hiện quá trình bootstrap cluster và sử dụng container runtime để chạy các Ceph daemon.

---
# 13. Ceph Daemon là gì?

**Daemon** là một process/service chạy nền để thực hiện một chức năng cụ thể.

Ceph có nhiều loại daemon.

Trong lab cơ bản chúng ta tập trung vào:

```text
MON
MGR
OSD
```

Ngoài ra Ceph còn có:

```text
MDS
RGW
```

tùy vào dịch vụ cần triển khai.

---

# 14. MON là gì?

**MON = Monitor**

MON quản lý thông tin quan trọng về trạng thái của Ceph cluster.

MON chịu trách nhiệm duy trì các cluster maps và tham gia vào cơ chế quorum.

Có thể hình dung:

```text
              Ceph Cluster
                   │
          ┌────────┼────────┐
          ▼        ▼        ▼
        MON.1    MON.2    MON.3
          │        │        │
          └────────┼────────┘
                   │
                 Quorum
```

Trong lab 3 node:

```text
Ceph-node-1 → MON
Ceph-node-2 → MON
Ceph-node-3 → MON
```

Mục tiêu là có quorum khi một MON gặp sự cố.

---

# 15. MGR là gì?

**MGR = Manager**

MGR cung cấp các chức năng quản lý và monitoring của Ceph.

Ví dụ:

- Monitoring
- Dashboard
- Cluster statistics
- Một số management modules
- Orchestrator integration

Mô hình:

```text
Ceph Cluster
      │
      ▼
     MGR
      │
 ┌────┼─────┐
 ▼    ▼     ▼
CLI Dashboard Metrics
```

Trong lab có thể triển khai MGR trên cả 3 node, nhưng tại một thời điểm sẽ có một MGR active và các MGR khác có thể ở trạng thái standby.

---

# 16. OSD là gì?

**OSD = Object Storage Daemon**

OSD là thành phần trực tiếp quản lý dữ liệu storage trong Ceph.

Đây là daemon đặc biệt quan trọng.

Ví dụ:

```text
100GB disk
    │
    ▼
  OSD
```

Trong lab của chúng ta:

```text
Ceph-node-1
 ├── 100G → OSD.0
 ├── 100G → OSD.1
 └── 100G → OSD.2

Ceph-node-2
 ├── 100G → OSD.3
 ├── 100G → OSD.4
 └── 100G → OSD.5

Ceph-node-3
 ├── 100G → OSD.6
 ├── 100G → OSD.7
 └── 100G → OSD.8
```

Tổng:

```text
9 OSD
900GB RAW
```

---

# 17. OSD Container và Disk

Một OSD được chạy dưới dạng container khi triển khai bằng cephadm.

Có thể hình dung:

```text
               Podman
                  │
            OSD container
                  │
                  │ access
                  ▼
              /dev/sdb
                  │
                  ▼
               100GB
```

Container không có nghĩa là disk bị nhốt bên trong container.

OSD container được cấp quyền truy cập vào storage device mà nó quản lý.

---

# 18. Toàn bộ quan hệ trong một Ceph Node

Ví dụ Ceph-node-1:

```text
Ceph-node-1
10.168.36.32
     │
     ├── Ubuntu 24.04
     │
     ├── cephadm
     │
     └── Podman
           │
           ├── MON container
           │
           ├── MGR container
           │
           ├── OSD.0 container
           │       │
           │       └── 100GB disk
           │
           ├── OSD.1 container
           │       │
           │       └── 100GB disk
           │
           └── OSD.2 container
                   │
                   └── 100GB disk
```

---

# 19. Toàn bộ Ceph Cluster

Lab của chúng ta:

```text
                         Ceph Cluster
                              │
                           cephadm
                              │
                     Podman / Container
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
          ▼                   ▼                   ▼
   Ceph-node-1         Ceph-node-2         Ceph-node-3
   10.168.36.32        10.168.36.33        10.168.36.34
          │                   │                   │
     MON / MGR            MON / MGR           MON / MGR
          │                   │                   │
     OSD.0-2              OSD.3-5             OSD.6-8
          │                   │                   │
     3 × 100GB             3 × 100GB           3 × 100GB
```

Tổng:

```text
3 Ceph nodes
9 OSD
900GB RAW
```

---

# 20. Liên hệ với OpenStack

OpenStack và Ceph có vai trò khác nhau.

## OpenStack

OpenStack là Cloud Infrastructure Platform.

Ví dụ:

```text
OpenStack
 ├── Nova
 ├── Neutron
 ├── Cinder
 ├── Glance
 ├── Keystone
 └── Horizon
```

## Ceph

Ceph cung cấp distributed storage.

```text
Ceph
 ├── RBD
 ├── CephFS
 ├── RGW
 └── RADOS
```

Khi tích hợp:

```text
                 OpenStack
          ┌──────────┼──────────┐
          │          │          │
        Nova       Cinder     Glance
          │          │          │
          └──────────┼──────────┘
                     │
                     ▼
                  Ceph RBD
                     │
                   RADOS
                     │
                Pool / PG
                     │
                  CRUSH
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
        OSD.1      OSD.4      OSD.7
          │          │          │
        Disk       Disk       Disk
```

---


