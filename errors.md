# rpc错误信息包
### errors  (调用rpc方法时返回的error)
```go
type Error struct {
	Status
	cause error
}

func (e *Error) Error() string {
	return fmt.Sprintf("error: code = %d reason = %s message = %s metadata = %v cause = %v", e.Code, e.Reason, e.Message, e.Metadata, e.cause)
}

// Unwrap provides compatibility for Go 1.13 error chains.
func (e *Error) Unwrap() error { return e.cause }

message Status {
int32 code = 1;
string reason = 2;
string message = 3;
map<string, string> metadata = 4;
};
```
* 自定义Error结构，实现error接口所需Error()，其中Status为pb传输结构对应message Status
```go
func (e *Error) Is(err error) bool {
	if se := new(Error); errors.As(err, &se) {
		return se.Code == e.Code && se.Reason == e.Reason
	}
	return false
}
```
* 默认err 为Error结构的派生，同时code reason相同则为同个错误类型
```go
func (e *Error) GRPCStatus() *status.Status {
	s, _ := status.New(httpstatus.ToGRPCCode(int(e.Code)), e.Message).
		WithDetails(&errdetails.ErrorInfo{
			Reason:   e.Reason,
			Metadata: e.Metadata,
		})
	return s
}
```
* 调用 transport/http/status下的status.go中的ToGRPCCode,目的是将http状态码转换为rpc对应状态码
* 调用grpc status的new及withdetails 构造对应grpc的Status结构
```go
func FromError(err error) *Error {
	...
	gs, ok := status.FromError(err)
    if !ok {
        return New(UnknownCode, UnknownReason, err.Error())
    }
    ret := New(
        httpstatus.FromGRPCCode(gs.Code()),
        UnknownReason,
        gs.Message(),
    )
    ...
}
```
* 将error 转换为 Error结构，非实现grpcstatus接口(即实现GRPCStatus() *Status) 默认构造code为500 reason为空的Error结构体

## types
```go
// BadRequest new BadRequest error that is mapped to a 400 response.
func BadRequest(reason, message string) *Error {
	return New(400, reason, message)
}

// IsBadRequest determines if err is an error which indicates a BadRequest error.
// It supports wrapped errors.
func IsBadRequest(err error) bool {
	return Code(err) == 400
}
```
* 主要负责调用errors的方法生成rpc对应的错误以及判断对应的错误