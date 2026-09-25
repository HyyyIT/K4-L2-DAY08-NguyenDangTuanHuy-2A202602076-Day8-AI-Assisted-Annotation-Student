# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Nguyễn Đăng Tuấn Huy
Công cụ gán nhãn đã dùng: sửa trực tiếp file nhãn

Báo cáo thực hành được hoàn thiện dựa trên số liệu thực nghiệm. Mọi con số được truy xuất chính xác
từ `reports/rounds_table.md`, `outputs/selection_round1.csv`, `outputs/metrics_round*.json` và
`outputs/round*_diff.md`. Không coi nhãn test do mô hình tạo là chân lý tuyệt đối.


## 1. Dữ liệu và cách chia tập

Tại sao tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian, có vùng
đệm ở giữa, thay vì chia ngẫu nhiên? Nếu chia ngẫu nhiên, số đo trên tập kiểm thử sẽ bị lệch theo
hướng nào, và vì sao?

Tập chưa gán nhãn (pool) và tập kiểm thử (test set) bắt buộc phải được chia theo trục thời gian kèm vùng đệm (buffer zone) ở giữa vì các lý do cốt lõi sau:
1. **Tránh rò rỉ dữ liệu do tương quan thời gian (Temporal Data Leakage):** Dữ liệu được trích xuất từ chuỗi video giám sát giao thông với tần suất lấy mẫu dày đặc (mỗi frame cách nhau chỉ 0.4 giây). Trong một đoạn video từ camera cố định, các frame lân cận có nền cảnh, điều kiện ánh sáng, góc chiếu và chính các phương tiện giao thông gần như không đổi.
2. **Vai trò của vùng đệm (Buffer Zone):** Việc đặt các khoảng đệm (như được cấu hình trong `data/frames.csv`, ví dụ các frame buffer giữa pool và test) nhằm đảm bảo các luồng xe đang lưu thông trong pool có đủ thời gian rời khỏi tầm nhìn của camera trước khi các frame thuộc test set xuất hiện, triệt tiêu sự trùng lặp đối tượng.
3. **Hướng lệch của số đo nếu chia ngẫu nhiên:** Nếu chia ngẫu nhiên (random split), số đo trên tập kiểm thử (như AP50, Precision, Recall) sẽ bị **thổi phồng và lệch theo hướng lạc quan giả tạo (artificially inflated / overly optimistic)**. Khi đó, mô hình chỉ cần "học vẹt" (overfitting) các xe cụ thể, góc đèn và bối cảnh ở frame train là đã có thể nhận diện chính xác trên frame test nằm ngay sát cạnh (cách vài phần mười giây). Điều này khiến số đo cao ngất ngưởng trong phòng thí nghiệm nhưng mô hình sẽ thất bại hoàn toàn (poor generalization) khi triển khai trên các phân đoạn thời gian thực tế mới.

## 2. Mô hình khởi đầu lạnh (cold start)

Chép dòng vòng 0 từ `rounds_table.md`. Dựa vào `outputs/compare_round0.jpg`, cho biết mô hình khởi
đầu lạnh không khớp nhãn tham chiếu ở những loại xe nào. Độ phủ (recall) theo kích thước xe cho
thấy điều gì? Một trường hợp nào cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai?

Dòng vòng 0 trích xuất nguyên văn từ `reports/rounds_table.md`:
```markdown
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
```

