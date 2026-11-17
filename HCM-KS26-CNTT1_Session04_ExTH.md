# [BTTH] BÀI TẬP THỰC HÀNH TỔNG HỢP – SESSION 04

**Chủ đề:** Kỹ thuật Prompting nâng cao (Logic & Chuỗi)

## 1. Thiết kế giải pháp

**Bài toán:** Em chọn việc tự động kiểm tra đơn hàng cho cửa hàng bán quần áo. Một số đơn có thể thiếu thông tin hoặc nhập sai số lượng, đơn giá.

- **Input:** Danh sách đơn hàng gồm `ma_don`, `ten_khach`, `so_dien_thoai`, `dia_chi`, `san_pham`, `so_luong`, `don_gia`.
- **Output:** JSON chứa từng đơn (`ma_don`, `trang_thai`, `tong_tien`, `ghi_chu`) và bảng tổng kết số đơn, tổng tiền.
- **Quy tắc:** Số lượng và đơn giá phải là số nguyên dương. Chấp nhận chuỗi số như `"2"` và đổi thành `2`. Số âm, chữ hoặc dữ liệu trống là lỗi số liệu; nếu số liệu đúng nhưng thiếu thông tin khách/địa chỉ/sản phẩm thì yêu cầu bổ sung. **Ưu tiên `LOI_DU_LIEU` trước `CAN_BO_SUNG`** nếu có cả hai loại lỗi. Không tự đoán dữ liệu thiếu.

**Chuỗi xử lý:** `Input → Prompt 1 (kiểm tra) → Prompt 2 (tính tiền) → Prompt 3 (tổng kết) → JSON cuối`.

Sau mỗi bước, lưu kết quả vào checkpoint để nếu bị ngắt có thể tiếp tục, không cần chạy lại toàn bộ.

## 2. Các prompt thực thi

### Prompt 1 – Kiểm tra và chuẩn hóa

> Bạn là trợ lý kiểm tra đơn hàng. Hãy kiểm tra JSON trong `<<<INPUT>>>` theo từng đơn.
>
> 1. Kiểm tra mã đơn, tên khách, số điện thoại, địa chỉ, sản phẩm có trống không.
> 2. Chuẩn hóa `so_luong` và `don_gia`: chỉ chấp nhận số nguyên dương hoặc chuỗi chỉ chứa chữ số; còn lại xem là sai.
> 3. Nếu số liệu sai, gán `LOI_DU_LIEU`; nếu số liệu đúng nhưng thiếu trường chữ, gán `CAN_BO_SUNG`; còn lại `HOP_LE`.
> 4. Không đoán dữ liệu. Mỗi bản ghi xuất `ma_don`, `so_luong`, `don_gia`, `trang_thai`, `ghi_chu`. Nếu số sai thì xuất `null` ở trường đó.
> 5. Chỉ xuất mảng JSON, không giải thích dài. Kiểm tra đủ số dòng đầu vào.
>
> `<<<INPUT>>>` Dán JSON đầu vào tại đây `<<<END>>>`.

### Prompt 2 – Tính tiền

> Nhận JSON từ Prompt 1 trong `<<<BUOC_1>>>`.
>
> - Chỉ tính `tong_tien = so_luong * don_gia` nếu `trang_thai = HOP_LE`.
> - Đơn `CAN_BO_SUNG` hoặc `LOI_DU_LIEU` thì `tong_tien = null`.
> - Giữ nguyên thứ tự, mã đơn, trạng thái và ghi chú; không tạo thêm đơn.
> - Chỉ trả về mảng JSON gồm `ma_don`, `trang_thai`, `tong_tien`, `ghi_chu`.
>
> `<<<BUOC_1>>>` Dán kết quả Prompt 1 tại đây `<<<END>>>`.

### Prompt 3 – Tổng hợp và kiểm tra

> Nhận JSON từ Prompt 2 trong `<<<BUOC_2>>>`.
>
> - Đếm tổng đơn, số đơn hợp lệ, cần bổ sung và lỗi dữ liệu.
> - Cộng `tong_tien` của các đơn hợp lệ, bỏ qua `null`.
> - Kiểm tra số lượng các nhóm có bằng tổng số đơn không; đơn lỗi phải có `tong_tien = null`; tổng tiền hợp lệ phải dương.
> - Xuất **một đối tượng JSON** có ba khóa: `don_hang`, `tong_ket`, `kiem_tra`. Đặt `kiem_tra = "OK"` nếu không có mâu thuẫn, ngược lại `"CAN_KIEM_TRA"`.
> - Không thêm lời giải thích ngoài JSON.
>
> `<<<BUOC_2>>>` Dán kết quả Prompt 2 tại đây `<<<END>>>`.

