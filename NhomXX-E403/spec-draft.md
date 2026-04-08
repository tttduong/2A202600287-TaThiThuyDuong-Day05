# SPEC — AI Product Hackathon

**Nhóm:** THE RATS
**Track:** VinFast
**Problem statement (1 câu):** *Khách hàng tìm hiểu mua xe VinFast gặp khó khăn khi tra cứu thông số, chính sách qua website và thường phải chờ đợi lâu ngoài giờ hành chính mới được hỗ trợ; AI Agent giúp tư vấn tự động 24/7, cung cấp chính xác thông số qua RAG và điều hướng người dùng đặt lịch lái thử ngay lập tức.*

---

## 1. AI Product Canvas

|   | Value | Trust | Feasibility |
|---|-------|-------|-------------|
| **Câu hỏi** | User nào? Pain gì? AI giải gì? | Khi AI sai thì sao? User sửa bằng cách nào? | Cost/latency bao nhiêu? Risk chính? |
| **Trả lời** | *Khách muốn mua xe. Pain: Chờ đợi CSKH lâu, tự đọc web khó. AI giải: Trả lời ngay tức khắc (giá, thông số, ưu đãi) & Lên lịch lái thử.* | *AI có thể báo sai giá. Giải pháp: AI kèm disclaimer và nếu không chắc sẽ fallback "Để lại SĐT cho CSKH tư vấn chuyên sâu".* | *Cost: ~$0.005/query. Latency <3s. Risk chính: Tư vấn sai lệch chính sách bán hàng hoặc so sánh toxic với xe đối thủ.* |

**Automation hay augmentation?** ☐ Automation · ☑ Augmentation
Justify: *Augmentation — AI đảm nhiệm khâu lọc "phễu đầu" (tư vấn cơ bản, xin thông tin, lên lịch), quyết định chốt sale và tư vấn tài chính chuyên sâu vững nhất vẫn là nhân sự (Sales/CSKH) tiếp nhận phần Context sau đó.*

**Learning signal:**

1. User correction đi vào đâu? *Feedback (rất hài lòng/không hài lòng) cho luồng chat và tỷ lệ Rớt (Drop-off) hội thoại đi vào Dashboard quản lý để cập nhật lại hệ RAG.*
2. Product thu signal gì để biết tốt lên hay tệ đi? *Tỷ lệ chuyển đổi Conversation -> Lead (đặt lịch thành công) tăng, tần suất xuất hiện keyword "Gặp tư vấn viên" giảm.*
3. Data thuộc loại nào? ☐ User-specific · ☑ Domain-specific · ☐ Real-time · ☐ Human-judgment · ☐ Khác: ___
   Có marginal value không? (Model đã biết cái này chưa?) *Có. Base model (ví dụ GPT-4o) không tự biết chính xác cấu hình mới nhất, giá lăn bánh tính toán cho từng tỉnh và các chính sách pin khuyến mãi thay đổi liên tục của VinFast.*

---

## 2. User Stories — 4 paths

### Feature: *Chatbot tư vấn xe & Đặt lịch lái thử*

**Trigger:** *User vào trang chủ VinFast, bấm vào widget Chat và hỏi "Tư vấn cho mình VF 8"*

| Path | Câu hỏi thiết kế | Mô tả |
|------|-------------------|-------|
| Happy — AI đúng, tự tin | User thấy gì? Flow kết thúc ra sao? | *AI chào hỏi, tóm tắt VF 8 kèm giá, hỏi user quan tâm bản Eco hay Plus. User chọn đặt lịch, AI thu thập SĐT & giờ, call Tool đặt lịch thành công.* |
| Low-confidence — AI không chắc | System báo "không chắc" bằng cách nào? User quyết thế nào? | *User hỏi "Sạc ở trạm bên thứ 3 có được bảo hành thay pin không?", AI có confidence score thấp. AI đáp: "Chính sách này khá đặc thù, anh/chị cho em xin SĐT để chuyên viên hãng gọi lại hỗ trợ chính xác nhất nhé?"* |
| Failure — AI sai | User biết AI sai bằng cách nào? Recover ra sao? | *AI trích lỗi RAG báo giá VF 3 là 500 triệu. User bắt lỗi: "Trên web báo 240 triệu mà?". AI xin lỗi, đính chính dựa trên confirm lại context, cờ (flag) phiên chat.* |
| Correction — user sửa | User sửa bằng cách nào? Data đó đi vào đâu? | *User bấm nút "Report chửi", Admin xem log session và cập nhật/xóa chunk data bị đụng độ trong Knowledge Base của RAG.* |

---

## 3. Eval metrics + threshold

**Optimize precision hay recall?** ☑ Precision · ☐ Recall
Tại sao? *Vì thông số kỹ thuật, bảo hành, và đặc biệt là giá bán xe mang tính pháp lý/thương hiệu rất cao. Thà AI báo "Xin phép chuyển CSKH" (Low Recall) còn hơn là "Bịa ra mức giá không có" (Low Precision) gây khủng hoảng truyền thông.*

