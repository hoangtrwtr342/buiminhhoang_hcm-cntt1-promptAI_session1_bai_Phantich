# [Phân tích] So sánh Trade-off các giải pháp ra lệnh cho Cấu trúc Prompt Tiêu chuẩn & Định dạng



## Bước 1: Đề xuất 2 Prompt mẫu

### Giải pháp 1: Đơn giản (Zero-shot, ràng buộc cơ bản)
```markdown
# [C - Context] Cửa hàng bán lẻ cần tự động nhập liệu đơn hàng từ tin nhắn khách trên Fanpage/Zalo.
# [R - Request] Trích xuất tin nhắn sau thành JSON gồm: `status` ("VALID"/"INCOMPLETE"), `customer` (`name`, `phone`, `address`), `items` (`product_name`, `quantity`), `note`.
# [A - Adjustments] Thiếu thông tin gán `null`. Chỉ `"VALID"` khi đủ SĐT, địa chỉ, sản phẩm.
# [T - Type of Output] Chỉ trả về mã JSON, không giải thích.
# [E - Extras] Input: "{TIN_NHẮN}"
```

### Giải pháp 2: Chi tiết (CREATE chuẩn + Few-shot + Bắt lỗi biên)
```markdown
# [C - Context] Cửa hàng bán lẻ nhận nhiều tin nhắn đặt hàng viết tắt, thiếu dấu mỗi ngày trên Fanpage/Zalo và cần bóc tách dữ liệu tự động để đẩy vào phần mềm quản lý đơn.
# [R - Request] Chuyển tin nhắn trong `<input>` thành JSON chuẩn và kiểm tra tính hợp lệ của đơn hàng.
# [A - Adjustments] 
- Thiếu dữ liệu gán `null`, tuyệt đối không tự suy diễn.
- SĐT hợp lệ phải đủ 10 số (bắt đầu bằng 0), sai định dạng gán `null`.
- Chỉ `"VALID"` khi đủ: `phone` hợp lệ, `address` rõ quận/tỉnh, và ≥1 sản phẩm. Thiếu thì gán `"INCOMPLETE"`.
- Không ghi số lượng mặc định là `1`.
# [E - Examples]
Input: "Chị Lan, 2 áo thun, 12 Nguyễn Trãi Q1 HCM, 0901234567"
Output: {"status": "VALID", "customer": {"name": "Lan", "phone": "0901234567", "address": "12 Nguyễn Trãi, Q1, HCM"}, "items": [{"product_name": "Áo thun", "quantity": 2}], "note": null}

Input: "Lấy mũ đen về Cầu Giấy HN, sđt 0988123"
Output: {"status": "INCOMPLETE", "customer": {"name": null, "phone": null, "address": "Cầu Giấy, HN"}, "items": [{"product_name": "Mũ đen", "quantity": 1}], "note": null}
# [T - Type of Output] Chỉ trả về 1 khối JSON duy nhất, không văn bản thừa.
# [E - Extras] <input>{TIN_NHẮN}</input>
```

---

## Bước 2: Bảng so sánh Trade-off

| Tiêu chí | Giải pháp 1 (Đơn giản) | Giải pháp 2 (Chi tiết) |
| :--- | :--- | :--- |
| **Độ dễ hiểu** | **Cao** (Ngắn, viết nhanh) | **Trung bình** (Cần thiết kế luật & ví dụ) |
| **Tốn token** | **Ít** (~95 tokens/lần) | **Nhiều hơn** (~290 tokens/lần) |
| **Chặn lỗi biên** | **Trung bình** (Dễ lọt SĐT sai, địa chỉ thiếu) | **Cực tốt** (Bắt chuẩn lỗi thiếu/sai định dạng) |
| **Độ ổn định JSON** | **Trung bình** (Dễ kèm lời chào thừa) | **Rất cao** (Nhờ Few-shot làm khuôn mẫu) |

---

## Bước 3: Lựa chọn tối ưu & Chạy thử

* **Phương án chọn:** **Giải pháp 2 (Chi tiết)**.
* **Lý do:** Chi phí chênh lệch vài trăm token không đáng kể, nhưng việc chặn được đơn sai SĐT/thiếu địa chỉ giúp cửa hàng tránh thiệt hại phí hoàn hàng. Ngoài ra, kỹ thuật Few-shot và thẻ `<input>` giữ cấu trúc JSON ổn định 100% để đẩy thẳng qua API.

**Chạy thử Giải pháp 2:**
* **Input:** *"Gửi bạn Minh 3 đôi tất xám về 45 Lê Lợi nha, sđt 0912.345.67a (nhầm, đuôi 678). Gói quà giúp mình."*
* **Output:**
```json
{
  "status": "INCOMPLETE",
  "customer": {
    "name": "Minh",
    "phone": "0912345678",
    "address": "45 Lê Lợi"
  },
  "items": [
    {"product_name": "Tất xám", "quantity": 3}
  ],
  "note": "Gói quà"
}
```
*(Kết luận: AI tự làm sạch SĐT gõ nhầm thành `0912345678` và bắt lỗi `"INCOMPLETE"` do địa chỉ thiếu quận/tỉnh).*
