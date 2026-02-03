# ISO 15765 - Giao thức truyền dữ liệu trên CAN (ISO-TP)

## Tổng quan

**ISO 15765** (còn gọi là **ISO-TP**, viết tắt của ISO Transport Protocol) là một chuẩn quốc tế quy định cách **truyền các gói dữ liệu lớn** trên mạng CAN Bus.

### Tại sao cần ISO 15765?

CAN Bus tiêu chuẩn (CAN 2.0) có một giới hạn quan trọng: **mỗi CAN frame chỉ chứa được tối đa 8 byte dữ liệu**. Điều này đủ cho các tín hiệu đơn giản (điện áp, nhiệt độ, trạng thái...), nhưng **không đủ** khi cần truyền lượng dữ liệu lớn hơn, ví dụ:

- Chẩn đoán xe (Diagnostics) - đọc mã lỗi DTC, đọc thông tin ECU
- Cập nhật firmware (Flash Programming)
- Đọc/ghi dữ liệu cấu hình dài

ISO 15765 giải quyết vấn đề này bằng cách **chia nhỏ dữ liệu lớn thành nhiều CAN frame** rồi ghép lại ở phía nhận.

## Cấu trúc ISO 15765

### Các phần của tiêu chuẩn

| Phần | Tên | Mô tả |
|:---:|---|---|
| ISO 15765-1 | General Information | Thông tin chung và tổng quan |
| ISO 15765-2 | Transport Protocol | **Giao thức truyền tải** - phần cốt lõi, quy định cách chia nhỏ và ghép dữ liệu |
| ISO 15765-3 | UDS on CAN | Chẩn đoán hợp nhất (UDS) trên CAN |
| ISO 15765-4 | OBD on CAN | Chẩn đoán on-board (OBD) trên CAN |

Trong file DBC `Pack_DBC_V0.1.dbc`, các message ISO 15765 là:
- **BMS1_ISO_Message** (CAN ID: 0x002)
- **BMS2_ISO_Message** (CAN ID: 0x003)

Đây là các kênh dùng cho giao tiếp chẩn đoán/cấu hình giữa BMS và thiết bị bên ngoài (ví dụ: công cụ chẩn đoán, máy tính).

## ISO 15765-2: Transport Protocol (Chi tiết)

### 4 loại Frame

ISO 15765-2 định nghĩa **4 loại frame** để thực hiện việc chia nhỏ và truyền dữ liệu:

#### 1. Single Frame (SF) - Frame đơn

Dùng khi dữ liệu **nhỏ hơn hoặc bằng 7 byte** (vừa đủ trong 1 CAN frame).

```
┌──────────┬─────────────────────────────┐
│ Byte 0   │ Byte 1 ... Byte 7           │
│ PCI: 0x0N│ Dữ liệu (N byte, N ≤ 7)    │
└──────────┴─────────────────────────────┘
  N = số byte dữ liệu
```

**Ví dụ:** Gửi 3 byte dữ liệu `[0x22, 0xF1, 0x90]`
```
CAN Data: [03] [22] [F1] [90] [00] [00] [00] [00]
           │    └─── 3 byte dữ liệu ───┘
           └── SF, length = 3
```

#### 2. First Frame (FF) - Frame đầu tiên

Dùng khi dữ liệu **lớn hơn 7 byte**. Frame này báo hiệu bắt đầu một chuỗi truyền và chứa **tổng độ dài dữ liệu**.

```
┌──────────┬──────────┬──────────────────┐
│ Byte 0   │ Byte 1   │ Byte 2 ... 7     │
│ 0x1H     │ HL       │ 6 byte dữ liệu   │
└──────────┴──────────┴──────────────────┘
  H:HL = tổng số byte dữ liệu (12 bit, tối đa 4095)
```

#### 3. Consecutive Frame (CF) - Frame tiếp theo

Các frame tiếp theo mang phần dữ liệu còn lại, được đánh **số thứ tự tuần hoàn** từ 1 đến F (0x1 → 0xF → 0x0 → 0x1...).

