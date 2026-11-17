# THIẾT KẾ GIẢI PHÁP PROMPTING TỰ ĐỘNG HÓA: QUY TRÌNH XỬ LÝ KHIẾU NẠI & BẢO HÀNH (CHAINING & COT)



## BƯỚC 1: THIẾT KẾ GIẢI PHÁP (I/O & ERROR HANDLING ANALYSIS)

### 1. Bối cảnh & Vấn đề thực tế

Tại cửa hàng công nghệ **TechShop**, hệ thống tiếp nhận hàng trăm yêu cầu khiếu nại, bảo hành mỗi ngày. Dữ liệu đầu vào từ khách hàng thường lộn xộn, thiếu thông tin định danh (mã đơn hàng), chứa từ ngữ cảm thán hoặc cố tình mỉa mai.

Nếu dùng một câu lệnh đơn lẻ (Single Prompt), mô hình AI rất dễ bị quá tải, bỏ sót các điều kiện chính sách đổi trả hoặc tự ý bịa đặt (Hallucination) mã đơn hàng khi gặp dữ liệu trống. Do đó, giải pháp thiết kế sử dụng **Prompt Chaining (Chuỗi Prompt tuần tự)** kết hợp **Chain-of-Thought (CoT)** và **Cơ chế bắt lỗi dữ liệu dị biệt (Anomaly & Error Handling)** nhằm chia nhỏ bài toán thành các bước xử lý an toàn và chính xác tuyệt đối.

### 2. Sơ đồ Cấu trúc Đầu vào & Đầu ra (I/O Design)

* **Input (Dữ liệu đầu vào):**
* Đoạn văn bản khiếu nại thô từ khách hàng (có thể chứa thông tin không đầy đủ hoặc nhiễu loạn cảm xúc).


* **Output (Cấu trúc đầu ra qua 2 bước xích):**
* **Bước 1 (CoT Analysis JSON):** Trích xuất thực thể, chuỗi suy luận từng bước (Step-by-step reasoning) và xác định mức độ nghiêm trọng.
* **Bước 2 (Policy Action & Fallback JSON/Text):** Đối chiếu quy chế bảo hành, kiểm tra lỗi dị biệt (Missing Data / Fallback) và xuất email phản hồi khách hàng chuyên nghiệp.



### 3. Quy tắc Điều kiện Xử lý Lỗi (Error Handling Rules)

* **Bẫy dữ liệu trống/sai lệch:** Nếu khách hàng không cung cấp mã đơn hàng (`order_id` trống, `null`, hoặc không đúng định dạng `#TS-XXXX`), hệ thống **bắt buộc** không được tự suy diễn.
* **Cơ chế kích hoạt Fallback:** Chuyển trạng thái hệ thống thành `FALLBACK_REQUIRED`, từ chối tự sinh mã giả mạo và yêu cầu chuyển giao sang nhân viên CSKH xử lý thủ công.

---

## BƯỚC 2: HỆ THỐNG PROMPT THỰC THI (PROMPT CHAINING SYSTEM)

Giải pháp được đóng gói thành một chuỗi gồm 2 Prompt liên kết chặt chẽ:

###  Prompt 1: Trích xuất & Suy luận Từng bước (Chain-of-Thought)

```text
[Context / Character]
Bạn là Chuyên viên phân tích dữ liệu khiếu nại cao cấp tại TechShop. Nhiệm vụ của bạn là đọc phản hồi thô của khách hàng và thực hiện suy luận từng bước (Chain-of-Thought) để bóc tách thông tin chính xác.

[Request & Chain-of-Thought Rules]
Hãy phân tích đoạn phản hồi thô dưới đây theo các bước:
1. Đọc kỹ văn bản, nhận diện các vấn đề kỹ thuật cốt lõi và cảm xúc khách hàng.
2. Trích xuất các thực thể: Mã đơn hàng (order_id), Tên khách hàng (customer_name), Vấn đề gặp phải (issue_description).
3. Đánh giá mức độ nghiêm trọng (severity: Thấp, Trung bình, Cao, Nghiêm trọng).

[Type of Output & Format]
Xuất kết quả dưới dạng JSON thuần túy cấu trúc sau:
{
  "cot_reasoning": "Chuỗi suy luận từng bước...",
  "order_id": "...",
  "customer_name": "...",
  "issue_description": "...",
  "severity": "..."
}

[Input Data cần xử lý]:
"Shop làm ăn chán quá, mua cái máy chơi game mã đơn hình như là #TS-9999 mà giao đến mở ra màn hình sọc ngang không lên nguồn, gọi điện tổng đài thì không ai bắt máy!"

```

