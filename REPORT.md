# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Đoàn Duy Bách |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602515 |
| GitHub | DuyBach2003 |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | split_seed 8138 / model_seed 4846 (5 seed huấn luyện 4846–4850); tốc độ cruise của lưới 14 m/s |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 (mặc định) |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1,248 | 19,98 (576/2883, CI95 [18,6; 21,5]) | BLOCK (R2, R3, R4, R6) |
| IDM | 1,537 | 0,03 (1/2883, CI95 [0,01; 0,2]) | (baseline) |
| EgoMLP | 0,767 ± 0,006 | 11,8 .. 43,4 (5 seed, TB 26,9) | BLOCK (R2, R3, R4, R5, R6) |
| PercMLP | 0,623 ± 0,004 | 1,1 .. 1,9 (5 seed, TB 1,5) | BLOCK (R2, R3, R4, R6) |
| PercMLP+ (sau khi sửa) | 0,822 ± 0,009 | 0,3 .. 0,5 (5 seed, TB 0,37) | RELEASE (qua cả 7 quy tắc) |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Ba vấn đề trong `evidence/data_audit.json`: (1) **Cột `Acc_LV` không dùng được**: tương quan của nó với đạo hàm `d(Speed_LV)/dt` chỉ **0,045**, trong khi cột tương ứng của xe theo sau đạt **0,929**, tức `Acc_LV` gần như nhiễu so với chính tốc độ xe dẫn. Vì vậy khi khai thác ca phanh tôi dùng `a_lead` = gia tốc suy ra từ `Speed_LV` **đã làm mượt** (không phải `acc_lead_raw`); phân vị p0,1 của gia tốc làm mượt là −2,77 m/s². (2) **Quỹ đạo trùng hệt**: 9.993 quỹ đạo thô có **383** bản sao chính xác, còn lại **9.610**. Bản sao nằm ở các id cách xa nhau, nên nếu chia ngẫu nhiên thì một bản vào train, bản kia vào test → mô hình được chấm trên dữ liệu đã thuộc lòng, điểm test bị thổi phồng (rò rỉ). (3) **Đơn vị thời gian chia**: các khung hình liên tiếp trong cùng một quỹ đạo gần như giống nhau (10 Hz), nên chia theo khung hình cũng rò rỉ y như trên; chia theo **quỹ đạo** đảm bảo test là những lần lái mô hình chưa từng thấy (seed chia 8138: 2.883 quỹ đạo giữ lại). Ngoài ra đã kiểm `Spatial_Gap = Pos_LV − Pos_FAV − 4,5` (lệch tối đa 3,6e-11 m) nên dùng gap trực tiếp được.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Log 600 đoạn (9,4 MB). Chạy lại cùng mã: 0/600 đoạn khác, 0 đoạn khác bit. Bản refactor float32 + đổi thứ tự phép tính: **600/600 đoạn không giống từng bit** nhưng **0/600 vượt dung sai** (atol = rtol = 1e-3 m/s², lệch lớn nhất 2,1e-05). Phép cộng số thực không kết hợp ((0.1+0.2)+0.3 = 0.6000000000000001 ≠ 0.6), nên so "bằng hệt" sẽ báo động giả cho mọi thay đổi vô hại; dung sai 1e-3 m/s² nhỏ hơn rất nhiều so với độ phân giải có ý nghĩa của lệnh gia tốc nhưng lớn hơn sai số float32 cỡ 1e-5. Thay đổi AEB (phanh −6 khi TTC < 1,5 s) làm lệch **13/600** đoạn (58, 140, 240, …, lệch tối đa 1,73 m/s²). Mở đoạn 58: chỉ các khung 98–101 lệch, ở đó gap 4,30 → 3,62 m, tốc độ tiến gần 2,9 → 2,4 m/s, **TTC 1,43–1,49 s < 1,5 s**, lệnh cũ −5,44 → mới −6,0; đoạn 140 lệch ở khung 34–40 với TTC 1,43–1,49 s. AEB chỉ kích hoạt khi TTC dưới ngưỡng, nên mọi khung khác (và 587 đoạn còn lại) giữ nguyên. Phát nhanh nhất: 0 khung trễ, 0 khung rớt vì không có hạn chót. Phát thời gian thật: **581 khung trễ + 575 khung rớt** trên 59.533 khung, làm **374/600** đoạn đổi lệnh (lệnh cũ bị giữ lại). Phát nhanh nhất che mất lỗi về thời gian tính toán (một khung xử lý lâu hơn chu kỳ 0,1 s của log thì lệnh ra sau khung kế tiếp, hoặc khung kế tiếp bị bỏ) vốn chỉ lộ ra khi thông điệp đến đúng thời điểm của nó.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới cắt vào của tôi: cruise **14 m/s**, gap 2,76–59,31 m (p2–p98 của log) × tốc độ tiến gần 0–14 m/s, 60 × 45 = **2.700 ô**. Biên bất khả thi `gap < dv·t_react + dv²/(2·9)` cho **3,6 %** ô ở t_react = 0 và **10,2 %** ở 0,6 s; bộ mô phỏng phanh hết cỡ khớp đúng biên này (**0 ô lệch** xa biên ở cả hai độ trễ), nên biên đáng tin. Vùng đâm của IDM tăng 3,6 % → 10,2 %, nhưng **trùng khít** vùng bất khả thi (diện tích "tránh được mà vẫn đâm" = 0,0 % ở cả hai), tức mọi va chạm của IDM là "ai cũng đâm", không phải lỗi bộ điều khiển. Ngược lại constant-velocity đâm 80,6 % ô, trong đó 70,4 % (t_react 0,6) là tránh được. Về độ phủ: log có 614 ca xe dẫn phanh (6,4 % quỹ đạo, đỉnh phanh p50 −2,5, p99 −4,2 m/s²); người lái trong log phanh mạnh nhất cỡ **3,6 m/s²** (phân vị 99,9), trong khi **13 %** ô lưới đòi giảm tốc mạnh hơn mức đó. Các chấm trắng (ca khai thác) tụ ở vùng tiến gần thấp, gap vừa; góc tiến gần lớn/gap nhỏ hầu như trống → mô hình học từ log chưa bao giờ thấy vùng khó nhất.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

