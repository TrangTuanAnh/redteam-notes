# GNN

Tên đầy đủ: Graph Neural Network
Lĩnh vực: Deep Learning / Graph-based Learning
Đọc trước: Bài này là nền tảng để hiểu [HGAT](../HGAT/)

---

## Đồ thị là gì và tại sao cần nó?

Hầu hết dữ liệu mà các mô hình ML/DL truyền thống xử lý tốt đều có dạng **bảng** — mỗi mẫu là một hàng, mỗi đặc trưng là một cột. Ảnh có thể coi là lưới pixel, văn bản là chuỗi từ — đều có cấu trúc đều đặn, có thứ tự cố định.

Vấn đề là rất nhiều dữ liệu trong thực tế không có dạng đó — chúng là **mạng lưới quan hệ**:

- Mạng xã hội: người dùng kết nối qua tình bạn
- Phân tử hóa học: nguyên tử liên kết qua liên kết hóa học
- Trang web: trang trỏ đến nhau qua hyperlink
- Văn bản: từ và câu liên quan đến nhau qua ngữ nghĩa

Dữ liệu dạng này được biểu diễn bằng **đồ thị** $G = (V, E)$ — gồm tập nút $V$ (mỗi nút $v$ mang một vector đặc trưng $x_v$) và tập cạnh $E$ (quan hệ giữa các nút). $A$ là ma trận kề, $A_{ij} = 1$ nếu có cạnh nối nút $i$ với $j$.

Câu hỏi đặt ra: mô hình học máy phải làm sao khi mỗi nút có số lượng láng giềng khác nhau, không có thứ tự cố định như ảnh hay chuỗi?

---

## Message Passing — ý tưởng cốt lõi

GNN trả lời câu hỏi đó bằng cơ chế **message passing**: mỗi nút học biểu diễn của mình bằng cách tổng hợp thông tin từ các nút láng giềng, qua nhiều lớp liên tiếp.

Mỗi lớp gồm 3 bước: nút $v$ thu thập thông tin từ tất cả láng giềng $u \in \mathcal{N}(v)$, kết hợp với biểu diễn hiện tại của chính nó, rồi cập nhật biểu diễn mới. Sau $L$ lớp, biểu diễn của một nút đã hấp thụ thông tin từ các nút cách nó tối đa $L$ bước trên đồ thị.

Cứ nghĩ như hỏi thăm hàng xóm về đặc điểm một khu phố — hỏi một lần thì biết về hàng xóm trực tiếp, hỏi hai lần thì biết cả hàng xóm của hàng xóm. Càng nhiều lớp, thông tin càng lan rộng.

---

## Graph Convolutional Network (GCN)

GCN (Kipf & Welling, 2017) là một trong những cách hiện thực hóa message passing đơn giản nhất, lấy cảm hứng từ phép tích chập trong CNN nhưng áp dụng lên đồ thị.

Công thức cập nhật giữa các lớp:

$$H^{(l+1)} = \sigma\!\left(\tilde{A} \cdot H^{(l)} \cdot W^{(l)}\right)$$

- $H^{(l)}$ là ma trận biểu diễn của tất cả nút ở lớp $l$; ban đầu $H^{(0)} = X$ (đặc trưng gốc)
- $\tilde{A} = D^{-1/2} A' D^{-1/2}$ là ma trận kề đã chuẩn hóa, với $A' = A + I$ (tự nối vào chính nó để mỗi nút cũng tổng hợp thông tin của bản thân)
- $W^{(l)}$ là ma trận trọng số học được ở lớp $l$, dùng chung cho toàn bộ đồ thị
- $\sigma$ là hàm kích hoạt, thường dùng ReLU

Phép nhân $\tilde{A} \cdot H^{(l)}$ thực chất là lấy trung bình có chuẩn hóa biểu diễn của các láng giềng. Sau đó $W^{(l)}$ chiếu kết quả sang không gian mới.

Điểm yếu: mọi láng giềng đều được xem **quan trọng như nhau** — không phân biệt nút nào mang nhiều thông tin hơn. Ngoài ra GCN chỉ hoạt động tốt với đồ thị **đồng nhất** — tất cả nút cùng loại, cùng không gian đặc trưng.

---

## Graph Attention Network (GAT)

GAT (Veličković et al., 2018) giải quyết vấn đề đầu tiên của GCN: thay vì lấy trung bình đều, nó học **trọng số chú ý** cho từng cạnh.

Với nút $v$ và láng giềng $u$, trọng số chú ý được tính:

$$e_{vu} = \text{LeakyReLU}\!\left(\vec{a}^T \cdot [W h_v \| W h_u]\right)$$

rồi chuẩn hóa bằng softmax:

$$\alpha_{vu} = \frac{\exp(e_{vu})}{\sum_{k \in \mathcal{N}(v)} \exp(e_{vk})}$$

Biểu diễn mới của nút $v$:

$$h_v' = \sigma\!\left(\sum_{u \in \mathcal{N}(v)} \alpha_{vu} \cdot W h_u\right)$$

Hai nút "nhìn" vào nhau để tính mức độ tương thích, rồi dùng con số đó làm trọng số khi tổng hợp thông tin. Láng giềng liên quan hơn thì đóng góp nhiều hơn.

---

## Vấn đề còn lại — đồ thị dị thể

Cả GCN lẫn GAT đều giả định đồ thị **đồng nhất**: tất cả nút có cùng loại, cùng không gian đặc trưng.

Nhiều bài toán thực tế không như vậy. Ví dụ bài toán phân loại văn bản ngắn, nếu xây đồ thị kết hợp:

- Nút văn bản — vector TF-IDF, kích thước bằng kích thước từ điển
- Nút chủ đề — vector phân phối xác suất từ LDA
- Nút thực thể — vector Word2Vec ghép TF-IDF của mô tả Wikipedia

Ba loại này có **kích thước khác nhau** và **ý nghĩa khác nhau**. Áp thẳng GCN hay GAT vào là mô hình bị lẫn lộn giữa các không gian đặc trưng.

> Đây là bài toán **HGAT** giải quyết — mở rộng GAT cho đồ thị dị thể bằng cách thêm ma trận chiếu riêng cho từng loại nút, kết hợp với cơ chế chú ý hai cấp (type-level + node-level).

---

*Tham khảo: [Kipf & Welling, 2017 — GCN](https://arxiv.org/abs/1609.02907) | [Veličković et al., 2018 — GAT](https://arxiv.org/abs/1710.10903)*
