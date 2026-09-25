# Vì sao chọn lô này?

## Năm frame ưu tiên nếu chỉ đủ công rà năm ảnh

Mình không chọn ngay năm dòng đầu vì các frame nằm sát nhau trên trục thời gian thường gần như trùng nhau. Trong 50 dòng đầu của `outputs/selection_round1.csv`, mình ưu tiên:

| ưu tiên | frame | rank trong CSV | thời điểm (s) | score | U | A | số box mập mờ | lý do |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | --- |
| 1 | `frame_0182.jpg` | 1 | 72.8 | 0.9591 | 0.9182 | 1.0000 | 18 | Điểm cao nhất và có nhiều box mập mờ nhất. Ảnh đông xe ở nhiều kích thước nên một lần rà có thể phát hiện được nhiều lỗi pre-label. |
| 2 | `frame_0369.jpg` | 2 | 147.6 | 0.9324 | 0.9315 | 0.8889 | 16 | Mức bất định cao trên 43 dự đoán. Contact sheet cho thấy giao thông dày ở cả hai chiều và có nhiều xe xa. |
| 3 | `frame_0326.jpg` | 4 | 130.4 | 0.9155 | 0.9310 | 0.8333 | 15 | Bất định cao nhưng nằm ở đoạn thời gian khác với hai frame trên, giúp giảm lặp lại cùng một trạng thái giao thông. |
| 4 | `frame_0099.jpg` | 8 | 39.6 | 0.9063 | 0.9460 | 0.7778 | 14 | U rất cao và ảnh nằm ở đoạn đầu video, giúp lô không tập trung hết vào cụm frame 130–153 giây. |
| 5 | `frame_0227.jpg` | 11 | 90.8 | 0.8915 | 0.9164 | 0.7778 | 14 | Điểm vẫn cao, nằm giữa các đoạn đã chọn và tránh tốn công cho một frame gần trùng với ưu tiên khác. |

Trong 50 dòng đầu không có frame `empty=True`, nên quyết định này không dựa vào phần thưởng dành cho frame không có dự đoán.

## Ba frame thuộc lô 12 ảnh model chọn

- `frame_0182.jpg` đứng hạng 1 với score 0.9591. Thành phần A đạt 1.0 vì có 18 box mập mờ. Đây là ví dụ rõ nhất về việc bộ chọn ưu tiên ảnh có nhiều dự đoán ở mức confidence trung gian.
- `frame_0369.jpg` đứng hạng 2 với score 0.9324, U = 0.9315 và 43 box dự đoán. Contact sheet cho thấy nhiều xe có kích thước và độ sáng khác nhau, nên chi phí rà cao nhưng cũng có nhiều quyết định nhãn cần kiểm tra.
- `frame_0099.jpg` đứng hạng 8 với score 0.9063 và U = 0.9460. Dù không nằm trong năm hạng đầu, frame này ở giây 39.6, tách xa các cụm 72–75 và 130–153 giây, nên có ích cho đa dạng thời gian.

## Một frame điểm cao nhưng không chọn

`frame_0372.jpg` đứng hạng 6, score 0.9101 tại giây 148.8 nhưng không thuộc lô. Nó chỉ cách `frame_0369.jpg` ở giây 147.6 có 1.2 giây, nhỏ hơn `MIN_GAP_S = 2.0`. Với camera cố định, hai ảnh có thể chứa gần như cùng xe và bố cục. Rà cả hai sẽ tốn thêm công nhưng mang lại ít thông tin mới.

## Điều phép chọn chưa chứng minh

Score là điểm tổng hợp từ độ bất định, số box mập mờ và độ đa dạng theo thời gian. Điểm cao không đồng nghĩa nhãn AI chắc chắn sai, cũng không bảo đảm ảnh đó sẽ giúp model tốt hơn. Muốn biết lô ảnh có ích hay không vẫn phải rà nhãn, huấn luyện với cùng cấu hình rồi đo lại trên cùng tập kiểm thử. Ở vòng 1, AP50 thực tế đã giảm dù các frame được chọn có điểm bất định cao.
