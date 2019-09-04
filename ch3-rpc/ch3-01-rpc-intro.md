# 3.1 Bắt đầu với RPC
[Remote Procedure Call](https://en.wikipedia.org/wiki/Remote_procedure_call) ( viết tắt RPC) là phương pháp gọi hàm từ một máy tính từ xa để lấy kết quả. Trong lịch sử phát triển của internet, RPC đã trở thành một cơ sở hạ tầng không thể thiếu cũng như là IPC (inter procress communication) ngoài việc chúng đều dùng để giao tiếp giữa các máy tính chứ không chỉ là các tiến trình. Ngoài ra RPC còn hay được sử dụng trong các hệ thống phân tán.

## 3.1.1 Chương trình “Hello World” bằng RPC

Thư viện chuẩn của GO chứa gói net/rpc dùng để cài đặt chương trình RPC, chương trình RPC đầu tiên của chúng ta sẽ in ra chuỗi “Hello World” được tạo ra và trả về từ máy khác.
service/hello.go: định nghĩa service Hello.

```go
package service
//định nghĩa struct register service
type HelloService struct { }
// định nghĩa hàm service Hello, quy tắc:
// 1. Hàm service phải public (viết Hoa)
// 2. Có hai tham số trong hàm
// 3. Tham số thứ hai phải kiểu con trỏ
// 4. Phải trả về kiểu error
func (p *HelloService) Hello(request string, reply *string) error {
    *reply = "Hello " + request
    // trả về error = nil nếu thành công
    return nil
}

```
**server/main.go**: chương trình phía server.

```go
package main
import (
  "log"
  "net"
  "net/rpc"
  // import rpc service
  "../service"
)
func main() {
    // đăng ký tên service với đối tượng rpc service
    rpc.RegisterName("HelloService", new(HelloService))
    // chạy rpc server trên port 1234
    listener, err := net.Listen("tcp", ":1234")
    // nếu có lỗi thì in ra
    if err != nil {
        log.Fatal("ListenTCP error:", err)
    }
    // vòng lặp để xử lý nhiều client connect
    for {
        // chấp nhận một connection đến
        conn, err := listener.Accept()  
        // in ra nếu bị lỗi khi Accept
        if err != nil {
            log.Fatal("Accept error:", err)
        }
        // service RPC cho client trên một goroutine khác để giải phóng
        // main thread tiếp tục connect client khác
        rpc.ServeConn(conn)
    }   
}
```
**client/main.go**: mã nguồn client để gọi HelloService.

```go
package main
import (
  "fmt"
  "log"
  "net/rpc"
)
func main() {
    // kết nối đến rpc server
    client, err := rpc.Dial("tcp", "localhost:1234")
    // in ra lỗi nếu có
    if err != nil {
        log.Fatal("dialing:", err)
    }
    // biến chứa giá trị trả về sau lời gọi rpc
    var reply string
    // gọi rpc với tên service đã register, tham số và biến
    err = client.Call("HelloService.Hello", "hello", &reply)
    if err != nil {
        log.Fatal(err)
    }
    // in ra kết quả
    fmt.Println(reply)
}
```
Kết quả khi chạy HelloSerivce:

```sh
$ go run server/main.go
```
Phía client:

```sh
$ go run client/main.go
Hello World
```
Qua ví dụ trên, có thể thấy rằng việc dùng RPC trong GO thật sự đơn giản.

## 3.1.2 Tạo interface cho RPC

Ứng dụng sử dụng RPC sẽ có ít nhất ba thành phần:

- Chương trình cài đặt phương thức RPC ở bên phía server.
- Chương trình gọi RPC bên phía client
- ***Service*** đóng vai trò là interface giữa server và client

Trong ví dụ trên, chúng ta đã đặt tất cả những thành phần trên trong 3 folders server, client, service. Nếu bạn muốn refactor lại mã nguồn HelloService, đầu tiên hãy tạo interface như sau:

***Interface của RPC service***:

```go
// tên của service, chứa tiền tố pkg để tránh xung đột tên về sau
const HelloServiceName = "path/to/pkg.HelloService"
// interface RPC của HelloService
type HelloServiceInterface = interface {
    // định nghĩa danh sách các function trong service
    Hello(request string, reply *string) error
}
// đăng ký service
func RegisterHelloService(svc HelloServiceInterface) error {
    // gọi hàm register của net/rpc package
    return rpc.RegisterName(HelloServiceName, svc)
}
```

Sau khi định nghĩa lớp interface của RPC service, client có thể viết mã nguồn để gọi RPC command:

Cài đặt phía client:

```go
// hàm main phía client
func main() {
    // kết nối rpc server qua port 1234
    client, err := rpc.Dial("tcp", "localhost:1234")
    // log ra lỗi nếu có
    if err != nil {
        log.Fatal("dialing:", err)
    }
    // biến chứa kết quả sau khi gọi rpc
    var reply string
    // gọi hàm RPC được định nghĩa phía server
    err = client.Call(HelloServiceName+".Hello", "hello", &reply)
    // log ra chi tiết lỗi nếu có
    if err != nil {
        log.Fatal(err)
    }
}
```

Tuy nhiên, gọi các phương thức RPC thông qua hàm client.Call vẫn rất cồng kềnh, để đơn giản chúng ta nên wrapper biến connection vào trong struct:
Wrapper các đối tượng:

```go
// struct chứa các đối tượng
type HelloServiceClient struct {
    // wrapper server connection
    *rpc.Client
}
var _ HelloServiceInterface = (*HelloServiceClient)(nil)
// tạo hàm wrapper lời gọi Dial tới server
func DialHelloService(network, address string) (*HelloServiceClient, error) {
    // gọi Dial tới server bên trong
    c, err := rpc.Dial(network, address)
    // trả về rỗng và lỗi nếu có
    if err != nil {
        return nil, err
    }
    /trả về rpc struct và error=nil nếu thành công
    return &HelloServiceClient{Client: c}, nil
}
//wrapper lại lời gọi hàm Hello phía client
func (p *HelloServiceClient) Hello(request string, reply *string) error {
    return p.Client.Call(HelloServiceName+".Hello", request, reply)
}
```
Dựa trên các hàm wrapper, chúng ta viết lại client:
Hàm main phía client sau khi refactor:

```go
// kết nối RPC Server bằng hàm wrapper
func main() {
    client, err := DialHelloService("tcp", "localhost:1234")
    if err != nil {
        log.Fatal("dialing:", err)
    }
    // biến lưu kết quả từ lời gọi RPC
    var reply string
    err = client.Hello("hello", &reply)
    // log ra lỗi nếu có
    if err != nil {
        log.Fatal(err)
    }
}
```
Cuối cùng, phía server được viết lại như sau:

Chương trình phía bên server:

```go
//đối tượng RPC HelloService
type HelloService struct {}
// implement từ RPC
func (p *HelloService) Hello(request string, reply *string) error {
    *reply = "hello:" + request
    return nil
}
// hàm main phía server
func main() {
    // gọi wrapper đăng ký đối tượng HelloService
    RegisterHelloService(new(HelloService))
    // lắng nghe kết nối từ phía client
    listener, err := net.Listen("tcp", ":1234")
    // log ra lỗi nếu có (VD: trùng port, vvv)
    if err != nil {
        log.Fatal("ListenTCP error:", err)
    }
    // vòng lặp tiếp nhận nhiều client connect
    for {
        // chấp nhận 1 client connect nào đó.
        conn, err := listener.Accept()
        if err != nil {
            log.Fatal("Accept error:", err)
        }
        // connect service trên một gorotine khác để main thread tiếp tục vòng lặp accept client khác
        go rpc.ServeConn(conn)
    }
}
```
Ở phiên bản refactor, chúng ta sử dụng hàm RegisterHelloService để đăng ký RPC service, nó tránh việc trực tiếp đặt tên cho serice, và đảm bảo mọi đối tượng implement các hàm trong interface của RPC service đều có thể phục vụ lời gọi RPC từ phía client.

## 3.1.3 Vấn đề gọi RPC trên các ngôn ngữ khác nhau

Trong hệ thống microservice, mỗi service có thể viết bằng các ngôn ngữ lập trình khác nhau, do đó để ***cross-language*** (vượt qua rào cả ngôn ngữ) là điều kiện thiết yếu cho sự tồn tại của RPC trong môi trường internet.

Thư viện chuẩn RPC của GO mặc định đóng gói dữ liệu theo đặc tả của [GO Encoding](https://golang.org/pkg/encoding/), do đó sẽ rất khó để gọi RPC Service từ những ngôn ngữ khác.

May mắn là thư viện ```net/rpc``` của GO có ít nhất hai thiết kế đặc biệt:
- Một là cho phép chúng ta có thể thay đổi quá trình encoding và decoding gói tin RPC.
- Hai là interface RPC được xây dựng dựa trên interface ```io.ReadWriteClose```, chúng ta có thể xây dựng RPC trên những protocol giao tiếp khác nhau.

Từ đây chúng ta có thể triển khai cross-language thông qua gói ```net/rpc/jsonrpc```:

```go
package main
import(
	"log"
	"net"
	"net/rpc"
	"net/rpc/jsonrpc"
)
// define register service struct
type HelloService struct {}
func (p *HelloService) Hello(request string, reply *string) error {
	*reply = "Hello, " + request
	// return error = null nếu thành công
	return nil
}
func main() {
	 // register HelloService (dùng cách cũ cho đơn giản)
    rpc.RegisterName("HelloService", new(HelloService))
    // lắng nghe connection từ client
    listener, err := net.Listen("tcp", ":1234")
    if err != nil {
        log.Fatal("ListenTCP error:", err)
    }
    for {
        conn, err := listener.Accept()
        if err != nil {
            log.Fatal("Accept error:", err)
        }
		 // client service trên một goroutine khác, lúc này:
		 // 1. rpc.ServeConn được thay thế bằng rpc.ServeCodec
		 // 2. dùng jsonrpc.NewServerCodec để bao đối tượng conn
        go rpc.ServeCodec(jsonrpc.NewServerCodec(conn))
    }
}
```
Mã nguồn bên phía client sẽ thay đổi như sau:

***Hàm main bên phía client***:

```go
package main
import (
	"fmt"
	"log"
	"net"
	"net/rpc"
	"net/rpc/jsonrpc"
)
func main() {
	 // connect to RPC server
    conn, err := net.Dial("tcp", "localhost:1234")
    if err != nil {
        log.Fatal("net.Dial:", err)
    }
	 // call RPC Service đc encoding bằng json Codec
    client := rpc.NewClientWithCodec(jsonrpc.NewClientCodec(conn))
	 // biến lưu giá trị sau lời gọi hàm rpc
    var reply string
    // call RPC Service
    err = client.Call("HelloService.Hello", "hello", &reply)
    if err != nil {
        log.Fatal(err)
    }
	 // in ra kết quả
    fmt.Println(reply)
}
```
***Kết quả***:

- Server

```sh
$ go run server/main.go
```

- Client:

```sh
$ go run client/main.go
Hello, World
```

Để thấy dữ liệu được client gửi cho server, đầu tiên tắt chương trình server gọi lệnh [nc](http://www.tutorialspoint.com/unix_commands/nc.htm):

```sh
$ go run server/main.go
// Ctrl+C
$nc -l 1234
```
Sau đó gọi chương trình client ```$ go run client/main.go``` một lần nữa, ta sẽ thấy kết quả:

```json
$ nc -l 1234
{"method":"HelloService.Hello","params":["World"],"id":0}
```
Dữ liệu json trên tương ứng với hai cấu trúc: client là ```clientRequest``` và server là ```serverRequest```. Nội dung của cấu trúc ```clientRequest``` và ```serverRequest``` về cơ bản là giống nhau:

```go
// cấu trúc json phía client
type clientRequest struct {
    Method string         `json:"method"`
    Params [1]interface{} `json:"params"`
    Id     uint64         `json:"id"`
}
// cấu trúc json phía server
type serverRequest struct {
    Method string           `json:"method"`
    Params *json.RawMessage `json:"params"`
    Id     *json.RawMessage `json:"id"`
}
```
Ở chiều ngược lại, nếu muốn thấy thông điệp mà phía server gửi cho client, chạy RPC service phía server: ``` $go run server/main.go ``` và ở một terminal khác chạy command:

```sh 
$ echo -e '{"method":"HelloService.Hello","params":["hello"],"id":1}' | nc localhost 1234
```

Kết quả mà RPC server trả về

```json
{"id":1,"result":"Hello, World","error":null}
```
Trong đó:

- "id": để nhận dạng kết quả ứng với yêu cầu vì việc thực thi lời gọi RPC là bất đồng bộ
- "result": kết quả trả về của lời gọi hàm
- "error": chứa thông điệp lỗi nếu có

Dữ liệu json được trả về ở trên sẽ tương ứng với hai cấu trúc client là ```clientResponse``` và server là ```serverResponse```. Nội dung của hai cấu trúc cũng tương tự nhau:

```go
type clientResponse struct {
    Id     uint64           `json:"id"`
    Result *json.RawMessage `json:"result"`
    Error  interface{}      `json:"error"`
}

type serverResponse struct {
    Id     *json.RawMessage `json:"id"`
    Result interface{}      `json:"result"`
    Error  interface{}      `json:"error"`
}
```

Vì vậy, dù sử dụng bất kỳ ngôn ngữ nào, chỉ cần tuân theo json format là có thể giao tiếp với RPC service được viết bởi GO hay bất kỳ ngôn ngữ nào khác, nói cách khác ta có thể thực hiện việc cross-language trong RPC.

## 3.1.4 Go RPC qua HTTP Protocol

RPC Framework vốn có trong GO đã hỗ trợ việc cung cấp các RPC Service trên HTTP Protocol. Tuy nhiên HTTP serivce của framework cũng tích hợp [GOB](https://golang.org/pkg/encoding/gob/) protocol và không cung cấp interface sử dụng các giao thức khác, do đó, nó vẫn không thể truy cập được từ các ngôn ngữ khác.

Trong ví dụ trước, chúng ta đã triển khai ```jsonrpc``` trên giao thức TCP và thực hiện thành công lệnh gọi RPC thông qua công cụ ```nc```. Bây giờ chúng ta sẽ thử cấp ```jsonrpc``` trên HTTP Protocol. RPC Service mới sẽ tuân thủ theo chuẩn [REST](https://restfulapi.net/),

```go
func main() {
    rpc.RegisterName("HelloService", new(HelloService))
	 // routing uri/jsonrpc đến hàm xử lý tương ứng
    http.HandleFunc("/jsonrpc", func(w http.ResponseWriter, r *http.Request) {
    	 // conn là một kiểu io.ReadWriteCloser
        var conn io.ReadWriteCloser = struct {
        	  // là struct gồm đọc và ghi
            io.Writer
            io.ReadCloser
        }{  // được khởi tạo với nội dụng:
        	  // ReadCloser là nội dung nhận được
            ReadCloser: r.Body,
            // Writer là đối tượng dùng ghi kết quả
            Writer: w,
        }
		  // truyền RPC service với biến conn
        rpc.ServeRequest(jsonrpc.NewServerCodec(conn))
    })
	 // lắng nghe kết nối từ client trên port 1234
    http.ListenAndServe(":1234", nil)
}
```
Lệnh gọi RPC để gửi json đến kết nối đó:

```sh
$ curl localhost:1234/jsonrpc -X POST \
    --data '{"method":"HelloService.Hello","params":["hello"],"id":0}'
```
Kết quả vẫn là một json:

```json
{"id":0,"result":"hello:hello","error":null}
```
Điều đó làm việc gọi RPC service từ những ngôn ngữ khác dễ dàng hơn.

[Tiếp theo](ch3-02-protobuf.md)