- **Mô hình khởi đầu lạnh không khớp nhãn tham chiếu ở những loại xe nào:**
  Dựa vào `outputs/compare_round0.jpg` và `outputs/metrics_round0.json`, mô hình khởi đầu lạnh (pretrained YOLOv8n trên tập COCO) gặp lỗi nghiêm trọng ở:
  1. *Xe ở khoảng cách xa / kích thước nhỏ:* Các xe ở hậu cảnh chỉ lộ 2 đốm sáng đèn pha hoặc vệt đèn hậu nhỏ mờ nhạt, thân xe chìm trong bóng tối. Mô hình bỏ sót hầu hết các xe này (False Negatives - khung màu đỏ/cam).
  2. *Xe ở làn đường biên tối:* Các xe di chuyển ở làn ngoài cùng bên phải hoặc làn đối diện bị thiếu đèn đường chiếu rọi trực tiếp.
  3. *Lỗi gộp box:* Ở các vị trí ùn ứ hoặc hai xe chạy song song quá sát nhau, mô hình đôi khi chỉ phát hiện một xe hoặc vẽ box lệch ranh giới.
- **Độ phủ (Recall) theo kích thước xe cho thấy:**
  - `R small` = **0.1818** (18.18% - chỉ bắt được 12/66 box xe nhỏ).
  - `R medium` = **0.5473** (54.73% - bắt được 162/296 box xe vừa).
  - `R large` = **0.5610** (56.10% - bắt được 23/41 box xe lớn).
  Số đo này chứng minh độ phủ tỷ lệ thuận rõ rệt với kích thước xe. Trọng tâm yếu kém nhất của mô hình ban đầu là nhóm xe nhỏ (bỏ sót hơn 81.8% số xe nhỏ), bởi tập dữ liệu COCO chủ yếu chụp ban ngày với các góc chụp cận cảnh, không có đặc trưng của các chấm sáng đèn ban đêm trên cao tốc.
- **Trường hợp cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai:**
  Trong `outputs/compare_round0.jpg`, có những vật thể ở rất xa gần đường chân trời chỉ hiện diện như một đốm sáng nhòe 1–2 pixel hoặc vệt đèn phản chiếu trên dải phân cách/mặt đường ướt. Nhãn tham chiếu (vốn được tạo tự động bởi mô hình khác) đã gán nhãn cho các đốm sáng này. Khi mô hình cold start không dự đoán box tại đó, nó bị tính phạt là False Negative. Trong trường hợp này, con người cần trực tiếp rà soát nhãn tham chiếu xem đó có thực sự là xe hợp lệ theo quy tắc kích thước tối thiểu và độ che khuất (theo `GUIDELINE_LABEL.md`) hay chỉ là box giả của bộ sinh nhãn tự động.

## 3. Chiến lược chọn mẫu

Giải thích bằng lời công thức `score = W_U·U + W_A·A + W_D·D` và vai trò của `MIN_GAP_S`.
Dẫn ba frame trong `reports/SELECTION.md` và một frame khác để chứng minh cách bạn cân nhắc
độ bất định, ảnh gần trùng và công gán nhãn. Điểm bất định có chứng minh ảnh đó sẽ cải thiện
mô hình không? Vì sao?

- **Giải thích công thức tính điểm ưu tiên:**
  Công thức `score = W_U·U + W_A·A + W_D·D` (với trọng số mặc định $W_U = 0.5$, $W_A = 0.3$, $W_D = 0.2$ trong `tools/al_select.py`) kết hợp 3 tiêu chí:
  1. $U$ (Uncertainty): Trung bình độ bất định của 5 box khó nhất trong frame ($u(c) = 1 - |2c - 1|$). Độ bất định đạt cực đại bằng 1 khi confidence $c = 0.50$ (lúc mô hình phân vân nhất giữa có xe hay không). Trọng số lớn nhất ($0.5$) ưu tiên các ảnh khiến mô hình hoang mang nhất.
  2. $A$ (Ambiguity Density): Tỷ lệ số box mập mờ ($0.15 \le c < 0.50$) của frame chia cho số box mập mờ cao nhất trong toàn bộ pool ($W_A = 0.3$). Ưu tiên các khung hình tập trung nhiều đối tượng khó để tối đa hóa hiệu suất gắn nhãn của con người (gán 1 ảnh sửa được nhiều lỗi).
  3. $D$ (Diversity): Khoảng cách thời gian tới frame đã gán gần nhất, chuẩn hóa qua ngưỡng chặn 10 giây ($W_D = 0.2$). Ở vòng đầu chưa gán ảnh nào thì $D = 1.0$; ở các vòng sau, $D$ giúp dàn trải các frame đều đặn dọc theo trục thời gian, tránh tập trung cục bộ.
