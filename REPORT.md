# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Võ Đức Tài |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 202603007 |
| GitHub | HydrogenDrinker |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | split_seed = 6085 / model_seeds = [7379, 7380, 7381, 7382, 7383] |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.264 m | 19.53% | BLOCK (FAIL: R2, R3, R4, R6) |
| IDM | 1.529 m | 0.07% | (baseline) |
| EgoMLP | 0.754 ± 0.003 m | 19.29% .. 42.14% (tệ nhất: 42.14%) | BLOCK (FAIL: R2, R3, R4, R5, R6) |
| PercMLP | 0.620 ± 0.003 m | 0.62% .. 1.35% (tệ nhất: 1.35%) | BLOCK (FAIL: R2, R3, R4, R6) |
| PercMLP+ (sau khi sửa) | 0.839 ± 0.015 m | 0.21% .. 0.28% (tệ nhất: 0.28%) | RELEASE (đạt 7/7 quy tắc) |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Tập dữ liệu thô ban đầu gồm 9993 quỹ đạo với 992,679 dòng ở chu kỳ lấy mẫu 10 Hz (dt = 0.1 s). Quá trình kiểm tra phát hiện đúng 383 quỹ đạo bị trùng lặp hoàn toàn (exact duplicate) nằm rải rác ở các ID cách xa nhau, sau khi khử trùng còn lại 9610 quỹ đạo sạch. Phân tích tương quan cho thấy gia tốc xe bám sau Acc_FV có tương quan cao với đạo hàm vận tốc dv/dt đạt 0.929, trong khi cột gia tốc xe trước Acc_LV chỉ có tương quan vỏn vẹn 0.0453 (gần như bằng 0 do cảm biến radar/camera thô đo khoảng cách rồi vi phân hai lần mà không lọc nhiễu, không phản ánh gia tốc vật lý thực tế). Do đó không dùng cột Acc_LV mà thay bằng gia tốc a_lead suy dẫn từ vi phân của chuỗi vận tốc xe trước đã làm mịn bằng bộ lọc Savitzky-Golay (sau làm mịn đạt tương quan > 0.90). Dữ liệu được chia theo quỹ đạo (seed 6085: 6727 train / 2883 test) thay vì chia ngẫu nhiên từng dòng vì các khung hình kế tiếp trong cùng một chuỗi thời gian có độ tự tương quan cực kỳ cao (~0.99); nếu chia theo dòng thì các frame liền kề của cùng một tình huống sẽ xuất hiện ở cả hai tập, gây rò rỉ dữ liệu (data leakage) nghiêm trọng và khiến mô hình chỉ ghi nhớ thay vì học được quy luật tổng quát.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Bản refactor float32 và thay đổi thứ tự tích lũy tổng khác biệt ở cấp độ bit trên toàn bộ 600/600 đoạn quỹ đạo (not_bit_identical = 600), nhưng hoàn toàn không có đoạn nào vượt ngưỡng dung sai (differ_beyond_tol = 0 với atol = 0.001, rtol = 0.001, sai lệch cực đại chỉ là 0.000403 m/s²). Khi thay đổi hành vi bằng việc bổ sung phanh khẩn cấp tự động (AEB với ngưỡng TTC < 1.5 s), có đúng 10/600 đoạn bị lệch vượt dung sai (các đoạn 57, 63, 158, 243, 295, 406, 442, 459, 488, 510 với sai lệch gia tốc lớn nhất đạt 1.436 m/s²). Mở các đoạn này ra kiểm tra cho thấy xe trước bất ngờ phanh gấp làm khoảng cách giảm nhanh khiến TTC giảm sâu dưới 1.5 s, kích hoạt AEB can thiệp phanh tối đa và tạo ra bước nhảy gia tốc lệch hẳn so với bộ điều khiển IDM thông thường. Chế độ phát lại thời gian thực (realtime replay có giả lập trễ 1%) làm lộ ra 580 khung hình bị trễ (late) và 566 khung hình bị bỏ rơi (dropped) trên tổng số 59,211 khung hình, dẫn đến 381/600 đoạn bị thay đổi hành vi và lệch dung sai (max diff lên tới 1934.9 m/s² do tích lũy lỗi trễ chấp hành). Ngược lại, chế độ phát nhanh nhất (fastest replay) che giấu hoàn toàn các rủi ro này khi báo cáo 0 khung trễ và 0 khung rớt, tạo ra ảo tưởng hoàn hảo về tính tất định.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới kịch bản tạt đầu (cut-in grid) được thiết lập tại vận tốc cruise v_ego = 14.0 m/s, dải khoảng cách từ 2.76 m đến 59.31 m (60 mốc khoảng cách) và dải chênh lệch tốc độ từ 0.0 m/s đến 14.0 m/s (45 mốc), tạo thành lưới 2700 ô kịch bản. Ở độ trễ phản ứng t_react = 0.0 s, có 3.63% diện tích lưới (98 ô) là vùng va chạm vật lý không thể tránh khỏi (unavoidable frontier) dù bộ điều khiển phanh tối đa ở mức gia tốc giới hạn 9.0 m/s². Khi độ trễ phản ứng tăng lên t_react = 0.6 s, vùng va chạm vật lý tăng gần gấp 3 lần lên mức 10.22% diện tích (276 ô), và bộ điều khiển chuẩn IDM khớp chính xác 100% với biên giới vật lý này (0 ô lệch ra ngoài biên giới ở cả hai mức độ trễ). Khai thác tập log cho thấy giới hạn phanh mạnh nhất của người lái trong log (phân vị 99.9%) chỉ là 3.62 m/s² (trong khi cơ cấu phanh xe có thể đạt tới 9.0 m/s²), và có tới 12.67% diện tích lưới kịch bản đòi hỏi mức phanh ngặt nghèo hơn bất kỳ tình huống nào từng xuất hiện trong log thực tế. Điều này chứng minh log vận hành tự nhiên có thiên kiến chọn lọc nặng về các tình huống êm dịu và thiếu hẳn các kịch bản nguy hiểm ở rìa vật lý.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Bảng so sánh 4 bộ điều khiển ban đầu giữa vòng hở và vòng kín:
- constant-velocity: L2 vòng hở 1.264 m (Hạng 3) | Va chạm vòng kín 19.53% (Hạng 3)
- IDM: L2 vòng hở 1.529 m (Hạng 4) | Va chạm vòng kín 0.07% (Hạng 1)
- EgoMLP: L2 vòng hở 0.754 m (Hạng 2) | Va chạm vòng kín 27.80% (Hạng 4)
- PercMLP: L2 vòng hở 0.620 m (Hạng 1) | Va chạm vòng kín 0.85% (Hạng 2)
Có tới 4 cặp bị đảo ngược thứ tự xếp hạng giữa hai cách chấm: (const-vel, IDM), (const-vel, EgoMLP), (IDM, EgoMLP) và (IDM, PercMLP). IDM có sai số L2 vòng hở kém nhất (1.529 m) vì nó là mô hình dựa trên quy tắc vật lý an toàn, không cố gắng bắt chước các dao động chân ga nhỏ nhặt của con người trong điều kiện bình thường; tuy nhiên trong vòng kín nó phản hồi động học chính xác và đạt tỷ lệ va chạm thấp nhất (0.07%). Ngược lại, EgoMLP có L2 vòng hở rất ấn tượng (0.754 m) do tối ưu hóa hàm mất mát trên phân phối trạng thái tĩnh của dữ liệu huấn luyện, nhưng khi đưa vào vòng kín nó gặp hiện tượng trôi dạt hiệp biến (covariate shift): các sai số nhỏ ở mỗi bước tích lũy khiến xe trôi ra khỏi phân phối quen thuộc, dẫn tới tỷ lệ va chạm thảm họa 27.80% (trên lát phanh gấp lên tới 64.57%), chứng minh rằng L2 vòng hở là một thước đo sai lệch nguy hiểm.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

