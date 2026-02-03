# CAN Frame - Cấu trúc khung truyền CAN Bus

## Tổng quan

Mỗi lần một node (ví dụ BMS1) gửi dữ liệu lên CAN Bus, nó gửi một **CAN Frame** (khung truyền). Frame này không chỉ chứa data mà còn chứa các trường phụ trợ để đảm bảo truyền đúng, phát hiện lỗi và phân xử quyền gửi.

## Cấu trúc CAN Frame (Standard - CAN 2.0A)

Hệ thống BMS trong file `Pack_DBC_V0.1.dbc` sử dụng **Standard CAN** (11-bit ID).

```
┌─────┬──────────────┬─────────┬───────────────────────┬──────────┬─────┬─────┬───────┐
│ SOF │  Arbitration │ Control │      Data Field       │   CRC    │ ACK │ ACK │  EOF  │
│     │    Field     │  Field  │                       │  Field   │Slot │ Del │       │
│1 bit│   12 bit     │  6 bit  │    0 ~ 64 bit         │  16 bit  │1 bit│1 bit│ 7 bit │
└─────┴──────────────┴─────────┴───────────────────────┴──────────┴─────┴─────┴───────┘
```

Tổng cộng: **44 + (0~64) bit** = tối thiểu 44 bit, tối đa 108 bit (chưa tính bit stuffing).

---

## Chi tiết từng trường

### 1. SOF - Start of Frame (1 bit)

```
│ 0 │
```

- Luôn là bit **dominant (0)**
- Báo hiệu bắt đầu một frame mới
- Tất cả các node trên bus đang ở trạng thái nghỉ (idle = recessive = 1), khi thấy SOF = 0 thì biết có frame đang được gửi

### 2. Arbitration Field - Trường phân xử (12 bit)

```
┌─────────────────────────┬─────┐
│      CAN ID (11 bit)    │ RTR │
└─────────────────────────┴─────┘
```

#### CAN ID (11 bit)

- Xác định **loại message** (không phải địa chỉ node)
- Giá trị: 0x000 ~ 0x7FF (0 ~ 2047)
- **ID càng nhỏ = ưu tiên càng cao** (quan trọng cho cơ chế arbitration)

Ví dụ trong file DBC:
```
BMS1_ControlSystem:  ID = 0x300 = 0b011_0000_0000
BMS2_ControlSystem:  ID = 0x330 = 0b011_0011_0000
BMS1_ISO_Message:    ID = 0x002 = 0b000_0000_0010  ← ưu tiên cao nhất
```

#### RTR - Remote Transmission Request (1 bit)

| Giá trị | Ý nghĩa |
|:---:|---|
| 0 (dominant) | **Data Frame** - frame chứa dữ liệu (dùng trong 99% trường hợp) |
| 1 (recessive) | **Remote Frame** - frame yêu cầu node khác gửi data (hiếm khi dùng) |

Trong hệ thống BMS, tất cả đều là **Data Frame** (RTR = 0).

### 3. Control Field - Trường điều khiển (6 bit)

```
┌─────┬─────┬──────────────────┐
│ IDE │  r  │   DLC (4 bit)    │
└─────┴─────┴──────────────────┘
```

#### IDE - Identifier Extension (1 bit)

| Giá trị | Ý nghĩa |
|:---:|---|
| 0 (dominant) | **Standard Frame** - CAN ID 11 bit |
| 1 (recessive) | **Extended Frame** - CAN ID 29 bit |

Hệ thống BMS dùng **Standard Frame** (IDE = 0).

#### r - Reserved (1 bit)

- Bit dự phòng, luôn = 0

#### DLC - Data Length Code (4 bit)

Số byte data trong frame:

| DLC | Số byte data |
|:---:|:---:|
| 0000 | 0 |
| 0001 | 1 |
| ... | ... |
| 1000 | 8 (tối đa cho CAN 2.0) |

Ví dụ từ file DBC:
```
BMS1_InfoPack:    DLC = 8  (0b1000)  ← 8 byte data
BMS1_InfoDemBMS:  DLC = 4  (0b0100)  ← 4 byte data
BMS1_InfoVoltageCell4: DLC = 2  (0b0010)  ← 2 byte data
```

### 4. Data Field - Trường dữ liệu (0 ~ 64 bit)

```
┌────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┐
│ Byte 0 │ Byte 1 │ Byte 2 │ Byte 3 │ Byte 4 │ Byte 5 │ Byte 6 │ Byte 7 │
└────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┘
  0 ~ 8 byte, số byte thực tế = DLC
```