| Bộ điều khiển | L2 vòng hở (m) | Hạng | L2 lát "xe dẫn phanh" (m) | Va chạm vòng kín (TB qua seed) | Hạng |
|---|---|---|---|---|---|
| const-vel | 1,248 | 3 | 1,897 | 20,0 % | 3 |
| IDM | 1,537 | 4 | 1,607 | 0,03 % | 1 |
| EgoMLP | 0,767 | 2 | 1,140 | 26,9 % | 4 |
| PercMLP | 0,623 | 1 | 0,958 | 1,5 % | 2 |

Bốn cặp đổi thứ tự: (const-vel, IDM), (const-vel, EgoMLP), (IDM, EgoMLP), (IDM, PercMLP). **IDM** có L2 tệ nhất (1,537 m) vì nó không bắt chước người: nó giữ một thời khoảng cách cố định và phản ứng theo luật, khác phong cách người lái trong log, nên dự báo 3 s lệch quỹ đạo người. Nhưng vòng kín chỉ hỏi "có giữ được khoảng cách an toàn không", và IDM có phản hồi trực tiếp từ gap và tốc độ xe dẫn nên chỉ đâm 1/2883. L2 thưởng cho việc *giống người trung bình*, không thưởng cho việc *phản ứng đúng*. **EgoMLP** chỉ thấy lịch sử tốc độ của chính nó: trên log, tốc độ thường đổi chậm nên ngoại suy lịch sử cho L2 0,767 m (tốt hơn const-vel). Nhưng nó **không có gap và tốc độ xe dẫn**, nên khi tự lái nó không biết xe trước phanh (lát phanh 68,6 % va chạm ở seed tệ nhất); lỗi nhỏ tích luỹ không có tín hiệu nào kéo về. Lát "xe dẫn phanh" chỉ chiếm 4,6 % cửa sổ, nên L2 trung bình che lỗi đó (ví dụ const-vel 1,248 → 1,897 m trên lát).

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Với 5 seed (4846–4850), va chạm của EgoMLP là 21,8 / 31,6 / **11,8** / **43,4** / 26,1 %, tức dải **11,8 .. 43,4 %**, dù L2 vòng hở gần như không đổi (0,761–0,780 m). Khoảng Wilson 95 % của seed tốt nhất (4848: 341/2883, **[10,7; 13,1] %**) và seed tệ nhất (4849: 1252/2883, **[41,6; 45,2] %**) **không chồng nhau**: mỗi khoảng hẹp vì n = 2.883 lớn, nên khác biệt không phải nhiễu lấy mẫu mà do chính seed huấn luyện. Wilson chỉ đo bất định do *số quỹ đạo test*, không đo bất định do *quá trình huấn luyện*. Nếu chỉ chạy một seed, tôi có thể báo "EgoMLP đâm 11,8 % ± 1,2" hoặc "43,4 % ± 1,8" (gấp 3,7 lần) với khoảng tin cậy trông rất chắc chắn, và kết luận hoàn toàn phụ thuộc may rủi. Vì thế cổng R5 đòi dải qua seed ≤ 1 điểm và R2 lấy **seed tệ nhất**; EgoMLP rớt R5 (dải 31,6 điểm), còn PercMLP ổn định hơn (1,1 .. 1,9 %).

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Hiệu chỉnh: người trong log giữ thời khoảng cách trung vị 1,90 s; quét T, khoảng cách Wasserstein giữa phân bố thời khoảng cách nhỏ nhất ở **T = 1,4 s (W1 = 0,35)**, so với **0,86** của xe không bao giờ phản ứng, tức xe sau IDM chân thực hơn rõ rệt (xe sau cách 21,6 m, trễ 0,6 s).

