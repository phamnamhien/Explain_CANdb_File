# Pack_DBC_V0.1 - CAN Database cho Hệ thống Battery Pack

## Tổng quan

File `Pack_DBC_V0.1.dbc` là file CAN Database (định dạng Vector DBC) mô tả giao thức truyền thông CAN Bus cho hệ thống quản lý pin (Battery Management System) của VinFast.

- **Phiên bản:** Vinfast Initial LGC
- **Bus Name:** Battery
- **Bus Type:** CAN
- **Database Name:** EV

> **LGC là gì?** LGC là viết tắt của **LG Chem** (nay là LG Energy Solution) - nhà sản xuất cell pin Lithium-ion cung cấp cho VinFast. Tên "Vinfast Initial LGC" cho biết đây là phiên bản DBC ban đầu dành cho hệ thống pin sử dụng cell LG Chem.

## Các Node trên mạng CAN

| Node | Mô tả |
|------|--------|
| **BMS1** | Battery Management System 1 - Hệ thống quản lý pin cho Pack pin thứ nhất (48V) |
| **BMS2** | Battery Management System 2 - Hệ thống quản lý pin cho Pack pin thứ hai (48V) |
| **DC_DC** | Bộ chuyển đổi DC-DC - nhận dữ liệu từ cả hai BMS |

### Kiến trúc hệ thống

Hệ thống sử dụng **2 Pack pin song song**, mỗi Pack có một BMS riêng biệt:

```
┌──────────┐     CAN Bus      ┌──────────┐
│  BMS1    │◄────────────────►│          │
│ (Pack 1) │                   │  DC_DC   │
│ 13 cells │                   │ Converter│
└──────────┘                   │          │
                               │          │
┌──────────┐                   │          │
│  BMS2    │◄────────────────►│          │
│ (Pack 2) │                   └──────────┘
│ 13 cells │
└──────────┘
```

- **BMS1** và **BMS2** là hai module BMS **ngang hàng**, mỗi module quản lý độc lập một Pack pin gồm **13 cell** mắc nối tiếp.
- Hai BMS đều gửi dữ liệu lên cùng một CAN Bus để DC_DC converter nhận và xử lý.
- Cả hai BMS có **cấu trúc message và signal giống hệt nhau**, chỉ khác nhau ở CAN ID (BMS1: 0x002, 0x300-0x32F; BMS2: 0x003, 0x330-0x35F).

## Bảng Value Table (Bảng tra cứu giá trị)

Value Table trong file DBC là **bảng ánh xạ (mapping)** giữa một giá trị số nguyên trong CAN data và ý nghĩa thực tế của nó dưới dạng chuỗi ký tự. Khi BMS gửi một giá trị số qua CAN, phần mềm đọc (như CANalyzer, CANoe) sẽ tra bảng này để hiển thị tên dễ hiểu thay vì số thô.

### bms_rentalinfoPackType - Loại Pack pin

Xác định **dòng sản phẩm** (model) của Pack pin đang được sử dụng.

| Giá trị số | Ý nghĩa | Giải thích |
|:---:|---|---|
| 0 | TBD | To Be Determined - Chưa xác định, giá trị mặc định khi chưa cấu hình |
| 1 | Pack E1 | Pack pin dòng E1 |
| 2 | Pack D1 | Pack pin dòng D1 |

### bms_rentalinfoTBD1 - Dung lượng danh định

Xác định **mức dung lượng năng lượng** (kWh) của Pack pin.

| Giá trị số | Ý nghĩa | Giải thích |
|:---:|---|---|
| 0 | TBD | Chưa xác định |
| 1 | 1.9 kWh | Pack pin có dung lượng danh định 1.9 kWh |
| 2 | 1.2 kWh | Pack pin có dung lượng danh định 1.2 kWh |

### bms_resultCheckScuSn - Kết quả kiểm tra SCU Serial Number

Kết quả kiểm tra tính hợp lệ của **SCU Serial Number** (SCU - System Control Unit: bộ điều khiển hệ thống). BMS kiểm tra xem số serial của SCU có khớp/hợp lệ hay không.

