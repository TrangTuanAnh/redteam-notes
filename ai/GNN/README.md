# GNN

Tên đầy đủ: Graph Neural Network
Lĩnh vực  : Deep Learning / Graph-based Learning
Đọc trước : Bài này là nền tảng để hiểu [HGAT](../HGAT/)

---

## Đồ thị là gì?

Trước khi nói đến GNN, cần hiểu tại sao đồ thị (graph) lại quan trọng.

- Hầu hết dữ liệu mà chúng ta quen thuộc có dạng **bảng** (tabular) — mỗi hàng là một mẫu, mỗi cột là một đặc trưng. Mô hình học máy truyền thống hoạt động rất tốt với dạng dữ liệu này.
- Nhưng rất nhiều thứ trong thực tế **không** có dạng bảng — chúng là mạng lưới:
  - Mạng xã hội: người dùng (nút) kết nối với nhau qua tình bạn (cạnh)
  - Phân tử hóa học: nguyên tử (nút) liên kết với nhau qua liên kết hóa học (cạnh)
  - Trang web: trang (nút) trỏ đến nhau qua hyperlink (cạnh)
  - Văn bản: từ/câu (nút) liên quan đến nhau qua ngữ nghĩa (cạnh)

- Một đồ thị $G = (V, E)$ gồm:
  - $V$ — tập hợp các **nút** (nodes/vertices), mỗi nút $v$ mang một vector đặc trưng $x_v$
  - $E$ — tập hợp các **cạnh** (edges), biểu diễn quan hệ giữa các nút
  - $A$ — **ma trận kề** (adjacency matrix): $A_{ij} = 1$ nếu có cạnh nối nút $i$ và $j$, ngược lại bằng 0

> Câu hỏi đặt ra: làm thế nào để một mô hình học máy xử lý được dữ liệu dạng đồ thị, nơi mà mỗi nút có số lượng láng giềng khác nhau và không có thứ tự cố định?

---

## Ý tưởng cốt lõi — Message Passing

GNN giải quyết câu hỏi trên bằng một ý tưởng đơn giản: **mỗi nút học biểu diễn của mình bằng cách tổng hợp thông tin từ các nút láng giềng**.

Quá trình này được gọi là **message passing** (lan truyền thông điệp), diễn ra qua nhiều lớp (layer):

- **Bước 1 — Aggregate:** Nút $v$ thu thập thông tin từ tất cả các nút láng giềng $u \in \mathcal{N}(v)$
- **Bước 2 — Combine:** Kết hợp thông tin thu thập được với biểu diễn hiện tại của chính nó
- **Bước 3 — Update:** Cập nhật biểu diễn của nút $v$

Sau $L$ lớp, biểu diễn của một nút đã "hấp thụ" thông tin từ các nút cách nó tối đa $L$ bước trên đồ thị.

Hình dung đơn giản: hỏi một người về đặc điểm của một khu phố — sau 1 lần hỏi, họ biết về hàng xóm trực tiếp; sau 2 lần hỏi, họ biết cả về hàng xóm của hàng xóm. Càng nhiều lớp, "tầm nhìn" càng rộng.

---

## Graph Convolutional Network (GCN)

GCN (Kipf & Welling, 2017) là một trong những hiện thực hóa đơn giản nhất của message passing, lấy cảm hứng từ phép tích chập trong CNN nhưng áp dụng lên đồ thị.

Công thức cập nhật giữa các lớp:

$$H^{(l+1)} = \sigma\!\left(\tilde{A} \cdot H^{(l)} \cdot W^{(l)}\right)$$

Giải thích từng thành phần:

- $H^{(l)}$ — ma trận biểu diễn của tất cả các nút ở lớp $l$. Ban đầu $H^{(0)} = X$ (đặc trưng gốc)
- $\tilde{A} = D^{-1/2} A' D^{-1/2}$ — ma trận kề đã chuẩn hóa (có thêm self-loop $A' = A + I$), giúp mỗi nút cũng tổng hợp thông tin của chính nó
- $W^{(l)}$ — ma trận trọng số học được ở lớp $l$ (shared toàn bộ đồ thị)
- $\sigma$ — hàm kích hoạt (thường dùng ReLU)

Phép nhân $\tilde{A} \cdot H^{(l)}$ thực chất là **lấy trung bình có chuẩn hóa** các biểu diễn láng giềng. Sau đó $W^{(l)}$ chiếu kết quả sang không gian mới.

**Hạn chế của GCN:**

- Mọi láng giềng đều được coi là **quan trọng như nhau** — không phân biệt nút nào mang nhiều thông tin hơn
- Chỉ hoạt động tốt với đồ thị **đồng nhất** — tất cả nút đều cùng loại và cùng không gian đặc trưng

---

## Graph Attention Network (GAT)

GAT giải quyết hạn chế đầu tiên của GCN: thay vì lấy trung bình đều, nó học **trọng số chú ý** cho từng cạnh — nút láng giềng nào quan trọng hơn thì được tổng hợp với tỉ trọng cao hơn.

Với một nút $v$ và láng giềng $u$, trọng số chú ý được tính:

$$e_{vu} = \text{LeakyReLU}\!\left(\vec{a}^T \cdot [W h_v \| W h_u]\right)$$

Sau đó chuẩn hóa bằng softmax trên tất cả láng giềng:

$$\alpha_{vu} = \frac{\exp(e_{vu})}{\sum_{k \in \mathcal{N}(v)} \exp(e_{vk})}$$

Biểu diễn mới của nút $v$:

$$h_v' = \sigma\!\left(\sum_{u \in \mathcal{N}(v)} \alpha_{vu} \cdot W h_u\right)$$

Ý nghĩa trực quan: hai nút $v$ và $u$ "nhìn" vào nhau, tính xem mức độ tương thích là bao nhiêu, rồi dùng con số đó làm trọng số khi tổng hợp thông tin.

---

## Vẫn còn hạn chế — Đồ thị dị thể (Heterogeneous Graph)

Cả GCN lẫn GAT đều được thiết kế cho **đồ thị đồng nhất** — tất cả nút có cùng loại và cùng không gian đặc trưng.

Trong thực tế, nhiều bài toán có đồ thị **dị thể** (heterogeneous), nơi các nút thuộc nhiều loại khác nhau:

- Văn bản (vector TF-IDF, kích thước = kích thước từ điển)
- Chủ đề (vector phân phối xác suất từ LDA)
- Thực thể (vector Word2Vec ghép TF-IDF của mô tả Wikipedia)

Ba loại này có **kích thước vector khác nhau** và **ý nghĩa khác nhau**. Nếu áp GCN/GAT thẳng vào, mô hình sẽ bị lẫn lộn giữa các không gian đặc trưng.

> Đây chính là bài toán mà **HGAT** giải quyết — mở rộng GAT để xử lý đồ thị dị thể bằng cách thêm ma trận chiếu riêng cho từng loại nút, kết hợp với cơ chế chú ý hai cấp (type-level + node-level).

---

*Tham khảo: [Kipf & Welling, 2017 — Semi-supervised Classification with GCN](https://arxiv.org/abs/1609.02907) | [Veličković et al., 2018 — GAT](https://arxiv.org/abs/1710.10903)*
