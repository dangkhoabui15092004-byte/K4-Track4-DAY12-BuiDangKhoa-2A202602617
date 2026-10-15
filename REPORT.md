# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Bùi Đăng Khoa |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602617 |
| GitHub | dangkhoabui15092004-byte |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | split seed: 2514, model seeds: [5982, 5983, 5984, 5985, 5986] |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.260 | 20.46% | BLOCK |
| IDM | 1.524 | 0.07% | (baseline) |
| EgoMLP | 0.755 .. 0.766 (mean 0.760) | 12.94% .. 30.49% (mean 22.41%) | BLOCK |
| PercMLP | 0.617 .. 0.629 (mean 0.623) | 0.42% .. 1.28% (mean 0.81%) | BLOCK |
| PercMLP+ (sau khi sửa) | 0.795 .. 0.820 (mean 0.809) | 0.21% .. 0.24% (mean 0.23%) | RELEASE |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Quá trình kiểm tra dữ liệu ở Bước 0 (`evidence/data_audit.json`) phát hiện 3 vấn đề lớn:
1. Cột gia tốc xe dẫn `Acc_LV` có hệ số tương quan với đạo hàm vận tốc $d(\text{Speed\_LV})/dt$ chỉ đạt 0.0454 (gần như bằng 0, trái ngược hoàn toàn với xe theo sau đạt 0.9291). Do đó cột này bị nhiễu nặng hoặc sai lệch hệ thống, không thể dùng để khai thác tình huống phanh. Ta thay thế bằng đạo hàm vận tốc xe dẫn đã qua làm mượt (`v_lead_s`, `a_lead`).
2. Có đúng 383 quỹ đạo trùng lặp tuyệt đối (exact duplicates) được phát hiện và loại bỏ, giúp tập dữ liệu sạch giảm từ 9993 xuống còn 9610 quỹ đạo. Nếu không loại bỏ, các cặp quỹ đạo giống hệt nhau có thể rơi vào cả tập train và test gây rò rỉ dữ liệu (data leakage) và phóng đại điểm đánh giá.
3. Việc phân chia tập dữ liệu bắt buộc phải chia theo quỹ đạo (trajectory) thay vì chia ngẫu nhiên theo từng khung hình (frame), bởi vì các khung hình 10 Hz liên tiếp trong cùng một chuỗi có tương quan thời gian cực kỳ mạnh. Chia theo khung hình sẽ khiến mô hình "nhìn trước tương lai" ở frame liền kề, làm mất tính khách quan khi kiểm thử.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Từ số liệu `evidence/replay.json`:
1. Bản refactor float32 và thay đổi thứ tự cộng khiến 600/600 đoạn không giống từng bit (`not_bit_identical` = 600), nhưng có 0/600 đoạn vượt dung sai (`differ_beyond_tol` = 0) với sai số tuyệt đối cực đại chỉ là 0.000403 m/s² (dưới ngưỡng `atol=1e-3, rtol=1e-3`). Điều này chứng minh số thực float32 không có tính kết hợp trong tính toán máy tính, do đó đòi hỏi phải kiểm thử bằng dung sai vật lý thay vì so sánh từng bit chuẩn xác.
2. Khi bổ sung AEB với ngưỡng TTC 1.5s, chỉ có 8/600 đoạn bị lệch (tại các đoạn [57, 132, 136, 176, 198, 258, 309, 321]) với độ lệch lệnh lái lớn nhất 1.678 m/s². Mở đoạn 57 cho thấy sự khác biệt chỉ xuất hiện ở các khung hình có khoảng cách gap thu hẹp mạnh và TTC giảm xuống dưới 1.5s, kích hoạt nhánh điều khiển phanh khẩn cấp mới.
3. Chế độ phát nhanh nhất (`fastest`) hoàn toàn che giấu độ trễ tính toán vì xử lý phi thời gian thực (ghi nhận 0 khung hình trễ, 0 khung hình rớt). Ngược lại, chế độ `realtime` làm lộ ra 585 khung hình bị trễ (late) và 578 khung hình bị rớt (dropped) trên tổng số 60,017 khung hình, dẫn tới 375/600 đoạn bị thay đổi lệnh lái trên thực tế do vi phạm thời hạn (deadline miss).

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới kịch bản ở Bước 2 (`evidence/scenarios.json`) được thiết lập với tốc độ cruise 14.0 m/s, khoảng cách gap từ 2.76 m đến 59.31 m, chênh lệch vận tốc dv từ 0.0 đến 14.0 m/s, gồm 2700 ô (60 gaps x 45 dvs):
1. Ở độ trễ phản ứng $t_{react} = 0.0$ s, có 3.63% số ô là bất khả thi về mặt vật lý (unavoidable area), tức không bộ điều khiển nào tránh được va chạm dù phanh tối đa $b_{max} = 9.0$ m/s². Khi độ trễ phản ứng tăng lên $t_{react} = 0.6$ s, vùng bất khả thi tăng vọt lên 10.22% số ô (tăng thêm 6.59 điểm phần trăm). Vùng va chạm của IDM tăng từ 3.63% lên 10.22%, đúng bằng biên vật lý bất khả thi (diện tích va chạm do lỗi bộ điều khiển attributable area = 0.0%).
2. Giới hạn phanh gắt nhất của người lái trong log (phân vị 99.9) chỉ đạt 3.62 m/s², trong khi cơ cấu chấp hành xe có thể phanh tới 9.0 m/s². Trên lưới kịch bản, có tới 12.67% số ô đòi hỏi mức giảm tốc vượt quá 3.62 m/s² mà con người trong log từng áp dụng.
3. Các sự kiện khai thác từ log (614 ca phanh trên 9610 quỹ đạo, chiếm 6.39%) có độ giảm tốc trung vị chỉ -2.53 m/s² và phân vị 99 là -4.21 m/s². Toàn bộ các chấm trắng từ log chỉ phủ vùng lái tương đối an toàn, hoàn toàn bỏ trống vùng góc trên bên trái của lưới (gap hẹp, closing speed lớn).

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Bảng so sánh 4 bộ điều khiển (`evidence/closedloop.json`):
- PercMLP: L2 vòng hở 0.623 m (Hạng 1) | Va chạm vòng kín 0.81% (Hạng 2).
- EgoMLP: L2 vòng hở 0.760 m (Hạng 2) | Va chạm vòng kín 22.41% (Hạng 4).
- constant-velocity: L2 vòng hở 1.260 m (Hạng 3) | Va chạm vòng kín 20.46% (Hạng 3).
- IDM: L2 vòng hở 1.524 m (Hạng 4) | Va chạm vòng kín 0.07% (Hạng 1).

