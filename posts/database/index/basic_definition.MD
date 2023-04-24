# Basic definition

Series này sẽ tập trung tìm hiểu và thảo luận về index trong SQL. Chúng ta sẽ nói về index là gì, cách tổ chức data trong index, index hoạt động ra sao, làm thế nào để có thể sử dụng index hiệu quả.

Trong series này, Postgres sẽ được sử dụng làm môi trường test chính, thỉnh thoảng các hệ quản trị cơ sở dữ liệu khác có thể được sử dụng để so sánh với nhau.

Các định nghĩa cơ bản cần nắm được:

- `explain analyze <query>` Dùng để phân tích câu query mình đang chạy. Câu lệnh này sẽ **ước tính** xem câu query này nên sử dụng plan nào, bao nhiêu record có thể bị scan qua, thời gian thực thi có thể là bao lâu