###  Prompt 2: Đối chiếu Chính sách, Bắt lỗi & Soạn thảo Phản hồi (Action & Fallback)

```text
[Context / Character]
Bạn là Trưởng bộ phận Chăm sóc khách hàng và Vận hành tại TechShop, chịu trách nhiệm ra quyết định bồi thường và soạn thảo phản hồi.

[Error Handling & Fallback Rules (Rất quan trọng)]
- Kiểm tra dữ liệu đầu vào từ bước trước: Nếu trường `order_id` có giá trị là `null`, `"Chưa xác thực"`, bị trống hoặc không đúng định dạng `#TS-XXXX`, **bắt buộc** kích hoạt quy trình ngoại lệ: Đặt `system_status` thành `"FALLBACK_REQUIRED"` và yêu cầu nhân viên CSKH liên hệ thủ công để xin lại mã đơn hàng, tuyệt đối không được tự bịa ra mã đơn trong email.
- Nếu `order_id` hợp lệ, đặt `system_status` thành `"OK"` và tiến hành soạn thảo email xin lỗi lịch sự, đề xuất đổi mới sản phẩm trong 24h theo chính sách TechShop.

[Request]
Dựa vào dữ liệu JSON đầu vào, hãy kiểm tra lỗi, tính toán phương án và soạn thảo phản hồi chuẩn chỉnh.

[Type of Output & Format]
Kết quả trả về gồm 2 phần rõ ràng:
1. Trạng thái kiểm tra lỗi (JSON object chứa `system_status` và mã lỗi nếu có).
2. Nội dung Email phản hồi khách hàng hoàn chỉnh.

```

---

## BƯỚC 3: KẾT QUẢ CHẠY THỬ THỰC TẾ (TEST RUN SIMULATION)

Dưới đây là mô phỏng kết quả đầu ra khi thực thi chuỗi prompt với dữ liệu phản hồi thực tế của khách hàng:

### 1. Kết quả đầu ra từ Prompt 1 (Trích xuất & CoT)

```json
{
  "cot_reasoning": "Bước 1: Khách hàng phản ánh tình trạng máy chơi game hỏng nặng khi nhận hàng (màn hình sọc, không lên nguồn) và bất mãn vì tổng đài không hỗ trợ. Bước 2: Trích xuất thành công mã đơn hàng #TS-9999, tên khách hàng không đề cập. Bước 3: Mức độ lỗi nghiêm trọng do sản phẩm hoàn toàn không sử dụng được.",
  "order_id": "#TS-9999",
  "customer_name": "Chưa xác thực",
  "issue_description": "Màn hình sọc ngang, không lên nguồn",
  "severity": "Nghiêm trọng"
}

```

### 2. Kết quả đầu ra từ Prompt 2 (Kiểm tra lỗi Fallback & Soạn thảo Phản hồi)

```json
{
  "system_status": "OK",
  "email_response": "Tiêu đề: [TechShop] Chân thành xin lỗi Quý khách về sự cố đơn hàng #TS-9999\n\nKính gửi Quý khách,\n\nTechShop đã tiếp nhận thông tin khiếu nại của Quý khách liên quan đến lỗi kỹ thuật (màn hình sọc, không lên nguồn) của đơn hàng #TS-9999. Chúng tôi vô cùng lấy làm tiếc vì trải nghiệm không mong muốn này.\n\nTheo chính sách bảo hành của TechShop, chúng tôi xin phép thực hiện phương án xử lý ngay lập tức:\n1. Cử nhân viên kỹ thuật phối hợp thu hồi sản phẩm lỗi và đổi mới 100% sản phẩm nguyên vẹn tận nơi trong vòng 24 giờ tới (Miễn phí toàn bộ chi phí vận chuyển).\n2. Ghi nhận góp ý về chất lượng tổng đài để chấn chỉnh dịch vụ.\n\nTrân trọng cảm ơn sự thông cảm của Quý khách!"
}

```