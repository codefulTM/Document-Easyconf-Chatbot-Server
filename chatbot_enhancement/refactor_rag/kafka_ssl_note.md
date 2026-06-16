- Một số khái niệm:
+ SSL: đường truyền mã hóa giữa server và client
+ public key: key của server mà các client muốn gửi dữ liệu tới phải có. dùng để tạo "mật mã dùng chung"
+ private key: key của riêng server, CHỈ SERVER CÓ, dùng để decode nội dung đã được encode bởi public key ở phía client.
+ mật mã dùng chung: do client không thể decode nội dung mà server gửi(do không có private key) nên phải tạo mật mã dùng chung được encode bằng public key và gửi cho server -> đây là mật mã dùng để trò chuyện cho cả client và server sau quá trình trao đổi public key.
+ certificate: là CCCD chứa public key. có certificate hợp lệ thì client mới tin rằng public key là real do chính server chính chủ gửi.
+ CA(certificate authority): tổ chức chứng thực chứng chỉ. server nào được chứng thực rồi thì lúc vào url của server sẽ có hình ổ khóa + https.
<!-- Tới đây là các khái niệm dùng cho kết nối SSL giữa 2 container. Lúc này, 2 bên đều cần có certificate riêng để khỏi phải cho một bên ký với một CA có thẩm quyền -->
+ CA chung/Root CA/Self-signed CA: là CA tự mình tạo ra, có vai trò đứng giữa 2 bên và nói: "2 người có thể tin tưởng lẫn nhau rồi đó."
+ ca-cert: Là certificate CỦA CA, có chức năng đặc biệt là bên nào có certificate này thì đều là người nhà với nhau, tin tưởng nhau tuyệt đối(nhưng đương nhiên phải có certificate ở cả hai bên mới tin).
+ ca-key: Là con dấu của CA để ký vào certificate của 2 bên.
+ keystore: nơi chứa certificate(chứa public key) + private key.
+ truststore: chứa chứng chỉ của CA chung
+ CSR(yêu cầu ký chứng chỉ): tờ đơn đăng ký làm certificate. sau khi điền thông tin vào "đơn" và gửi cho CA thì CA sẽ kiểm tra và cấp certificate cho. Có thể tạo CSR qua công cụ như OpenSSL hay Keytool. CSR thường chứa: 
    + Thông tin định danh:
        + common name(CN) - quan trọng nhất: Tên miền như kafka, localhost, google.com,...
        + organization(O): Tên công ty/tổ chức
        + country(C): Mã quốc gia
    + Khóa công khai(public key)
<!-- END -->

- Luồng SSL giữa server và client:
+ B1: browser chào server
+ B2: server gửi certificate(hiểu nôm na là cái CCCD của server chứa public key)
+ B3: browser thấy certification được public CA công nhận, tạo key chung rồi lấy public key encode lại, sau đó gửi key chung cho server
+ B4: server lấy private key decode key chung đó. lúc này cả 2 bên đều có key chung
+ B5: browser và server trao đổi dữ liệu được encoded bởi key chung này. Không ai biết key chung này là gì, chỉ browser(người tạo key chung) và server(người có khả năng decode sử dụng private key) mới biết -> mấy người nghe lén có nghe lén được thì cũng không decode được.

- Luồng SSL giữa 2 container:
+ B1: Tạo CA chung
```
openssl req -new -x509 -keyout ca-key -out ca-cert -days 3650 -subj "/CN=MySelfSignedCA" -nodes
```
-> lệnh sinh ra ca-cert để bỏ vào truststore của 2 bên.
+ B2: Tạo keystore và truststore cho cả hai bên
    + Mỗi bên làm như sau:
        + Tạo keystore
        + Tạo file yêu cầu ký(CSR) từ keystore vừa tạo
        + Dùng CA ở B1 ký vào file CSR này
        + Nhập chứng chỉ CA(cần để bên kia so khớp với chứng chỉ CA trong truststore) và chứng chỉ đã ký ngược lại vào Keystore
        + Tạo truststore
    -> Output là thư mục ./secrets chứa:
    + Các file *.keystore.jks, *.truststore.jks. 
    + Cả keystore và truststore ở hai bên đều có chứng chỉ CA.
    + Note: Do truststore của 2 bên chỉ chứa đúng 1 thứ duy nhất là chứng chỉ CA chung -> chỉ cần tạo global.truststore.jks duy nhất, nạp CA certificate chung vào rồi mount dùng chung cho cả 2 bên. Không cần tạo kafka.truststore.jks, connect.truststore.jks riêng biệt.