| Giá trị số | Ý nghĩa | Giải thích |
|:---:|---|---|
| 0 | FAIL | Kiểm tra thất bại - Serial Number không hợp lệ |
| 1 | PASS | Kiểm tra thành công - Serial Number hợp lệ |

---

## Danh sách CAN Message

### BMS1 - Pack pin thứ nhất (ID: 0x002 - 0x32F)

| CAN ID | Tên Message | DLC | Cycle (ms) | Mô tả |
|:---:|---|:---:|:---:|---|
| 0x002 | BMS1_ISO_Message | 8 | - | Truyền thông ISO 15765 |
| 0x300 | BMS1_ControlSystem | 8 | 150 | Điều khiển hệ thống (giới hạn dòng/công suất sạc-xả) |
| 0x301 | BMS1_InfoCharging | 8 | 1000 | Thông tin trạng thái sạc |
| 0x303 | BMS1_InfoBms | 8 | 150 | Trạng thái BMS (user mode, ignition, emergency) |
| 0x304 | BMS1_InfoCellBalancing | 8 | 1000 | Trạng thái cân bằng cell (CB01-CB13) |
| 0x306 | BMS1_InfoDemCell | 8 | 150 | Chẩn đoán lỗi Cell (quá áp, thấp áp, quá dòng, quá nhiệt) |
| 0x309 | BMS1_InfoPack | 8 | 150 | Trạng thái Pack (điện áp, dòng điện, FET) |
| 0x30A | BMS1_InfoSox | 8 | 1000 | SOC, SOH, Cycle Count, thời gian sạc còn lại |
| 0x30B | BMS1_InfoAccumDsgChgCapacity | 8 | 3000 | Dung lượng tích lũy sạc/xả (Ah) |
| 0x30E | BMS1_InfoContactor | 8 | 150 | Trạng thái Contactor (điện áp ADC) |
| 0x310 | BMS1_InfoVoltageCell | 8 | 500 | Tổng hợp điện áp Cell (Avg, Min, Max) |
| 0x311 | BMS1_InfoVoltageCell1 | 8 | 500 | Điện áp Cell 01 - 04 |
| 0x312 | BMS1_InfoVoltageCell2 | 8 | 500 | Điện áp Cell 05 - 08 |
| 0x313 | BMS1_InfoVoltageCell3 | 8 | 500 | Điện áp Cell 09 - 12 |
| 0x314 | BMS1_InfoVoltageCell4 | 2 | 500 | Điện áp Cell 13 |
| 0x315 | BMS1_InfoDemBMS | 4 | 500 | Chẩn đoán lỗi BMS (FET, Fuse, ASIC) |
| 0x320 | BMS1_InfoTemperatureCell | 8 | 1000 | Nhiệt độ Cell (Avg, Min, Max) |
| 0x322 | BMS1_InfoTemperatureCB | 8 | 500 | Nhiệt độ mạch cân bằng và FET |
| 0x32F | BMS1_InfoPackVersion | 8 | 1000 | Phiên bản SW/HW/Bootloader, ngày sản xuất |

### BMS2 - Pack pin thứ hai (ID: 0x003 - 0x35F)

