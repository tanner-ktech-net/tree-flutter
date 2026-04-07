# Report: Kiến trúc Cây (Trees Architecture) trong Flutter

## 1. Giới thiệu tổng quan

Trong **Declarative UI paradigm (Mô hình giao diện khai báo)** của Flutter, giao diện người dùng (UI) không chỉ được tạo nên bởi một "cây" duy nhất. Thực chất, Flutter vận hành một hệ thống gồm **3 cây (Trees)** hoạt động song song để đảm bảo hiệu suất kết xuất (rendering) cao cấp ở mức 60-120fps. Ba cây này là:

- **Widget Tree** (Cây Widget)
- **Element Tree** (Cây Element)
- **RenderObject Tree** (Cây RenderObject)

## 2. Phân tích chi tiết 3 loại Cây

### A. Widget Tree (Cây cấu hình)

- **Định nghĩa**: Là nơi chứa **Configuration (cấu hình)** UI do lập trình viên khai báo. Nó mô tả UI trông như thế nào ở một thời điểm nhất định (state). Widget thực chất chỉ là một **Blueprint (bản thiết kế)** hay **Data class** đơn thuần.
- **Đặc điểm**:
  - **Immutable (Bất biến)**: Không thể thay đổi các thuộc tính sau khi đã khởi tạo (`@immutable`). Bất cứ khi nào **State** thay đổi, Flutter sẽ phá hủy và tạo ra một Widget Tree hoàn toàn mới.
  - **Lightweight (Siêu nhẹ)**: Việc tạo đi tạo lại hàng ngàn đối tượng Widget cực kỳ nhanh và chi phí thấp, vì chúng không chứa các mã vẽ (painting code) hay tính toán bố cục tiêu tốn tài nguyên.

### B. Element Tree (Cây quản lý - Người điều phối)

- **Định nghĩa**: Là bản thể hiện thực hoá (Instantiation) của Widget, đóng vai trò cầu nối giao tiếp giữa Widget và RenderObject. Đáng chú ý, đối tượng **`BuildContext`** mà lập trình viên thường dùng ở tham số hàm `build()` chính là interface hiển thị ra của một `Element` tại vị trí đó trên cây.
- **Đặc điểm**:
  - **Mutable (Có thể biến đổi)**: Không giống như Widget, Element tồn tại lâu dài trên bộ nhớ và giữ **References (tham chiếu)** tới cả Widget (cấu hình hiện tại) và RenderObject (bản vẽ hiển thị).
  - **State and Lifecycle Management**: Lưu trữ và quản lý `State` (đối với StatefulWidget). Khi Widget Tree xây mới, Element Tree không bị bẻ gãy. Nó sử dụng cơ chế **Diffing / Reconciliation (Thuật toán đối soát)** để so sánh Widget mới và cũ dựa trên `runtimeType` và `Key`.
  - Nếu `runtimeType` và `Key` khớp nhau, Element tiếp tục giữ nguyên `State` mà chỉ đơn giản là cập nhật tham chiếu sang vùng nhớ của Widget phiên bản mới.

### C. RenderObject Tree (Cây hiển thị)

- **Định nghĩa**: Là cấu trúc dữ liệu nền tảng thực hiện mọi công việc tính toán nặng nhọc: **Layout (tính toán không gian, kích thước)**, **Painting (vẽ các pixels)** ra màn hình và **Hit-testing (cảm biến sự kiện chạm)**.
- **Đặc điểm**:
  - **Heavyweight (Rất nặng)**: Việc khởi tạo và tính toán RenderObject rất phức tạp và tiêu tốn tài nguyên thiết bị di động (CPU/GPU).
  - **Tối ưu hoá cao độ**: Nhờ thuật toán đối soát của Element Tree, RenderObject hiếm khi bị dỡ bỏ. Nó chỉ thay đổi thuộc tính (mutate internal properties) khi Element ra lệnh.
  - **Quy tắc cốt lõi (Layout Protocol)**: Áp dụng quy tắc vàng nguyên thủy của nền tảng UI Flutter: *"Constraints go down. Sizes go up. Parent sets position."* (Các giới hạn ràng buộc không gian đi xuống, kích thước ước lượng đi lên báo lại, và Widget cha quyết định toạ độ kết xuất của con).

## 3. Vòng luân chuyển (Lifecycle Workflow)

1. **Initialization (Khởi tạo)**: Hàm `runApp()` tiếp nhận Widget gốc của bạn.
2. **Mounting (Gắn kết)**: Flutter duyệt qua Widget Tree. Ứng với mỗi Widget, phương thức `createElement()` được gọi ra để sinh ra một Node trên Element Tree nhằm theo dõi Widget đó.
3. **Rendering & Painting**: Khi một Element được Mount, nếu Widget tương ứng của nó thuộc nhánh `RenderObjectWidget`, nó sẽ kích hoạt `createRenderObject()` để sinh ra một Node trên RenderObject Tree làm nhiệm vụ phân tích đo đạc màn hình và vẽ.
4. **Rebuilding (Khi `setState` được gọi)**:
   - Các cấu hình Model bị thay đổi làm cho phương thức `build()` chạy lại sinh ra một mạng lưới Widget Tree mới cho khu vực đó.
   - Quá trình **Reconciliation** kích hoạt, Element Tree sẽ bước vào quá trình traverse (duyệt) và so sánh cây Widget mới với thông tin cũ đã lưu (tương tự như React Virtual DOM).
   - Nếu phát hiện thuộc tính (color, padding...) thay đổi, Element gọi hàm `updateRenderObject()` để gán các thuộc tính mới vào RenderObject. RenderObject lập tức tự gán nhãn là **dirty (bẩn)** để chờ tính toán vẽ lại khung hình (**Next Frame**) trên màn ảnh.

## 4. Tại sao lại cần tới 3 kiến trúc Cây?

Kiến trúc phân xử 3 lớp Trees cho phép Flutter đạt được khả năng kiến trúc toàn diện: Tận dụng trọn vẹn quy trình **Declarative, Reactive Programming** với tốc độ phát triển cực nhanh của React (thích thay đổi, rebuild lại toàn bộ UI ở bất kì sự kiện nào) nhưng vẫn duy trì đỉnh cao **High-Performance (hiệu suất trên 60fps)** mượt mà.

Sự tự do này được trả giá bằng thiết kế bất biến giá rẻ của **Widget**, sự quản lý bền bỉ thông qua phép tính so khớp (diffing) của **Element**, và khối lượng tính toán vật lý không lãng phí của **RenderObject**.

---
*Report được tổng hợp tự động để hỗ trợ nghiên cứu kiến trúc hệ thống Flutter.*
