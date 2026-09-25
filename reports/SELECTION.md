# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có
ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét
ảnh gần trùng hoặc trường hợp model không dự đoán được box:

Nếu chỉ có ngân sách rà đúng 5 ảnh, ta không thể chọn mù quáng top 5 dòng đầu theo score (từ rank 1 đến rank 5) vì các frame rank 4 (`frame_0326.jpg` tại 130.4s) và rank 5 (`frame_0331.jpg` tại 132.4s) nằm quá sát nhau (cách 2.0s), gây hiện tượng cụm cảnh (temporal clustering) làm lãng phí 40% ngân sách vào cùng một thời điểm ùn tắc. Danh sách 5 frame ưu tiên được chọn lọc cân bằng giữa điểm bất định, số box mập mờ và độ bao phủ toàn diện trục thời gian (từ 0s đến 160s):

1. **`frame_0182.jpg`** (Thứ tự 1 | Rank 1 | Điểm: 0.9591 | Thời điểm: 72.8s | U: 0.9182 | A: 1.0000 | 28 box, 18 box mập mờ):
   - *Lý do:* Điểm score cao nhất toàn bộ pool (0.9591), số box mập mờ đạt mức tối đa trong pool (A = 1.0 với 18 box có conf trong khoảng 0.15–0.50), U rất cao (0.9182). Nằm ở giữa video (t = 72.8s), đại diện cho cảnh giao thông mật độ vừa nhưng nhiều xe bị đèn pha làm lóa mắt camera, rất cần con người can thiệp chuẩn hóa.
2. **`frame_0369.jpg`** (Thứ tự 2 | Rank 2 | Điểm: 0.9324 | Thời điểm: 147.6s | U: 0.9315 | A: 0.8889 | 43 box, 16 box mập mờ):
   - *Lý do:* Điểm cao thứ hai toàn pool, độ bất định U = 0.9315 rất cao. Mật độ xe cực lớn (43 box dự đoán, 16 box mập mờ) tại phân đoạn cuối video (t = 147.6s). Đại diện cho tình huống giao thông ùn ứ dày đặc, nhiều xe đan xen ở khoảng cách xa gần.
   - *Quyết định loại trừ ảnh gần trùng:* Việc chọn `frame_0369.jpg` dẫn đến quyết định dứt khoát **loại bỏ** `frame_0372.jpg` (Rank 6, score 0.9101, t = 148.8s, cách 1.2s < MIN_GAP_S 2.0s) và `frame_0368.jpg` (Rank 9, score 0.9003, t = 147.2s, cách 0.4s). Do camera cố định, các xe chỉ nhích vài mét sau 1.2s; loại bỏ các frame gần trùng này giúp tiết kiệm ngân sách quý giá cho các phân đoạn khác.
3. **`frame_0099.jpg`** (Thứ tự 3 | Rank 8 | Điểm: 0.9063 | Thời điểm: 39.6s | U: 0.9460 | A: 0.7778 | 29 box, 14 box mập mờ):
   - *Lý do:* Đại diện quan trọng cho phân đoạn đầu video (t = 39.6s), tránh việc toàn bộ tập train bị dồn về nửa sau video. Frame này có độ bất định U cực cao (0.9460, cao hơn cả rank 1 và rank 2), chứa 14 box mập mờ trên tổng số 29 box dự đoán.
4. **`frame_0326.jpg`** (Thứ tự 4 | Rank 4 | Điểm: 0.9155 | Thời điểm: 130.4s | U: 0.9310 | A: 0.8333 | 39 box, 15 box mập mờ):
   - *Lý do:* Đại diện cho cao điểm ùn ứ tại mốc t ~ 130s với 39 box dự đoán và 15 box mập mờ, U = 0.9310.
   - *Quyết định tránh trùng cụm:* Giữa `frame_0326.jpg` (rank 4, 130.4s) và `frame_0331.jpg` (rank 5, 132.4s, score 0.9154), ta chỉ chọn duy nhất `frame_0326.jpg` và bỏ qua `frame_0331.jpg` cùng `frame_0330.jpg` (rank 12, 132.0s). Dù khoảng cách 130.4s và 132.4s vừa chạm ngưỡng 2.0s của thuật toán tự động, nhưng với ngân sách chỉ 5 ảnh của con người, giữ cả hai sẽ chiếm tới 40% công sức cho cùng một cảnh kẹt xe tương tự.