| CAN ID | Tên Message | DLC | Cycle (ms) | Mô tả |
|:---:|---|:---:|:---:|---|
| 0x003 | BMS2_ISO_Message | 8 | - | Truyền thông ISO 15765 |
| 0x330 | BMS2_ControlSystem | 8 | 150 | Điều khiển hệ thống (giới hạn dòng/công suất sạc-xả) |
| 0x331 | BMS2_InfoCharging | 8 | 1000 | Thông tin trạng thái sạc |
| 0x333 | BMS2_InfoBms | 8 | 150 | Trạng thái BMS (user mode, ignition, emergency) |
| 0x334 | BMS2_InfoCellBalancing | 8 | 1000 | Trạng thái cân bằng cell (CB01-CB13) |
| 0x336 | BMS2_InfoDemCell | 8 | 150 | Chẩn đoán lỗi Cell (quá áp, thấp áp, quá dòng, quá nhiệt) |
| 0x339 | BMS2_InfoPack | 8 | 150 | Trạng thái Pack (điện áp, dòng điện, FET) |
| 0x33A | BMS2_InfoSox | 8 | 1000 | SOC, SOH, Cycle Count, thời gian sạc còn lại |
| 0x33B | BMS2_InfoAccumDsgChgCapacity | 8 | 3000 | Dung lượng tích lũy sạc/xả (Ah) |
| 0x33E | BMS2_InfoContactor | 8 | 150 | Trạng thái Contactor (điện áp ADC) |
| 0x340 | BMS2_InfoVoltageCell | 8 | 500 | Tổng hợp điện áp Cell (Avg, Min, Max) |
| 0x341 | BMS2_InfoVoltageCell1 | 8 | 500 | Điện áp Cell 01 - 04 |
| 0x342 | BMS2_InfoVoltageCell2 | 8 | 500 | Điện áp Cell 05 - 08 |
| 0x343 | BMS2_InfoVoltageCell3 | 8 | 500 | Điện áp Cell 09 - 12 |
| 0x344 | BMS2_InfoVoltageCell4 | 8 | 500 | Điện áp Cell 13 |
| 0x345 | BMS2_InfoDemBMS | 4 | 500 | Chẩn đoán lỗi BMS (FET, Fuse, ASIC) |
| 0x350 | BMS2_InfoTemperatureCell | 8 | 1000 | Nhiệt độ Cell (Avg, Min, Max) |
| 0x352 | BMS2_InfoTemperatureCB | 8 | 500 | Nhiệt độ mạch cân bằng và FET |
| 0x35F | BMS2_InfoPackVersion | 8 | 1000 | Phiên bản SW/HW/Bootloader, ngày sản xuất |

---

## Chi tiết Signal theo nhóm chức năng

> Ghi chú: Cả BMS1 và BMS2 đều có cấu trúc signal giống nhau. Dưới đây mô tả theo cấu trúc chung, áp dụng cho cả hai node.

### 1. Điều khiển hệ thống (ControlSystem)

| Signal | Bit Position | Length | Factor | Offset | Range | Unit | Mô tả |
|--------|:---:|:---:|:---:|:---:|---|:---:|---|
| controlSysMaxChgCurrent | 7 | 16 bit | 0.02 | 0 | -655 ~ 655 | A | Dòng sạc tối đa cho phép |
| controlSysMaxDsgCurrent | 23 | 16 bit | 0.02 | 0 | -655 ~ 655 | A | Dòng xả tối đa cho phép |
| controlSysMaxChgPower | 39 | 16 bit | 1 | 0 | -32767 ~ 32767 | W | Công suất sạc tối đa cho phép |
| controlSysMaxDsgPower | 55 | 16 bit | 1 | 0 | -32767 ~ 32767 | W | Công suất xả tối đa cho phép |

### 2. Thông tin sạc (InfoCharging)

| Signal | Bit Position | Length | Factor | Range | Unit | Mô tả |
|--------|:---:|:---:|:---:|---|:---:|---|
| chargingVoltageLimit | 7 | 16 bit | 0.002 | 0 ~ 131.07 | V | Giới hạn điện áp sạc (= 0 khi có lỗi) |
| chargingCurrentLimit | 23 | 16 bit | 0.02 | -655 ~ 655 | A | Giới hạn dòng sạc (= 0 khi có lỗi) |
| chargingFullyChargedSts | 39 | 4 bit | 1 | 0 ~ 15 | - | 0=N/A, 1=Charging, 2=Fully Charged |
| controlModeCharger | 35 | 4 bit | 1 | 0 ~ 15 | - | 0=Charge OFF, 1=Charge Normal, 2=Trickle Mode |
| endCurrentCharge | 47 | 8 bit | 0.01 | 0 ~ 2.55 | A | Dòng kết thúc sạc |
| trickleCurrentCharger | 55 | 8 bit | 0.01 | 0 ~ 2.55 | A | Dòng sạc Trickle tối đa |
| chargingDiagCharger | 63 | 8 bit | 1 | 0 ~ 255 | - | Chẩn đoán bộ sạc |

### 3. Trạng thái BMS (InfoBms)