Phần lưới có va chạm bất kỳ (trước hoặc sau), t_react = 0,6 s (va chạm phía sau: phát lại / phản ứng):

| Bộ điều khiển | Xe sau phát lại | Xe sau phản ứng | Va chạm sau |
|---|---|---|---|
| const-vel | 80,6 % | 80,6 % | 0,0 / 0,0 % |
| IDM | 80,9 % | **10,2 %** | 70,6 / 0,0 % |
| EgoMLP | 82,1 % | 82,1 % | 0,0 / 0,0 % |
| PercMLP | 74,1 % | 39,5 % | 34,7 / 0,0 % |

(t_react = 0: IDM 80,3 % → 3,6 %, va chạm sau 76,7 % → 0,0 %.) Xe sau phát lại đi theo quỹ đạo ghi sẵn, không biết xe thử vừa phanh, nên bộ điều khiển nào **phanh đúng** trước xe cắt vào thì bị tông từ phía sau, và lỗi bị tính cho nó: IDM bị **70,6 %** va chạm sau giả. Trong thế giới phát lại, IDM (80,9 %) trông **tệ ngang** constant-velocity (80,6 %), thậm chí PercMLP (74,1 %) trông tốt hơn IDM. Khi xe sau phản ứng, va chạm sau về 0 và IDM chỉ còn đúng vùng bất khả thi 10,2 %, trong khi const-vel/EgoMLP vẫn 80–82 % (vì chúng không phanh nên không bị tông từ sau, mà đâm phía trước). Thứ hạng đảo ngược hoàn toàn: loại tác nhân phản ứng khiến ta phạt bộ điều khiển an toàn nhất.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị chặn bởi **R2** (seed tệ nhất 1,91 %, cận trên 2,47 % > 0,03 % + 1 điểm), **R3** (lát phanh 7,9 %, cận trên 12,6 % > 0 + 5), **R4** (29,3 % lưới tránh được mà vẫn đâm > 0 + 2) và **R6** (39,5 % vs 10,2 % + 2). Hai nguyên nhân gốc: (a) **kịch bản log không chứa** — R4/R6: cắt vào với tiến gần lớn gần như không có trong log (13 % lưới đòi phanh mạnh hơn 3,6 m/s² người từng làm), nên mô hình không biết phanh ở đó; (b) **sai số tích luỹ (covariate shift)** — R2/R3: khi tự lái, sai số nhỏ đưa xe vào trạng thái người chưa bao giờ ở (gap ngắn hơn, tiến gần hơn), mô hình chỉ học trạng thái của người nên không biết cách hồi phục. Sửa: (1) thêm 197.220 cửa sổ từ 20.000 ca cắt vào mô phỏng, nhãn = IDM → chữa (a); (2) **DAgger** 2 vòng: để mô hình tự lái log train, gắn nhãn dự báo 3 s của IDM **ở đúng trạng thái nó tự đi vào** → chữa (b). Kết quả PercMLP+: va chạm log 0,3 .. 0,5 % (seed tệ nhất 0,45 %, cận trên 0,77 %), lát phanh 0,5 % (cận trên 2,9 %), lưới 0,6 s 29,3 % → **0,3 %**, thế giới phản ứng 10,5 % vs 10,2 %; qua cả 7 quy tắc → RELEASE. Ở tốc độ chưa thấy 20 m/s nó cũng giữ được (0,3 .. 1,2 % so với PercMLP 17,9 .. 42,4 %).

