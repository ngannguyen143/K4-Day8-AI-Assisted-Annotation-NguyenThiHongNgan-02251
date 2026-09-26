# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có
ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét
ảnh gần trùng hoặc trường hợp model không dự đoán được box: Tôi ưu tiên năm frame sau, xét theo
điểm trong 50 dòng đầu nhưng vẫn kiểm tra sự trùng cảnh để tránh dùng cả ngân sách cho một đoạn
video gần như giống nhau:

| Ưu tiên | Frame | Thời điểm (s) | Hạng | Điểm | U | A | Số box | Box mơ hồ | Lý do |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | --- |
| 1 | `frame_0182.jpg` | 72.8 | 1 | 0.9591 | 0.9182 | 1.0000 | 28 | 18 | Điểm cao nhất; có nhiều box mơ hồ nên có khả năng thu được nhiều sửa chữa có ích. |
| 2 | `frame_0369.jpg` | 147.6 | 2 | 0.9324 | 0.9315 | 0.8889 | 43 | 16 | Điểm và U đều rất cao; nhiều xe và nhiều box cần kiểm tra, phù hợp để phát hiện cả bỏ sót lẫn box lệch. |
| 3 | `frame_0380.jpg` | 152.0 | 3 | 0.9170 | 0.9340 | 0.8333 | 40 | 15 | U cao nhất trong năm frame; bổ sung một thời điểm khác của đoạn cuối video thay vì chỉ dựa vào A. |
| 4 | `frame_0326.jpg` | 130.4 | 4 | 0.9155 | 0.9310 | 0.8333 | 39 | 15 | Điểm cao, số box lớn và U cao; là ứng viên có chi phí rà nhãn đáng kể nhưng giá trị kiểm tra cũng cao. |
| 5 | `frame_0331.jpg` | 132.4 | 5 | 0.9154 | 0.8308 | 1.0000 | 47 | 18 | Nhiều box nhất trong top 5 và nhiều box mơ hồ nhất; ưu tiên để kiểm tra cảnh đông xe, dù gần `frame_0326.jpg`. |

Tôi không chọn thêm các frame cách nhau quá ngắn chỉ vì điểm cao. Cụ thể, `frame_0369.jpg`
ở 147.6 s được chọn, còn `frame_0372.jpg` ở 148.8 s có điểm 0.9101 (hạng 6) nhưng không
được chọn; khoảng cách chỉ 1.2 s, nhỏ hơn `min_gap_s = 2.0` trong `to_label/round1/batch.json`.
Contact sheet cho thấy đây là cặp khung gần nhau trong cùng đoạn cảnh, nên chọn cả hai sẽ làm
giảm độ đa dạng của năm lần rà. Tương tự, `frame_0326.jpg` và `frame_0331.jpg` cách 2.0 s;
tôi vẫn giữ `frame_0331.jpg` vì nó có 47 box và 18 box mơ hồ, nhưng đây là điểm cần ghi nhận
về chi phí và nguy cơ trùng thông tin.

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet: `frame_0182.jpg`
đứng hạng 1 với điểm 0.9591, U = 0.9182, A = 1.0, có 28 box và 18 box mơ hồ; đây là
ứng viên có tín hiệu uncertainty mạnh nhất ở đầu danh sách. `frame_0369.jpg` đứng hạng 2,
điểm 0.9324 và U = 0.9315, có 43 box/16 box mơ hồ; contact sheet đặt nó trong đoạn cuối
video nơi các khung liền kề cần được xem xét để tránh chọn trùng. `frame_0331.jpg` đứng hạng
5, điểm 0.9154, có 47 box và 18 box mơ hồ; số lượng box cao cho thấy việc rà sẽ tốn công,
nhưng cũng có nhiều cơ hội phát hiện box thiếu hoặc sai. Cả ba đều thuộc danh sách 12 frame
trong `to_label/round1/batch.json`; CSV cho thấy `empty = False` và `D = 1.0`, nên quyết
định này dựa vào uncertainty/ambiguity và độ đa dạng thời gian, không phải ảnh không có box.

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do:
`frame_0372.jpg` là trường hợp điểm cao nhưng không chọn: điểm 0.9101, U = 0.9202, hạng 6,
42 box và 15 box mơ hồ, nhưng chỉ cách `frame_0369.jpg` 1.2 s và bị ràng buộc khoảng cách
tối thiểu. Nếu còn ngân sách, đây là frame nên xem bổ sung để kiểm tra độ ổn định của nhãn giữa
hai khung gần trùng. Ngược lại, trong 50 dòng đầu không có frame nào `empty = True`; vì vậy
không thể dùng bảng này để tuyên bố model đã bỏ sót hoàn toàn một ảnh. Việc chọn frame điểm
thấp chỉ nên làm khi muốn kiểm tra một vùng thời gian khác hoặc một failure mode cụ thể.

Điều phép chọn này chưa chứng minh về chất lượng mô hình: Điểm `score` chỉ là tiêu chí ưu tiên
do chiến lược uncertainty tổng hợp từ U, A và D; nó không phải precision, recall hay AP.
Các frame được chọn cũng không chứng minh rằng nhãn của model đúng: ở vòng 1, 12 ảnh có 169
box pre-label nhưng sau khi rà còn 321 box, gồm 18 box được sửa, 11 box bị xóa và 163 box
được thêm (`outputs/round1_diff.md`). Vì vậy top 5 chỉ tối ưu hóa giá trị rà nhãn trong ngân
sách giả định, còn chất lượng mô hình phải được đánh giá riêng trên 20 ảnh test cố định.

Kết quả mới nhất trong notebook cho thấy cold start đạt AP50 = 0.771, precision = 0.925,
recall = 0.489 và F1 = 0.640; sau fine-tune trên 12 ảnh/321 box, vòng 1 chỉ đạt AP50 =
0.378, precision = 1.000, recall = 0.025 và F1 = 0.048. Như vậy vòng 1 giảm 0.393 AP50
so với cold start, không thể kết luận rằng lô được chọn đã cải thiện model. Bảng notebook còn
cho thấy recall xe nhỏ giảm từ 0.182 xuống 0.000, xe trung bình từ 0.547 xuống 0.013 và xe
lớn từ 0.561 xuống 0.146. Đây là tín hiệu nên dừng hoặc đổi chiến lược chọn mẫu, đồng thời
kiểm tra lại nhãn và cấu hình fine-tune trước khi chọn vòng tiếp theo. Các số mới lấy từ output
đánh giá trên test trong `notebooks/day8_active_learning.ipynb`; không dùng mAP trên tập train
làm số đo chất lượng.