| Signal | Bit Position | Length | Mô tả | Giá trị |
|--------|:---:|:---:|---|---|
| statusIgnitionRecognition | 7 | 4 bit | Nhận diện chân Ignition | 0=Low, 1=High |
| statusRollingCounter | 3 | 4 bit | Bộ đếm tuần hoàn (0-3), dừng nếu BMS bị treo | 0,1,2,3 |
| statusBmsUserMode | 15 | 4 bit | Chế độ hoạt động | 0=Standby, 1=Working |
| statusEmergency | 11 | 4 bit | Trạng thái khẩn cấp | - |
| statusCurrentDirection | 23 | 8 bit | Hướng dòng điện | 0=Rest, 1=Charging, 2=Discharging |
| stsDiagCellTopPriorResult | 31 | 8 bit | Kết quả chẩn đoán cell nghiêm trọng nhất | 0 ~ 255 |
| wakeupSource | 63 | 8 bit | Nguồn đánh thức | 0=RTC, 1=BMS SW |

### 4. Trạng thái Pack (InfoPack)

| Signal | Bit Position | Length | Factor | Range | Unit | Mô tả |
|--------|:---:|:---:|:---:|---|:---:|---|
| statusCFET | 7 | 4 bit | 1 | 0 ~ 15 | - | Trạng thái C-FET (0=Off, 1=On) |
| statusDFET | 3 | 4 bit | 1 | 0 ~ 15 | - | Trạng thái D-FET (0=Off, 1=On) |
| statusPrechargeFET | 11 | 4 bit | 1 | 0 ~ 15 | - | Trạng thái Precharge-FET (0=Off, 1=On) |
| statusLowCapacityAlarm | 15 | 4 bit | 1 | 0 ~ 15 | - | Cảnh báo dung lượng thấp SOC<10% (0=Clear, 1=Set) |
| statusPackVoltageBAT | 23 | 16 bit | 0.002 | 0 ~ 131.07 | V | Tổng điện áp nối tiếp các cell |
| statusPackCurrentAvg | 39 | 16 bit | 0.02 | -655.36 ~ 655.34 | A | Dòng điện trung bình Pack |
| statusPackCurrent | 55 | 16 bit | 0.02 | -655.36 ~ 655.34 | A | Dòng điện tức thời Pack (+: sạc, -: xả) |

### 5. SOC/SOH (InfoSox)

| Signal | Bit Position | Length | Factor | Range | Unit | Mô tả |
|--------|:---:|:---:|:---:|---|:---:|---|
| socFullChargeCapacityAh | 7 | 8 bit | 0.4 | 0 ~ 76.5 | Ah | Dung lượng sạc đầy |
| socRemainingCapacityAh | 15 | 8 bit | 0.4 | 0 ~ 102 | Ah | Dung lượng còn lại |
| socRSOC | 23 | 8 bit | 1 | 0 ~ 255 | % | Relative State of Charge |
| socSOH | 31 | 8 bit | 1 | 0 ~ 255 | % | State of Health |
| socCycleCount | 39 | 16 bit | 1 | 0 ~ 65535 | - | Số chu kỳ sạc/xả |
| socRemainingChargingTime | 55 | 16 bit | 1 | 0 ~ 65535 | min | Thời gian sạc còn lại |

### 6. Dung lượng tích lũy (InfoAccumDsgChgCapacity)

| Signal | Bit Position | Length | Factor | Range | Unit | Mô tả |
|--------|:---:|:---:|:---:|---|:---:|---|
| AccumChargeCapacityAh | 7 | 32 bit | 1 | 0 ~ 4294967295 | Ah | Tổng dung lượng sạc tích lũy |
| AccumDischargeCapacityAh | 39 | 32 bit | 1 | 0 ~ 4294967295 | Ah | Tổng dung lượng xả tích lũy |

### 7. Điện áp Cell

#### InfoVoltageCell - Tổng hợp

