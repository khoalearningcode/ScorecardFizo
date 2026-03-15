# ScorecardFizo

Repo này hiện chứa một notebook duy nhất: [Scorecard.ipynb](/home/caokhoa/Documents/ScorecardFizo/Scorecard.ipynb). Notebook dùng để xây dựng scorecard tín dụng từ file dữ liệu người dùng, chọn biến bằng IV/WOE + logistic regression, scale điểm, rồi xuất rule SQL để chấm điểm trên BigQuery.

## Kỹ thuật đang dùng

Đây là một pipeline scorecard kiểu truyền thống trong credit risk, không phải mô hình cây hay boosting. Ý tưởng chính là biến đổi feature về dạng dễ giải thích, fit logistic regression trên không gian WOE, rồi quy đổi xác suất/rủi ro thành điểm số.

Các kỹ thuật chính:

- `WOE` (Weight of Evidence): biến mỗi bin của feature thành một giá trị log-odds giữa nhóm good và bad
- `IV` (Information Value): đo sức phân biệt của từng biến để sàng lọc feature ban đầu
- `Binning`: chia biến số thành các khoảng, gom category hiếm về nhóm chung để mô hình ổn định hơn
- `Correlation pruning`: loại bớt biến WOE quá tương quan để giảm trùng thông tin
- `Logistic Regression`: mô hình tuyến tính trên các biến WOE để sinh scorecard dễ giải thích
- `Score scaling`: chuyển log-odds thành thang điểm theo `BASE_SCORE` và `PDO`

## Điểm mạnh kỹ thuật

- Dùng scorecard truyền thống nên rất mạnh về `explainability`: có thể giải thích điểm số đến từng feature, từng bin.
- Pipeline dùng `WOE + Logistic Regression`, là tổ hợp rất chuẩn cho bài toán credit risk và dễ được business hoặc risk team chấp nhận.
- Có bước `IV screening` để loại nhanh các biến yếu, giúp mô hình gọn hơn và giảm nhiễu ngay từ đầu.
- Có `correlation pruning`, nên hạn chế giữ các biến trùng thông tin và giảm rủi ro đa cộng tuyến.
- Dùng `L1` để chọn biến và `L2` để fit cuối, tức là vừa có tính chọn lọc vừa giữ được độ ổn định của hệ số.
- Có `time-based split` theo `loan_date`, nên cách đánh giá gần với bối cảnh production hơn random split.
- Binning có xử lý `missing` và category hiếm, giúp scorecard robust hơn khi dữ liệu thực tế bẩn hoặc lệch phân phối.
- Chỉ giữ nhóm feature hành vi/velocity/trigger, nên thiên về tín hiệu hành vi thật thay vì phụ thuộc nhiều vào thông tin định danh.
- Kết quả cuối có thể xuất trực tiếp thành `SQL` BigQuery, rất mạnh về khả năng triển khai thực tế.
- Đầu ra không chỉ có mô hình mà còn có bảng bin, bảng summary và rule SQL, nên dễ audit, review và bàn giao.
- Score được scale theo `BASE_SCORE` và `PDO`, tức là đầu ra ở dạng thang điểm chuẩn hoá, phù hợp cho vận hành và thiết kế policy.

## Notebook đang làm gì

Pipeline trong notebook gồm các bước chính:

1. Đọc dữ liệu CSV từ `DATA_PATH`.
2. Chọn target từ `BAD` hoặc `BAD_lock`.
3. Time split theo `loan_date` nếu cột này tồn tại.
4. Lọc feature theo nhóm hành vi/velocity/trigger.
5. Loại cột thiếu dữ liệu quá nhiều, cột hằng, cột bị loại cố định.
6. Binning số và category, tính WOE/IV.
7. Chọn biến bằng IV, tiếp tục prune tương quan.
8. Fit logistic regression để lấy scorecard cuối.
9. Scale score với `BASE_SCORE = 600`, `PDO = 50`.
10. Sinh bảng bin/điểm và SQL để triển khai chấm điểm.

## Phương pháp chi tiết

### 1. Time-based split

Nếu có `loan_date`, notebook chia train/test theo thời gian với tỷ lệ 80/20 thay vì random split. Cách này hợp lý hơn cho bài toán risk vì mô phỏng đúng tình huống triển khai thực tế: huấn luyện trên dữ liệu quá khứ và đánh giá trên dữ liệu tương lai.

### 2. Feature filtering theo domain

Notebook không quét toàn bộ cột một cách mù quáng mà chỉ giữ các pattern hành vi cụ thể như:

