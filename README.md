# Aphplication - Máy chủ ứng dụng PHP siêu nhẹ


## PHP chậm là lẽ tự nhiên bởi vì tất cả mọi thứ được thực hiện trên mọi yêu cầu

Thông thường khi bạn chạy một tập lệnh PHP, điều sau sẽ xảy ra:

![Without an app server](https://r.je/img/aphplication-without.png)
* **_Ô màu đỏ là diễn ra trên mọi request_**
* **_Ô màu xanh là chỉ diễn ra trên request đầu tiên_**

Điều duy nhất xảy ra khác nhau trên mỗi yêu cầu là bước cuối cùng. Tất cả các công việc (không tầm thường) khởi động một ứng dụng được thực hiện đều đặn trên mỗi yêu cầu. Mỗi khi một trang được xem, các lớp được nạp, framework được khởi tạo, cơ sở dữ liệu được kết nối, các thư viện được cấu hình. Tất cả công việc khó khăn này được thực hiện theo mọi yêu cầu. Mỗi lần bạn truy cập một trang, tất cả các tệp cấu hình đều được tải và các lớp được khởi tạo.

Aphplication là một nỗ lực để giải quyết vấn đề này bằng cách thay đổi bản chất của cách PHP xử lý các yêu cầu.

Điều gì sẽ xảy ra nếu chúng ta có thể chụp nhanh (snapshot) một kịch bản (script) PHP ở bước 5 (xem lại hình trên nhé), sau khi mọi thứ đã khởi động và sẵn sàng xử lý các yêu cầu riêng lẻ? Đây chính là cách Aphplication hoạt động:

![Without an app server](https://r.je/img/aphplication-with.png)

Một khi sử dụng Aphplication, code thường sẽ trên mỗi yêu cầu được chạy một lần và sau đó lắng nghe các kết nối. Trên thực tế bạn có thể nhảy vào bất kỳ phần nào của một tập lệnh PHP đang chạy.

Kết quả là mỗi yêu cầu sẽ chỉ thực hiện các tác vụ cần thiết. [Gia tăng hiệu suất cho ứng dụng Laravel lên đến 2400%](https://laracasts.com/discuss/channels/laravel/proof-of-concept-application-server-2400-laravel-startup-speed-increase)

## Aphplication là gì (Không phải Application đâu nhé)?

Aphplication là một ứng dụng máy chủ PHP. Nó hoạt động giống như là Node.js, ứng dụng của bạn luôn luôn chạy và khi ai đó kết nối, họ đang kết nối với một ứng dụng đang hoạt động. Điều này cho phép bạn duy trì trạng thái trên các yêu cầu và tránh code khởi động nhiều lần trên mỗi yêu cầu

Có hai phần của dự án Aphplication:

1) Phía máy chủ: sẽ chứa tất cả code của bạn và tiếp tục chạy ngay cả khi không có ai truy cập trang.

2) Phía client: Đây là một middle-man đứng giữa trình duyệt và ứng dụng đang chạy. Khi ai đó kết nối với `client.php` kịch bản (script) PHP chạy, middle-man này sẽ nói chuyện với máy chủ, yêu cầu máy chủ thực hiện một số xử lý và sau đó trả về kết quả. Máy chủ tiếp tục chạy nhưng kịch bản (script) máy khách dừng lại.

## Yêu cầu

Aphplication sử dụng hàng đợi tin nhắn trong nội bộ. Điều này nhanh hơn đáng kể so với socket hoặc các tools khác tương tự. Bạn phải có phần mở rộng `extension=sysvmsg.so` trong `php.ini` bằng cách cài đặt mới phần mở rộng hoặc bỏ comment trong `php.ini` nếu có rồi.


Aphplication yêu cầu một máy chủ Linux với phần mở rộng sysvmsg.so được kích hoạt. Phần mở rộng này tồn tại theo mặc định trên hầu hết các cài đặt mặc định của Linux.

## Cách sử dụng

1) Tạo máy chủ của bạn bằng cách tạo một file class và implements Aphplication\Aphplication interface

2) Truyền vào trong class đó một instance của `Aphplication\Server()`;

3) Lưu file đó lại, ví dụ: `server.php`