- **Vai trò của `MIN_GAP_S`:** Là khoảng cách thời gian tối thiểu giữa hai ảnh được chọn trong cùng một lô (đặt là $2.0$ giây). Do camera cố định, hai frame cách nhau dưới 2 giây có các xe hầu như chưa đổi vị trí, góc chiếu và mật độ y hệt nhau. `MIN_GAP_S` ngăn thuật toán chọn tham lam các ảnh gần trùng, tránh lãng phí ngân sách gán nhãn của con người.
- **Chứng minh bằng 3 frame trong `SELECTION.md` và 1 frame khác:**
  1. `frame_0182.jpg` (Rank 1, Score 0.9591, $t=72.8s$, $U=0.9182$, $A=1.0$): Điểm số và số box mập mờ (18 box) cao nhất pool, nằm ở giữa video với luồng xe lóa đèn pha mạnh.
  2. `frame_0369.jpg` (Rank 2, Score 0.9324, $t=147.6s$, $U=0.9315$, $A=0.8889$): Đoạn cuối video mật độ xe đông đúc với 43 box dự đoán, 16 box mập mờ.
  3. `frame_0270.jpg` (Rank 13, Score 0.8878, $t=108.0s$, $U=0.9089$, $A=0.7778$): Cầu nối thời gian quan trọng ở $t=108s$, chứa xe tải lớn và góc tối khớp với ghi nhận độc lập trong `BLIND_SCAN.md`.
  4. Frame cân nhắc loại trừ: `frame_0372.jpg` (Rank 6, Score 0.9101, $t=148.8s$): Có điểm số rất cao (đứng thứ 6/268 frame) nhưng **bị loại bỏ** (`selected = False`) vì chỉ cách `frame_0369.jpg` đúng $1.2s < \text{MIN\_GAP\_S} = 2.0s$. Quyết định này giúp tiết kiệm công gán nhãn trên một ảnh gần trùng lặp hoàn toàn.
- **Điểm bất định có chứng minh ảnh đó sẽ cải thiện mô hình không? Vì sao?**
  **Không.** Điểm bất định cao chỉ phản ánh rằng mô hình hiện tại đang thiếu tự tin ở mức xác suất $c \sim 0.5$, nhưng không bảo đảm rằng sau khi học ảnh đó mô hình sẽ tăng AP50. Lý do là độ bất định có thể bắt nguồn từ **nhiễu không thể học (aleatoric uncertainty)**, chẳng hạn như ánh đèn pha rọi thẳng gây lóa mù mắt cảm biến, mặt đường ướt phản quang thành vệt sáng giả, mắt lưới hàng rào che khuất, hoặc các đốm sáng xa mờ ảo chỉ vài pixel không đủ thông tin thị giác để xác định xe. Việc gán nhãn các frame chứa nhiều nhiễu thị giác không giúp mô hình học thêm tri thức tổng quát hóa, thậm chí có thể gây nhiễu nhãn và làm giảm hiệu năng.

## 4. Các vòng học chủ động (active learning)

Chép bảng từ `rounds_table.md`. Với mỗi vòng, trình bày:

- mức độ bạn đã sửa nhãn gợi ý (số box giữ nguyên, chỉnh sửa, xoá, thêm mới, lấy từ
  `outputs/round*_diff.md`);
- AP50 thay đổi bao nhiêu so với khởi đầu lạnh và so với vòng trước;
- nhóm xe nào tốt lên hoặc xấu đi theo số đo trên cùng tập test.

