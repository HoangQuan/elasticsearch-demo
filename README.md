## **Full Text Search và Full Text Search Trong Rails**

### **Giới Thiệu**

`Full Text Search` (Viết tắt là FTS) là Kỹ thuật tìm kiếm chuỗi (ký tự) trên toàn bộ các trường có định dạng chuỗi trong một table trên Database. Định nghĩa cụ thể bạn có thể xem trên [wikipedia](https://en.wikipedia.org/wiki/Full_text_search) .

Đến đây, Nhiều bạn có thể đặt câu hỏi: tại sao phải dùng Full text Search, Có nhiều cách để tìm kiếm trong database cơ mà?

Trong bài viết này mình sẽ tìm giới thiệu Một số khái niệm cơ bản trong Full Text Search và Ứng dụng nó trong ROR.

### **Đặt vấn đề**

Giả sử bạn có một bảng `articles` với các trường: id, title, descrition và content.


Yêu cầu đặt ra là bạn phải tìm kiếm dữ liệu trong bảng này, Với thông tin là tất cả các trường chứa nội dung “Viblog”

Để giải quyết vấn dề này bạn có thể dùng câu truy vấn LIKE trong SQL như sau:

```Mysql
declare @keysearch nvarchar(500
set @keysearch = N'Viblog'
select * from articles
where scode like '%'+@keysearch+'%'
or sename like '%'+@keysearch+'%'
or slocalname like '%'+@keysearch+'%'
or semail like '%'+@keysearch+'%'

```
và cũng có thể dùng Full text search để thực hiện như sau:

```
declare @keysearch nvarchar(500)
set @keysearch = N'Viblog'
select * from articles where freetext(*,@keysearch)

```

Kết quả của 2 truy vấn này là như nhau.

### **Tại sao phải dùng FTS**
Như bạn đã thấy trongh ví dụ trên chúng ta có thể dùng 2 cách để thực hiện truy vấn và đưa ra kết quả khác nhau. Tuy nhiên FULL TEXT SEARCH lại được sử dụng vì các lý do sau:

Một là: Perfomance tốt hơn. Đối với câu truy vấn LIKE bạn sẽ thực hiện tìm kiếm đơn thuần không sử dụng `index`, trong khi đó FTS lại đánh index cho các trường được chọn và tìm kiếm trên đó.

Hai là: Độ chính xác tìm kiếm cao hơn. Đối với truy vấn LIKE nếu bạn tìm `Title LIKE “%one%”` thì kết quả trả ra sẽ chứa “one”, “phone”, “zone” hay “money” … nói dung là kết quả không mong muốn. Tuy nhiên FTS lại làm tốt điều này.

Một lý do quan trọng nữa là nếu bạn muốn tìm kiếm những từ khóa có dấu như “Giải trí” mà lại gõ “Giai tri” thì chắc chắn truy vấn LIKE sẽ không tìm ra kết quả mong muốn.

Những lý do trên đã đủ thuyết phục bạn chuyển sang dùng FTS chưa? ^^

### **FTS Hoạt động như  thế nào?**

Full Text Search sử dụng kỹ thuật `Inverted Index` trong quá trình tìm kiếm.

#### **Inverted Index** 

Bình thường để tìm kiếm theo index người ta sẽ đánh index theo đơn vị cột (tức là để đánh index trong một bảng thì người ta sẽ đánh index cho một trường trong bảng đó như id, hoắc tổ hợp [id, email, name] ..)
 Tuy nhiên FTS sẽ đánh index theo đơn vị term (trong Mysql). Cụ thể, Inverted Index là một cấu trúc dữ liệu nhằm mục đích map giữa các term và các trường (doccument) chứa tẻm đó.

ví dụ ta có 2 trường như sau:

```
D1 = "This is first document"
D2 = "This is second one"
D3 = "one two"

```
Vậy thì Inverted Index của nó sẽ như sau:

```
"this" => {D1, D2}
"is" => {D1, D2}
"first" => {D1}
"document" => {D1}
"second" => {D2}
"one" => {D2, D3}
"two" => {D3}

```
Từ đó chúng ta sẽ tìm kiếm từ khóa trên việc tổ hợp các term. Việc tím kiếm sẽ dễ dàng và nhanh chóng hơn rất nhiều. Ngoài ra khi bạn muốn tìm từ khóa “This is fisrt” hay “First is this” thì kết quả vẫn được tìm thấy và độ phức tạp của thuật toán tìm kiếm là  như nhau.

Việc tách kết quả tìm kiếm ra thành Term rất hiệu quả đúng không? Nhưng bạn có tự hỏi: “Làm thể nào mà FTS có thể tách được các chuỗi ra thành các term được không?”

#### **Tokenize**

`Tokenize` là một bài toán quan trọng trong thuật toán tìm kiếm của FTS. Trong đó có 2 kỹ thuật cơ bản là:

N-Gram
Morphological Analysis

 `N-Gram` là Kỹ thuật chia các chuỗi thành các chuỗi con thông qua việc chia đều các chuỗi đầu vào thành các chuỗi nhỏ hơn có độ dài bằng nhau và đó độ dài N. Thông thường thì N sẽ nằm trong khoảng từ 1-3, với các tên gọi là: `unigram (N=1)`,`biggram (N=2)`, `trigram (N=3)`.

Ví dụ chuỗi “GOOD MORNING” sẽ được tách thành các term bằng phương pháp `biggram` như sau:

```
"good morning" => {"go", "oo", "od", "d ", " m", "mo", "or", "rn", "ni", "in", "ng"}
```

`Morphological Analysis` http://en.wikipedia.org/wiki/Morphology_(linguistics) (MA) là kỹ thuật xử lý ngôn ngữ tự nhiên (National Language Procesing) đây là một kỹ thuật phân tích cấu trúc của từ. Nói cách khác Morphological là Kỹ thuật tách một chuỗi thành các từ có nghĩa.

Ví dụ từ Good Morning ở trên sẽ phân tích thành:

```
good morning => {“good”, “moring”}
```

Để làm được điều này đòi hỏi chúng ta phải có một bộ thư viện ngôn ngữ với các từ đầy đủ.Ví dụ như các Search enginee lớn (“Google”, “yhaoo”...) họ cần phải có một đội ngũ nghiên cứu để đưa ra bộ thư viện MA riêng của họ để thích hợp nhiều ngôn ngữ. Ngoài ra cũng có một số bộ thư viện open source như `lucene`..

Đến đây, Nhiều bạn sẽ thắc mắc: ”để tách chuỗi thì dùng MA là hợp lý rồi tại sao lại phải dùng N-gram nữa”  thì như đã nói ở trên. MA đòi hỏi phải có một thư viện ngôn ngữ phong phú. tuy nhiên ngôn ngữ thì đa dạng và luôn thay đổi. nên đôi khi MA không đưa ra kết quả đung. Vì vậy tốt nhất chúng ta nên kết hợp khéo léo cả hai phương pháp trên.

### **Full Text Search ứng dụng trong ROR**

Có lẽ đọc đến đây các bạn cũng đã phần nào hiểu được FTS là gì và có ý nghĩa như thế nào đối với công việc tìm kiếm. 

Ở phần này mình sẽ làm một ứng dụng nhỏ để thực hiện FTS trong ROR. Trong ROR có một số gem áp dụng FTS rất nhanh chóng và hiệu quả như: `sunspot_rails` và `elasticsearch`.

`sunspot` xử dụng một search enginee khá nổi tiếng là SOLR. Có lẽ nhiều bạn đã biết và thực hiện. 

Trong bài này mình sẽ giới thiệu và cài đặt demo `elasticsearch`.

*  Tạo một Project và một bảng `articles` với các trường: id, title, text

```RUBY
$ rails new elasticsearch-demo
$ cd elasticsearch-demo
$ bundle install
$ rails s
```

```
$ rails g model Article title:string text:text
$ rake db:migrate
```

* Cài đặt elasticsearch

Ubuntu
vào  elasticsearch.org/download http://www.elasticsearch.org/download/ và download DEB file. Mở thư mục đã download elasticsearch và chạy lệnh:

```
$ sudo dpkg -i elasticsearch-[version].deb
```

Mac
```
$ brew install elasticsearch
```
Chạy elasticsearch:

```
sudo services elasticsearch start
```

Để kiểm tra xem elasticsearch chạy hay chưa bạn vào địa chỉ “http://localhost:9200/” nếu thấy response về thông tin server như này là ok:

```
{
  "status" : 200,
  "name" : "Anvil",
  "version" : {
    "number" : "1.2.1",
    "build_hash" : "6c95b759f9e7ef0f8e17f77d850da43ce8a4b364",
    "build_timestamp" : "2014-06-03T15:02:52Z",
    "build_snapshot" : false,
    "lucene_version" : "4.8"
  },
  "tagline" : "You Know, for Search"
}

```

thêm Gem vào Gemfile:

```
gem 'elasticsearch-model'
gem 'elasticsearch-rails'
```

`Bundle install`

 Tạo `SearchController.rb`

```
def search
  if params[:q].nil?
    @articles = []
  else
    @articles = Article.search params[:q]
  end
end

```

`Include elasticsearch/model` vào trong model

```
require 'elasticsearch/model'

class Article < ActiveRecord::Base
  include Elasticsearch::Model
  include Elasticsearch::Model::Callbacks
end
Article.import # for auto sync model with elastic search

```

Đó là nhưng bước cài đặt mặc định của elasticsearch.

Tuy nhiên, Với elasticsearch cung cấp nhiều tính năng để bạn có thể search nâng cao.

#### **Custom Query**

Elasticsearch cho phép bạn `custom` Query thông qua JSON object:

```ruby
def self.search(query)
  __elasticsearch__.search(
    {
      query: {
        multi_match: {
          query: query,
          fields: ['title^10', 'text']
        }
      }
    }
  )
end
```

Nói chung bạn có thể gộp các điều kiện phức tạp và khai báo trong đối tượng JSON này, dưới phần `query`.

Trường hợp này tôi muốn tìm kiếm các từ khóa trong 2 trường đó là `title` và `text`. Do đó, Tôi khai báo điều kiện `multi_match` và thêm 2 trường mong muốn vào thuộc tính `fields`.

Ngoài ra Elasticsearch còn cung cấp rất nhiều tính năng khác bạn có thể tham khảo trên github của [Elasticsearch](https://github.com/elastic/elasticsearch)


Bạn có thể tham khảo code demo tại [đây](https://github.com/HoangQuan/elasticsearch-demo)

Tài liệu tham khảo:
http://dev.mysql.com/doc/refman/5.7/en/fulltext-natural-language.html
https://msdn.microsoft.com/en-us/library/ms142571.aspx
https://github.com/HoangQuan/elasticsearch-demo
Mong bài viết sẽ hữu ích với các bạn.
Cảm ơn!