- `app_cnt_*d`
- `approve_cnt_*d`
- `outSideApp_cnt_*d`
- `idTrigger_cnt_*d`
- `phoneTrigger_cnt_*d`
- `userTriggerNew_cnt_*d`
- `velocity_*d`
- `acceleration_*d`
- `avg_value_ratio`
- `metrics_exceeded`
- `days_since_last_activity`
- `region_tier`

Một số cột bị loại cứng như thông tin tỉnh/thành hoặc cột mang tính định danh vận hành:

- `vnpostProvinceName`
- `vnpostUsername`
- `vnpostUsername_ne`
- `reasonLock`

Điều này cho thấy notebook đang ưu tiên scorecard hành vi, giảm leakage và tránh nhét các biến khó triển khai sang SQL scoring.

### 3. Binning

Với biến số:

- ép kiểu numeric an toàn
- chia tối đa `6` bins
- mỗi bin phải có ít nhất khoảng `3%` dữ liệu train
- có xử lý riêng cho `missing`

Với biến category:

- nhóm hiếm dưới `1%` về nhãn chung như `__OTHER__`
- giữ `__MISSING__` riêng

Binning là bước quan trọng vì scorecard không dùng trực tiếp raw value. Mỗi feature trước tiên được đổi thành các bin có ý nghĩa hơn về risk, giúp quan hệ với target mượt hơn và dễ diễn giải hơn.

### 4. WOE/IV

Sau khi có bin:

- notebook tính `WOE` cho từng bin
- cộng dồn để ra `IV` cho từng biến
- giữ các biến có `IV >= 0.02`
- riêng `region_tier` chỉ được giữ nếu `IV >= 0.05`
- sau đó giới hạn còn tối đa `20` biến sau vòng IV

Ý nghĩa kỹ thuật:

- `WOE` giúp biến category và numeric bins về cùng một không gian log-odds, rất hợp với logistic regression
- `IV` là bước pre-screen nhanh, đặc biệt hữu ích khi đầu vào có nhiều feature hành vi dạng đếm/trigger

### 5. Refit WOE trên full train

Notebook chỉ dùng sample train để tính IV cho nhanh, nhưng sau khi chọn được biến thì nó fit lại binning và WOE trên toàn bộ tập train. Đây là lựa chọn đúng về mặt kỹ thuật: screening rẻ hơn, còn tham số cuối cùng của scorecard vẫn được học từ full train.

### 6. Correlation pruning

Sau khi đã có ma trận WOE:

- notebook loại biến tương quan cao với ngưỡng `0.85`
- nếu hai biến trùng thông tin, ưu tiên giữ biến có `IV` cao hơn

Bước này giúp scorecard gọn hơn, ổn định hơn, và giảm hiện tượng coefficient bị méo do đa cộng tuyến.

### 7. Logistic regression hai pha

Notebook dùng 2 mô hình logistic:

- `L1 logistic` với `C = 0.5` để chọn biến
- `L2 logistic` với `C = 1.0` để fit mô hình cuối

Lý do:

- pha `L1` đóng vai trò sparse selection, đẩy bớt hệ số về 0
- pha `L2` cho hệ số cuối thường mượt và ổn định hơn để scale thành score

Model được fit với `class_weight="balanced"`, phù hợp khi bad rate lệch đáng kể so với good rate.

### 8. Đánh giá bằng AUC

Notebook tính:

- `AUC train`
- `AUC test`

Đây là metric hợp lý cho scorecard nhị phân vì phản ánh khả năng rank-order giữa good và bad tốt hơn accuracy. Tuy vậy, hiện notebook chưa có KS, PSI hay calibration check.

### 9. Score scaling

Sau khi có logistic regression trên không gian WOE, notebook đổi log-odds sang điểm theo công thức chuẩn của scorecard:

```text
factor = PDO / ln(2)
offset = BASE_SCORE - factor * ln(base_odds)
score = offset - factor * log(odds)
```

Trong notebook:

- `BASE_SCORE = 600`
- `PDO = 50`
- `BASE_ODDS = None`, tức là odds gốc lấy từ tỷ lệ good/bad của train

Điều này có nghĩa là cứ mỗi khi odds tốt/xấu tăng gấp đôi thì điểm sẽ thay đổi 50 điểm.

### 10. Sinh điểm theo từng bin

Sau khi có hệ số `beta` cho từng feature WOE:

- mỗi bin nhận một số điểm riêng
- điểm bin được tính từ `-factor * beta * woe`
- intercept được đưa vào `base_points_int`