```php
// Lớp này được thực hiện một lần và tiếp tục chạy trong nền
class MyApplication implements \Aphplication\Aphplication {
	// Trạng thái được duy trì qua các lần yêu cầu lại. Đây không phải là serialized, nó được giữ như là có thể
	// được kết nối cơ sở dữ liệu, đồ thị đối tượng phức tạp, vv
	// Lưu ý: Mỗi worker thread có một bản sao của trạng thái này theo mặc định
	private $num = 0;

	// Phương thức chấp nhận được thực hiện trên mỗi yêu cầu.
	// Bởi vì trường hợp này đã chạy, các superglobals được truyền từ máy khách

	// Giá trị trả về là một chuỗi được gửi lại cho máy khách.
	// Lưu ý: Để có sự tương thích tốt hơn, mọi cuộc gọi header() cũng được gửi lại cho máy khách
	public function accept(): string {
		// The only code that is run on each request.
		$this->num++;
		return $this->num;
	}
}

$server = new \Aphplication\Server(new MyApplication());
$server->start();
```

Máy chủ bây giờ sẽ chạy và chỉ mỗi một thực thể `MyApplication` sẽ được giữ để chạy PHP process trên server. Mỗi khi máy khách kết nối, phương thức `accept` của máy chủ được gọi và có thể thực hiện xử lý cụ thể cho trang đó.


Bạn cũng có thể làm điều này:

```php
// Lớp này được thực hiện một lần và tiếp tục chạy trong nền
class MyApplication implements \Aphplication\Aphplication {
	private $frameworkEntryPoint;

	public function __construct() {
		// Khởi tạo framework và lưu trữ nó trong bộ nhớ. Điều này chỉ xảy ra một lần và được duy trì hoạt động trên máy chủ
		$db = new PDO('...');
		$this->frameworkEntryPoint = new MyFramework($db);
	}

	// Phương thức chấp nhận được thực hiện trên mỗi yêu cầu.
	// Bởi vì trường hợp này đã chạy, các superglobals được truyền từ máy khách

	// Giá trị trả về là một chuỗi được gửi lại cho máy khách.
	// Lưu ý: Để có sự tương thích tốt hơn, mọi cuộc gọi header() cũng được gửi lại cho máy khách
	public function accept(): string {
		// Each time a client requests, route the request as normal
		return $this->frameworkEntryPoint->route($_SERVER['REQUEST_URI']);
	}
}

$server = new \Aphplication\Server(new MyApplication());
$server->start();
```


Bằng cách này, tất cả các framework class của bạn chỉ được tải một lần. Điều này thậm chí còn tốt hơn so với [opcaching](https://chungnguyen.xyz/posts/php-7-4-preload-toc-do-ban-tho-cho-php) vì không chỉ các tệp chỉ được phân tích cú pháp một lần, mà code khởi động cũng chỉ được thực thi một lần.

2) Khởi động ứng dụng trên dòng lệnh:

Giả sử máy chủ của bạn được lưu trữ trong file `server.php`, khởi động máy chủ ứng dụng:

```
php server.php
```

3) Chạy kịch bản lệnh CLI Client từ cùng thư mục mà máy chủ được khởi động (Cả server và client *phải* chạy từ cùng một thư mục làm việc)

Bây giờ kết nối với máy chủ từ máy khách.

```php
require '../Aphplication/Client.php';
$client = new \Aphplication\Client();
echo $client->connect();

```

Để sử dụng máy chủ web làm ứng dụng khách, chỉ cần tạo tập lệnh PHP:

```php
require '../Aphplication/Client.php';
$client = new \Aphplication\Client();
echo $client->connect();
```



### Tắt máy chủ

Để tắt máy chủ chạy cùng một kịch bản thông qua dòng lệnh với lệnh dừng:

```bash
php server stop
```

### Ví dụ về máy chủ Web

Đầu tiên tạo một máy chủ web trong file `server.php`. Nó không cần phải ở trong thư mục `public_html` hay mà bất cứ đâu bạn thích:


```php
require 'vendor/autoload.php';
class MyApplication implements \Aphplication\Aphplication {
	// Trạng thái ứng dụng. Điều này sẽ được lưu trong bộ nhớ khi ứng dụng bị đóng
	// Điều này thậm chí có thể là các kết nối MySQL và các tài nguyên khác

	private $num;

	public function accept(): string {
		$this->num++;

		// trả lại phản hồi để gửi lại cho trình duyệt, ví dụ: một số mã HTML
		return $this->num;
	}

}


// Bây giờ tạo một instance máy chủ
$server = new \Aphplication\Server(new MyApplication());

// Kiểm tra argv để cho phép bắt đầu và dừng máy chủ
if (isset($argv[1]) && $argv[1] == 'stop') $server->shutdown();
else $server->start();
```

