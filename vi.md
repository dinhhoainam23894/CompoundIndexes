[Source](https://mariadb.com/kb/en/library/compound-composite-indexes/ "Permalink to Compound (Composite) Indexes - MariaDB Knowledge Base")

# Các chỉ số tổng hợp (Composite) - cơ sở tri thức MariaDB

## Một bài học nhỏ trong "chỉ số hợp nhất" ("chỉ số tổng hợp")

Tài liệu này bắt đầu có vẻ tầm thường và nhàm chán, nhưng càng về sau xây dựng lên các thông tin thú vị hơn về những điều có lẽ bạn chưa biết về cách hoạt động của chỉ mục trong MariaDB và MySQL.

Điều này cũng giải thích [EXPLAIN][1] (đến một mức độ nào đó).

(Hầu hết điều này cũng áp dụng cho các cơ sở dữ liệu không phải MySQL)

## Truy vấn để thảo luận

Câu hỏi đặt ra "Khi nào Andrew Johnson là tổng thống của Hoa Kỳ?".

Bảng `Presidents` có sẵn như sau :
    
    
    +-----+------------+----------------+-----------+
    | seq | last_name  | first_name     | term      |
    +-----+------------+----------------+-----------+
    |   1 | Washington | George         | 1789-1797 |
    |   2 | Adams      | John           | 1797-1801 |
    ...
    |   7 | Jackson    | Andrew         | 1829-1837 |
    ...
    |  17 | Johnson    | Andrew         | 1865-1869 |
    ...
    |  36 | Johnson    | Lyndon B.      | 1963-1969 |
    ...
    

("Andrew Johnson" đã được chọn cho bài học này vì các bản sao.)

Chỉ số nào sẽ là tốt nhất cho câu hỏi đó? Cụ thể hơn, điều tốt nhất cho
    
    
        SELECT  term
            FROM  Presidents
            WHERE  last_name = 'Johnson'
              AND  first_name = 'Andrew';
    

một vài INDEX để thử...

* No indexes 
* INDEX(first_name), INDEX(last_name) (two separate indexes) 
* "Index Merge Intersect" 
* INDEX(last_name, first_name) (a "compound" index) 
* INDEX(last_name, first_name, term) (a "covering" index) 
* Variants 

## No indexes

Vâng, Tôi đang giả lập 1 chút ở đây. Tôi có 1 PRIMARY KEY trên `seq`, nhưng điều đó không có lợi thế về truy vấn chúng tôi đang nghiên cứu.
    
    
    mysql>  SHOW CREATE TABLE Presidents G
    CREATE TABLE `presidents` (
      `seq` tinyint(3) unsigned NOT NULL AUTO_INCREMENT,
      `last_name` varchar(30) NOT NULL,
      `first_name` varchar(30) NOT NULL,
      `term` varchar(9) NOT NULL,
      PRIMARY KEY (`seq`)
    ) ENGINE=InnoDB AUTO_INCREMENT=45 DEFAULT CHARSET=utf8
    
    mysql>  EXPLAIN  SELECT  term
                FROM  Presidents
                WHERE  last_name = 'Johnson'
                  AND  first_name = 'Andrew';
    +----+-------------+------------+------+---------------+------+---------+------+------+-------------+
    | id | select_type | table      | type | possible_keys | key  | key_len | ref  | rows | Extra       |
    +----+-------------+------------+------+---------------+------+---------+------+------+-------------+
    |  1 | SIMPLE      | Presidents | ALL  | NULL          | NULL | NULL    | NULL |   44 | Using where |
    +----+-------------+------------+------+---------------+------+---------+------+------+-------------+
    
    # Or, using the other form of display:  EXPLAIN ... G
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: ALL        <-- Implies table scan
    possible_keys: NULL
              key: NULL       <-- Implies that no index is useful, hence table scan
          key_len: NULL
              ref: NULL
             rows: 44         <-- That's about how many rows in the table, so table scan
            Extra: Using where
    

## Chi tiết triển khai

Trước tiên, hãy mô tả cách InnoDB lưu trữ và sử dụng các chỉ mục.

* Dữ liệu và PRIMARY KEY được "nhóm" với nhau trên BTree. 
* Tra cứu BTree khá nhanh và hiệu quả. Đối với một bảng hàng triệu có thể có 3 cấp độ của BTree, và hai cấp cao nhất có thể được lưu trữ. 
* Mỗi chỉ số phụ nằm trong một BTree khác, với PRIMARY KEY ở lá.
* Tìm nạp các mục 'liên tiếp' (theo chỉ mục) từ BTree rất hiệu quả vì chúng được lưu trữ liên tiếp. 
* Để đơn giản, chúng ta có thể đếm từng tra cứu BTree dưới dạng 1 đơn vị công việc và bỏ qua các lần quét cho các mục liên tiếp. Điều này xấp xỉ số lần truy cập đĩa cho một bảng lớn trong một hệ thống bận.

Đối với MyISAM, PRIMARY KEY không được lưu trữ với dữ liệu, vì vậy hãy nghĩ về nó như là một khóa thứ cấp (quá đơn giản).

## INDEX(first_name), INDEX(last_name)

Người mới làm quen, một khi anh ta biết về lập chỉ mục, quyết định lập chỉ mục nhiều cột, mỗi cột một lần. Nhưng...

MySQL hiếm khi sử dụng nhiều hơn một chỉ mục tại một thời điểm trong một truy vấn. Vì vậy, nó sẽ phân tích các chỉ số có thể.

* first_name -- có 2 hàng có thể (một tra cứu BTree, sau đó quét liên tục)
* last_name -- có 2 hàng có thể Giả sử nó chọn last_name. Dưới đây là các bước để thực hiện SELECT: 1. Sử dụng INDEX (last_name), tìm 2 mục chỉ mục với last_name = 'Johnson'. 2. Lấy chìa khóa PRIMARY (được bổ sung hoàn toàn vào mỗi chỉ số phụ trong InnoDB); nhận được (17, 36). 3. Tiếp cận dữ liệu bằng cách sử dụng seq = (17, 36) để lấy các hàng cho Andrew Johnson và Lyndon B. Johnson. 4. Sử dụng phần còn lại của mệnh đề WHERE lọc tất cả trừ hàng mong muốn. 5. Cung cấp câu trả lời (1865-1869). 
    
    
    mysql>  EXPLAIN  SELECT  term
                FROM  Presidents
                WHERE  last_name = 'Johnson'
                  AND  first_name = 'Andrew'  G
      select_type: SIMPLE
            table: Presidents
             type: ref
    possible_keys: last_name, first_name
              key: last_name
          key_len: 92                 <-- VARCHAR(30) utf8 may need 2+3*30 bytes
              ref: const
             rows: 2                  <-- Two 'Johnson's
            Extra: Using where
    

## "Hợp nhất phần giao của các chỉ mục"

OK,Vậy bạn thực sự thông minh và quyết định rằng MySQL đủ thông minh để sử dụng cả hai chỉ mục tên để t được câu trả lời. Điều này được gọi là "intersect". 1. Sử dụng INDEX (last_name), tìm 2 mục chỉ mục với last_name = 'Johnson'; nhận được (7, 17) 2. Sử dụng INDEX (first_name), tìm 2 mục chỉ mục với first_name = 'Andrew'; nhận được (17, 36) 3. "Và" hai danh sách cùng nhau (7,17) & (17,36) = (17) 4. Tiếp cận dữ liệu bằng cách sử dụng seq = (17) để lấy hàng cho Andrew Johnson. 5. Cung cấp câu trả lời (1865-1869).
    
    
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: index_merge
    possible_keys: first_name,last_name
              key: first_name,last_name
          key_len: 92,92
              ref: NULL
             rows: 1
            Extra: Using intersect(first_name,last_name); Using where
    

The EXPLAIN không cung cấp thông tin chi tiết về số lượng hàng được thu thập từ mỗi chỉ mục, v.v.

## INDEX(last_name, first_name)

Đây được gọi là chỉ mục "hợp nhất" hoặc "tổng hợp" vì nó có nhiều hơn một cột. 1. Tìm hiểu về BTree để chỉ mục có được chính xác hàng chỉ mục cho Johnson + Andrew; get seq = (17). 2. Tiếp cận dữ liệu bằng cách sử dụng seq = (17) để lấy hàng cho Andrew Johnson. 3. Cung cấp câu trả lời (1865-1869). Thế này tốt hơn. Trong thực tế, điều này thường là "tốt nhất".
    
    
        ALTER TABLE Presidents
            (drop old indexes and...)
            ADD INDEX compound(last_name, first_name);
    
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: ref
    possible_keys: compound
              key: compound
          key_len: 184             <-- The length of both fields
              ref: const,const     <-- The WHERE clause gave constants for both
             rows: 1               <-- Goodie!  It homed in on the one row.
            Extra: Using where
    

## "Covering": INDEX(last_name, first_name, term)

Ngạc nhiên chưa! Chúng ta thực sự có thể làm tốt hơn một chút. Chỉ mục "Covering" là chỉ mục trong đó tất cả các trường của SELECT được tìm thấy trong chỉ mục. Nó có thêm tiền thưởng không cần phải tiếp cận vào "dữ liệu" để hoàn thành nhiệm vụ. 1. Tìm hiểu về BTree để chỉ mục có được chính xác hàng chỉ mục cho Johnson + Andrew; get seq = (17). 2. Cung cấp câu trả lời (1865-1869). "dữ liệu" BTree không được chạm vào; đây là một cải tiến về "hợp nhất".
    
    
        ... ADD INDEX covering(last_name, first_name, term);
    
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: ref
    possible_keys: covering
              key: covering
          key_len: 184
              ref: const,const
             rows: 1
            Extra: Using where; Using index   <-- Note
    

Mọi thứ đều tương tự như sử dụng "hợp nhất", ngoại trừ việc bổ sung "Sử dụng chỉ mục".

## Biến thể

* Điều gì sẽ xảy ra nếu bạn xáo trộn các trường trong mệnh đề WHERE? Trả lời: Thứ tự của những thứ ANDed không quan trọng.
* Điều gì sẽ xảy ra nếu bạn xáo trộn các trường trong INDEX? Trả lời: Nó có thể tạo nên sự khác biệt lớn. Xem thêm sau một phút nữa.
* Điều gì sẽ xảy ra nếu có thêm trường vào cuối? Trả lời: Tác hại rất ít; có thể mang về rất nhiều lợi ích (ví dụ, 'covering'). 
* Reduncancy? Đó là, nếu bạn có cả hai thứ này: INDEX (a), INDEX (a, b)? Trả lời: Reduncy trả phí một cái gì đó trên INSERTs; nó hiếm khi hữu ích cho các SELECT.
* Prefix? Tức là, INDEX (last_name (5). First_name (5)) Trả lời: Đừng bận tâm; nó hiếm khi giúp ích, và thường có hại. (Chi tiết là một chủ đề khác.)

## Nhiều ví dụ hơn:
    
    
        INDEX(last, first)
        ... WHERE last = '...' -- good (even though `first` is unused)
        ... WHERE first = '...' -- index is useless
    
        INDEX(first, last), INDEX(last, first)
        ... WHERE first = '...' -- 1st index is used
        ... WHERE last = '...' -- 2nd index is used
        ... WHERE first = '...' AND last = '...' -- either could be used equally well
    
        INDEX(last, first)
        Both of these are handled by that one INDEX:
        ... WHERE last = '...'
        ... WHERE last = '...' AND first = '...'
    
        INDEX(last), INDEX(last, first)
        In light of the above example, don't bother including INDEX(last).
    

## Postlog

Đã làm mới -- Oct, 2012; nhiều link hơn -- Nov 2016