Đây là phần chứa **dữ liệu thực tế** mà file DBC mô tả cách giải mã.

**Ví dụ cụ thể:** BMS1_InfoPack (ID = 0x309, DLC = 8)

```
Byte:    [  0  ] [  1  ] [  2  ] [  3  ] [  4  ] [  5  ] [  6  ] [  7  ]
Bit:     7    0  15    8  23   16  31   24  39   32  47   40  55   48  63
         ├──┬──┤ ├──┬──┤ ├───────┤       ├───────┤       ├───────┤
         │  │  │ │  │  │ │       │       │       │       │       │
        CFET│ DFET│  │  PackVolt│       PackCurAvg       PackCurrent
            │    │  │  BAT      │
         LowCap  Precharge     │
            Alarm    FET       │
```

Mỗi signal trong DBC mô tả:
- **Bit position:** bắt đầu ở bit nào
- **Length:** dài bao nhiêu bit
- **Byte order:** Big Endian (Motorola) hay Little Endian (Intel)
- **Factor + Offset:** công thức chuyển đổi giá trị

### 5. CRC Field - Kiểm tra lỗi (16 bit)

```
┌──────────────────────┬───────┐
│  CRC Sequence (15 bit)│CRC Del│
└──────────────────────┴───────┘
```

- **CRC Sequence (15 bit):** Mã kiểm tra lỗi, tính từ SOF đến hết Data Field
- **CRC Delimiter (1 bit):** Luôn = 1 (recessive), đánh dấu kết thúc CRC

Node gửi tính CRC và đặt vào frame. Node nhận tính lại CRC từ data nhận được rồi so sánh - nếu khác nhau thì frame bị lỗi.

### 6. ACK Field - Xác nhận (2 bit)

```
┌──────────┬──────────┐
│ ACK Slot │ ACK Del  │
│  (1 bit) │ (1 bit)  │
└──────────┴──────────┘
```

- **ACK Slot:** Node gửi đặt = 1 (recessive). Nếu có **bất kỳ node nào** nhận đúng (CRC khớp), node đó sẽ **ghi đè thành 0** (dominant)
- **ACK Delimiter:** Luôn = 1

```
Node gửi (BMS1):  gửi ACK Slot = 1
Node nhận (DC_DC): nhận CRC đúng → ghi đè ACK Slot = 0
                                      ↓
                   BMS1 đọc lại bus, thấy 0 → "OK, có node nhận được"
                   BMS1 đọc lại bus, thấy 1 → "Không ai nhận!" → Error
```

### 7. EOF - End of Frame (7 bit)

```
│ 1 │ 1 │ 1 │ 1 │ 1 │ 1 │ 1 │
```

- 7 bit **recessive (1)** liên tiếp
- Đánh dấu kết thúc frame
- Sau EOF là 3 bit **IFS** (Inter Frame Space) trước khi frame tiếp theo được gửi

---

## Ví dụ thực tế: BMS1 gửi InfoPack

BMS1 gửi điện áp Pack = 51.2V, dòng điện = 10A, CFET = On, DFET = On:

```
Giá trị thực → Raw:
  PackVoltageBAT: 51.2 / 0.002 = 25600 = 0x6400
  PackCurrent:    10.0 / 0.02  = 500   = 0x01F4
  CFET: 1, DFET: 1

CAN Frame trên bus:
┌─────┬─────────────┬─────┬─────┬─────┬──────────────────────────────────────┬─────────┬─────┬─────────┐
│ SOF │  ID = 0x309 │ RTR │ IDE │ DLC │           Data (8 bytes)             │   CRC   │ ACK │   EOF   │
│  0  │ 01100001001 │  0  │  0  │1000 │ 11 00 64 00 00 00 01 F4             │ (15bit) │ 0/1 │ 1111111 │
└─────┴─────────────┴─────┴─────┴─────┴──────────────────────────────────────┴─────────┴─────┴─────────┘
                                        │  │  │     │           │
                                        │  │  └──┬──┘           └── PackCurrent = 0x01F4
                                        │  │     └── PackVoltageBAT = 0x6400
                                        │  └── [LowCapAlarm=0][PrechargeFET=0]
                                        └── [CFET=1][DFET=1]
```

---

## Cơ chế Arbitration (Phân xử)

Khi 2 node gửi cùng lúc, CAN Bus dùng **bit-level arbitration** dựa trên CAN ID:

