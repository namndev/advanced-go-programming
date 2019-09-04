# 3.6 gRPC và Protobuf extensions
Hiện nay, cộng đồng Open source đã phát triển rất nhiều extensions xung quanh Protobuf và gRPC, tạo thành một hệ sinh thái to lớn. Ở phần này sẽ trình bày về một số extensions thông dụng.

## 3.6.1 Validtor

Cho đến nay, Protobuf đã có phiên bản thứ 3. Ở version 2 có một thuộc tính ```default``` ở các trường nhằm định nghĩa giá trị mặc định cho nó là một giá trị thuộc kiểu ```string``` hoặc kiểu ```number```.

Chúng ta sẽ tạo ra file proto sử dụng Protobuf v2:

***hello.proto (proto2)***:

```go
syntax = "proto2";
package main;
message Message {
    optional string name = 1 [default = "gopher"];
    optional int32 age = 2 [default = 10];
}
```
Cú pháp này sẽ được implement thông qua phần mở rộng tính năng của Protobuf. Giá trị mặc định không còn được hỗ trợ trong v3, nhưng chúng ta có thể mô phỏng giá trị mặc định của chúng bởi một phần mở rộng của option.

Sau đây là phần viết lại của file proto trên với phần mở rộng thuộc cú pháp v3:

***hello.proto (proto3)***:

```go
syntax = "proto3";

package main;

import "google/protobuf/descriptor.proto";

extend google.protobuf.FieldOptions {
    string default_string = 50000;
    int32 default_int = 50001;
}

message Message {
    string name = 1 [(default_string) = "gopher"];
    int32 age = 2[(default_int) = 10];
}
```
Trong dấu [ ] là một cú pháp mở rộng. Chúng ta sẽ tạo lại mã nguồn Go dựa trên những thông tin liên quan đến phần mở rộng của options. Phần mã nguồn sinh ra có một số nội dung dựa trên phần mở rộng như sau:

***hello.pb.go***:

```go
var E_DefaultString = &proto.ExtensionDesc{
    ExtendedType:  (*descriptor.FieldOptions)(nil),
    ExtensionType: (*string)(nil),
    Field:         50000,
    Name:          "main.default_string",
    Tag:           "bytes,50000,opt,name=default_string,json=defaultString",
    Filename:      "helloworld.proto",
}

var E_DefaultInt = &proto.ExtensionDesc{
    ExtendedType:  (*descriptor.FieldOptions)(nil),
    ExtensionType: (*int32)(nil),
    Field:         50001,
    Name:          "main.default_int",
    Tag:           "varint,50001,opt,name=default_int,json=defaultInt",
    Filename:      "helloworld.proto",
}
```
Chúng ta có thể parse out phần mở rộng của option được định nghĩa trong mỗi thành viên của Message tại thời điểm thực thi bởi kiểu ```reflection```, và sau đó parse out giá trị mặc định mà chúng ta đã định nghĩa sẵn từ những thông tin liên quan khác cho phần mở rộng.

