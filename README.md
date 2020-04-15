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

Như phân tích ở trên, thì Goroutines không phải là một thread. Mỗi hoạt động đồng thời được gọi là Goroutine. Một ví dụ cụ thể như sau, trong đó chúng ta đang xử lý nhiều loại dữ liệu và ghi vào nhiều tệp. Thông thường có 2 cách để làm điều này

- Lập trình tuần tự

- Lập trình đồng thời

Ví dụ dưới đây sẽ chỉ cho bạn thấy, để chuyển từ chương trình tuần tự sang đồng thời sẽ vô cùng đơn giản, nó không hề khó như bạn nghĩ. 

** Chương trình t:**

```
package main

import (
       "os"
       "fmt"
)

var filenamearray []string

func processData(index int) {
       f, _ := os.OpenFile(filenamearray[index],os.O_RDWR | os.O_CREATE,0777) // skipping error
       defer f.Close()
       for i := 0;i<10000000;i++ {
              a := i + index
              fmt.Fprintln(f,a)
       }
}

func main() {

       filenamearray = []string{"a1", "a2", "a3","a4"}

       for i:=0;i<len(filenamearray) ;i++ {
              processData(i)
       }


}
```

** Chương trình song song: **

```
import (
       "os"
       "fmt"
       "sync"
)

var filenamearray []string

var wg sync.WaitGroup

func processData(index int) {
       f, _ := os.OpenFile(filenamearray[index],os.O_RDWR | os.O_CREATE,0777) // skipping error
       defer wg.Done()
       defer f.Close()

       for i := 0;i<10000000;i++ {
              a := i + index
              fmt.Fprintln(f,a)
       }
}

func main() {

       filenamearray = []string{"a1", "a2", "a3","a4"}

       for i:=0;i<len(filenamearray) ;i++ {
              wg.Add(1)
              go processData(i)
       }

       wg.Wait()

}

```

 KHi chạy hai chương trình, nó có sự khác biệt lớn về mặt performance: 
 
 - Chạy tuần tự: 1 Minute and 21 seconds 
 - Chạy song song : 28 seconds. 

# Coins Change

Hãy thử xét bài toán coins change sau: Bạn có một số dolla nhất định, và một danh sách các đồng tiền, trên mỗi đồng tiền là một mệnh giá khác nhau, tìm và in ra tất các các cách khác nhau mà bạn có thể chia số dollar ban đầu, với những đồng tiền bạn có, điều kiện là số đồng tiền là vô hạn. 

Solution: 

Trước khi code, chúng ta thử phân tích yêu cầu bài toán và đưa ra giải thuật xem sao. Nếu bạn có các đồng tiền mệnh giá {2, 3, 5, 6} thì có bao nhiêu cách để bạn có thể chia được 10$. Cách đơn giản nhất mà bạn có thể nghĩ để thực hiện công việc trên là gì? Nếu là tôi thì tôi sẽ nghĩ đến đệ quy đầu tiên, cứ thử tất cả trường hợp, thỏa mãn điều kiện thì in ra và tăng số ượng. Nhưng thuật toán đệ quy thì bạn biết rồi đó, độ phức tạp của nó là vô cùng lớn, mã code dùng đệ quy như sau:

```
package main

import (
       "fmt"
       "sort"
)

var ways int

func coinchange(startIndex int, totalMoney int , coins []int) {
       if startIndex < 0 {
              return
       }
       if totalMoney < 0 {               return        }        if totalMoney == 0 {               ways++               return        }        for i := startIndex;i>=0;i-- {
              coinchange(i,totalMoney-coins[i],coins)
       }

}

func main() {
       var money, numCoins int
       fmt.Scanf("%d%d",&moneys )
       k := make([]int,numCoins)
       for i:=0;i<numCoins;i++ {
              fmt.Scanf("%d",&k   }

       // sort to optimize the calulation
       sort.Ints(k)

       coinchange(numCoins-1,money,k)

       fmt.Println(ways)

}
```

Cách này cũng không đến nỗi, nếu bộ dữ liệu của bạn là dữ liệu nhỏ, Nhưng với dữ liệu lớn thì thật sự là vấn đề, chắc chắn nó sẽ bị timeout khi chạy. Chúng ta duyệt qua dữ liệu hết lần này tới lần khác. Ví dụ {6, 4} = 10, {6, 2, 2} = 10. Để giải quyết vấn đề này, hãy thử xem chương trình sau đây:

```
package main
import "fmt"

func main() {
       var money, numCoins int
       fmt.Scanf("%d%d",&moneys )
       k := make([]int,numCoins)
       for i:=0;i<numCoins;i++ {
              fmt.Scanf("%d",&k   }
       // make a DP array
       dp := make([]int,money+1)
       dp[0] = 1
       for i:=0;i<numCoins;i++ {
              start := k[i]
              for j:=start;j<=money;j++ {
                     // use the saved solution
                     dp[j] += dp[j-start]
              }
       }
       fmt.Println(dp[money])
}
```
Với việc sử dụng chanel vào bài toán, chúng ta sẽ k sợ bị timeout khi chạy viows dữ liệu lớn. 
