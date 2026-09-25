# Quét độc lập trước khi xem pre-label

Frame: `frame_0331.jpg`

Số xe nhìn thấy bằng mắt: khoảng 28 xe, tính cả các xe chỉ xuất hiện một phần ở mép ảnh.

Hai vị trí dễ bị AI bỏ sót hoặc vẽ sai, kèm mô tả xe:

1. Cụm xe ở xa phía trên–giữa khung hình: xe rất nhỏ, chủ yếu chỉ thấy hai đèn hậu đỏ sát nhau nên dễ bị bỏ sót hoặc gộp nhầm nhiều xe vào một box.
2. Mép dưới và mép phải khung hình: có xe ở rất gần camera nhưng bị cắt mất một phần thân và nhòe do chuyển động/ánh đèn, nên box dễ vượt khỏi ảnh hoặc không ôm đúng phần xe còn nhìn thấy.

Chạy `python3 tools/lock_blind.py` ngay sau khi điền. Sau đó giữ file này nguyên vẹn.
