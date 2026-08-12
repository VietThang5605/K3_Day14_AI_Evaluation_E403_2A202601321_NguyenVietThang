# Báo cáo Thực hiện Task 1 — Data Models (`QAPair` & `EvalResult`)

**Ngày hoàn thành:** 12/08/2026  
**File thực hiện:** [`template.py`](file:///Users/nguyenvietthang/Coding/VinUni/Lab/K3_Day14_AI_Evaluation_E403_2A202601321_NguyenVietThang/template.py)  
**Tác giả:** AI Evaluation Pipeline Developer  

---

## 1. Những gì đã làm (What Was Done)

Trong Task 1, chúng tôi đã triển khai hoàn thiện hai data models chuẩn hóa bằng Python `dataclasses` trong file [`template.py`](file:///Users/nguyenvietthang/Coding/VinUni/Lab/K3_Day14_AI_Evaluation_E403_2A202601321_NguyenVietThang/template.py):

### 1.1 Khai báo Dataclass `QAPair`
Đại diện cho một câu hỏi – câu trả lời mẫu thuộc Golden Dataset (gồm 20 test cases):

```python
@dataclass
class QAPair:
    question: str
    expected_answer: str
    context: str | None = ""
    metadata: dict = field(default_factory=dict)
    retrieved_contexts: list = field(default_factory=list)
```

### 1.2 Khai báo Dataclass `EvalResult`
Đại diện cho kết quả chấm điểm của 1 `QAPair` sau khi đi qua hệ thống RAG:

```python
@dataclass
class EvalResult:
    qa_pair: QAPair
    actual_answer: str
    faithfulness: float
    relevance: float
    completeness: float
    passed: bool
    failure_type: str | None = None
    context_precision: float | None = None
    context_recall: float | None = None

    def overall_score(self) -> float:
        return (self.faithfulness + self.relevance + self.completeness) / 3.0
```

---

## 2. Tại sao thực hiện như vậy (Design Rationale & Why)

1. **Tuân thủ quy tắc Dataclasses trong Python (Fields Ordering)**:
   - Đặt các thuộc tính không có giá trị mặc định (`question`, `expected_answer` trong `QAPair`; `qa_pair`, `actual_answer`, `faithfulness`, `relevance`, `completeness`, `passed` trong `EvalResult`) lên trên.
   - Các thuộc tính có mặc định (`context`, `metadata`, `retrieved_contexts`, `failure_type`, `context_precision`, `context_recall`) đặt ở dưới để tránh lỗi `TypeError: non-default argument follows default argument`.

2. **Tránh lỗi Mutable Default Arguments (`default_factory`)**:
   - Trong Python, việc viết `metadata: dict = {}` hay `retrieved_contexts: list = []` sẽ khiến mọi instance dùng chung một object dict/list ngầm định trong bộ nhớ.
   - Do đó, bắt buộc phải dùng `field(default_factory=dict)` và `field(default_factory=list)` để mỗi instance tạo ra có một dict/list hoàn toàn độc lập.

3. **Tính linh hoạt kiểu dữ liệu (`context: str | None = ""`)**:
   - Đôi khi dữ liệu mock hoặc test cases truyền `None` vào vị trí của `context`. Việc định nghĩa kiểu `str | None` giúp loại bỏ hoàn toàn cảnh báo/lỗi Type Checker và runtime error.

4. **Tách biệt Answer Metrics và Retrieval Metrics trong `overall_score()`**:
   - `overall_score()` được định nghĩa chính xác là trung bình cộng của 3 metrics phía Generator: $\text{Faithfulness}$, $\text{Answer Relevance}$, và $\text{Completeness}$.
   - Hai chỉ số `context_precision` và `context_recall` phục vụ việc chẩn đoán khâu Retriever, giữ độc lập (`None` nếu không cung cấp `retrieved_contexts`) và **không** tham gia làm biến động `overall_score()`, tuân thủ đúng yêu cầu thiết kế pipeline RAGAS trong lab.

---

## 3. Kết quả đạt được (Results)

### 3.1 Kết quả chạy Automated Unit Tests
Đã tiến hành kiểm thử các hàm và thuộc tính của Task 1 bằng `pytest`:

```bash
.venv/bin/pytest tests/test_solution.py -k TestEvalResultOverallScore -v
```

**Kết quả từ Terminal:**
```text
============================= test session starts ==============================
platform darwin -- Python 3.11.15, pytest-9.1.1, pluggy-1.6.0
collected 42 items / 39 deselected / 3 selected

tests/test_solution.py::TestEvalResultOverallScore::test_all_ones_returns_one PASSED [ 33%]
tests/test_solution.py::TestEvalResultOverallScore::test_all_zeros_returns_zero PASSED [ 66%]
tests/test_solution.py::TestEvalResultOverallScore::test_correct_average PASSED [100%]

======================= 3 passed, 39 deselected in 0.01s =======================
```

### 3.2 Kiểm thử Khởi tạo Instance (Sanity Check)
Chạy thử script kiểm tra khởi tạo `QAPair` và `EvalResult` bằng Python CLI:

```python
from template import QAPair, EvalResult

qa = QAPair("Q", "A")
res = EvalResult(qa, "A", 0.9, 0.9, 0.9, True)

print("Overall Score:", res.overall_score())  # Outputs: 0.9
print("Metadata default:", qa.metadata)       # Outputs: {}
print("Retrieved default:", qa.retrieved_contexts) # Outputs: []
```
$\rightarrow$ Tất cả các trường mặc định và phương thức tính toán hoạt động chính xác $100\%$.

---

## 4. Các lưu ý quan trọng (Important Notes & Caveats)

1. **Thứ tự tham số khi khởi tạo không dùng keyword**:
   - Khi khởi tạo `QAPair` theo dạng Positional: `QAPair(question, expected_answer, context, metadata, retrieved_contexts)`.
   - Khi khởi tạo `EvalResult` theo dạng Positional: `EvalResult(qa_pair, actual_answer, faithfulness, relevance, completeness, passed, failure_type, context_precision, context_recall)`.

2. **Tiêu chuẩn Đạt (`passed`)**:
   - Trường `passed` là boolean đại diện cho câu trả lời đạt chất lượng. Quy tắc chuẩn của pipeline: `passed = True` khi $\text{faithfulness} \ge 0.5$, $\text{relevance} \ge 0.5$ và $\text{completeness} \ge 0.5$.

3. **Liên kết với các Task tiếp theo**:
   - Ở **Task 2b** và **Task 4**, khi chạy evaluation full pipeline với `run_full_eval()`, cần chắc chắn truyền `pair.retrieved_contexts` để hai thuộc tính `context_precision` và `context_recall` trong `EvalResult` được tính toán và điền giá trị thay vì để mặc định `None`.