Có 4 cặp đổi thứ tự xếp hạng giữa vòng hở và vòng kín: (const-vel, IDM), (const-vel, EgoMLP), (IDM, EgoMLP), và (IDM, PercMLP).
- IDM có L2 vòng hở tệ nhất (1.524 m) vì nó là một phương trình vi phân điều khiển phản hồi mực thước, không cố học bắt chước các dao động nhiễu và độ trễ phản xạ của tài xế con người trong log. Tuy nhiên, khi đưa vào vòng kín, IDM liên tục nhận phản hồi về gap và vận tốc tương đối hiện tại để tự sửa sai và dập tắt sai lệch, giúp xe duy trì khoảng cách an toàn tuyệt đối và chỉ va chạm 0.07% (2/2883 log).
- EgoMLP có L2 vòng hở tốt (0.760 m, tốt hơn mức 1.260 m của const-vel) vì nó học khớp tốt phân bố tốc độ trung bình của xe đi sau trong log tĩnh. Nhưng nó hoàn toàn không có đầu vào về khoảng cách gap và tốc độ xe dẫn (thiếu nhận thức môi trường). Khi xe dẫn phanh gấp, EgoMLP không có bất kỳ tín hiệu nào để phanh, dẫn tới tỷ lệ va chạm vòng kín lên tới 22.41% (và tới 61.62% trên lát cắt xe dẫn phanh, tệ hơn cả const-vel 52.02%).

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Số liệu 5 seed huấn luyện của EgoMLP từ `evidence/closedloop.json`:
- Tỷ lệ va chạm dao động rất lớn qua 5 seed: seed 5982 (22.20%), seed 5983 (12.94%), seed 5984 (25.98%), seed 5985 (20.43%), và seed 5986 (30.49%). Mức chênh lệch cực đại giữa các seed lên tới 17.55 điểm phần trăm (từ 12.94% đến 30.49%).
- Khoảng tin cậy Wilson 95% của seed tốt nhất (seed 5983) là [11.76%, 14.21%], trong khi của seed xấu nhất (seed 5986) là [28.84%, 32.19%]. Hai khoảng tin cậy này hoàn toàn tách rời nhau và không có bất kỳ điểm giao nhau nào.
- Nếu chỉ chạy duy nhất một seed ngẫu nhiên (chẳng hạn vô tình trúng seed 5983), người làm kiểm thử sẽ đưa ra kết luận sai lệch rằng EgoMLP chỉ va chạm 12.9%, thấp hơn thực tế hơn 2.3 lần so với seed 5986 (30.5%). Sự ngẫu nhiên trong khởi tạo trọng số và xáo trộn mini-batch của mạng nơ-ron tạo ra các mô hình có độ an toàn rất chênh lệch. Việc chỉ báo cáo một lần chạy duy nhất che giấu độ bất ổn định nghiêm trọng này, gây nguy cơ phát hành một mô hình kém an toàn ra thực địa.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Bảng so sánh tác động của xe sau trên lưới kịch bản cắt vào ($t_{react} = 0.6$ s, 2700 ô, từ `evidence/agents.json`):
| Bộ điều khiển | Xe sau phát lại: tổng va chạm (va chạm sau) | Xe sau phản ứng: tổng va chạm (va chạm sau) |
|---|---|---|
| constant-velocity | 80.63% (0.00%) | 80.63% (0.00%) |
| IDM | 80.85% (70.63%) | 10.22% (0.00%) |
| EgoMLP | 84.67% (0.00%) | 84.67% (0.00%) |
| PercMLP | 76.26% (38.78%) | 37.59% (0.00%) |
| PercMLP+ | 80.44% (69.93%) | 11.26% (0.00%) |

