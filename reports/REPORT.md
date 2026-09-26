# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Nguyễn Thị Hồng Ngân

Công cụ gán nhãn đã dùng: CVAT

Báo cáo này dùng số liệu từ `outputs/metrics_round0.json`, `outputs/round1_diff.md`,
`reports/SELECTION.md` và các kết quả đã lưu trong `notebooks/day8_active_learning.ipynb`.
Nhãn test do mô hình tạo chỉ là nhãn tham chiếu, không phải chân lý tuyệt đối.

## 1. Dữ liệu và cách chia tập

Tại sao tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian, có vùng
đệm ở giữa, thay vì chia ngẫu nhiên? Nếu chia ngẫu nhiên, số đo trên tập kiểm thử sẽ bị lệch theo
hướng nào, và vì sao?

Pool và test được chia theo trục thời gian, có vùng đệm, để giảm khả năng các khung hình gần
nhau của cùng một cảnh xuất hiện ở cả train/pool và test. Nếu chia ngẫu nhiên, các khung gần
nhau có thể chứa cùng xe, góc nhìn và điều kiện ánh sáng; mô hình sẽ được đánh giá trên ảnh rất
giống ảnh đã thấy, nên AP50 và recall có thể bị cao giả tạo. Cách chia theo thời gian khó hơn
nhưng đo tốt hơn khả năng tổng quát sang đoạn video khác. Tập test gồm 20 ảnh với 403 box tham
chiếu; 14 box rất nhỏ bị bỏ qua khi chấm.

## 2. Mô hình khởi đầu lạnh (cold start)

Chép dòng vòng 0 từ `rounds_table.md`. Dựa vào `outputs/compare_round0.jpg`, cho biết mô hình khởi
đầu lạnh không khớp nhãn tham chiếu ở những loại xe nào. Độ phủ (recall) theo kích thước xe cho
thấy điều gì? Một trường hợp nào cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai?

Vòng 0: `yolov8n cold start (COCO car+bus+truck) | 0 ảnh train | 0 box train | AP50 0.771 |
P 0.925 | R 0.489 | F1 0.640 | R nhỏ 0.182 | R vừa 0.547 | R lớn 0.561`.

Theo `metrics_round0.json` (và ảnh so sánh round 0), model nhận tốt hơn các xe vừa/lớn nhưng
bỏ sót nhiều xe nhỏ, xa hoặc bị che/khuất trong cảnh ban đêm. Recall xe nhỏ chỉ 0.182 so với
0.547 ở xe vừa và 0.561 ở xe lớn; tổng FN là 206 so với 197 TP. Tuy nhiên cần rà lại các box
tham chiếu rất nhỏ trước khi gọi đó là lỗi model, vì 14 box cao dưới 16 px bị bỏ qua và việc
nhìn hai điểm đèn trong ảnh đêm có thể không đủ để xác định đường viền xe theo guideline.

## 3. Chiến lược chọn mẫu

Giải thích bằng lời công thức `score = W_U·U + W_A·A + W_D·D` và vai trò của `MIN_GAP_S`.
Dẫn ba frame trong `reports/SELECTION.md` và một frame khác để chứng minh cách bạn cân nhắc
độ bất định, ảnh gần trùng và công gán nhãn. Điểm bất định có chứng minh ảnh đó sẽ cải thiện
mô hình không? Vì sao?

Điểm chọn mẫu được tính theo `score = W_U·U + W_A·A + W_D·D`: U đo độ bất định, A đo mức
mơ hồ của các dự đoán, còn D khuyến khích độ đa dạng; các trọng số W quyết định mức ảnh hưởng
của từng thành phần. `MIN_GAP_S = 2.0` loại các ứng viên quá gần nhau theo thời gian, giúp
ngân sách rà nhãn không bị dồn vào cùng một cảnh. Vì vậy tôi chọn `frame_0182.jpg`,
`frame_0369.jpg`, `frame_0380.jpg`, `frame_0326.jpg` và `frame_0331.jpg` như đã giải thích
trong `reports/SELECTION.md`, đồng thời xem `frame_0372.jpg` là ứng viên điểm cao nhưng bị
loại do chỉ cách `frame_0369.jpg` 1.2 giây. Ba ví dụ chính là `frame_0182.jpg` (score 0.9591,
18 box mơ hồ), `frame_0369.jpg` (0.9324, 43 box) và `frame_0331.jpg` (0.9154, 47 box);
`frame_0372.jpg` chứng minh cách cân nhắc ảnh gần trùng và chi phí rà.

Điểm bất định chỉ giúp xếp hạng nơi nên kiểm tra, không chứng minh ảnh đó sẽ cải thiện model.
Kết quả vòng 1 thực tế giảm mạnh, cho thấy việc chọn đúng ảnh vẫn cần nhãn chính xác, cách vẽ
nhất quán và cấu hình fine-tune phù hợp.

## 4. Các vòng học chủ động (active learning)

Chép bảng từ `rounds_table.md`. Với mỗi vòng, trình bày:

- mức độ bạn đã sửa nhãn gợi ý (số box giữ nguyên, chỉnh sửa, xoá, thêm mới, lấy từ
  `outputs/round*_diff.md`);