Dựa vào các ảnh `compare_round*.jpg`, chỉ ra một ca kết quả đổi sau fine-tune (tốt hơn hoặc xấu
đi), cùng lý do có thể kiểm. Dùng `BLIND_SCAN.md`, `REVIEW_LOG.csv` và `round1_diff.md` phân biệt
quan sát độc lập, lỗi pre-label đã sửa và kết quả mô hình sau train. Mô tả một ca khó theo guideline.

Bảng so sánh các vòng trích xuất từ `reports/rounds_table.md`:
```markdown
| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 310 | 0.698 | -0.073 | 1.000 | 0.166 | 0.285 | 0.000 | 0.149 | 0.561 |
```

- **Mức độ sửa nhãn gợi ý vòng 1 (từ `outputs/round1_diff.md`):**
  Lô 12 ảnh được chọn. Mô hình pre-label đề xuất 169 box, sau khi rà soát và chỉnh sửa đạt 310 box:
  - `accepted` (giữ nguyên): **129 box** (tỷ lệ chấp nhận đạt 76%).
  - `edited` (chỉnh sửa ranh giới box): **23 box**.
  - `deleted` (xóa box giả - FP của model): **17 box**.
  - `added` (thêm mới xe bị bỏ sót - FN của model): **158 box** (cho thấy pre-label ban đầu bỏ sót gần một nửa số xe thực tế).
- **Biến thiên AP50:**
  - AP50 giảm từ **0.771** xuống **0.698** ($\Delta \text{AP50} = -0.073$, tức giảm 7.3% so với cold start).
- **Nhóm xe tốt lên / xấu đi trên cùng tập test:**
  - *Tốt lên vượt bậc về độ chính xác (Precision):* Precision@0.25 tăng từ **0.925 lên 1.000** (đạt 100%, không còn bất kỳ dự đoán sai False Positive nào trên tập test).
  - *Xấu đi nghiêm trọng về độ phủ (Recall):* Recall@0.25 sụt giảm mạnh từ **0.489 xuống 0.166** (chỉ còn nhận diện được 67/403 box tham chiếu).
  - *Phân tích theo kích thước:* Nhóm xe lớn (`R large`) được duy trì ổn định ở mức **0.5610** (56.10%); nhóm xe vừa (`R medium`) giảm từ **0.5473 xuống 0.1486** (14.86%); nhóm xe nhỏ (`R small`) sụt giảm hoàn toàn từ **0.1818 về 0.0000** (0%). Mô hình sau fine-tune trở nên cực kỳ thận trọng, chỉ đưa ra dự đoán khi độ tin cậy rất cao ở các xe lớn/gần, và triệt tiêu dự đoán ở các xe nhỏ ở xa.
- **Ca kết quả đổi sau fine-tune từ `outputs/compare_round1.jpg`:**
  Quan sát `frame_0050` và `frame_0150` ở cột `round 1`:
  - *Điểm tốt:* Số False Positive (khung vàng/cam vẽ sai) đã giảm hoàn toàn về **0** (FP = 0 ở cả 4 ảnh test trực quan). Các vết bóng đèn phản chiếu trên mặt đường trước đây bị cold start bắt nhầm thành xe nay đã bị loại bỏ triệt để.
  - *Điểm xấu:* Rất nhiều xe nhỏ ở các làn xa (các chấm đỏ FN) bị mô hình bỏ qua hoàn toàn.
  - *Lý do có thể kiểm chứng:* Tập train chỉ có 12 ảnh với 310 box. Khi fine-tune 50 epoch với hàm mất mát trừng phạt mạnh FP, mô hình học được xu hướng co cụm, nâng ngưỡng tự tin ngầm lên cao để tránh lỗi, dẫn đến hiện tượng under-prediction đối với các đối tượng khó.