5. **`frame_0270.jpg`** (Thứ tự 5 | Rank 13 | Điểm: 0.8878 | Thời điểm: 108.0s | U: 0.9089 | A: 0.7778 | 35 box, 14 box mập mờ):
   - *Lý do:* Đóng vai trò là cầu nối đa dạng thời gian ở mốc t = 108.0s (nằm giữa mốc 72.8s và 130.4s). Frame có 35 box dự đoán, 14 box mập mờ, U = 0.9089. Giúp phân bổ 5 ảnh trải đều tuyệt đối theo trục thời gian (~40s, ~73s, ~108s, ~130s, ~148s), bao trọn các trạng thái lưu thông khác nhau. Ngoài ra, đây chính là frame học viên đã quét độc lập trong `BLIND_SCAN.md`, phát hiện nhiều xe tối màu và xe ở xa bị mô hình bỏ sót.

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet:

1. **`frame_0182.jpg`**:
   - *Bằng chứng CSV:* Rank 1, score = 0.9591 (cao nhất pool), t_sec = 72.8s, U = 0.9182, A = 1.0000 (18 box mập mờ – cao nhất toàn bộ 268 frame), n_boxes = 28, empty = False, selected = True.
   - *Bằng chứng contact sheet (`selection_round1.jpg`):* Nằm ở hàng 1, ô thứ 3. Ảnh chụp đoạn đường ban đêm có dải đèn pha xe ngược chiều và luồng xe làn giữa rực sáng làm lóa camera. Nhiều xe phía sau và bên rìa bị bóng tối hoặc ánh lóa che khuất một phần, khiến mô hình phân vân ranh giới box (nhiều box conf rơi vào dải 0.15–0.50).
2. **`frame_0369.jpg`**:
   - *Bằng chứng CSV:* Rank 2, score = 0.9324, t_sec = 147.6s, U = 0.9315, A = 0.8889 (16 box mập mờ), n_boxes = 43 (mật độ box rất cao), empty = False, selected = True.
   - *Bằng chứng contact sheet (`selection_round1.jpg`):* Nằm ở hàng 2, ô thứ 4. Khung cảnh ùn ứ xe dày đặc ở giai đoạn cuối clip, các vệt đèn đỏ đuôi xe bên phải và đèn pha trắng bên trái ken đặc. Model nhận diện được tới 43 box nhưng gặp khó khăn lớn trong việc tách biệt các xe đi sát nhau ở hậu cảnh.
3. **`frame_0270.jpg`**:
   - *Bằng chứng CSV:* Rank 13, score = 0.8878, t_sec = 108.0s, U = 0.9089, A = 0.7778 (14 box mập mờ), n_boxes = 35, empty = False, selected = True.
   - *Bằng chứng contact sheet (`selection_round1.jpg`):* Nằm ở hàng 1, ô thứ 6. Bối cảnh có xe tải lớn di chuyển ở làn giữa, cùng nhiều đốm sáng đỏ của xe ở xa làn bên phải lọt thỏm trong bóng tối. Model có điểm bất định cao (U > 0.90) và khoảng cách thời gian lớn (D = 1.0) so với các cụm trước, giúp bổ sung biến thiên cảnh quan quý giá cho tập huấn luyện.

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do:

- **Frame điểm cao nhưng KHÔNG chọn: `frame_0372.jpg`**
  - *Thông số:* Rank 6, score = 0.9101 (đứng thứ 6/268 frame trong pool), U = 0.9202, A = 0.8333, n_boxes = 42, n_ambiguous = 15, nhưng `selected = False`.
  - *Lý do:* Thời điểm t_sec = 148.8s của `frame_0372.jpg` chỉ cách `frame_0369.jpg` (Rank 2, t = 147.6s, đã được chọn) đúng **1.2 giây**, nhỏ hơn ngưỡng khoảng cách thời gian tối thiểu `MIN_GAP_S = 2.0s`. Do camera giám sát đặt cố định, khoảng cách 1.2s khiến các phương tiện hầu như chưa thay đổi vị trí đáng kể, góc chiếu sáng và hậu cảnh gần như trùng lặp hoàn toàn. Việc chọn frame này sẽ làm lãng phí chi phí gán nhãn của con người vào một dữ liệu dư thừa (redundant data) mà không cung cấp thêm tri thức mới cho mô hình. (Tương tự với `frame_0368.jpg` rank 9 cách 0.4s và `frame_0330.jpg` rank 12 cách 0.4s).

- *(Mở rộng) Frame điểm thấp nhưng VẪN NÊN XEM: `frame_0000.jpg` (hoặc `frame_0014.jpg`)*
  - *Thông số:* `frame_0000.jpg` (Rank 92, score = 0.7870, t = 0.0s, n_boxes = 24, n_ambiguous = 8, A = 0.4444).
  - *Lý do:* Điểm score bị kéo thấp do số box mập mờ ít (A thấp vì đường thông thoáng hơn, ít xe dồn cục). Tuy nhiên, đây là khung cảnh đầu tiên của video với luồng xe di chuyển tốc độ nhanh, có thể xuất hiện hiện tượng nhòe chuyển động (motion blur) hoặc góc chụp thoáng nhìn rõ thân xe thay vì chỉ thấy đèn xe. Nếu chỉ dựa vào score để chọn những ảnh ùn tắc đèn chói lọi, mô hình sẽ bị lệch phân phối (distribution bias) và kém nhận diện ở các khung cảnh thông thoáng.

Điều phép chọn này chưa chứng minh về chất lượng mô hình:

1. **Điểm bất định cao không đồng nghĩa với việc cải thiện chất lượng mô hình:** Điểm bất định (Uncertainty) chỉ phản ánh mức độ thiếu tự tin của mô hình hiện tại (confidence gần 0.50), chứ không bảo đảm rằng việc con người gán nhãn và nạp frame đó vào huấn luyện sẽ chắc chắn làm tăng AP50 hay cải thiện khả năng tổng quát hóa.
2. **Nhiễu thị giác không thể học (Aleatoric uncertainty):** Điểm bất định cao có thể sinh ra từ các yếu tố nhiễu thuần túy như ánh đèn pha rọi thẳng vào ống kính gây lóa diện rộng, mặt đường ướt phản chiếu ánh sáng giả, mắt lưới rào sắt chắn trước camera, hoặc các chấm sáng ở đường chân trời chỉ vỏn vẹn vài pixel không đủ thông tin nhận dạng. Việc ép mô hình học những mẫu nhiễu này thậm chí có thể gây hại (overfitting hoặc làm tăng False Positive).
3. **Phép chọn chỉ là hàm thu thập mẫu (Acquisition function), không phải thước đo hiệu năng:** Công thức `score = W_U·U + W_A·A + W_D·D` là thuật toán heuristic dùng để sàng lọc mẫu ứng viên. Chất lượng thực tế của mô hình chỉ có thể được chứng minh qua kiểm thử thực nghiệm trên một tập test độc lập, chuẩn hóa với các độ đo khách quan (mAP, Precision, Recall theo kích thước box).
4. **Giới hạn của tập test và nhãn tham chiếu:** Việc chọn lô ảnh này chưa chứng minh mô hình sẽ vượt trội trong môi trường thực tế vì tập test hiện tại chỉ có 20 ảnh, và nhãn tham chiếu test ban đầu do mô hình khác tạo ra (chưa được con người rà soát toàn diện), nên sự biến thiên số đo sau một vòng chưa thể khẳng định chắc chắn về năng lực thật sự của mô hình.