| Signal | Bit Position | Length | Factor | Range | Unit | Mô tả |
|--------|:---:|:---:|:---:|---|:---:|---|
| statusVoltageCellAvg | 7 | 16 bit | 0.0001 | 0 ~ 6.5535 | V | Điện áp Cell trung bình |
| statusVoltageCellMin | 23 | 16 bit | 0.0001 | 0 ~ 6.5535 | V | Điện áp Cell thấp nhất |
| statusVoltageCellMinNo | 39 | 8 bit | 1 | 0 ~ 255 | Bank | Số thứ tự Cell có điện áp thấp nhất |
| statusVoltageCellMax | 47 | 16 bit | 0.0001 | 0 ~ 6.5535 | V | Điện áp Cell cao nhất |
| statusVoltageCellMaxNo | 63 | 8 bit | 1 | 0 ~ 255 | Bank | Số thứ tự Cell có điện áp cao nhất |

#### InfoVoltageCell1-4 - Điện áp từng Cell

Mỗi signal điện áp cell riêng lẻ (Cell 01 đến Cell 13) đều có cấu trúc:
- **Độ dài:** 16 bit
- **Factor:** 0.0001
- **Range:** 0 ~ 6.5535 V
- **Byte Order:** Big Endian (Motorola)

### 8. Nhiệt độ Cell (InfoTemperatureCell)

| Signal | Bit Position | Length | Factor | Range | Unit | Mô tả |
|--------|:---:|:---:|:---:|---|:---:|---|
| statusTemperatureCellAvg | 7 | 8 bit | 1 | -127 ~ 127 | °C | Nhiệt độ Cell trung bình |
| statusTemperatureCellMin | 15 | 8 bit | 1 | -127 ~ 127 | °C | Nhiệt độ Cell thấp nhất |
| statusTemperatureCellMinNo | 23 | 8 bit | 1 | 0 ~ 255 | Th | Số thứ tự thermistor thấp nhất |
| statusTemperatureCellMax | 31 | 8 bit | 1 | -127 ~ 127 | °C | Nhiệt độ Cell cao nhất |
| statusTemperatureCellMaxNo | 39 | 8 bit | 1 | 0 ~ 255 | Th | Số thứ tự thermistor cao nhất |

### 9. Nhiệt độ mạch cân bằng (InfoTemperatureCB)

| Signal | Bit Position | Length | Factor | Range | Unit | Mô tả |
|--------|:---:|:---:|:---:|---|:---:|---|
| statusTemperatureCB1 | 7 | 8 bit | 1 | -127 ~ 127 | °C | Nhiệt độ mạch CB1 |
| statusTemperatureCB2 | 15 | 8 bit | 1 | -127 ~ 127 | °C | Nhiệt độ mạch CB2 |
| statusTemperatureFET | 23 | 8 bit | 1 | -127 ~ 127 | °C | Nhiệt độ FET |

### 10. Trạng thái Contactor (InfoContactor)

| Signal | Bit Position | Length | Factor | Range | Unit | Mô tả |
|--------|:---:|:---:|:---:|---|:---:|---|
| statusAdcBattVolt | 7 | 16 bit | 0.002 | 0 ~ 131.07 | V | Điện áp đo ADC phía Battery |
| statusAdcPackVolt | 23 | 16 bit | 0.002 | 0 ~ 131.07 | V | Điện áp đo ADC phía Pack |
| statusAdcFuseVolt | 39 | 16 bit | 0.002 | 0 ~ 131.07 | V | Điện áp đo ADC phía Fuse |
| statuschargetime | 55 | 16 bit | 1 | 0 ~ 65535 | - | Thời gian sạc |

### 11. Cân bằng Cell (InfoCellBalancing)

Gồm 13 signal (CB01 đến CB13), mỗi signal 4 bit:
- **0 = Off:** Mạch cân bằng tắt
- **1 = On:** Mạch cân bằng đang hoạt động

### 12. Phiên bản Pack (InfoPackVersion)

| Signal | Bit Position | Length | Mô tả |
|--------|:---:|:---:|---|
| versionSoftwareMajor | 11 | 4 bit | Phiên bản phần mềm chính (major) |
| versionSoftwareMinor | 7 | 4 bit | Phiên bản phần mềm phụ (minor) |
| versionSoftwareSubminor | 3 | 4 bit | Phiên bản phần mềm con (subminor) |
| versionBootloaderMajor | 15 | 4 bit | Phiên bản bootloader chính |
| versionBootloaderMinor | 23 | 4 bit | Phiên bản bootloader phụ |
| versionBootloaderSubminor | 19 | 4 bit | Phiên bản bootloader con |
| versionHardwareMajor | 31 | 4 bit | Phiên bản phần cứng |
| versionManufacturerDate | 39 | 16 bit | Ngày sản xuất |
| OtpBq | 55 | 16 bit | Số serial sản xuất theo ngày |

