---
layout: post
title: Giao dịch phân tán
description: Giao dịch phân tán
published: true
---

Đây là ghi chú cho thảo luận của nhóm Hệ thống phân tán của Grokking Việt Nam.

Atomicity mà một trong bốn thuộc tính ACID của một hệ quan trị CSDL. Atomicity
cung cấp một cơ chế xử lý trong trường hợp có một (hoặc nhiều) hành động ghi nào
đó bị lỗi.

Ví dụ: chuyến một số tiền X từ tài khoản A sang B bao gồm 2 hành động:
(a) Balance(A) = Balance(A) - X;
(b) Balance(B) = Balance(B) + X.

Với một Atomic transaction thì chỉ có thể xảy ra một trong hai trường hợp:
1. Tài khoản A và B giữ nguyên nếu một trong hai hành động (a) (b) lỗi hoặc cả
   hai bị lỗi
2. Tài khoản A bị trừ số tiền X, B nhận được số tiền X nếu (a) và (b) thành
   công.


## 1. Một node

Với transaction được thực hiện trong một node CSDL, atomicity được thực thi bởi
storage engine. Khi một client yêu cầu DB commit một transaction, DB trước tiên
viết các hành động ghi ổ đĩa (thường là WAL), sau đó thêm một `commit record`
vào log này.

Thứ tự ghi này là rất quan trọng. Nếu DB bị crash giữa quá trình này,
transaction sẽ được khôi phục từ log: nếu commit record được tìm thấy từ log,
transaction được xem là thành công' ngược lại tất cả các hành động ghi sẽ bị thu
hồi. Thời điểm ghi bàn trong một transaction là thời điểm ổ đĩa hoàn thành việc
ghi commit record: Trước đó transaction có thể bị hủy bỏ, nhưng sau đó
transaction chính thức được commit ngay cả khi DB bị crash.

Trong trường hợp nhiều node tham gia vào hệ thống, một transaction không đơn
giản là gởi đến tất cả các node và để chúng độc lập commit transaction, vì có
thể dẫn đến kết quả một số node hoành thành transaction, một số không vì:
- một node có theer phát hiện thấy dữ liệu không hợp lệ, dẫn đến hủy
  transaction, trong khi các node khác thì không
- một node không nhận được request vì yêu cầu bị mất trên đường truyền mạng,
  trong khi các node khác thì không
- một node bị crash trước khi ghi `commit record` do đó  transaction bị thu hồi,
  trong khi các node khác thì không.

Nếu một vài node commit, các node khác không, hệ thống không đạt được sự đúng
đắn của atomicity: tất cả đều được commit hoặc tất cả bị bỏ qua.

## 2. Two-phase commit (2PC)

2PC là một giải thuật kinh điển để giải quyết vấn đề atomicity trong hệ thống có
nhiều bên (participant) để đảm bảo rằng tất cả hoặc commit, hoặc bỏ qua.

![2PC](/images/2021-05-06-distributed-transaction/2pc.png){:width="800px" :class="img-responsive"}

Hình: Sơ đồ thời gian cho 2PC, sử dụng 3N message. Mỗi participant quản lý mỗi
recovery log riêng biệt. Tham khảo [1].

Một 2PC bắt đầu với app viết và đọc vào các node participant như thường lệ. Khi
client sẵn sàng cho commit, coordinator (C) bắt đầu phase 1: C gởi `prepare`
request đến tất cả các node, hỏi rằng chúng đã sẵng sàng để commit chưa. C sau
đó xử lý các phản hồi:
- Nếu tất cả các participant trả lới Có, C sẽ gởi ra yêu cầu `commit` trong
  phase 2. Lúc này transaction được commit.
- Nếu một trong các participant trả lời Không, C sẽ gởi yêu cầu `abort` đến tất
  cả các participant trong phase 2.

Chi tiết:
1. Khi một app muốn bắt đầu một transaction, nó yêu cầu transactionId từ C.
   transactionId này là duy nhất.
2. App bắt đầu transaction trên mỗi participant đính kèm transactionId ở bước 1.
   Hành động đọc/ghi được thực thi trong transaction trên node này. Nếu bất cứ
   đều gì xảy ra sai, C có thể abort transaction.