```
Thời điểm:    SOF │ ID bit 10 │ ID bit 9 │ ID bit 8 │ ...
              ────┼───────────┼──────────┼──────────┼────
BMS1 (0x300): │ 0 │     0     │    1     │    1     │ ...  0b011_0000_0000
BMS2 (0x330): │ 0 │     0     │    1     │    1     │ ...  0b011_0011_0000
                                                      ↑
                                              Giống nhau đến đây
              ────┼───────────┼──────────┼──────────┼────
              │ ID bit 5 │ ID bit 4 │
              ┼──────────┼──────────┤
BMS1 (0x300): │    0     │    0     │ ← dominant (0) → THẮNG, tiếp tục gửi
BMS2 (0x330): │    1     │    ...   │ ← recessive (1) → THUA, dừng gửi
              ┼──────────┼──────────┤
Bus thực tế:  │    0     │          │ ← dominant thắng recessive
```

**Nguyên tắc:** Dominant (0) luôn thắng Recessive (1). Node có **ID nhỏ hơn** sẽ có nhiều bit dominant hơn → thắng arbitration → được gửi trước.

Trong hệ thống BMS:
- **0x002** (BMS1 ISO) = ưu tiên cao nhất
- **0x003** (BMS2 ISO) = ưu tiên thứ hai
- **0x300** (BMS1 ControlSystem) thắng **0x330** (BMS2 ControlSystem)
- Message nào thua arbitration sẽ **tự động thử lại** ngay lập tức, không mất data

---

## Các loại Error trong CAN

CAN Bus có 5 cơ chế phát hiện lỗi:

| Loại lỗi | Phát hiện bởi | Mô tả |
|---|---|---|
| **Bit Error** | Node gửi | Gửi bit 0 nhưng đọc lại trên bus thấy bit 1 (hoặc ngược lại, ngoài vùng arbitration) |
| **Stuff Error** | Node nhận | Sau 5 bit giống nhau liên tiếp mà không thấy stuff bit đối nghịch |
| **CRC Error** | Node nhận | CRC tính lại không khớp với CRC trong frame |
| **Form Error** | Node nhận | Các bit cố định (CRC Del, ACK Del, EOF) không đúng giá trị quy định |
| **ACK Error** | Node gửi | Không có node nào xác nhận (ACK Slot vẫn = 1) |

Khi phát hiện lỗi, node sẽ gửi **Error Frame** (6 bit dominant liên tiếp) để thông báo cho toàn bộ bus biết frame đang truyền bị lỗi. Node gửi sẽ tự động **truyền lại** frame đó.

---

## Bit Stuffing

CAN Bus sử dụng **bit stuffing** để đồng bộ clock giữa các node. Quy tắc:

> Sau **5 bit giống nhau liên tiếp**, tự động chèn thêm **1 bit đối nghịch**.

```
Dữ liệu gốc:     1 1 1 1 1 0 0 0 0 0 1 1
                            ↓               ↓
Sau bit stuffing:  1 1 1 1 1 [0] 0 0 0 0 0 [1] 1 1
                             ↑               ↑
                          stuff bit        stuff bit
```

Bit stuffing áp dụng từ SOF đến CRC (không áp dụng cho ACK và EOF). Đây là lý do frame thực tế trên bus có thể dài hơn tính toán lý thuyết.

---

## So sánh Standard vs Extended Frame

| Đặc điểm | Standard (CAN 2.0A) | Extended (CAN 2.0B) |
|---|---|---|
| Độ dài ID | 11 bit | 29 bit |
| Số ID tối đa | 2,048 | 536,870,912 |
| IDE bit | 0 (dominant) | 1 (recessive) |
| Ứng dụng | Phổ biến trong ô tô | J1939 (xe tải), CANopen |

File DBC `Pack_DBC_V0.1.dbc` sử dụng **Standard Frame** (11-bit ID). Tất cả CAN ID đều nằm trong khoảng 0x002 ~ 0x35F, vừa đủ trong 11 bit.

---

## Tóm tắt

Một CAN frame hoàn chỉnh khi BMS1 gửi InfoPack (0x309):

```
[SOF] [  CAN ID  ] [RTR] [IDE] [r] [ DLC ] [    8 byte DATA    ] [   CRC   ] [ACK] [  EOF  ]
  0    01100001001    0     0    0    1000    11 00 64 00 ...       (15 bit)    0/1   1111111
  │         │         │     │    │     │            │                  │         │       │
  │    Message ID     │  Standard│  8 bytes    Dữ liệu thực      Kiểm tra   Xác nhận  Kết
  │    = 0x309        │  Frame   │             (điện áp,          lỗi        nhận      thúc
  │                   │          │              dòng, FET...)
  Bắt đầu         Data Frame  Dự phòng
  frame
```