Trong cộng đồng Open source, thư viện [go-proto-validators](https://github.com/mwitkow/go-proto-validators) là một extension của protobuf có chức năng validtor rất mạnh mẽ dựa trên phần mở rộng tự nhiên của Protobuf. Để sử dụng validator đầu tiên ta cần phải tải plugin sinh mã nguồn bên dưới:

```sh
$ go get github.com/mwitkow/go-proto-validators/protoc-gen-govalidators
```
Sau đó thêm phần ```validation rules``` vào các thành viên của Message dựa trên rules của go-proto-validators.

***hello.proto***:

```go
syntax = "proto3";

package main;

import "github.com/mwitkow/go-proto-validators/validator.proto";

message Message {
	// dấu ngoặc vuông mang ý nghĩa là phần tuỳ chọn
    string important_string = 1 [
        (validator.field) = {regex: "^[a-z]{2,5}$"}
    ];
    int32 age = 2 [
        (validator.field) = {int_gt: 0, int_lt: 100}
    ];
}
```
Tất cả những valiadation rules được định nghĩa trong message ```FieldValidator``` trong file [validator.proto](https://github.com/mwitkow/go-proto-validators/blob/master/validator.proto). Trong đó ta sẽ thấy một số trường được dùng ở ví dụ trên như sau:

***mwitkow/go-proto-validators/validator.proto***:

```go
syntax = "proto2";
package validator;
import "google/protobuf/descriptor.proto";
 extend google.protobuf.FieldOptions {
 	optional FieldValidator field = 65020;
}

message FieldValidator {
	// sử dụng Golang RE2-syntax regex để match với nội dung các field
	optional string regex = 1;
	// giá trị của biến integer bình thường lớn hơn giá trị này. 
	optional int64 int_gt = 2;
	// giá trị của biến integer bình thường nhỏ hơn giá trị này.
	optional int64 int_lt = 3;
	// ...
}           
```
Phần chú thích của mỗi trường ở trên sẽ cho chúng ta thông tin về chức năng của chúng. Sau khi chọn được các chức năng validate cần thiết, chúng ta dùng lệnh sau để sinh ra mã nguồn validator:

```go
protoc  \
    --proto_path=${GOPATH}/src \
    --proto_path=${GOPATH}/src/github.com/google/protobuf/src \
    --proto_path=. \
    --govalidators_out=. --go_out=plugins=grpc:.\
    hello.proto
```
Trong đó:

- proto_path: đường dẫn đến tất cả các file .proto được sử dụng 
- govalidators_out: plugin sinh ra mã nguồn validator

Chú ý: Trong Windows, ta thay thế ```${GOPATH}``` thành ```%GOPATH%```

Lệnh trên sẽ gọi chương trình protoc-gen-govalidators để sinh ra file với tên ```hello.validator.pb.go``` , nội dung của nó sẽ như sau:

***hello.validator.pb.go***:

```go
// định nghĩa chuỗi regex
var _regex_Message_ImportantString = regexp.MustCompile("^[a-z]{2,5}$")
// hàm validate() sẽ chạy các rules và bắt lỗi nếu có
func (this *Message) Validate() error {
	// rule 1 kiểm tra ImportantString có theo regex hay không, nếu có lỗi sẽ ném ra
    if !_regex_Message_ImportantString.MatchString(this.ImportantString) {
        return go_proto_validators.FieldError("ImportantString", fmt.Errorf(
            `value '%v' must be a string conforming to regex "^[a-z]{2,5}$"`,
            this.ImportantString,
        ))
    }
    // rule 2 kiểm tra Age > 0 hay không, nếu có lỗi sẽ ném ra
    if !(this.Age > 0) {
        return go_proto_validators.FieldError("Age", fmt.Errorf(
            `value '%v' must be greater than '0'`, this.Age,
        ))
    }
    // rule 3 kiểm tra Age < 100 hay không, nếu có lỗi sẽ ném ra
    if !(this.Age < 100) {
        return go_proto_validators.FieldError("Age", fmt.Errorf(
            `value '%v' must be less than '100'`, this.Age,
        ))
    }
    // trả về nil nếu kiểm tra tất cả các rules trên đều hợp lệ
    return nil
}
```
Thông qua hàm Validate() được sinh ra, chúng có thể được kết hợp với gRPC interceptor , chúng ta có thể dễ dàng validate giá trị của tham số đầu vào và kết quả trả về của mỗi hàm.

## 3.6.2 REST interface

Hiện nay RESTful JSON API vẫn là sự lựa chọn hàng đầu cho các ứng dụng web hay mobile. Vì tính tiện lợi và dễ dùng của RESTful API nên chúng ta vẫn sử dụng nó để frontend có thể giao tiếp với hệ thống backend. Nhưng khi chúng ta sử dụng framework gRPC của Google để xây dựng các service. Các service sử dụng gRPC thì dễ dàng trao đổi dữ liệu với nhau dựa trên giao thức HTTP/2 và protobuf, nhưng ở phía frontend lại sử dụng RESTful API hoạt động trên giao thức HTTP/1. Vấn đề đặt ra là chúng ta cần phải chuyển đổi các yêu cầu RESTful API thành các yêu cầu gRPC để hệ thống các service gRPC có thể hiểu được.

Cộng đồng Open source đã implement một project với tên gọi là [grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway), nó sẽ sinh ra một proxy có vai trò chuyển các yêu cầu REST HTTP thành các yêu cầu gRPC HTTP/2.
<div align="center">
	<img src="../images/ch3-6-grpc-gateway.png">
	<br/>
	<span align="center">
		<i>gRPC-Gateway workflow</i>
	</span>
</div>
<br/>
Trong file Protobuf (chỉ có ở proto3), chúng ta sẽ thêm thông tin phần routing ứng với các hàm trong gRPC service, để dựa vào đó grpc-gateway sẽ sinh ra mã nguồn proxy tương ứng. 

***rest_service.proto***:

```go
syntax = "proto3";

package main;

import "google/api/annotations.proto";

message StringMessage {
  string value = 1;
}

service RestService {
	// định nghĩa hàm RPC Get trong service
    rpc Get(StringMessage) returns (StringMessage) {
    	// nội dung phần option trong này định nghĩa Rest API ra bên ngoài
        option (google.api.http) = {
	        // get: là tên phương thức được sử dụng
            get: "/get/{value}"  // "/get/{value}" : là đường dẫn uri
        };
    }
    rpc Post(StringMessage) returns (StringMessage) {
        option (google.api.http) = {
            post: "/post"
            body: "*"
        };
    }
}
```
Chúng ta cài đặt plugin protoc-gen-grpc-gateway với những lệnh sau:

```sh
$ go get -u github.com/grpc-ecosystem/grpc-gateway/protoc-gen-grpc-gateway
```
Sau đó chúng ta sinh ra mã nguồn routing cho grpc-gateway thông qua plugin sau:

```sh
$ protoc -I/usr/local/include -I. \
    -I$GOPATH/src \
    -I$GOPATH/src/github.com/grpc-ecosystem/grpc-gateway/third_party/googleapis \
    --grpc-gateway_out=. --go_out=plugins=grpc:.\
    hello.proto
```
Trong windows: Thay thế ```${GOPATH}``` với ```%GOPATH%```.

Plugin sẽ sinh ra hàm ```RegisterRestServiceHandlerFromEndpoint()``` cho RestService service như sau:

```go
func RegisterRestServiceHandlerFromEndpoint(
    ctx context.Context, mux *runtime.ServeMux, endpoint string,
    opts []grpc.DialOption,
) (err error) {
    ...
}
```

Hàm ```RegisterRestServiceHandlerFromEndpoint``` được dùng để chuyển tíêp những request được định nghĩa trong REST interface đến gRPC service. Sau khi registering các Route handle, chúng ta sẽ chạy proxy web service trong hàm main như sau:

***proxy/main.go***:

```go
func main() {
    ctx := context.Background()
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()

    mux := runtime.NewServeMux()

    err := RegisterRestServiceHandlerFromEndpoint(
        ctx, mux, "localhost:5000",
        []grpc.DialOption{grpc.WithInsecure()},
    )
    if err != nil {
        log.Fatal(err)
    }

    http.ListenAndServe(":8080", mux)
}
// $ go run proxy/main.go
```
Tiếp theo ta sẽ chạy gRPC service: 

***restservice/main.go***:

```go
type RestServiceImpl struct{}

func (r *RestServiceImpl) Get(ctx context.Context, message *StringMessage) (*StringMessage, error) {
    return &StringMessage{Value: "Get hi:" + message.Value + "#"}, nil
}

func (r *RestServiceImpl) Post(ctx context.Context, message *StringMessage) (*StringMessage, error) {
    return &StringMessage{Value: "Post hi:" + message.Value + "@"}, nil
}
func main() {
    grpcServer := grpc.NewServer()
    RegisterRestServiceServer(grpcServer, new(RestServiceImpl))
    lis, _ := net.Listen("tcp", ":5000")
    grpcServer.Serve(lis)
}
// $ go run restservice/main.go
```
Sau khi chạy hai chương trình gRPC và REST services, chúng ta có thể tạo request REST service với lệnh [curl](https://thoainguyen.github.io/2019-06-15-using-curl/):

```sh
// gọi service Get
$ curl localhost:8080/get/gopher
{"value":"Get: gopher"}
// gọi service Post
$ curl localhost:8080/post -X POST --data '{"value":"grpc"}'
{"value":"Post: grpc"}
```
Khi chúng ta publishing REST interface thông qua [Swagger](https://swagger.io/), một swagger file có thể được sinh ra nhờ vào công cụ grpc-gateway bằng lệnh bên dưới:

```sh
$ go get -u github.com/grpc-ecosystem/grpc-gateway/protoc-gen-swagger

$ protoc -I. \
  -I$GOPATH/src/github.com/grpc-ecosystem/grpc-gateway/third_party/googleapis \
  --swagger_out=. \
  hello.proto
```
Trong đó: ```--swagger_out=.```: dùng plugin swagger để sinh ra swagger file tại thư mục hiện tại.

File ```hello.swagger.json``` sẽ được sinh ra sau đó. Trong trường hợp này, chúng ta có thể dùng ```swagger-ui project``` để cung cấp tài liệu ```REST interface``` và testing dưới dạng web pages.

## 3.6.3 Dùng Docker grpc-getway

Với những lập trình viên phát triển gRPC Services trên các ngôn ngữ không phải Golang như Java, C++, ... có nhu cầu sinh ra grpc gateway cho các services của họ nhưng gặp khá nhiều khó khăn từ việc cài đặt môi trường Golang, protobuf, các lệnh generate v.v.. Có một giải pháp đơn giản hơn đó là sử dụng Docker để xây dựng grpc-gateway theo bài hướng dẫn chi tiết sau: [buildingdocker-grpc-gateway](https://medium.com/zalopay-engineering/buildingdocker-grpc-gateway-e2efbdcfe5c).

## 3.6.4 Nginx

Những phiên bản [Nginx](https://www.nginx.com/) về sau cũng đã hỗ trợ gRPC với khả năng register nhiều gRPC service instance giúp load balancing dễ dàng hơn. Những extension của Nginx về gRPC là một chủ đề lớn, ở đây chúng tôi không trình bày hết được, các bạn có thể tham khảo các tài liệu trên trang chủ của [Nginx](https://www.nginx.com/blog/nginx-1-13-10-grpc/).

[Tiếp theo](ch3-07-protobuf-based-framework.md)