EgoMLP dao động cực kỳ mạnh mẽ giữa 5 seed huấn luyện ngẫu nhiên: tỷ lệ va chạm trải dài từ 19.29% (seed 7381) đến 42.14% (seed 7380), tức chênh lệch tới hơn 22.85 điểm phần trăm chỉ do việc khởi tạo trọng số mạng nơ-ron và xáo trộn lô dữ liệu. Khoảng tin cậy Wilson 95% cho seed tệ nhất 7380 là [40.35%, 43.96%], chứng minh rằng tỷ lệ đâm cao hơn 40% là một đặc tính hệ thống chắc chắn chứ không phải do biến động ngẫu nhiên của mẫu kiểm thử. Nếu kỹ sư chỉ chạy duy nhất một seed may mắn như 7381, họ sẽ ghi nhận con số 19.29% và lầm tưởng mô hình hoạt động ổn định ngang ngửa bộ điều khiển vận tốc không đổi (const-vel 19.53%), trong khi thực tế mô hình có thể suy biến nặng nề lên gấp đôi ở một lần khởi tạo khác. Việc chỉ dựa vào một lần chạy duy nhất là vô cùng nguy hiểm vì nó che giấu tính bất định cố hữu của quá trình học mô hình, tạo ra cảm giác an toàn giả tạo và có thể dẫn tới việc phê duyệt triển khai một hệ thống lái tự động có độ rủi ro tử thần.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Bảng so sánh tác nhân xe sau trên lưới kịch bản tại độ trễ t_react = 0.6 s:
- constant-velocity: Xe sau phát lại đâm 80.63% | Xe sau phản ứng đâm 80.63% (va chạm sau: 0.0% / 0.0%)
- IDM: Xe sau phát lại đâm 80.85% | Xe sau phản ứng đâm 10.22% (va chạm sau: 70.63% / 0.0%)
- EgoMLP: Xe sau phát lại đâm 86.41% | Xe sau phản ứng đâm 86.41% (va chạm sau: 0.0% / 0.0%)
- PercMLP: Xe sau phát lại đâm 80.37% | Xe sau phản ứng đâm 27.81% (va chạm sau: 53.78% / 0.0%)
- PercMLP+: Xe sau phát lại đâm 80.11% | Xe sau phản ứng đâm 10.85% (va chạm sau: 69.26% / 0.0%)
Thời gian giãn cách headway hiệu chỉnh tối ưu cho tác nhân IDM xe sau là T = 1.4 s, mang lại khoảng cách Wasserstein W1 = 0.34 so với phân phối thực tế của con người (median headway log là 1.87 s), chân thực hơn nhiều so với xe không phản ứng có W1 = 0.81. Khi xe sau chỉ phát lại cứng nhắc theo quỹ đạo ghi trong log, nếu xe tự hành (ego) phanh gấp để tránh xe trước, xe sau không phanh và húc thẳng vào đuôi xe ego, tạo ra 70.63% va chạm sau giả cho IDM (tổng va chạm tăng vọt lên 80.85%). Khi chuyển sang môi trường tương tác phản ứng (reactive world), xe sau biết phanh kịp thời khiến va chạm đuôi giảm về đúng 0.0%, đưa tỷ lệ va chạm của IDM về đúng 10.22% (trùng khít với giới hạn vật lý). Điều này làm thay đổi hoàn toàn kết luận: tác nhân phản ứng giúp bóc tách va chạm do lỗi tự thân của xe ego khỏi các vụ đâm giả do môi trường tĩnh vô cảm gây ra.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

