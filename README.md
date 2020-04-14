# Một vài thủ thuật trong Golang

Trong thực tế, các vấn đề của cuộc sống thường vô cùng phức tạp, nó cũng tạo ra vô số các bài toán phức tạp và hóc búa. Ví dụ như bây giờ có một máy chủ xác nhận đăng nhập của bạn qua OTP SMS thực hiện cho 10000 người dùng mỗi giây. Vấn đề của chúng ta là làm cách nào để đảm bảo hệ thông được vận hành 1 cách trơn chu nhất mà không có bất kỳ vấn đề gì với lượng user truy cập lớn như vậy. 

Giải pháp: Thiết kế một cách đơn giản nhất thì tôi sẽ tạo ra các công việc độc lập, một sẽ xử lý phần đăng nhập, phần còn lại sẽ đảm nhiệm việc OPT SMS. Vì vậy, công việc OTP sẽ chạy độc lập với việc đăng nhập. Đó là cách mà bước xử lý OTP sẽ được thực hiện lại cho các lần đăng nhập khác của người dùng. Tất nhiên, nếu để 2 việc này cứ chạy riêng biệt, thì chúng ta chưa có được một chức năng login hoàn chỉnh đúng không nào. Nhiệm vụ bây giờ là cần phải cho 2 công việc này 'nói chuyện' được với nhau.

Để giải quyết vấn đề trên, thì trong Golang cung cấp cho chúng ta một cơ chế được gọi là channels. 

# Channels

Hiểu một cách đơn giản, channels là cầu nối giữa các goroutines. Có thể hiểu tóm tắt qua các định nghĩa sau:

- Mỗi channels có một kiểu dữ liệu như là int, string, struct{}
- Sử dụng make() để tạo một channels. Ví dụ, ch:= make(chan int). Nếu bạn không tạo channels thì nó không thể sử dụng vào sẽ báo lỗi panic
- Channels có thể được truyền như là một tham số của function 

# Sử dụng channels

Một channels có 3 hoạt động như sau:

1. Đọc (ch-<x) // ghi x vào channel ch
2. Ghi (x:= <-ch) // Đọc chennel vào x
3. Đóng channel

# Đóng channels

- Nếu đã đóng channel thì không thể gửi thông tin trên nó được nữa. 
- Nếu bạn đọc trên một channel đã đóng, thì trước tiên  tất cả các giá trị chưa được đọc sẽ được đọc và sau đó lệnh đọc sẽ luôn luôn gọi giá trị 0 của channel

# Phân loại channels

Channels trong Golang được chia làm 2 loại

1. Unbuffered
2. Buffered

Bài viết này, chúng ta sẽ tập trung vào Unbufferd channels. Một tên gọi khác của Unbufferd chanels là kênh đồng bộ (Synchronous channels). Bởi vì chúng hoạt động theo nguyên tắc là chỉ có thể thực hiện 1 thao tác tại 1 thời điểm, là đọc hoặc viết. 

Ví dụ 

```
ch := make(chan int)
```

Với cách tạo channels này, bạn sẽ tạo ra 1 Unbufferd channel. Có một số rule bắt buộc như sau:

- Nếu channel mà không rỗng (is not empty), hành động send sẽ bị block. Thực hiện hành động recieve sẽ giúp channel sẵn sàng để send.
- Nếu channel rỗng (is empty) hành động tiếp nhận (recieve) sẽ bị block. Chúng ta có thể thực hiện hành động send

Quá khó hiểu đúng không nào, gửi, và nhận, block và không block. @@. Cụ thể hơn, hãy xem ví dụ sau đây

```
package main

import (
       "fmt"
       "sync"
)
var wg sync.WaitGroup

func toggle(ch <-chan int) {


       for k := range ch { // reads from channel until it's closed
              fmt.Println("================================>" , k*k)
       }

       // this will be called only if we close the channel.. :-)
       wg.Done()

}



func main() {
       // create a channels
       ch1 := make(chan int)
       ch2 := make(chan int)
       ch3 := make(chan int)


       // launch go routines

       wg.Add(3)
       defer wg.Wait()


       go toggle(ch2)
       go toggle(ch3)


       go toggle(ch1)


       for i:=2;i<10;i++ {
              //writing data to channels
              ch1<-i
              ch2<-i
              ch3<-i

              fmt.Println(i)
       }

       //close the channels we need range toggle to return
       close(ch1)
       close(ch2)
       close(ch3)
}
```

Chú ý răng: Nếu bạn sử dụng wait-group với channels, hãy ghi nhớ rằng, mọi goroutine đang chờ đều sẽ bị thoát ra. Vì vậy, nếu bạn sử để thoát khỏi vòng lặp.

# Khác biệt giữa Goroutines và Threads

Khác biệt đầu tiên là về Size:

**Thread** về cơ bản là ngăn xếp kích thước cố định, tốt nhất là 2MB. Nó có một không gian rộng lớn, ví dụ gọi 1024 luồng (thread) sẽ dẫn đến chiếm 2GB dung lượng. 
 
**Goroute** bắt đầu một chu trình (life) với một ngăn xếp nhỏ (small stack) thường là 2 KB. Bởi vì một số goroutine quá nhỏ so với ngăn xép 2MB và ngăn xếp Goroutine có thể phát triển được. 

Luồng (Thread) sẽ bị giới hạn cho hoạt động đệ quy vì ngăn xếp đệ quy lớn hơn 2M sẽ gây ra tràn ngăng xếp. Go thì khác hoàn toàn, ngăn xếp trong go có thể phát triển lên tới 1 GB. Với 1GB stack, bạn có thể gọi đệ quy tùy thích

Khác biệt thứ 2 là về Schedule:

Hệ thống Thread được lên lịch bởi Kernel. Phụ thuộc hẹn giờ phần cứng. 

Go run-time có lịch trình riêng của mình. Không phụ thuộc bộ đếm thời gian phần cứng. Nó là m:n schedule,có nghĩa là m (goroutines) => n (thread)

Định danh (Identity)

Goroutine thì k có định danh, với một số người thì nó thực sự là vấn đề, những có những người có thể làm quen được với nó. Vì go không có TLS nên Identity là không cần thiết. 

Hiện nay, lập trình đồng thời là vô cùng quan trọng và cần thiết. Ví dụ như, máy chử cần xử lý đồng thời nhiều kết nối. Go cung cấp cho chúng ta hai loại lập trình đồng thời như sau:

1. CSP - Communicating sequential processes - Truyền đạt các quy trình tuần tự
2. Chia sẻ bộ nhớ lập trình đa luồng

# Goroutines

Như phân tích ở trên, thì Goroutines không phải là một thread. Mỗi hoạt động đồng thời được gọi là Goroutine. Một ví dụ cụ thể như sau, trong đó chúng ta đang xử lý nhiều loại dữ liệu và ghi vào nhiều tệp. Th