- Thời khoảng cách $T$ tối ưu được hiệu chỉnh là $T = 1.4$ s, đạt khoảng cách Wasserstein-1 $W_1 = 0.350$ so với phân bố thời khoảng cách của tài xế con người trong log (trung vị 1.84 s). Con số này chân thực hơn hẳn so với một tác nhân không bao giờ phản ứng ($W_1 = 0.832$).
- Khi xe sau chỉ "phát lại" hành vi cũ từ log, nó hoàn toàn mù trước việc xe thử nghiệm (IDM) vừa chủ động phanh gấp để tránh xe phía trước cắt vào. Do không biết xe trước phanh, xe sau lao thẳng tới và đâm vào đuôi xe IDM, tạo ra tới 70.63% "va chạm giả" từ phía sau, nâng tổng va chạm của IDM lên 80.85% (trông tệ như const-vel 80.63%).
- Khi thay bằng xe sau có phản ứng (IDM $T=1.4$ s), va chạm phía sau được loại bỏ hoàn toàn về 0.00%, đưa tổng va chạm của IDM trở về đúng giá trị vật lý bất khả thi 10.22%. Điều này làm thay đổi hoàn toàn kết luận: một bộ điều khiển phanh an toàn sẽ bị kết tội oan là "nguy hiểm" nếu môi trường mô phỏng sử dụng các tác nhân thụ động không biết tương tác.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

Quy trình đánh giá cổng phát hành (`evidence/gate.json`, `fix.json`):
1. PercMLP bị chặn (BLOCK) bởi 4 quy tắc: R2_closed_loop_logs (va chạm 1.28%, cận trên 1.76% > baseline IDM 0.07% + 1.0%), R3_braking_slice (va chạm 4.04%, cận trên 7.77% > baseline 0.51% + 5.0%), R4_scenario_grid (va chạm trên lưới 27.37% > baseline 0.00% + 2.0%), và R6_reactive_world (va chạm 37.59% > baseline 10.22% + 2.0%).
2. Hai nguyên nhân gốc:
   - Thiếu dữ liệu phân phối lạ (Covariate shift / OOD): Log đường thật chủ yếu là lái xe đều đặn, không có tình huống xe cắt vào ở khoảng cách ngắn và tốc độ chênh lệch cao, khiến PercMLP đâm tới 27.37% trên lưới kịch bản (R4).
   - Tích tụ sai số điều khiển (Compounding errors): Khi xe tự lái trong vòng kín, các sai số nhỏ đẩy xe vào trạng thái lạ chưa từng gặp trong log, làm mô hình hoảng loạn và đâm (R2, R3).
