# Hướng dẫn cấu hình lab OSPF đa vùng (R1–R2–R3–R4)

## 0. Phân tích sơ đồ

| Vùng OSPF | Thiết bị / Interface | Mạng |
|---|---|---|
| **Area 0** | R1(Fa0/0), R2(Fa0/0), R3(Fa0/0) — qua switch "mang" | 192.168.123.0/24 |
| **Area 0** | R1(Se0/2/0) — R3(Se0/2/0) | 192.168.13.0/24 |
| **Area 0** | R1(Fa1/0) — PC1 | 172.16.1.0/24 |
| **Area 0** | R3(Fa0/1) — PC3 | 172.16.3.0/24 |
| **Area 1** | R1(Fa0/1) — R4(Fa0/0) | 192.168.14.0/24 |
| **Area 1** | R4 Loopback0 | 8.8.8.8 |
| **Area 2** | R2(Fa0/1) — PC2 | 172.16.2.0/24 |

→ **R1** là ABR giữa Area 0 và Area 1. **R2** là ABR giữa Area 0 và Area 2. **R3, R4** chỉ thuộc 1 vùng.

---

## 1. Kết nối dây & đặt IP

### Loại cáp cần dùng trong Packet Tracer
- PC ↔ Router (VPC1–R1, VPC2–R2, VPC3–R3): dùng **Automatic** hoặc **Copper Cross-Over**.
- Router ↔ Switch (R1/R2/R3 ↔ switch "mang"): dùng **Copper Straight-Through** (hoặc Automatic).
- Router ↔ Router qua Ethernet (R1 Fa0/1 ↔ R4 Fa0/0): dùng **Automatic** hoặc **Copper Cross-Over**.
- Router ↔ Router qua Serial (R1 Se0/2/0 ↔ R3 Se0/2/0): dùng **Serial DCE**. Đầu nào có biểu tượng đồng hồ (⏰) là DCE, cần cấu hình `clock rate`.

### R1 (đã cấu hình đúng theo log của bạn — giữ nguyên)
```
hostname R1
!
interface FastEthernet0/0
 ip address 192.168.123.1 255.255.255.0
 no shutdown
!
interface FastEthernet0/1
 ip address 192.168.14.1 255.255.255.0
 no shutdown
!
interface FastEthernet1/0
 ip address 172.16.1.1 255.255.255.0
 speed auto
 duplex auto
 no shutdown
!
interface Serial0/2/0
 ip address 192.168.13.1 255.255.255.0
 no shutdown
```

### R2
```
enable
configure terminal
hostname R2
!
interface FastEthernet0/0
 ip address 192.168.123.2 255.255.255.0
 no shutdown
!
interface FastEthernet0/1
 ip address 172.16.2.1 255.255.255.0
 no shutdown
```

### R3
```
enable
configure terminal
hostname R3
!
interface FastEthernet0/0
 ip address 192.168.123.3 255.255.255.0
 no shutdown
!
interface FastEthernet0/1
 ip address 172.16.3.1 255.255.255.0
 no shutdown
!
interface Serial0/2/0
 ip address 192.168.13.3 255.255.255.0
 no shutdown
 ! Nếu đầu này là DCE thì thêm: clock rate 64000
```

### R4
```
enable
configure terminal
hostname R4
!
interface FastEthernet0/0
 ip address 192.168.14.4 255.255.255.0
 no shutdown
!
interface Loopback0
 ip address 8.8.8.8 255.255.255.0
```

### Switch "mang" (2960-24TT)
Không cần IP để chạy OSPF, chỉ cần các cổng nối router ở trạng thái up (mặc định đã up nếu cắm cáp đúng):
```
enable
configure terminal
interface range fastEthernet0/1-3
 no shutdown
```

### Đặt IP cho PC (VPC1/2/3) — vào tab Desktop > IP Configuration
| PC | IP | Subnet Mask | Gateway |
|---|---|---|---|
| VPC1 | 172.16.1.2 | 255.255.255.0 | 172.16.1.1 |
| VPC2 | 172.16.2.2 | 255.255.255.0 | 172.16.2.1 |
| VPC3 | 172.16.3.2 | 255.255.255.0 | 172.16.3.1 |

---

## 2. Cấu hình OSPF + Router-ID (yêu cầu 2 & 3)

### R1
```
router ospf 1
 router-id 0.0.0.1
 network 172.16.1.0 0.0.0.255 area 0
 network 192.168.123.0 0.0.0.255 area 0
 network 192.168.13.0 0.0.0.255 area 0
 network 192.168.14.0 0.0.0.255 area 1
```

