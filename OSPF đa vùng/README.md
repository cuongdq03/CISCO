# Lab OSPF đa vùng (Multi-Area OSPF) — Packet Tracer

Lab cấu hình OSPF đa vùng trên 4 router (R1–R4), 1 switch Layer 2 và 3 PC, gồm bầu chọn DR/BDR theo yêu cầu và điều chỉnh cost để định tuyến theo đường ưu tiên.

## 📌 Yêu cầu bài lab

1. Kết nối dây và đặt IP theo sơ đồ.
2. Cấu hình định tuyến OSPF trên các router, đảm bảo mọi địa chỉ IP trong sơ đồ được tìm thấy (reachable toàn mạng).
3. Router-ID: `R1 = 0.0.0.1`, `R2 = 0.0.0.2`, `R3 = 0.0.0.3`, `R4 = 0.0.0.4`.
4. Bầu chọn DR/BDR:
   - Đoạn multi-access R1–R2–R3 (qua switch): **R2 = DR, R1 = BDR, R3 = DROTHER**.
   - Đoạn multi-access R1–R4: **R1 luôn là DR**.
5. Điều chỉnh cost sao cho R3 → Loopback0 của R4 đi theo đường **Serial là chính**, đường **Ethernet (qua switch) là dự phòng**.

## 🗺️ Sơ đồ mạng

![Topology](images/topology.png)

## 📋 Bảng địa chỉ IP

| Thiết bị | Interface | IP Address | Subnet Mask | Vùng OSPF |
|---|---|---|---|---|
| R1 | Fa0/0 | 192.168.123.1 | /24 | Area 0 |
| R1 | Fa0/1 | 192.168.14.1 | /24 | Area 1 |
| R1 | Fa1/0 | 172.16.1.1 | /24 | Area 0 |
| R1 | Se0/2/0 | 192.168.13.1 | /24 | Area 0 |
| R2 | Fa0/0 | 192.168.123.2 | /24 | Area 0 |
| R2 | Fa0/1 | 172.16.2.1 | /24 | Area 2 |
| R3 | Fa0/0 | 192.168.123.3 | /24 | Area 0 |
| R3 | Fa0/1 | 172.16.3.1 | /24 | Area 0 |
| R3 | Se0/2/0 | 192.168.13.3 | /24 | Area 0 |
| R4 | Fa0/0 | 192.168.14.4 | /24 | Area 1 |
| R4 | Lo0 | 8.8.8.8 | /24 | Area 1 |
| PC1 (VPC1) | Fa0 | 172.16.1.2 | /24 | GW: 172.16.1.1 |
| PC2 (VPC2) | Fa0 | 172.16.2.2 | /24 | GW: 172.16.2.1 |
| PC3 (VPC3) | Fa0 | 172.16.3.2 | /24 | GW: 172.16.3.1 |

**Vai trò router:** R1 = ABR (Area 0 ↔ Area 1) · R2 = ABR (Area 0 ↔ Area 2) · R3, R4 = Internal router.

## ⚙️ Cấu hình DR/BDR

| Đoạn mạng | Router | Priority | Vai trò |
|---|---|---|---|
| 192.168.123.0/24 (switch) | R2 (Fa0/0) | 200 | DR |
| | R1 (Fa0/0) | 100 | BDR |
| | R3 (Fa0/0) | 0 | DROTHER |
| 192.168.14.0/24 | R1 (Fa0/1) | 200 | DR |
| | R4 (Fa0/0) | 0 | DROTHER (không bao giờ tranh DR) |

## ⚙️ Cấu hình Cost (đường chính Serial / dự phòng Ethernet)

| Router | Interface | Cost |
|---|---|---|
| R1 | Serial0/2/0 | 5 |
| R1 | FastEthernet0/0 | 50 |
| R3 | Serial0/2/0 | 5 |
| R3 | FastEthernet0/0 | 50 |

→ Từ R3 đến Loopback0 của R4 (8.8.8.8): đường **Serial (R3–R1)** là chính, đường **Ethernet qua switch (R3–switch–R1)** là dự phòng khi Serial đứt.

## 📁 Cấu trúc thư mục

```
lab-ospf-multiarea/
├── README.md
├── images/
│   └── topology.png              # Sơ đồ mạng gốc
└── configs/
    ├── R1-running-config.txt
    ├── R2-running-config.txt
    ├── R3-running-config.txt
    ├── R4-running-config.txt
    └── Switch-running-config.txt
```

## ✅ Kết quả kiểm chứng

- `show ip ospf neighbor` trên R1: xác nhận R2 = DR, R1 = BDR, R3 = DROTHER trên đoạn switch; R4 = DROTHER trên đoạn R1–R4.
- `show ip route ospf` trên cả 4 router: đầy đủ toàn bộ subnet trong sơ đồ (không thiếu route nào).
- `ping`/`traceroute 8.8.8.8` từ R3: đi qua `Serial0/2/0` (192.168.13.1) trong điều kiện bình thường.
- **Kiểm thử failover:** `shutdown` Serial0/2/0 trên R3 → OSPF hội tụ lại, route đến `8.8.8.8` tự động chuyển sang đường Ethernet qua switch (192.168.123.x) → xác nhận cơ chế dự phòng hoạt động đúng.
- Ping toàn mạng giữa PC1 ↔ PC2 ↔ PC3 ↔ Loopback R4: **thành công 100%**.

## 🔧 Cách áp dụng lại cấu hình

Trên mỗi router trong Packet Tracer, vào chế độ cấu hình và dán nội dung file `.txt` tương ứng trong thư mục `configs/` (bỏ qua các dòng bắt đầu bằng `!` vì đó là comment).

## 📝 Ghi chú

- Cáp Router–PC và Router–Router (Ethernet) dùng loại **Automatic / Crossover**; cáp Router–Switch dùng **Straight-through**; liên kết Serial cần xác định đúng đầu **DCE** để cấu hình `clock rate`.
- Có thể cần chạy `clear ip ospf process` sau khi đổi `router-id` hoặc `priority` để ép bầu chọn lại ngay lập tức thay vì chờ timer.