3. Biện pháp sửa: Thêm 197,180 cửa sổ kịch bản cắt vào mô phỏng (`cutin_windows`) và áp dụng 2 vòng DAgger (`dagger_windows`, tổng cộng gần 895,000 cửa sổ) để cho mô hình tự lái, thu thập đúng các trạng thái nó tự đi vào và gán nhãn bằng dự báo của chuyên gia IDM. DAgger dạy mô hình cách tự khôi phục khi lệch khỏi quỹ đạo chuẩn, giải quyết triệt để 2 nguyên nhân gốc.
4. Sau khi sửa, PercMLP+ đạt RELEASE với cả 7 quy tắc: R2 giảm còn 0.24% (cận trên 0.50% <= 1.07%), R3 còn 0.51% (cận trên 2.80% <= 5.51%), R4 giảm ngoạn mục từ 27.37% xuống 1.04% (<= 2.0%), R6 giảm còn 11.26% (<= 12.22%).
5. PercMLP+ KHÔNG hề "tốt hơn" IDM; nó chỉ đạt mức "không kém hơn" baseline (non-inferior), bởi vì nhãn huấn luyện sửa sai của nó được sinh ra hoàn toàn từ chính thuật toán IDM (học trò học từ thầy thì không thể vượt qua thầy trong miền bài toán giả định). Đáng chú ý, L2 vòng hở của PercMLP+ còn tăng từ 0.623 m lên 0.809 m, tái khẳng định điểm số vòng hở không phản ánh độ an toàn. Đề xuất: Chưa nên phát hành ngay để thay thế IDM; cần triển khai thử nghiệm ở chế độ bóng (Shadow Mode) trên xe thật, bổ sung kiểm thử với cảm biến đa dạng và luôn giữ bộ quy tắc luật IDM làm cơ chế an toàn dự phòng (safety fallback).

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Ba giới hạn quan trọng nhất của bài lab đối với quyết định phát hành:
1. Giả định chuyển động một chiều dọc (1D Longitudinal Motion Only): Bài lab giả định xe chỉ di chuyển trên một đường thẳng, bỏ qua hoàn toàn các yếu tố động học 2D/3D như đánh lái tránh chướng ngại vật, chuyển làn khẩn cấp, trượt bánh khi phanh trên đường ướt/băng tuyết, và lực gia tốc trọng trường khi lên/xuống dốc. Trong thực tế, việc phanh gấp với gia tốc 9 m/s² mà không kiểm soát vô lăng có thể làm mất lái hoặc văng đuôi xe.
2. Bỏ qua hoàn toàn chuỗi xử lý cảm biến (Sensor Stack & Perception Noise): Các mô hình được cung cấp trực tiếp khoảng cách và vận tốc xe dẫn ở dạng ground-truth lý tưởng (chỉ thử nghiệm mức nhiễu Gaussian tĩnh rất nhỏ). Thực tế, hệ thống perception gồm camera, lidar, radar luôn đối mặt với hiện tượng mất dấu vật thể (false negative), nhận diện nhầm bóng râm/nắp cống thành chướng ngại vật dẫn đến phanh ma (phantom braking), và độ trễ pipeline thay đổi liên tục.
3. Tác nhân môi trường còn quá đơn giản và mang tính tiền định: Xe dẫn trong log chỉ phát lại tốc độ cũ mà không phản ứng lại với xe thử nghiệm, và xe sau phản ứng chỉ là một luật IDM đơn lẻ với tham số $T$ cố định. Giao thông đời thực là một hệ thống tương tác phức tạp với nhiều loại phương tiện (xe máy, xe buýt, xe tải), người đi bộ khó lường, và các tài xế có phong cách lái xe hung hăng hoặc bất cẩn khác nhau.

Các bước tiếp theo để thu hẹp khoảng cách đến thế giới thực:
- Triển khai mô phỏng vòng kín trên môi trường 3D độ trung thực cao (như CARLA hoặc mô phỏng nội bộ của hãng xe) có tích hợp chuỗi perception đầy đủ từ hình ảnh camera và pointcloud lidar.
- Xây dựng hệ thống kiểm thử tự động tìm kiếm kịch bản góc chết (adversarial scenario generation / fuzzing) để chủ động săn lùng các tình huống hiếm mà mạng nơ-ron xử lý sai.
- Chạy thử nghiệm chế độ bóng (Shadow Mode) trên đội xe thương mại trong hàng triệu kilômét để ghi nhận các trường hợp mạng MLP đưa ra quyết định khác với tài xế con người hoặc luật an toàn trước khi cấp quyền can thiệp vào phanh/ga thực tế.