3. Khi app sẵng sàng để commit, C gởi yêu cầu prepare đến các participant, đính
   kèm với transactionId. Nếu bất kỳ một trong các yêu cầu này bị từ chối hoặc
   timeout, C gởi yêu cầu abort cho transactionId đến tất cả participant.
4. Khi participant nhận một yêu cầu chuẩn bị commit, nó đảm bảo rằng có thể
   commit transaction dưới tất cả các điều kiện, bao gồm viết các dữ liệu
   transaction xuống đĩa, và kiểm tra conflict, constrain. Khi một node trả lời
   Có, nó phải đảm bảo rằng transaction phải được commit mà không có bất cứ lỗi
   nào khi có yêu cầu.
5. Khi C nhận được trả lời từ tất cả participant, nó được ra quyết định commit
   hoặc abort. C cần phải viết quyết định này xuống transaction log ở ổ đĩa, nhờ
   đó có thể khôi phục trong trường hợp C bị crash. Đây gọi là thời điểm `commit
   point`. Lưu ý: lúc này ta có 2 `point of no failure`, một ở C, một ở các
   participant.
6. Sau khi quyết định của C được ghi xuống đĩa, yêu cầu commit/abort được gởi
   tới các participant. Trường hợp fail, C cần thử lại vô tận cho đến khi thành
   công.


## 3. Vấn đề của 2PC:

1. C về bản chất là một DB, và nó cần được đối xử như các DB khác. Nếu C không
   được replicated mà chạy trên một node, đó là SPF.
2. Participant có thể phải chờ vô tận


## 4. Hỏi đáp

1. Có thể sử dụng Raft thay cho 2PC?
2. Tại sao participant cần giữ lock cho đến khi transaction được commit/abort?

## 5. Sự thay thế:

SAGA: https://microservices.io/patterns/data/saga.html

// TODO: write here

Transactional message (TM): Khi tất cả bạn muồn là atomic {cập nhật DB, publish
một event/message}.

Có 2 cách thực thi cho TM: transaction outbox, và transaction log tailing.

### 5.1. Transaction outbox

![Mô hình Transactional
Outbox](/images/2021-05-06-distributed-transaction/transaction-outbox.png){:width="800px"
:class="img-responsive"}

Hình: Mô hình Transaction Outbox pattern với `Message Relay`. Tham khảo [3].

Giải pháp:
- Với RDBM, thêm các message/event vào một bảng outbox trong cùng một
  transaction
- Với NoSQL, DB thêm các message/event vào record (document/item) đang được cập
  nhật

Một `Message Relay` process độc lập đẩy message/event này vào message broker.

Vấn đề:
1. `Message Relay` có thể publish message nhiều hơn một lần (Vì sao?). Vì vậy
   message consumer cần phải có thuộc tính idempotent. Tuy nhiên `Message
   Broker` cũng có thể giao lặp mại một message/event nên đây thường không phải
   là vấn đề.
2. Polling DB thì đơn giản, nhưng chỉ hoạt động tốt với scale nhỏ.

```
SELECT * FROM OUTBOX ORDERED BY ... ASC
BEGIN
  DELETE FROM OUTBOX WHERE ID in (....)
COMMIT
```

### 5.2. Transaction log tailing

![](https://microservices.io/i/patterns/data/TransactionLogMining.png){:width="800px"
:class="img-responsive"}

Hình: Message/event, lấy từ Outbox table bằng cách khai thác DB transaction log, được
đưa vào broker. Tham khảo [3].

Ví dụ
1. https://debezium.io/ publish thay đổi trong DB vào Kafka
2. DynamoDB stream
   https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html
3. https://github.com/eventuate-tram/eventuate-tram-core sử dụng MySQL binlog,
   Postgres WAL để lấy các thay đổi trong DB rồi publish vào Kafka

## 6. Tham khảo:

- [1] Principles of Computer System Design, Chapter 9 Atomicity: All-or-Nothing
  and Before-or-After
- [2] Designing Data-Intensive Applications, Chapter 9 Consistency and Consensus
- [3] Microservices Patterns With examples in Java, Chris Richardson