- **Phân biệt 3 mức thông tin:**
  1. *Quan sát độc lập (`BLIND_SCAN.md` trên `frame_0270`):* Trước khi xem AI gợi ý, học viên đếm được 27 xe, đặc biệt chú ý xe nằm ở góc phải ngoài cùng màu đen tối chỉ thấy đèn đít xe và 2 xe ở làn ngược chiều bên phải phía xa. Đây là đánh giá khách quan của con người dựa trên nhận thức ngữ cảnh.
  2. *Lỗi pre-label đã sửa (`round1_diff.md` & `REVIEW_LOG.csv`):* Pre-label ban đầu chỉ vẽ 13 box trên `frame_0270`, học viên đã thêm 11 box xe bị bỏ sót (tổng 24 box), sửa các box bị dính 2 xe thành từng box riêng biệt và xóa các box đèn lóa.
  3. *Kết quả mô hình sau train:* Mô hình mới học được việc phân định rõ ranh giới thân xe lớn nhưng lại đánh mất khả năng phát hiện xe nhỏ ở xa (R small = 0), cho thấy việc học từ 12 ảnh là chưa đủ mẫu bao quát phân phối ban đêm.
- **Mô tả một ca khó theo guideline:** Ca xe ở góc rìa tối của `frame_0270` chỉ xuất hiện đốm sáng đỏ của đèn hậu, thân xe gần như chìm vào bóng đêm. Theo `GUIDELINE_LABEL.md`, chỉ được vẽ box khi thấy được ước lượng thân xe hoặc cụm đèn có ranh giới rõ ràng, không được vẽ box bao trùm quầng sáng phát tán ra mặt đường; nếu xe bị che khuất trên 80% thì phải bỏ qua. Việc xác định ranh giới thật của thân xe trong vùng tối là thử thách lớn nhất giữa việc bỏ sót (FN) và vẽ box sai (FP).

## 5. Kết luận và giới hạn

Kết quả vòng này so với cold start ra sao? Vì sao bạn dừng hoặc tiếp tục? Đề xuất hai ca còn yếu
hoặc bất định cho vòng sau, kèm chi phí rà nhãn và nguy cơ ảnh gần trùng. Tập kiểm thử chỉ 20 ảnh,
có luật bỏ qua xe quá nhỏ và nhãn tham chiếu do mô hình tạo chưa được rà thủ công; các giới hạn đó
ảnh hưởng thế nào đến kết luận? Nếu AP50 giảm, bạn sẽ kiểm tra điều gì trước khi train thêm?

- **Đánh giá kết quả vòng 1 so với cold start:**
  Vòng 1 cho kết quả mang tính hai mặt đặc trưng của bài toán nhận diện ban đêm: AP50 giảm nhẹ từ 0.771 xuống 0.698 (-0.073), Recall giảm mạnh từ 0.489 xuống 0.166, nhưng Precision đạt mức hoàn hảo 1.000 (loại sạch 100% False Positive). Mô hình đã chuyển dịch từ trạng thái "đoán mò nhiều để bắt xe nhỏ" (cold start) sang trạng thái "chỉ dự đoán khi chắc chắn là xe rõ nét".
- **Lý do dừng hoặc tiếp tục:**
  Nếu tiếp tục, mục tiêu là thực hiện Vòng 2 để bổ sung thêm dữ liệu (12 ảnh tiếp theo) nhằm cải thiện Recall xe nhỏ. Tuy nhiên, nếu dừng lại ở Vòng 1, lý do kỹ thuật hoàn toàn hợp lý: Vòng 1 đã hoàn thành trọn vẹn chu trình Active Learning (thuật toán chọn mẫu -> rà soát độc lập -> sửa nhãn trên CVAT -> fine-tune -> đánh giá đối chứng), và kết quả phản ánh đúng hiện thực khách quan rằng khi huấn luyện một mô hình mạng nơ-ron sâu trên tập dữ liệu rất nhỏ (12 ảnh), AP50 có thể giảm do trade-off giữa Precision và Recall.