Cổng an toàn đã chặn đứng PercMLP do vi phạm đồng thời 4/7 quy tắc: R2 (va chạm vòng kín 1.35%, cận trên 95% là 1.84% so với ngưỡng IDM + 1% = 1.07%), R3 (va chạm trên lát phanh gấp 2.86%, cận trên là 6.51% so với ngưỡng 5.57%), R4 (diện tích va chạm tránh được trên lưới tạt đầu 17.59% so với ngưỡng 2.00%), và R6 (va chạm trong thế giới phản ứng 27.81% so với ngưỡng 12.22%). Nguyên nhân gốc gồm hai yếu tố: dữ liệu log thiếu hoàn toàn các tình huống tạt đầu nguy hiểm ở biên vật lý (gây lỗi ở R4, R6) và lỗi bắt chước nhỏ tích lũy dần theo thời gian trong vòng kín (gây lỗi ở R2). Phương pháp sửa kết hợp hai kỹ thuật: (1) bổ sung 197,130 cửa sổ từ 20,000 kịch bản tạt đầu mô phỏng tổng hợp (cut-in scenarios) để bao phủ phân phối biên, và (2) áp dụng thuật toán DAgger qua 2 vòng lặp để đưa chính mô hình vào tương tác vòng kín, thu thập các trạng thái xe tự ghé thăm và gán nhãn hành động chuẩn từ chuyên gia IDM. PercMLP+ sau khi sửa đã vượt qua cả 7/7 quy tắc cổng và được phê duyệt RELEASE (R2 giảm xuống 0.28%, R3 còn 1.14%, R4 chỉ còn 0.63%, R6 còn 10.85%). Tuy nhiên, PercMLP+ chưa thực sự tốt hơn IDM (va chạm vòng kín 0.28% so với IDM 0.07%, diện tích tránh được 0.63% so với IDM 0.00%, và khi kiểm thử ở tốc độ chưa từng thấy 20 m/s thì PercMLP+ vẫn đâm 0.3%..2.3% trong khi IDM là 0.0%). Do đó, chưa nên phát hành rộng rãi ngay mà cần bổ sung kiểm thử OOD (out-of-distribution), đánh giá với cảm biến có nhiễu thời tiết và kiểm thử trên xe thật tại sa hình đóng (closed-course track testing).

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Bài lab có ít nhất ba giới hạn nghiêm trọng đối với việc quyết định phát hành xe tự hành:
1. Mô hình động học 1D đơn giản hóa: Mô hình giả định chuyển động dọc thuần túy trên một làn đường thẳng, bỏ qua hoàn toàn động lực học ngang, chuyển làn, trượt lốp, độ bám đường thay đổi theo thời tiết (đường mưa, băng tuyết), độ trễ truyền động của phanh và độ nhún của hệ thống treo. Trong thực tế, phanh gấp ở tốc độ cao có thể làm mất lái hoặc văng đuôi nếu không kiểm soát lực bám ngang.
2. Cảm biến hoàn hảo và không có vật cản che khuất: Môi trường mô phỏng cung cấp khoảng cách và vận tốc tương đối tức thời, chính xác gần như tuyệt đối, trong khi thực tế cảm biến camera/LiDAR/Radar luôn có góc khuất, nhiễu đo lường, độ trễ nhận diện vật thể, mục tiêu giả (ghost targets) và bị hạn chế tầm nhìn trong điều kiện thời tiết xấu như sương mù, mưa lớn hay ngược sáng.
3. Hành vi tác nhân xung quanh mang tính khuôn mẫu: Xe trước và xe sau chỉ được mô phỏng bằng mô hình toán học IDM đơn giản với phản ứng thuần túy dựa trên khoảng cách. Thế giới thực phức tạp hơn rất nhiều với hành vi khó lường của tài xế (lạng lách, phanh nhử, mất tập trung), người đi bộ qua đường bất ngờ, xe máy tạt đầu cắt mặt và các tương tác xã hội phi ngôn ngữ.
Để thu hẹp khoảng cách tới thế giới thực (sim-to-real gap), các bước tiếp theo cần thực hiện gồm: nâng cấp mô hình mô phỏng lên 3D đa vật thể có động lực học lốp chân thực (như CARLA hoặc Carmaker); thu thập dữ liệu thế giới thực đa dạng tình huống thời tiết và đường sá; tích hợp mô đun nhận thức đầy đủ từ ảnh thô; và tiến hành kiểm thử thực nghiệm trên sa hình đóng có chướng ngại vật mềm trước khi chạy thử nghiệm có tài xế an toàn trên đường công cộng.