```
┌──────────┬──────────────────────────────┐
│ Byte 0   │ Byte 1 ... Byte 7            │
│ 0x2N     │ 7 byte dữ liệu tiếp theo     │
└──────────┴──────────────────────────────┘
  N = Sequence Number (0x0 ~ 0xF, tuần hoàn)
```

#### 4. Flow Control (FC) - Frame điều khiển luồng

Phía nhận gửi lại frame này để **thông báo cho phía gửi biết** tốc độ và cách gửi các Consecutive Frame.

```
┌──────────┬──────────┬──────────┬────────┐
│ Byte 0   │ Byte 1   │ Byte 2   │ ...    │
│ 0x3S     │ BS       │ STmin    │ Padding│
└──────────┴──────────┴──────────┴────────┘
  S     = Flow Status (0=Continue, 1=Wait, 2=Overflow)
  BS    = Block Size (số CF gửi trước khi chờ FC tiếp)
  STmin = Separation Time minimum (thời gian tối thiểu giữa 2 CF)
```

## Ví dụ truyền 20 byte dữ liệu

Giả sử BMS cần gửi 20 byte dữ liệu `[D0 D1 D2 ... D19]` qua ISO-TP:

```
Bước 1: BMS gửi First Frame
┌─────────────────────────────────────────────────┐
│ [10 14] [D0] [D1] [D2] [D3] [D4] [D5]          │
│  FF, length=20  ── 6 byte dữ liệu đầu tiên     │
└─────────────────────────────────────────────────┘

Bước 2: DC_DC trả lời Flow Control
┌─────────────────────────────────────────────────┐
│ [30] [00] [0A] [00] [00] [00] [00] [00]          │
│  FC   BS=0  STmin=10ms                           │
│  (Continue, không giới hạn block, tối thiểu 10ms)│
└─────────────────────────────────────────────────┘

Bước 3: BMS gửi Consecutive Frame #1
┌─────────────────────────────────────────────────┐
│ [21] [D6] [D7] [D8] [D9] [D10] [D11] [D12]      │
│  CF, seq=1 ── 7 byte dữ liệu tiếp theo          │
└─────────────────────────────────────────────────┘

Bước 4: BMS gửi Consecutive Frame #2
┌─────────────────────────────────────────────────┐
│ [22] [D13] [D14] [D15] [D16] [D17] [D18] [D19]  │
│  CF, seq=2 ── 7 byte dữ liệu cuối cùng          │
└─────────────────────────────────────────────────┘
```

Kết quả: 20 byte dữ liệu được truyền thành công qua **3 CAN frame** (1 FF + 2 CF), với 1 FC để điều khiển luồng.

## Tóm tắt

```
Dữ liệu ≤ 7 byte  →  Single Frame (1 CAN frame)
Dữ liệu > 7 byte  →  First Frame + Flow Control + Consecutive Frames
```

| Đặc điểm | Giá trị |
|---|---|
| Dữ liệu tối đa (12-bit length) | 4095 byte |
| Dữ liệu tối đa (32-bit length, extended) | 4 GB |
| Tốc độ phụ thuộc | CAN Bus baudrate và STmin |
| Ứng dụng phổ biến | UDS Diagnostics, Flash Programming, DTC Reading |

## Trong hệ thống Pack_DBC_V0.1

Hai message ISO 15765 trong file DBC:

| Message | CAN ID | Node | Mục đích |
|---|:---:|:---:|---|
| BMS1_ISO_Message | 0x002 | BMS1 | Kênh ISO-TP cho chẩn đoán/cấu hình Pack 1 |
| BMS2_ISO_Message | 0x003 | BMS2 | Kênh ISO-TP cho chẩn đoán/cấu hình Pack 2 |

Cả hai message đều có **DLC = 8 byte** và chứa **1 signal 64-bit** (`bms1_iso_signal` / `bms2_iso_signal`) bao trọn toàn bộ 8 byte data. Dữ liệu thực tế bên trong sẽ được phân tích theo cấu trúc 4 loại frame (SF/FF/CF/FC) như mô tả ở trên.

Các message này **không có chu kỳ gửi cố định** (không có GenMsgCycleTime) vì chúng chỉ được gửi **khi có yêu cầu** (on-demand), ví dụ khi kết nối công cụ chẩn đoán vào hệ thống.