### R2
```
router ospf 1
 router-id 0.0.0.2
 network 192.168.123.0 0.0.0.255 area 0
 network 172.16.2.0 0.0.0.255 area 2
```

### R3
```
router ospf 1
 router-id 0.0.0.3
 network 192.168.123.0 0.0.0.255 area 0
 network 192.168.13.0 0.0.0.255 area 0
 network 172.16.3.0 0.0.0.255 area 0
```

### R4
```
router ospf 1
 router-id 0.0.0.4
 network 192.168.14.0 0.0.0.255 area 1
 network 8.8.8.8 0.0.0.0 area 1
```

> Sau khi gõ `router-id`, nếu OSPF đã chạy trước đó, chạy thêm `end` rồi `clear ip ospf process` (gõ `yes` khi được hỏi) để router-id có hiệu lực ngay.

---

## 3. Bầu chọn DR/BDR (yêu cầu 4)

### Đoạn multi-access R1–R2–R3 (qua switch, mạng 192.168.123.0/24)
Mục tiêu: **R2 = DR, R1 = BDR, R3 = DROTHER** → chỉnh priority trên **Fa0/0** của cả 3 router:

```
! Trên R2
interface FastEthernet0/0
 ip ospf priority 200

! Trên R1
interface FastEthernet0/0
 ip ospf priority 100

! Trên R3
interface FastEthernet0/0
 ip ospf priority 0
```

### Đoạn R1–R4 (mạng 192.168.14.0/24)
Mục tiêu: **R1 luôn là DR** → cách chắc chắn nhất là loại R4 khỏi cuộc bầu chọn:

```
! Trên R1
interface FastEthernet0/1
 ip ospf priority 200

! Trên R4
interface FastEthernet0/0
 ip ospf priority 0
```

> Priority = 0 nghĩa là router đó **không bao giờ** trở thành DR/BDR, dù có reload hay adjacency reset thế nào.

### Áp dụng lại bầu chọn
Priority chỉ có tác dụng khi DR/BDR **chưa được bầu** hoặc bầu lại từ đầu. Sau khi chỉnh priority, ép bầu lại bằng cách reset OSPF trên các router liên quan:
```
clear ip ospf process
```
(gõ `yes` để xác nhận) — thực hiện trên R1, R2, R3 (cho đoạn switch) và R1, R4 (cho đoạn R1–R4).

Kiểm tra kết quả:
```
show ip ospf neighbor
show ip ospf interface fastEthernet0/0
```

---

## 4. Điều chỉnh cost — Serial là đường chính, Ethernet dự phòng (yêu cầu 5)

Mặc định: FastEthernet cost = 1, Serial cost = 64 → đường Ethernet (qua switch) sẽ luôn được chọn, **ngược** với yêu cầu. Cần hạ cost Serial xuống thấp hơn cost Ethernet trên cặp cổng R1–R3:

```
! Trên R1
interface Serial0/2/0
 ip ospf cost 5
interface FastEthernet0/0
 ip ospf cost 50

! Trên R3
interface Serial0/2/0
 ip ospf cost 5
interface FastEthernet0/0
 ip ospf cost 50
```

Kết quả: R3 đến Loopback0 (8.8.8.8) của R4 sẽ đi theo đường **R3 → Se0/2/0 → R1 → Fa0/1 → R4** (cost thấp) là đường chính. Nếu Serial đứt, route tự động chuyển sang **R3 → Fa0/0 → switch → R1 → Fa0/1 → R4** làm đường dự phòng.

Kiểm tra:
```
show ip route ospf
show ip ospf interface brief
traceroute 8.8.8.8
```
(chạy `traceroute` từ R3 để xác nhận đi qua Serial trước, sau đó thử `shutdown` Se0/2/0 trên R1 hoặc R3 để kiểm tra chuyển sang đường Ethernet)

---

## 5. Kiểm tra tổng thể sau khi cấu hình xong

```
show ip ospf neighbor        ! kiểm tra láng giềng + vai trò DR/BDR/DROTHER
show ip protocols            ! kiểm tra router-id, area
show ip route                ! xem toàn bộ route đã học được
ping 8.8.8.8                 ! từ R2, R3, PC1, PC2, PC3 — phải ping thấy loopback R4
```

Nếu tất cả các PC ping được lẫn nhau và ping được 8.8.8.8, nghĩa là mọi địa chỉ trên sơ đồ đã được OSPF quảng bá đầy đủ (đúng yêu cầu 2).