Nhưng PercMLP+ **không tốt hơn IDM**: cổng chỉ chứng minh **không kém hơn quá biên** (0,45 % vs 0,03 %; 0,3 % vs 0,0 % lưới; 10,5 % vs 10,2 %); trên **mọi** số đo nó vẫn kém IDM một chút. Nó học **từ chính IDM** (nhãn cắt vào và DAgger đều là IDM) nên trần của nó là IDM; nó không thể vượt thầy trên những tiêu chí thầy định nghĩa. L2 vòng hở còn **tăng** 0,623 → 0,822 m (vì giờ bắt chước IDM chứ không bắt chước người) trong khi vòng kín tốt hơn hẳn — thêm bằng chứng điểm vòng hở không đo an toàn. Tôi **không** đề xuất thay IDM ngay: trước hết chạy bóng (shadow mode) song song IDM trên dữ liệu mới, mở rộng kịch bản (xe dẫn phanh gấp, nhiễu cảm biến, tốc độ cao), xác định một lợi ích cụ thể (êm hơn, tiết kiệm hơn) mà IDM không có, rồi mới cân nhắc phát hành có giới hạn.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

1. **Thế giới mô phỏng là giả định của chúng ta.** Vật lý một chiều, giới hạn phanh 9 m/s², trễ 0,6 s và chính biên bất khả thi đều do ta đặt; IDM "0 % tránh được mà vẫn đâm" chỉ đúng trong mô hình này. Mặt đường trơn, tải, độ trễ cơ cấu chấp hành thật có thể dời biên, nên con số 0,3 % của PercMLP+ không chuyển sang đường thật.
2. **Không có cảm biến thật, chỉ có trạng thái hoàn hảo (cộng nhiễu Gauss đơn giản).** Phép thử nhiễu ±1 m / ±0,5 m/s cho PercMLP+ 0,35 .. 0,45 %, nhưng lỗi thật của nhận thức là mất dấu, nhận nhầm vật thể, trễ khung — đúng loại lỗi mà một mô hình học có thể nhạy hơn IDM. Cổng chưa kiểm loại lỗi này.
3. **Xe dẫn trong log không phản ứng với xe thử, và chỉ có bám xe một làn.** Bước 5 cho thấy loại tác nhân phản ứng có thể đảo kết luận (IDM 80,9 % → 10,2 %); ta mới chỉ cho xe sau phản ứng, xe dẫn và xe cắt vào vẫn là kịch bản cứng. Không có chuyển làn, người đi bộ, giao lộ.
4. **Độ phủ dữ liệu và thống kê có hạn.** 2.883 quỹ đạo test, 191 ca xe dẫn phanh: với n = 191, Wilson cho cận trên của PercMLP+ trên lát phanh là 2,9 % (IDM 0/191), quá thô so với mức rủi ro chấp nhận được ngoài đời, vốn nhỏ hơn nhiều bậc. Log Argoverse 2 là đô thị tốc độ thấp (tốc độ xe dẫn trung vị 8,06 m/s, p99 15,55 m/s), và nhãn huấn luyện đến từ IDM nên lỗi của IDM được sao chép mà cổng (so với IDM) không thể phát hiện.

Việc cần làm tiếp: chạy bóng trên xe thật/dữ liệu mới và so quyết định với IDM; mô phỏng có cảm biến (closed-loop sensor sim) và tác nhân phản ứng cho mọi xe xung quanh; khai thác thêm ca hiếm (tìm kiếm đối kháng quanh biên thất bại) đến khi khoảng tin cậy đủ hẹp; kiểm chứng mô phỏng bằng dữ liệu đường thử; và định nghĩa trước tiêu chí lợi ích để phát hành vượt khỏi "không kém hơn".
