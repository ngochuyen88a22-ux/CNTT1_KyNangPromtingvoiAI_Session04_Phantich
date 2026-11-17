# Bước 1: Đề xuất 2 Prompt Mẫu
## Giải pháp 1: Đơn giản, ngắn gọn (Zero-shot CoT & Single Call)
```
Đọc thư khiếu nại dưới đây, suy nghĩ từng bước để phân loại sự cố, tính toán mức đền bù theo chính sách (hư hỏng đền 100% giá trị đơn, giao trễ đền 20%) và viết email trả lời khách hàng.
Dữ liệu: "Đơn hàng #TFD91 trị giá 2.000.000 VNĐ bị giao trễ 3 ngày."
```

## Giải pháp 2: Chi tiết, nhiều ràng buộc (Prompt Chaining + Few-shot CoT + Neo ngữ cảnh)
```
Bạn là Hệ thống Điều phối Tự động hóa CSKH của FastDelivery. Có nhiệm vụ - Trạm 1 & 2 (Logic & Tính toán): Phân tích dữ liệu trong thẻ <data>.
1. Ràng buộc xử lý lỗi biên: Nếu thiếu mã đơn hoặc giá trị đơn hàng không hợp lệ (<= 0 hoặc trống), dừng xử lý và xuất ngay JSON: `{"status": "ERROR_MISSING_DATA"}`.
2. Áp dụng Chain-of-Thought (CoT): Lập luận minh bạch 3 bước:
   - Bước 1: Xác định loại sự cố (`Hư hỏng` = 100% | `Giao trễ` = 20%).
   - Bước 2: Tính số tiền đền bù = Giá trị đơn $\times$ Tỷ lệ đền bù.
   - Bước 3: Kiểm tra ngưỡng phê duyệt (Nếu tiền đền bù > 1.000.000 VNĐ -> Gắn cờ `CẦN_DUYỆT_THỦ_CÔNG`).
3. Data Handoff: Chỉ xuất duy nhất chuỗi JSON sạch ở dòng cuối trong khối ```json ... ``` để chuyển sang Trạm 3 (Soạn Email), tuyệt đối không viết lời chào thừa.

<data>
"Đơn hàng #TFD91 trị giá 2.000.000 VNĐ bị giao trễ 3 ngày."
</data>
```

# Bước 2: Lập Bảng So sánh Trade-off

| Tiêu chí | Giải pháp 1 (Đơn giản) | Giải pháp 2 (Chi tiết) |
| :--- | :--- | :--- |
| **Độ dễ hiểu** | **Cao** (Viết nhanh, cấu trúc tự nhiên, không phức tạp). | **Trung bình** (Đòi hỏi tư duy, thiết lập điều kiện logic và định dạng JSON) |
| **Tốn token** | **Ít** (Tiết kiệm Input/Output tokens do gói gọn trong 1 lần gọi duy nhất) | **Nhiều hơn** (Do các quy tắc ràng buộc, lập luận CoT chi tiết và cấu trúc Handoff)|
| **Chặn lỗi biên** | **Trung bình** (Dễ bị sót bước tính toán, ảo giác số liệu hoặc gãy định dạng khi gặp dữ liệu trống/lỗi) | **Cực tốt** (Chặn dữ liệu rác tại nguồn, minh bạch từng bước tính |

# Bước 3: Lựa chọn Phương án Tối ưu và Chạy thử

* **Chọn giải pháp 2 (Chi tiết)**
* **Lý do chọn:** Còn nhiều rủi ro tính sai tiền hoặc gửi nhầm email cho khách hàng gây thiệt hại lớn hơn nhiều so với chi phí vài trăm token phát sinh thêm. Giải pháp 2 đảm bảo tính chính xác nhờ **CoT**, dễ bảo trì và kiểm soát rủi ro nhờ Prompt Chaining và chống trôi dạt vai trò nhờ Context Anchoring

## Kết quả chạy thử nghiệm (Output từ Giải pháp 2):

**Lộ trình lập luận (Chain-of-Thought):**
* Bước 1: Mã đơn `#FD99` hợp lệ. Loại sự cố xác định là `Giao trễ` $\rightarrow$ Áp dụng mức đền bù $20\%$
* Bước 2: Tính toán chi phí: $2.000.000 \text{ VNĐ} \times 20\% = 400.000 \text{ VNĐ}$
* Bước 3: Đối chiếu ngưỡng: $400.000 \text{ VNĐ} \le 1.000.000 \text{ VNĐ}$ $\rightarrow$ Đủ điều kiện tự động chuyển sang Trạm 3 soạn email

```json
{
  "order_id": "FD99",
  "issue_type": "Giao trễ",
  "compensation_amount": 400000,
  "approval_status": "AUTO_APPROVED",
  "status": "SUCCESS"
}
```