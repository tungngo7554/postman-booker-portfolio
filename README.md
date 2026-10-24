# Postman API Testing: Booking Lifecycle

Bộ kiểm thử API tự động bằng Postman cho [Restful-Booker](https://restful-booker.herokuapp.com) (API luyện tập công khai). Một lần bấm Run chạy trọn vòng đời dữ liệu: đăng nhập, tạo, đọc, sửa, xóa, xác nhận đã xóa, cùng các trường hợp lỗi.

## Kỹ năng thể hiện

- Gửi và đọc response các method GET, POST, PUT/PATCH, DELETE
- Collection và Environment (`baseUrl`, `username`, `password`, `token`, `bookingId`)
- Chaining: lưu `token` và `bookingId` từ response để request sau dùng lại
- Pre-request script sinh dữ liệu ngẫu nhiên (tên, giá)
- Assertion tự động: status code, kiểu dữ liệu, so khớp dữ liệu gửi và nhận
- Negative test: thiếu token, ID không tồn tại, thiếu field bắt buộc
- Data-driven testing với CSV (5 iteration)
- Chạy bằng Collection Runner

## Cấu trúc collection

| # | Request | Mục đích | Kiểm tra |
|---|---|---|---|
| 1 | `Auth_Get token` | Đăng nhập, lưu token | 200, token là chuỗi không rỗng |
| 2 | `Booking-Create` | Tạo booking với dữ liệu ngẫu nhiên hoặc CSV | 200, `bookingid` là số, dữ liệu trả về khớp dữ liệu gửi |
| 3 | `Booking-Get by id` | Đọc booking vừa tạo | 200, đúng booking vừa tạo |
| 4 | `Booking-Update` | Sửa tên (PATCH, kèm token) | 200, tên đã đổi |
| 5 | `Booking-Delete` | Xóa booking | 201 |
| 6 | `Get after delete` | Xác nhận đã xóa | 404 |
| 7 | `Delete without token` | Negative: xóa không có token | 403 |
| 8 | `Booking - Create invalid` | Negative: thiếu `firstname` | Bị từ chối (4xx/5xx) |
| 9 | `Booking - Get not found` | Negative: ID không tồn tại | 404 |

## Script tiêu biểu

**Lưu token (`Auth_Get token`, After response)**
```js
const data = pm.response.json();
pm.environment.set("token", data.token);

pm.test("Status là 200", () => pm.response.to.have.status(200));
pm.test("Có token", () => {
  pm.expect(data.token).to.be.a("string").and.not.empty;
});
```

**Sinh dữ liệu (`Booking-Create`, Before request)** ưu tiên dữ liệu từ CSV, không có thì tự sinh ngẫu nhiên:
```js
const fn = pm.iterationData.get("firstname") || pm.variables.replaceIn("{{$randomFirstName}}");
const ln = pm.iterationData.get("lastname") || pm.variables.replaceIn("{{$randomLastName}}");
const pr = pm.iterationData.get("totalprice") || _.random(100, 999);

pm.variables.set("firstName", fn);
pm.variables.set("lastName", ln);
pm.variables.set("price", pr);
```

**So khớp dữ liệu và chaining (`Booking-Create`, After response)**
```js
const res = pm.response.json();

pm.test("Status 200", () => pm.response.to.have.status(200));
pm.test("bookingid là số", () => pm.expect(res.bookingid).to.be.a("number"));
pm.test("Dữ liệu trả về khớp dữ liệu gửi", () => {
  pm.expect(res.booking.firstname).to.eql(pm.variables.get("firstName"));
  pm.expect(res.booking.lastname).to.eql(pm.variables.get("lastName"));
  pm.expect(res.booking.totalprice).to.eql(Number(pm.variables.get("price")));
});

pm.environment.set("bookingId", res.bookingid);
pm.environment.set("savedFirstName", pm.variables.get("firstName"));
```

## Cách chạy lại

1. Mở Postman, nhập collection và environment trong repo này.
2. Chọn environment `Booker - Prod` ở góc trên bên phải.
3. Chuột phải collection, chọn **Run collection**.
4. Với data-driven: chọn file `bookings.csv` ở mục Iteration data, đặt Iterations là 5.
5. Bấm **Start run**. Kết quả mong đợi: 0 lỗi, tất cả test pass.

## Kết quả

![Kết quả Collection Runner](screenshots/runner-result.png)

## Phát hiện trong quá trình kiểm thử

- API trả **500** thay vì **400** khi thiếu field bắt buộc (`firstname`). Lỗi do dữ liệu phía client gửi lên nên đúng ra phải là mã 4xx.
- `DELETE` thành công trả **201** thay vì 200/204, khác với quy ước REST thông thường.
- Token có thời hạn, nên luôn chạy `Auth_Get token` ngay trước các request cần xác thực.

## Bài học rút ra

- Khi test lỗi: đọc status code, rồi tab Body, rồi mới đến script.
- Lỗi đầu tiên trong một chuỗi thường kéo theo các lỗi sau (ví dụ Create fail thì `bookingId` trống).
- Negative test phải độc lập, không phụ thuộc biến do request khác tạo ra, để kết quả giống nhau dù chạy lẻ hay chạy cả collection.
- Phân biệt **Failed** (test sai kết quả) và **Errors** (request không gửi được).

## Cấu trúc repo

```
postman-booker-portfolio/
├── README.md
├── collection/        # collection Postman
├── environment/       # environment (đã xóa thông tin nhạy cảm)
├── data/bookings.csv
└── screenshots/runner-result.png
```