- AP50 thay đổi bao nhiêu so với khởi đầu lạnh và so với vòng trước;
- nhóm xe nào tốt lên hoặc xấu đi theo số đo trên cùng tập test.

Dựa vào các ảnh `compare_round*.jpg`, chỉ ra một ca kết quả đổi sau fine-tune (tốt hơn hoặc xấu
đi), cùng lý do có thể kiểm. Dùng `BLIND_SCAN.md`, `REVIEW_LOG.csv` và `round1_diff.md` phân biệt
quan sát độc lập, lỗi pre-label đã sửa và kết quả mô hình sau train. Mô tả một ca khó theo guideline.

### Bảng kết quả trên cùng tập test

| Vòng | Model | Ảnh train | Box train | AP50 | Δ so cold start | P | R | F1 | R nhỏ | R vừa | R lớn |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vòng 1 | 12 | 321 | 0.378 | -0.393 | 1.000 | 0.025 | 0.048 | 0.000 | 0.013 | 0.146 |

Ở vòng 1, model đề xuất 169 box trên 12 ảnh. Sau khi rà bằng CVAT còn 321 box: giữ nguyên
140, chỉnh 18, xóa 11 và thêm 163, accept rate 83% (`outputs/round1_diff.md`). Các ca này
cho thấy pre-label ban đầu bỏ sót nhiều xe: `frame_0099.jpg` thêm 11 box, `frame_0369.jpg`
thêm 22 box và `frame_0326.jpg` thêm 17 box. Một điểm sáng bị model nhầm thành xe ở
`frame_0326.jpg` đã bị xóa; đây là lỗi false positive được `REVIEW_LOG.csv` ghi lại. Một ca khó
theo guideline là xe bị cắt ở mép ảnh trong `frame_0182.jpg`: chỉ vẽ phần thân nằm trong ảnh,
không cố dựng phần bị khuất. Ca độc lập trước khi xem pre-label là `frame_0099`: nhìn thấy 26
xe, trong đó có vị trí khuất ở góc trái chỉ thấy bóng đèn.

Kết quả sau fine-tune xấu đi rõ rệt trên test: AP50 giảm 0.393, recall giảm 0.489 xuống 0.025;
recall xe nhỏ giảm 0.182 xuống 0, xe vừa 0.547 xuống 0.013 và xe lớn 0.561 xuống 0.146.
Notebook không có `outputs/compare_round1.jpg` trong repo hiện tại, nên chưa thể chỉ ra một
thay đổi hình ảnh cụ thể sau fine-tune; kết luận xấu đi ở đây chỉ dựa trên bảng đánh giá test
trong notebook. Cần xuất lại ảnh so sánh để kiểm tra xem model có học lệch, box bị co/mất hay
cấu hình train đã không phù hợp.

## 5. Kết luận và giới hạn

Kết quả vòng này so với cold start ra sao? Vì sao bạn dừng hoặc tiếp tục? Đề xuất hai ca còn yếu
hoặc bất định cho vòng sau, kèm chi phí rà nhãn và nguy cơ ảnh gần trùng. Tập kiểm thử chỉ 20 ảnh,
có luật bỏ qua xe quá nhỏ và nhãn tham chiếu do mô hình tạo chưa được rà thủ công; các giới hạn đó
ảnh hưởng thế nào đến kết luận? Nếu AP50 giảm, bạn sẽ kiểm tra điều gì trước khi train thêm?

Vòng 1 không cải thiện so với cold start nên tôi dừng việc chọn tiếp theo cấu hình hiện tại.
Hai ca nên ưu tiên kiểm tra nếu làm vòng sau là: (1) `frame_0369.jpg`, vì có 43 box, U =
0.9315 và sau sửa phải thêm 22 box, cho thấy pre-label bỏ sót nhiều; (2) `frame_0331.jpg`,
vì có 47 box và 18 box mơ hồ, đồng thời gần `frame_0326.jpg`, nên vừa có giá trị kiểm tra
cảnh đông xe vừa có nguy cơ trùng cảnh và chi phí rà cao. Cần tránh chọn thêm `frame_0372.jpg`
nếu vẫn giữ khoảng cách tối thiểu 2 giây.

Kết luận bị giới hạn bởi chỉ 20 ảnh test, luật bỏ qua 14 xe quá nhỏ và nhãn tham chiếu do model
tạo chưa được rà thủ công. Do đó AP50 có thể thay đổi đáng kể theo vài ca nhỏ hoặc khó; không
nên xem điểm 0.378 là đo lường tuyệt đối về năng lực thật. Trước khi train thêm, tôi sẽ kiểm tra
đầu tiên: file nhãn đã sửa có đúng class id 0 và đúng định dạng YOLO không; ảnh và nhãn có ghép
đúng tên không; tỷ lệ box bị thêm rất lớn có phản ánh lỗi pre-label hay lỗi gán nhãn không; và
đường dẫn/config, số epoch, checkpoint dùng để đánh giá có đúng vòng 1 không. Sau đó mới cân
nhắc đổi chiến lược chọn mẫu hoặc train lại. Chi phí rà nhãn và ảnh gần trùng cần được tính
trong quyết định, thay vì chọn mọi frame có score cao.
