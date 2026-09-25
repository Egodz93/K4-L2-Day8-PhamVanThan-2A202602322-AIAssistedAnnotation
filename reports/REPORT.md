# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Phạm Văn Thân

Công cụ gán nhãn đã dùng: CVAT Docker local, import và export bằng định dạng Ultralytics YOLO Detection 1.0.

## 1. Dữ liệu và cách chia tập

Video là một cảnh liên tục từ camera cố định và được lấy mẫu 2.5 frame/giây, nên hai ảnh kế tiếp chỉ cách nhau 0.4 giây và thường chứa cùng xe. Dữ liệu vì thế được chia theo thời gian: 20 ảnh test nằm trong bốn đoạn quanh giây 20, 60, 100 và 140. 112 ảnh vùng đệm bị loại, 268 ảnh còn lại tạo pool. Ảnh pool gần test nhất vẫn cách 4.4 giây.

Nếu chia ngẫu nhiên, các ảnh gần như trùng và thậm chí cùng một chiếc xe có thể xuất hiện ở cả train và test. Đây là rò rỉ dữ liệu, mô hình được chấm trên nội dung nó gần như đã thấy nên AP50, precision và recall có xu hướng cao hơn khả năng tổng quát hóa thực tế. Vùng đệm theo thời gian làm giảm loại lạc quan giả này.

## 2. Mô hình khởi đầu lạnh (cold start)

Dòng vòng 0 trong `reports/rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

`outputs/compare_round0.jpg` cho thấy cold start phát hiện khá tốt các xe lớn, sáng và gần camera, nhưng thường bỏ sót xe nhỏ ở xa, xe tối chỉ còn cụm đèn và xe trong vùng chói. Số liệu theo kích thước cũng thể hiện rõ điểm yếu này. Recall của nhóm small chỉ đạt 0.1818, thấp hơn medium 0.5473 và large 0.5610. Một số false positive nằm quanh cụm đèn hậu hoặc vùng phản chiếu, nơi khó xác định ranh giới thân xe.

Trước khi kết luận model sai ở một xe rất xa trong `frame_0350`, cần người rà lại box tham chiếu quanh các cặp đèn đỏ sát đường chân trời. Nhãn test do một mô hình khác tạo và chưa được rà từng box. Riêng 14 box cao dưới 16 px đã được bỏ qua khi chấm vì quá khó gán nhất quán.

## 3. Chiến lược chọn mẫu

Công thức `score = 0.5·U + 0.3·A + 0.2·D` kết hợp ba tín hiệu. U là trung bình độ bất định của tối đa năm box khó nhất và đạt giá trị cao khi confidence gần 0.5. A là số box có confidence từ 0.15 đến dưới 0.50, sau khi chuẩn hóa theo pool. D thể hiện khoảng cách thời gian tới ảnh đã gán gần nhất. Ở vòng đầu chưa có ảnh đã gán nên D = 1 cho mọi frame. `MIN_GAP_S = 2.0` giúp tránh lấy nhiều ảnh gần như trùng nhau từ camera cố định.

Theo `reports/SELECTION.md`, `frame_0182.jpg` có score 0.9591 và 18 box mập mờ. `frame_0369.jpg` có score 0.9324, U = 0.9315 và 43 dự đoán. `frame_0099.jpg` có U = 0.9460, đồng thời bổ sung một ảnh ở đoạn sớm hơn của video, tại giây 39.6. Ngược lại, `frame_0372.jpg` dù đứng hạng 6 với score 0.9101 vẫn không được chọn vì chỉ cách `frame_0369.jpg` 1.2 giây, nhỏ hơn khoảng cách tối thiểu. Trong 50 ứng viên đầu không có frame trống dự đoán.

Điểm bất định chỉ cho biết model đang phân vân ở đâu, không bảo đảm ảnh đó sẽ giúp model tốt hơn. Ảnh được chọn vẫn có thể gần trùng, tốn nhiều công sửa hoặc chứa các trường hợp khó gán nhãn nhất quán. Kết quả vòng 1 cho thấy rõ điều này, lô ảnh có điểm cao nhưng AP50 sau fine-tune vẫn giảm.

## 4. Các vòng học chủ động (active learning)

Bảng từ `reports/rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 292 | 0.595 | -0.176 | 1.000 | 0.109 | 0.197 | 0.000 | 0.057 | 0.658 |

Ở vòng 1, model đề xuất 169 box. Sau khi rà bằng CVAT, mình giữ 110, chỉnh 41, xóa 18 và thêm 141 box, còn tổng cộng 292 box. Tỷ lệ giữ nguyên là 65%, theo `outputs/round1_diff.md`. Số box được thêm khá lớn, cho thấy pre-label bỏ sót nhiều xe, nhất là xe nhỏ hoặc tối. Vì vậy nhãn cuối cũng cần được kiểm tra kỹ về tính nhất quán.