| Metric | Threshold | Red flag (dừng khi) |
|--------|-----------|---------------------|
| *Answer Accuracy (Độ chính xác nội dung so với Fact)* | *≥ 95%* | *< 85% trong 1 tuần liên tục (Cần Freeze để fix DB)* |
| *Booking Conversion Rate (Số lượng chat tạo ra Lead)* | *≥ 10%* | *< 2% (Logic luồng gọi API Tool bị fail)* |
| *Hallucination / Toxic Rate* | *≤ 1%* | *> 5% (Nguy cơ khủng hoảng thương hiệu)* |

---

## 4. Top 3 failure modes

| # | Trigger | Hậu quả | Mitigation |
|---|---------|---------|------------|
| 1 | *User gài bẫy hỏi AI so sánh tiêu cực VinFast với loại xe hãng khác.* | *AI có thể bị lừa nói xấu đối thủ hoặc chê hãng mình.* | *Prompt Engineer strict guardrails: "Tuyệt đối không bình luận khen chê hãng khác, luôn lịch sự bẻ hướng tập trung về VinFast."* |
| 2 | *RAG chunking không tốt, nhầm lẫn thông số giữa VF 8 và VF 9 trong cùng một tài liệu PDF.* | *Tư vấn báo sai thông số cho khách hàng (vd: VF 8 có ghế massage).* | *Clean data chuẩn bị RAG thay vì tự động nhai PDF. Dùng Routing agent để search DB riêng theo từng model xe.* |
| 3 | *Hệ thống API gọi CRM (Save lead) bị timeout hoặc down server.* | *Khách nhập số điện thoại nhưng AI báo lỗi thất bại, làm khách cụt hứng.* | *AI fail-safe, trả lời gracefully: "Hệ thống đang quá tải tí xíu, nhưng em đã ghi nhớ thông tin của anh/chị rồi. Sẽ có CSKH gọi lại xác nhận nhé." Fallback đẩy log offline lưu vào CSV đợi retry.* |

---

## 5. ROI 3 kịch bản

|   | Conservative | Realistic | Optimistic |
|---|-------------|-----------|------------|
| **Assumption** | *500 user/ngày, 5% đặt lịch* | *2000 user/ngày, 8% đặt lịch* | *5000 user/ngày, 12% đặt lịch* |
| **Cost** | *$3/ngày (Inference LLM)* | *$10/ngày* | *$25/ngày* |
| **Benefit** | *Tiết kiệm 4h trực chat (~$12), thêm 25 Lead* | *Giảm 16h CSKH, thêm 160 Lead* | *Bao trọn 80% L1 Support, thêm 600 Lead* |
| **Net** | *Dương nhẹ + Lead* | *Dương rất lớn* | *Đóng góp trực tiếp làm thay đổi Data Sale của hãng* |

**Kill criteria:** *Có hiện tượng chụp màn hình bóc phốt lan truyền MXH vì AI tư vấn sai luật (Critical Brand Harm). Hoặc sau 2 tháng Cost duy trì AI > Benefit lượng Lead mang về (Ví dụ: khách vào trêu đùa quá đông nhưng không ai đặt lịch).*

---

## 6. Mini AI spec (1 trang)

**1. Sản phẩm & Giải pháp:**
VinFast Auto-Agent là Chatbot thế hệ mới nhúng vào website VinFast và app. Sử dụng kỹ thuật ReAct Agent kết hợp RAG (Truy xuất tài liệu từ DB) và Function-Calling (Lên lịch test-drive qua API CRM) để thay thế cho Chatbot Rule-based và giảm tải cho Human CSKH.

**2. Khách hàng/User:** 
Khách hàng tiềm năng đang trong giai đoạn tìm hiểu (Consideration) muốn tra cứu thông số, so sánh giá cả siêu tốc và lên lịch lái thử không bị làm phiền; Khách hàng hiện tại tìm kiếm trợ giúp kỹ thuật xe.

**3. Bản chất sản phẩm (Auto/Aug):**
Augmentation (Tiền tuyến - L1 Support). Tự động chặn và giải quyết nhanh 80% câu hỏi FAQ (cấu hình, giá cọc, so sánh bản Plus/Eco...). Ngay khi có mùi "Khó", "Tài chính phức tạp" hoặc "Khiếu nại", lập tức Freeze và đề nghị chuyển ngữ cảnh chat cho tư vấn viên để chốt sale/chăm sóc. Đảm bảo user có Human touch ở chặng cuối.

**4. Metrics & Quality Trade-offs:**
Hệ thống ưu tiên tuyệt đối vào **Precision > Recall**. Chấp nhận việc AI có lúc thoái thác trả lời để nhường cho Human, tuyệt đối tránh hiện tượng bịa giá (đảo lộn bảng giá). 

**5. Cơ chế Data Flywheel:**
Càng nhiều khách hàng hỏi -> Lộ ra các vấn đề Edge cases chưa nằm trong Cẩm nang HDSD -> Đội quản lý RAG gom các câu "Low-confidence" từ Dashboard để thêm vào CSDL -> AI học đường ống mới ngay lập tức. Cứ thế, Coverage của RAG rộng ra trong khi Cost/query giảm dần do dùng caching.