- **Đề xuất hai ca còn yếu/bất định cho vòng sau:**
  1. *Ca 1 - Xe nhỏ ở cự ly xa lúc đường thông thoáng (ví dụ: `frame_0020.jpg` lúc $t=8.0s$ hoặc `frame_0099.jpg`):* Nhằm cung cấp thêm mẫu xe nhỏ ban đêm để cứu vãn chỉ số `R small`. Chi phí rà nhãn ở mức trung bình (khoảng 20–25 box). Nguy cơ ảnh gần trùng thấp do cách xa các frame đã gán ở vòng 1 hơn 30 giây.
  2. *Ca 2 - Xe tải lớn và xe bị che khuất ở dải phân cách (ví dụ: `frame_0232.jpg` lúc $t=92.8s$):* Cung cấp mẫu xe tải to và xe đan xen phức tạp. Chi phí rà nhãn cao (khoảng 40 box), cần rà soát kỹ ranh giới. Nguy cơ ảnh gần trùng cần được kiểm soát bằng cách giữ khoảng cách tối thiểu $\ge 2.0s$ với `frame_0227.jpg` ($90.8s$).
- **Tác động của giới hạn tập kiểm thử (20 ảnh, bỏ qua xe < 16px, nhãn tham chiếu tự động):**
  1. *Kích thước test nhỏ (20 ảnh, 403 box):* Độ tin cậy thống kê bị hạn chế; sự thay đổi của một vài box có thể làm biến động AP50 thêm hàng phần trăm, chưa đại diện trọn vẹn cho toàn bộ phân phối giao thông ban đêm.
  2. *Luật bỏ qua xe quá nhỏ (< 16px, 14 box):* Tạo ra ranh giới cứng ngắc tại 16 pixel; nếu mô hình nhận diện tốt một xe kích thước 15 pixel thì lại không được cộng điểm hoặc bị tính phạt sai lệch.
  3. *Nhãn tham chiếu do mô hình tự tạo (pseudo-groundtruth):* Nhãn kiểm thử chưa qua con người chuẩn hóa 100%, bản thân nhãn test vẫn chứa lỗi (bỏ sót xe trong bóng tối hoặc vẽ box sai vào vệt sáng). Do đó, sự sụt giảm AP50 của mô hình fine-tune không đồng nghĩa với việc mô hình hoạt động kém hơn ngoài thực tế, mà một phần do mô hình không còn dự đoán trùng với các lỗi cố hữu của bộ sinh nhãn tham chiếu.
- **Nếu AP50 giảm, cần kiểm tra các yếu tố sau trước khi train thêm:**
  1. *Kiểm tra tính nhất quán của nhãn gán:* Rà soát xem nhãn người gán ở vòng 1 có nhất quán với tiêu chí của nhãn test không (ví dụ: quy ước bao trùm đèn xe, mức độ che khuất được phép gán).
  2. *Kiểm tra phân phối kích thước box:* Xem tập train 12 ảnh có bị mất cân đối nghiêm trọng, thiếu hẳn các box xe nhỏ khiến mô hình bị "lãng quên" đặc trưng xe nhỏ hay không.
  3. *Kiểm tra ngưỡng tự tin và chiến lược đánh giá:* Kiểm tra đường cong Precision-Recall tại các ngưỡng confidence khác nhau ($c = 0.05, 0.10, 0.25$) thay vì chỉ nhìn vào một điểm cắt $c = 0.25$.
  4. *Kiểm tra hiện tượng quá khớp (Overfitting):* Huấn luyện 50 epoch trên 12 ảnh có thể khiến mô hình tối ưu hóa quá mức vào các đặc trưng cục bộ của 12 ảnh đó. Cần cân nhắc điều chỉnh learning rate, giảm số epoch hoặc tăng cường dữ liệu (data augmentation) trước khi mở rộng vòng tiếp theo.