Sau 50 epoch trên 12 ảnh, AP50 giảm 0.176 so với cold start. Đây cũng là mức giảm so với vòng trước vì vòng 1 là lần fine-tune đầu tiên. TP giảm từ 197 xuống 44, FP giảm từ 16 xuống 0 nhưng FN tăng từ 206 lên 359. Precision đạt 1.0 không có nghĩa model tốt hơn, mà cho thấy model đang dự đoán quá dè dặt. Recall giảm từ 0.4888 xuống 0.1092 và F1 giảm từ 0.6396 xuống 0.1969. Recall của nhóm small giảm từ 0.1818 xuống 0, nhóm medium giảm từ 0.5473 xuống 0.0574. Chỉ nhóm large tăng từ 0.5610 lên 0.6585.

`outputs/compare_round1.jpg` cho thấy kết quả xấu đi rõ ràng ở `frame_0050`. Cold start có TP 11, FP 2 và FN 7, trong khi vòng 1 chỉ còn TP 4, FP 0 và FN 14. Các box đúng còn lại chủ yếu nằm trên xe lớn hoặc sáng. Nhiều xe nhỏ và trung bình bị bỏ sót, đúng với xu hướng của số liệu recall theo kích thước. Trước khi train tiếp, cần kiểm tra lại phân bố confidence, tính đại diện của 12 ảnh và khả năng model đã học quá sát lô nhỏ này.

`reports/BLIND_SCAN.md` ghi lại quan sát độc lập trước khi mình xem nhãn AI. Ở `frame_0331.jpg`, mình thấy khoảng 28 xe và nhận ra vùng xe xa cùng các mép ảnh là những chỗ dễ sai. Sau khi mở pre-label, `reports/REVIEW_LOG.csv` ghi lại những việc mình thực sự đã làm, gồm thêm xe nhỏ bị bỏ sót, xóa box trùng, chỉnh box xe xa và giữ một box đúng. `round1_diff.md` tổng hợp thay đổi nhãn của cả lô. Trong khi đó, `metrics_round1.json` và `compare_round1.jpg` mô tả model sau khi train. Đây là ba loại bằng chứng khác nhau và cần được đọc riêng.

Một trường hợp khó theo guideline là xe rất xa chỉ còn hai chấm đèn. Khi đó cần ước lượng phần thân quanh đèn thay vì chỉ khoanh hai điểm sáng. Nếu box cao dưới khoảng 16 px thì gán hay không đều được. Xe bị cắt ở mép dưới hoặc mép phải cũng khó vì box chỉ được ôm phần còn nằm trong ảnh, không kéo ra ngoài khung.

## 5. Kết luận và giới hạn

Mình dừng sau vòng bắt buộc. AP50 giảm từ 0.7714 xuống 0.5951, lớn hơn nhiều so với mức dao động nhỏ 0.01. Recall của nhóm small và medium cũng giảm mạnh. Nếu làm vòng 2 ngay lúc này, lỗi nhãn hoặc độ lệch của lô nhỏ có thể bị khuếch đại. Trước hết cần rà lại 141 box được thêm, 18 box bị xóa, cách vẽ box cho xe xa hoặc bị che và phân bố confidence của model sau train.

Nếu có vòng sau, mình sẽ ưu tiên kiểm tra hai nhóm. Nhóm thứ nhất là xe nhỏ hoặc tối gần đường chân trời, vì recall small đang bằng 0 nhưng việc rà từng xe khá tốn công và dễ thiếu nhất quán. Nhóm thứ hai là xe cỡ trung trong vùng chói hoặc bị che, cắt ở mép ảnh, vì recall medium chỉ đạt 0.0574. Khi lấy thêm ảnh cần tránh các frame cách nhau dưới 2 giây. Những frame này thường gần trùng, làm tăng công gán nhãn nhưng bổ sung rất ít thông tin. Mình sẽ chọn một số ít frame ở các đoạn thời gian khác nhau rồi kiểm tra chéo nhãn trước khi train.

Kết luận còn bị giới hạn vì tập test chỉ có 20 ảnh, 14/417 box quá nhỏ bị bỏ qua khi chấm và nhãn tham chiếu do mô hình tạo ra chưa được người rà thủ công. Vì vậy các số đo chỉ cho biết mức khớp với bộ tham chiếu này, không phải độ đúng tuyệt đối ngoài thực tế. Trước khi train thêm, mình sẽ xem lại nhãn vòng 1 trên ảnh gốc, kiểm tra box trùng hoặc thiếu và cách xử lý xe chỉ thấy đèn hay nằm sát mép. Sau đó mình sẽ xem learning curve, thử lại ngưỡng confidence và bổ sung một tập validation độc lập thay vì chỉ nhìn vào train mAP cao.