### Prompt xử lý đứt gãy

> Phiên trước bị ngắt. Đây là checkpoint của bước gần nhất đã hoàn tất: `<<<CHECKPOINT>>>` ... `<<<END>>>`.
>
> Hãy tiếp tục từ **bước kế tiếp**. Nếu đang xử lý dở một danh sách, chỉ làm những mã đơn chưa có trong `<<<DA_XONG>>>` ... `<<<END>>>`. Không lặp đơn đã hoàn thành. Giữ cùng cấu trúc JSON. Nếu checkpoint trống/hỏng, báo `THIEU_CHECKPOINT`, không tự tạo dữ liệu.

**Cách vận hành:** Lưu `checkpoint_b1.json` sau Prompt 1, `checkpoint_b2.json` sau Prompt 2; chỉ chuyển sang bước sau khi JSON hợp lệ. Nhờ vậy lỗi ở một bước không làm hỏng cả chuỗi.

## 3. Chạy thử và kết quả

**Dữ liệu thử:**

| Mã đơn | Tên khách | Địa chỉ | Số lượng | Đơn giá |
|---|---|---|---|---|
| DH01 | Lan | TP.HCM | 2 | 150000 |
| DH02 | Nam | *(trống)* | 1 | 150000 |
| DH03 | Hoa | Hà Nội | -1 | 200000 |
| DH04 | Minh | Đà Nẵng | `"2"` | `"120000"` |

*Cả 4 đơn đều có mã, số điện thoại và tên sản phẩm; riêng DH02 thiếu địa chỉ.*

**Kết quả Prompt 1 (kiểm tra):**

```json
[
  {
    "ma_don": "DH01",
    "so_luong": 2,
    "don_gia": 150000,
    "trang_thai": "HOP_LE",
    "ghi_chu": ""
  },
  {
    "ma_don": "DH02",
    "so_luong": 1,
    "don_gia": 150000,
    "trang_thai": "CAN_BO_SUNG",
    "ghi_chu": "Thiếu địa chỉ"
  },
  {
    "ma_don": "DH03",
    "so_luong": null,
    "don_gia": 200000,
    "trang_thai": "LOI_DU_LIEU",
    "ghi_chu": "Số lượng phải là số nguyên dương"
  },
  {
    "ma_don": "DH04",
    "so_luong": 2,
    "don_gia": 120000,
    "trang_thai": "HOP_LE",
    "ghi_chu": ""
  }
]
```

**Kết quả Prompt 2 (tính tiền):**

```json
[
  {
    "ma_don": "DH01",
    "trang_thai": "HOP_LE",
    "tong_tien": 300000,
    "ghi_chu": ""
  },
  {
    "ma_don": "DH02",
    "trang_thai": "CAN_BO_SUNG",
    "tong_tien": null,
    "ghi_chu": "Thiếu địa chỉ"
  },
  {
    "ma_don": "DH03",
    "trang_thai": "LOI_DU_LIEU",
    "tong_tien": null,
    "ghi_chu": "Số lượng phải là số nguyên dương"
  },
  {
    "ma_don": "DH04",
    "trang_thai": "HOP_LE",
    "tong_tien": 240000,
    "ghi_chu": ""
  }
]
```

**Kết quả Prompt 3 (JSON cuối):**

```json
{
  "don_hang": [
    {
      "ma_don": "DH01",
      "trang_thai": "HOP_LE",
      "tong_tien": 300000,
      "ghi_chu": ""
    },
    {
      "ma_don": "DH02",
      "trang_thai": "CAN_BO_SUNG",
      "tong_tien": null,
      "ghi_chu": "Thiếu địa chỉ"
    },
    {
      "ma_don": "DH03",
      "trang_thai": "LOI_DU_LIEU",
      "tong_tien": null,
      "ghi_chu": "Số lượng phải là số nguyên dương"
    },
    {
      "ma_don": "DH04",
      "trang_thai": "HOP_LE",
      "tong_tien": 240000,
      "ghi_chu": ""
    }
  ],
  "tong_ket": {
    "tong_don": 4,
    "hop_le": 2,
    "can_bo_sung": 1,
    "loi_du_lieu": 1,
    "doanh_thu_hop_le": 540000
  },
  "kiem_tra": "OK"
}
```