Khi máy chủ đã được viết xong, hãy khởi động nó bằng

```
php server.php
```

Điều này sẽ bắt đầu một tiến trình PHP trong bộ nhớ và bắt đầu chờ các yêu cầu. Để dừng máy chủ bạn có thể gọi `php server.php stop`

Bây giờ máy chủ đang chạy, tạo một file `client.php` bên trong thư mục `public_html` hay `httpdocs` nơi mà máy chủ web có thể truy cập được.

`client.php` chỉ nên chứa mã này:

```php
require '../Aphplication/Client.php';
$client = new \Aphplication\Client();
echo $client->connect();
```
(Điều chỉnh đường dẫn đến `Aphplication/Client.php` cho phù hợp). Bạn *có thể* sử dụng copmoser để tự động load, tuy nhiên nó không phải là một ý tưởng tốt bởi vì dùng composer autoload tiêu tốn một chi phí đáng kể chỉ để loading một file. Bạn sẽ nhận được hiệu suất tốt hơn chỉ bằng cách sử dụng `require` để include trong client code.

Mã máy khách được cung cấp kết nối với máy chủ, gửi dữ liệu get/post/vv từ yêu cầu hiện tại và trả về phản hồi. Tệp PHP này **là** chạy trên mọi yêu cầu, do đó hãy cố gắng giữ cho nó luôn có điện!

Bây giờ nếu bạn ghé thăm `client.php` trên trình duyệt, bạn sẽ thấy đầu ra từ máy chủ. Trong trường hợp này, nó sẽ hiển thị một bộ đếm vì mỗi lần máy chủ được kết nối, biến `$ num` được tăng thêm một.


### Giờ thì sao?

Máy chủ của bạn có thể thực hiện *mọi thứ mà một tập lệnh PHP bình thường có thể thực hiện*. Khi một câu lệnh `require` đã được proceessd trên máy chủ, tệp đó là bắt buộc và sẽ không được yêu cầu lại cho đến khi máy chủ được khởi động lại!


### Phát triển

Điều này làm cho việc phát triển trở nên khó khăn hơn khi bạn phải khởi động lại máy chủ mỗi lần. Các bản phát hành trong tương lai sẽ có một chế độ phát triển không thực sự khởi chạy máy chủ nhưng cho phép bạn chạy các ứng dụng khách như thể nó (giống như một kịch bản php bình thường, nơi mọi thứ được nạp mỗi lần).



## Hiệu suất

Aphplication có thể lên đến 1000% nhanh hơn so với một kịch bản PHP tiêu chuẩn. Khi bạn chạy Laravel, Wordpress hay Zend project bằng thông dịch PHP nó sẽ thực thi và làm rất nhiều việc: tải tất cả các tệp tin required.php, kết nối với cơ sở dữ liệu và cuối cùng xử lý yêu cầu của bạn. Với Aphplication, tất cả các mã boostrapping được thực hiện một lần, khi máy chủ bắt đầu. Khi ai đó kết nối họ đang kết nối với ứng dụng đang chạy đã thực hiện tất cả công việc boostrapping, máy chủ sau đó chỉ xử lý yêu cầu và đưa nó cho khách hàng.

Bạn có thể nghĩ về máy chủ ứng dụng một chút như MySQL, nó luôn chạy và chờ xử lý các yêu cầu. Khi một yêu cầu được thực hiện, nó thực hiện một số xử lý và trả về kết quả. Không giống như một tập lệnh PHP truyền thống, nó tiếp tục chạy sẵn sàng để xử lý yêu cầu tiếp theo.

Điều này mang lại cho Aphplication một lợi ích hiệu suất rất lớn so với phương pháp truyền thống để tải tất cả các tệp được yêu cầu và thực hiện tất cả các kết nối cần thiết trên mỗi yêu cầu.


### Đa luồng

Aphplication là đa luồng. Nó sẽ khởi động như nhiều quá trình như bạn muốn. Điều này nên lên đến 3x số lõi (hoặc lõi ảo) CPU của bạn có. Điều này là do các tập lệnh PHP thường tạm dừng (ví dụ: trong khi chờ MySQL trả về một số dữ liệu).

Bạn có thể đặt số lượng thread bằng cách sử dụng

```php
$server->setThreads($number);
```

trước `$server->start()`