Kết quả cuối cùng là một scorecard cộng điểm:

```text
Total Score = Base Points + Sum(Bin Points của từng feature)
```

Đây là lý do notebook có thể xuất thẳng SQL `CASE WHEN` cho từng biến và triển khai trên BigQuery mà không cần mang model binary sang hệ thống scoring.

## Các thư viện cần có

Notebook đang dùng:

- `numpy`
- `pandas`
- `scikit-learn`
- `jupyter`

Cài nhanh:

```bash
pip install numpy pandas scikit-learn jupyter
```

## Cấu hình cần sửa trước khi chạy

Trong notebook có một block `CONFIG` đang hard-code đường dẫn:

```python
DATA_PATH = "/root/test/score/raw_data/user_data.csv"
OUT_DIR   = "/root/test/score/output"
```

Bạn gần như chắc chắn sẽ cần sửa lại 2 giá trị này cho đúng môi trường local/server của mình trước khi chạy.

Các config quan trọng khác:

- `TARGET_CANDIDATES = ["BAD", "BAD_lock"]`
- `DATE_COL = "loan_date"`
- `DROP_ALWAYS = {"vnpostProvinceName", "vnpostUsername", "vnpostUsername_ne", "reasonLock"}`
- `KEEP_PATTERNS`: chỉ giữ các feature dạng `app_cnt_*`, `approve_cnt_*`, `velocity_*`, `acceleration_*`, `region_tier`, ...
- `MISSING_DROP_TH = 0.995`
- `NUM_BINS_MAX = 6`
- `MIN_BIN_PCT = 0.03`
- `CAT_MIN_COUNT_PCT = 0.01`
- `IV_MIN_KEEP = 0.02`
- `REGION_TIER_IV_KEEP = 0.05`
- `MAX_FEATURES_AFTER_IV = 20`
- `CORR_DROP_TH = 0.85`
- `LR_C_SELECT = 0.5`
- `LR_C_FINAL = 1.0`
- `BASE_SCORE = 600`
- `PDO = 50`

## Yêu cầu dữ liệu đầu vào

CSV đầu vào nên có tối thiểu:

- Một cột target: `BAD` hoặc `BAD_lock`
- Nếu muốn chia train/test theo thời gian: cột `loan_date`
- Các cột feature khớp với pattern mà notebook đang giữ lại

Notebook đang ưu tiên các feature hành vi như:

- `app_cnt_*d`
- `approve_cnt_*d`
- `outSideApp_cnt_*d`
- `idTrigger_cnt_*d`
- `phoneTrigger_cnt_*d`
- `velocity_*d`
- `acceleration_*d`
- `avg_value_ratio`
- `metrics_exceeded`
- `days_since_last_activity`
- `region_tier`

## Cách chạy

Mở notebook:

```bash
jupyter notebook
```

Sau đó chạy file [Scorecard.ipynb](/home/caokhoa/Documents/ScorecardFizo/Scorecard.ipynb) từ trên xuống dưới.

Nếu muốn chạy không cần giao diện:

```bash
jupyter nbconvert --to notebook --execute Scorecard.ipynb
```

## Kết quả đầu ra

Notebook sẽ tạo thư mục output nếu chưa tồn tại và ghi ra 3 file:

- `scorecard_bins_no_province.csv`
- `scorecard_feature_summary_no_province.csv`
- `user_score_scorecard_no_province.sql`

Ý nghĩa:

- `scorecard_bins_no_province.csv`: định nghĩa bin, WOE, điểm cho từng feature
- `scorecard_feature_summary_no_province.csv`: tóm tắt feature được chọn và thông tin mô hình
- `user_score_scorecard_no_province.sql`: SQL BigQuery để tính score

Khi chạy xong notebook sẽ in:

- target được dùng
- IV của `region_tier`
- danh sách feature cuối
- `AUC train` và `AUC test`
- đường dẫn các file đã lưu

## Lưu ý

- Notebook hiện chỉ có 1 cell code dài, chưa tách module.
- Đường dẫn input/output đang là đường dẫn tuyệt đối.
- Đây là scorecard interpretable, phù hợp khi cần explainability và triển khai SQL, nhưng thường sẽ thua các model tree boosting nếu mục tiêu chỉ là tối đa predictive power.
- Vì WOE/IV phụ thuộc khá mạnh vào phân phối train, nên khi đưa production nên theo dõi thêm drift như `PSI`.
- Nếu chia sẻ cho team, nên cân nhắc refactor notebook thành script hoặc package nhỏ để dễ review và chạy tự động.