---

## Chẩn đoán lỗi (Diagnostic)

### Chẩn đoán Cell (InfoDemCell)

Tất cả các signal chẩn đoán Cell đều sử dụng **4 bit** với các mức lỗi:

| Giá trị | Mức độ | Mô tả |
|:---:|---|---|
| 0 | Normal | Bình thường |
| 1 | Warning1 | Cảnh báo mức 1 |
| 2 | Warning2 | Cảnh báo mức 2 |
| 3 | Fault | Lỗi |
| 4 | Failure | Hỏng |
| 5-7 | RSVD | Dự phòng |

**Danh sách mã lỗi Cell:**

| Signal | Mô tả |
|--------|--------|
| demOverVoltageCell | Quá áp Cell |
| demOverVoltagePack | Quá áp Pack |
| demUnderVoltageCell | Thấp áp Cell |
| demUnderVoltagePack | Thấp áp Pack |
| demOverCurrentChg | Quá dòng sạc |
| demOverCurrentDis | Quá dòng xả |
| demOverTemperatureChg | Quá nhiệt khi sạc |
| demOverTemperatureDis | Quá nhiệt khi xả |
| demUnderTemperatureChg | Nhiệt độ quá thấp khi sạc |
| demUnderTemperatureDis | Nhiệt độ quá thấp khi xả |
| demImbCellChg | Mất cân bằng điện áp Cell khi sạc |
| demImbCellRest | Mất cân bằng điện áp Cell khi nghỉ |
| demImbCellTemperature | Mất cân bằng nhiệt độ Cell |
| demImbCellSOC | Mất cân bằng SOC Cell |

### Chẩn đoán BMS (InfoDemBMS)

Các signal chẩn đoán BMS sử dụng **4 bit** với 2 trạng thái:
- **0 = Normal:** Bình thường
- **3 = Fault:** Lỗi

| Signal | Mô tả |
|--------|--------|
| demBMSCFET | Lỗi C-FET (Charge FET) |
| demBMSDFET | Lỗi D-FET (Discharge FET) |
| demBMSPFET | Lỗi P-FET (Precharge FET) |
| demBMSASICComm | Lỗi giao tiếp ASIC |
| demBMSASICShutdown | ASIC bị shutdown |
| demBMSShortCurrent | Lỗi ngắn mạch (ASIC Short Current) |
| demBMSFETTemperature | Quá nhiệt FET |
| demBMSFuseBlow | Cầu chì bị đứt |

---

## Chu kỳ gửi Message

| Chu kỳ | Message |
|:---:|---|
| **150 ms** | ControlSystem, InfoPack, InfoDemCell, InfoContactor, InfoBms |
| **500 ms** | InfoVoltageCell (1-4), InfoVoltageCell, InfoTemperatureCB, InfoDemBMS |
| **1000 ms** | InfoTemperatureCell, InfoSox, InfoPackVersion, InfoCharging, InfoCellBalancing |
| **3000 ms** | InfoAccumDsgChgCapacity |

## Cấu hình Signal

- **Byte Order:** Big Endian (Motorola) - ký hiệu `@0` trong DBC
- **Kiểu dữ liệu:** Unsigned (`+`) cho hầu hết signal, Signed (`-`) cho dòng điện và nhiệt độ
- **Tất cả message** đều gửi theo chế độ **Fixed Periodic** (GenMsgSendType = 0)

## Quy ước đặt tên Signal

```
bms{N}_{category}{Name}
```

- `{N}`: Số thứ tự BMS (1 hoặc 2)
- `{category}`: Nhóm chức năng
  - `status` - Trạng thái
  - `dem` - Chẩn đoán (Diagnostic Event Manager)
  - `soc` - State of Charge
  - `control` - Điều khiển
  - `charging` - Sạc
  - `version` - Phiên bản
- `{Name}`: Tên cụ thể của signal
