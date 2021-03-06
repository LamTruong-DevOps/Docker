# Tập trung logs Docker sử dụng EFK Stack

![Docker Logs with EFK](https://user-images.githubusercontent.com/97789851/165694689-bca684a7-4c1e-4def-8168-ac23a22c3baf.jpeg)

## Giới thiệu

Khi bạn đưa các containers Docker vào sử dụng, bạn sẽ thấy nhu cầu thu thập logs rất cần thiết để giám sát hoạt động của container. Để duy trì nhận logs ở một nơi nào đó dễ sử dụng, dễ phân tích hơn ở chính containers. Thì **Fluentd** là một công cụ thu thập dữ liệu mã nguồn mở được thiết kế để thống nhất cơ sở hạ tầng ghi logs của bạn. Nó tập hợp các kỹ sư vận hành, kỹ sư ứng dụng và kỹ sư dữ liệu lại với nhau bằng cách làm cho việc thu thập và lưu trữ logs trở nên đơn giản và có thể mở rộng.

**Fluentd** giúp bạn dễ dàng thu thập logs và định tuyến chúng đến một nơi khác, chẳng hạn như trong hướng dẫn này là **Elasticsearch**, để bạn có thể phân tích dữ liệu logs. Và sau đó chúng được chuyển tới **Kibana**, một giao diện thân thiện với bạn hơn.

### Thiết lập cần thiết khi bắt đầu

**Để hoàn thành hướng dẫn này, bạn sẽ cần những thứ sau:**

* Trong hướng dẫn này sử dụng 2 server Ubuntu để thực hiện: 

    **192.168.5.30 cài EFK nhận logs** và **192.168.5.40 gửi logs**
* Sử dụng Ubuntu server và có quyền root
* Docker được cài đặt trên máy chủ của bạn bằng cách làm theo **[Install Docker on Ubuntu](https://github.com/LamTruong-DevOps/Docker#readme)**
* Cập nhật múi giờ trên server Ubuntu:
```console
sudo timedatectl set-timezone Asia/Ho_Chi_Minh
```
# Phần I - Setup và config server tập trung logs EFK (192.168.5.30)  
## Bước 1: Xây dựng image Fluentd
![Fluentd](https://user-images.githubusercontent.com/97789851/165901837-9a83ebeb-05c6-4092-8d67-689193a897d5.jpg)

Tạo một thư mục mới cho các tài nguyên Fluentd Docker của bạn và chuyển vào đó:
```console
sudo mkdir Fluent-Aggregator-Docker && cd Fluent-Aggregator-Docker
```
Tạo **Dockerfile**:
```console
sudo nano Dockerfile
```
Thêm chính xác các nội dung sau vào tệp của bạn. Tệp này yêu cầu Docker cài đặt **Ruby**, **Fluentd** và **plugin kết nối Elasticsearch** kèm 1 số lệnh thực thi khi khởi tạo container:
```console
FROM ruby:2.6.6
RUN apt-get update
RUN gem install fluentd -v "~>1.14.0"
RUN mkdir /etc/fluent
RUN apt-get install libcurl4-gnutls-dev -y
RUN /usr/local/bin/gem install fluent-plugin-elasticsearch
RUN /usr/local/bin/gem install fluent-plugin-secure-forward
ADD ./fluent.conf /etc/fluent/
ENTRYPOINT ["/usr/local/bundle/bin/fluentd", "-c", "/etc/fluent/fluent.conf"]
```
Bạn cũng cần tạo một tệp **fluent.conf** trong cùng thư mục đó:
```console
sudo nano fluent.conf
```
Tệp **fluent.conf** sẽ config như sau. Bạn có thể sao chép chính xác tệp này:
```console
<match **.*>
  @type copy
  <store>
    @type elasticsearch
    host 192.168.5.30
    port 9200
    index_name ${tag}
    type_name ${tag}
    enable_ilm true
    include_timestamp true
    flush_interval 5s
  </store>
  <store>
    @type stdout
  </store>
</match>

<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>
```
**-> Cùng phân tích cú pháp cấu hình:**

File config Fluentd được chia làm 2 phần chính là **match** và **source**. Trong đó:

**match** là đầu ra, định tuyến cũng như chỉ đường cho logs biết chúng được vận chuyển tới đâu
* <**.*> giúp fluentd nhận tất cả các "index - chỉ mục theo tag bên phía server fluentd gửi logs"
* @type, host, port: Gửi logs đến Elasticsearch, địa chỉ IP của server chứa Elastic (hướng dẫn này thì bộ thu thập logs EFK cài chung trên 1 server), mặc định Elastic hoạt động trên port 9200
* index_name, type_name, enable_ilm: Nhận chỉ mục, tên,... theo tag từ bên server Fluentd gửi logs đến
* include_timestamp, flush_interval: Khi tạo chỉ mục trên Kibana sẽ có giao diện, hình ảnh trực quan, dễ hiểu. Thời gian làm mới nhận logs và gửi logs
* @type stdout: Plugin output stdout in các sự kiện đầu ra theo tiêu chuẩn (hoặc logs nếu được khởi chạy dưới dạng daemon). Plugin đầu ra này rất hữu ích cho mục đích gỡ lỗi.

**source** là đầu vào, chỉ cho fluentd biết nhận logs từ những nguồn nào
*   @type forward, port, bind: Lắng nghe logs gửi đến mặc định ở port 24224 và bind để nhận từ tất cả các nguồn

Sau đó, xây dựng image Docker của bạn, có thể đặt tên image là **fluentd-aggregator**:
```console
docker build -t fluentd-aggregator .
```
Quá trình này sẽ mất vài phút để hoàn thành. Kiểm tra xem bạn đã tạo thành công các image chưa:
```console
docker image ls
```
Bạn sẽ thấy đầu ra như thế này:

    REPOSITORY           TAG       IMAGE ID       CREATED          SIZE
    fluentd-aggregator   latest    858465f09f14   17 seconds ago   898MB
    ruby                 2.6.6     6d86b0beade7   13 months ago    840MB

## Bước 2: Khởi động container Elasticsearch
![Elasticsearch](https://user-images.githubusercontent.com/97789851/165902276-0abe366c-a6e8-4241-8836-a165ed42b0ac.jpg)

Bây giờ, hãy quay lại thư mục gốc hoặc thư mục ưa thích của bạn cho container **Elasticsearch** và trong hướng dẫn này sẽ sử dụng **version 8.1.1**:
```console
cd && docker pull elasticsearch:8.1.1
```
Trước khi khởi động container Elasticsearch, chúng ta cần tăng giá trị **max_map_count** trên máy chủ Docker của bạn vì image này nhanh hơn so với việc tự cấu hình
```console
sudo sysctl -w vm.max_map_count=262144
```
Sau khi tải image Elasticsearch xuống thành công, hãy **khởi chạy** container cùng tham số và sử dụng biến trong container Elasticsearch
```console
docker run -d --name elasticsearch -p 9200:9200 -v /etc/localtime:/etc/localtime:ro -e "ES_JAVA_OPTS=-Xms128m -Xmx128m" -e "discovery.type=single-node" -e "xpack.security.enabled=false" elasticsearch:8.1.1
```
**-> Giải thích 1 chút cho các bạn hiểu :))**
* Mặc định Elasticsearch sử dụng port 9200
* -v /etc/localtime:/etc/localtime:ro ánh xạ múi giờ trên server vào container
* ES_JAVA_OPTS=-Xms128m -Xmx128m cấu hình JVM thường để bằng 1/3 dung lượng RAM server
* discovery.type=single-node hiện tại chỉ sử dụng 1 node Elasticsearch
* xpack.security.enabled=false tắt gói bảo mật của Elasticsearch, hướng dẫn này chưa dùng đến tính năng xác thực account để login

Tiếp theo, hãy đảm bảo rằng **container Elasticsearch** đang chạy bình thường bằng cách kiểm tra các container đang hoạt động:
```console
docker ps
```
Bạn sẽ thấy đầu ra như thế này:

    CONTAINER ID   IMAGE                 COMMAND                  CREATED          STATUS          PORTS                                                 NAMES
    577c7db077d8   elasticsearch:8.1.1   "/bin/tini -- /usr/l…"   33 minutes ago   Up 33 minutes   0.0.0.0:9200->9200/tcp, :::9200->9200/tcp, 9300/tcp   elasticsearch
### Kiểm tra thông số container Elasticsearch trên local
Mở trình duyệt browser trên máy của bạn và truy cập: **192.168.5.30:9200**

Bạn sẽ thấy kết quả hiển thị tương tự như sau:

![Elasticsearch](https://user-images.githubusercontent.com/97789851/165886662-5d7bc6d2-6a89-41f4-87a3-270e75fea0e7.png)

## Bước 3: Khởi động container Kibana kết nối Elasticsearch
![Kibana](https://user-images.githubusercontent.com/97789851/165902554-a264c186-0ccb-46b9-8e78-ccad94c8ab47.jpg)

**Chú ý: Version của Kibana và Elasticsearch phải cùng nhau**

Tải **Kibana version 8.1.1** về server
```console
docker pull kibana:8.1.1
```
Sau khi tải image Kibana xuống thành công, hãy **khởi chạy** container Kibana
```console
docker run -d --name kibana --link elasticsearch:es -p 5601:5601 -v /etc/localtime:/etc/localtime:ro kibana:8.1.1
```
**-> Giải thích nhé :>**
- --link elasticsearch:es liên kết container Elasticsearch với container Kibana
- Port hoạt động mặc định của Kibana là 5601
- ...

**->** Có rất nhiều cách để khởi chạy **container Kibana** nhưng theo mình thì cách trên khá ngắn gọn và dễ hiểu. Mình sẽ chia sẻ thêm 1 số cách khác nếu bạn nào thích có thể tìm hiểu thêm:

- **Cách 2:** Thay phần --link elasticsearch:es **->** -e "ELASTICSEARCH_URL=http://192.168.5.30:9200/" -e "ELASTICSEARCH_HOSTS=http://192.168.5.30:9200/"
  
- **Cách 3:** Không dùng link và biến liên kết ELASTICSEARCH. Các bạn chỉ mở port 5601, ánh xạ time trên server rồi chạy. Nhưng khi check port 5601 trên local bạn sẽ phải config import link URL Elasticsearch để lấy **verification-code**, sau đó trên server truy cập vào container Kibana để **verification-code**. Khá là cồng kềnh, mới đầu khi tìm hiểu mình cũng làm như vậy =))
### Kiểm tra container Kibana và check giao diện trên local
Xem container đã hoạt động chưa
```console
docker ps
```
    CONTAINER ID   IMAGE                 COMMAND                  CREATED          STATUS          PORTS                                                 NAMES
    3921a53f91a2   kibana:8.1.1          "/bin/tini -- /usr/l…"   59 minutes ago   Up 59 minutes   0.0.0.0:5601->5601/tcp, :::5601->5601/tcp             kibana
    577c7db077d8   elasticsearch:8.1.1   "/bin/tini -- /usr/l…"   21 hours ago     Up 3 hours      0.0.0.0:9200->9200/tcp, :::9200->9200/tcp, 9300/tcp   elasticsearch
Mở trình duyệt browser trên máy của bạn và truy cập: **192.168.5.30:5601**

Cách sử dụng **Kibana** như thế nào thì trong những phần tới mình sẽ nói rõ hơn. Giao diện trực quan khá đẹp :v
![Kibana](https://user-images.githubusercontent.com/97789851/165914781-4f28ac7c-eef6-4fe6-971a-5a9728a9d3d1.png)

## Bước 4: Khởi động container Fluentd-Aggregator liên kết Elasticsearch
Bây giờ chúng ta sẽ khởi động **container chạy Fluentd-Aggregator**, thu thập logs và gửi chúng đến Elastcisearch

Từ bước 1 chúng ta đã xây dựng image fluentd-aggregator nên giờ sẽ chạy image đó thành container để lấy và chuyển logs
```console
docker run -d --name fluentd-aggregator --link elasticsearch:es -p 24224:24224/tcp -p 24224:24224/udp -e "TZ=Asia/Ho_Chi_Minh" fluentd-aggregator
```
Trong đó:
- Fluentd hoạt động trên port 24224
- "TZ=Asia/Ho_Chi_Minh": để đa dạng mình sử dụng biến trong container để cập nhật lại múi giờ thay vì ánh xạ múi giờ từ server như các container trước. Các bạn có thể dùng lại cách ánh xạ vẫn đúng
- ...

**-> Cuối cùng**, hãy kiểm tra xem bộ thu thập logs EFK có đang chạy hay không bằng cách kiểm tra các quy trình Docker container đang hoạt động trên server:
```console
docker ps
```
Bạn sẽ thấy cả container **Elasticsearch**, **Kibana** và container **fluentd--aggregator** mới:

    CONTAINER ID   IMAGE                 COMMAND                  CREATED          STATUS          PORTS                                                                                          NAMES
    509e61f7ac48   fluentd-aggregator    "/usr/local/bundle/b…"   16 minutes ago   Up 16 minutes   0.0.0.0:24224->24224/tcp, 0.0.0.0:24224->24224/udp, :::24224->24224/tcp, :::24224->24224/udp   fluentd-aggregator
    3921a53f91a2   kibana:8.1.1          "/bin/tini -- /usr/l…"   3 hours ago      Up 3 hours      0.0.0.0:5601->5601/tcp, :::5601->5601/tcp                                                      kibana
    577c7db077d8   elasticsearch:8.1.1   "/bin/tini -- /usr/l…"   23 hours ago     Up 5 hours      0.0.0.0:9200->9200/tcp, :::9200->9200/tcp, 9300/tcp                                            elasticsearch

## Bước 5: Xác nhận rằng Elasticsearch đang nhận sự kiện. Test index - chỉ mục trên Kibana:
Hãy xác nhận rằng **Elasticsearch** đang nhận các sự kiện:
```console
curl -XGET 'http://localhost:9200/_all/_search?q=*'
```
Đầu ra phải chứa các sự kiện giống như sau:

    {"took":1,"timed_out":false,"_shards":{"total":1,"successful":1,"skipped":0,"failed":0},"hits":{"total":{"value":9,"relation":"eq"},"max_score":1.0,"hits":[{"_index":"fluent.info","_id":"r3ecdIAB6Cq5dgvkXYbU","_score":1.0,"_source":{"pid":11,"ppid":1,"worker":0,"message":"starting fluentd worker pid=11 ppid=1 worker=0","@timestamp":"2022-04-29T16:17:20.209262808+07:00"}},{"_index":"fluent.info","_id":"sHecdIAB6Cq5dgvkXYbV","_score":1.0,"_source":{"port":24224,"bind":"0.0.0.0","message":"listening port port=24224 bind=\"0.0.0.0\"","@timestamp":"2022-04-29T16:17:20.209830444+07:00"}},{"_index":"fluent.info","_id":"sXecdIAB6Cq5dgvkXYbV","_score":1.0,"_source":{"worker":0,"message":"fluentd worker is now running worker=0","@timestamp":"2022-04-29T16:17:20.210894155+07:00"}},{"_index":"fluent.info","_id":"snejdIAB6Cq5dgvkFobh","_score":1.0,"_source":{"pid":10,"ppid":1,"worker":0,"message":"starting fluentd worker pid=10 ppid=1 worker=0","@timestamp":"2022-04-29T16:24:41.935037705+07:00"}},{"_index":"fluent.info","_id":"s3ejdIAB6Cq5dgvkFobh","_score":1.0,"_source":{"port":24224,"bind":"0.0.0.0","message":"listening port port=24224 bind=\"0.0.0.0\"","@timestamp":"2022-04-29T16:24:41.935544274+07:00"}},{"_index":"fluent.info","_id":"tHejdIAB6Cq5dgvkFobh","_score":1.0,"_source":{"worker":0,"message":"fluentd worker is now running worker=0","@timestamp":"2022-04-29T16:24:41.936044111+07:00"}},{"_index":"fluent.info","_id":"tXejdIAB6Cq5dgvk6oZi","_score":1.0,"_source":{"pid":10,"ppid":1,"worker":0,"message":"starting fluentd worker pid=10 ppid=1 worker=0","@timestamp":"2022-04-29T16:25:35.065391635+07:00"}},{"_index":"fluent.info","_id":"tnejdIAB6Cq5dgvk6oZi","_score":1.0,"_source":{"port":24224,"bind":"0.0.0.0","message":"listening port port=24224 bind=\"0.0.0.0\"","@timestamp":"2022-04-29T16:25:35.066373598+07:00"}},{"_index":"fluent.info","_id":"t3ejdIAB6Cq5dgvk6oZi","_score":1.0,"_source":{"worker":0,"message":"fluentd worker is now running worker=0","@timestamp":"2022-04-29T16:25:35.067103266+07:00"}}]}}
**->** Bạn có thể có khá nhiều sự kiện được ghi lại tùy thuộc vào thiết lập của bạn. Một sự kiện phải bắt đầu bằng **{"took":** và kết thúc bằng **thời gian**. Như kết quả này hiển thị, **Elasticsearch** đang nhận dữ liệu và rất nhiều thông số khác

**->** Có thể thấy rất nhiều thông số về logs khi chúng ta thiết lập và cấu hình trong file **fluent.conf**
### Để có thể tập trung và phân tích logs một cách dễ nhìn và trực quan nhất, chúng ta sẽ sử dụng Kibana
  - Trên trang giao diện **Kibana**. Tại thanh tìm kiếm search: **Data Views**

![Kibana](https://user-images.githubusercontent.com/97789851/165927798-a63864cb-bbc5-420a-a9a2-8f5a8b0643fc.png)

- Nhấn **Create data view**. Nhìn góc bên phải bạn thấy 1 index mặc định **fluent.info** do cấu hình trong Fludentd. Copy và paste vào phần **Name** bên phải, phần **Timestamp field** sử dụng **@timestamp**. Cuối cùng nhấn vào **Create data view** bên dưới để tạo index

![Untitled](https://user-images.githubusercontent.com/97789851/165929552-3f053c28-077d-4aee-9f24-31a740b5061c.png)

- Tại mục menu chính, phần **Analytics** chọn **Discover** để xem chi tiết logs của index **fluent.info** vừa tạo. Tại đây bạn có thể thấy các thông số như: thời gian, thông tin hoạt động của fluent, id,... Bạn có thể chọn **Try Document Explorer** và bỏ tích phần **doc_table:legacy** để nhìn giao diện logs một cách trực quan nhất

![Kibana](https://user-images.githubusercontent.com/97789851/166153151-669bd388-7ed6-4e6a-b2e7-91852b8b1f7f.png)

**-> Như vậy cơ bản là đã setup và config xong server cài EFK nhận logs 192.168.5.30**

# Phần II - Setup và config server gửi logs (192.168.5.40)
**Tiếp theo mình sẽ hướng dẫn các bạn cài server để gửi logs đến EFK**

**->** Vì mục đích của hướng dẫn này là giám sát logs sinh ra từ container Docker nên server gửi logs sẽ cài 1 số dịch vụ (ví dụ dịch vụ **Nginx**) và được đọc bởi **fluentd**. Đây là server gửi logs nên mình sẽ đặt tên cho thư mục chứa các tài nguyên cài đặt là **fluent-forwarder**

## Bước 1: Sử dụng dịch vụ Nginx bằng Docker
Đầu tiên hãy chỉnh múi giờ cho server như các bước **Thiết lập cần thiết khi bắt đầu** ở đàu hướng dẫn, sau đó tải xuống **image Nginx** và khởi động nó
```console
docker pull nginx
```
```console
docker run -d --name nginx -p 80:80 -v /etc/localtime:/etc/localtime:ro nginx:latest
```
Bạn có thể kiểm tra dịch vụ **Nginx** hoạt động hay chưa bằng cách truy cập trên browser: **192.168.5.40:80**

Bạn sẽ thấy kết quả hiển thị tương tự như sau:

![Nginx](https://user-images.githubusercontent.com/97789851/166155990-f66cbf7c-fe23-4469-8b9c-809a5197786e.png)

**->** Sau khi **container Nginx** đã hoạt động. Mặc định logs sẽ được ghi vào dường dẫn **/var/lib/docker/containers/**
- Kiểm tra bằng lệnh: 
```console
sudo ls -l /var/lib/docker/containers/
```
- Bạn sẽ thấy 1 thư mục tương tự như sau:

      drwx--x--- 4 root root 4096 May  4 14:22 0c4d9eb7311717f166289342b743391d1a499a1e0f9b1364f5558e3e5a6bf9c6
- Logs sinh ra từ Docker sẽ được lưu mặc định dưới dạng **json.log**. Bạn có thể xem cụ thể file logs bằng câu lệnh tương tự như trên nhưng vào sâu hơn
```console
sudo ls -l /var/lib/docker/containers/0c4d9eb7311717f166289342b743391d1a499a1e0f9b1364f5558e3e5a6bf9c6
```
### Ở bước tiếp theo chúng ta sẽ sử dụng đến đường dẫn logs này
## Bước 2: Xây dựng và khởi động image Fluentd gửi logs 
Tương tự như **Phần I - Bước 1** nên mình sẽ giải thích ngắn gọn hơn, nếu khó hiểu các bạn có thể kéo lên trên xem lại

Tạo thư mục **Fluent-Forwarder-Docker**, trong đó tạo **Dockerfile** và tệp **fluent.conf**
```console
sudo mkdir Fluent-Forwarder-Docker && cd Fluent-Forwarder-Docker
```
```console
sudo nano Dockerfile
```
**Dockerfile** sẽ lược bỏ 2 plugin vì nó gửi logs trực tiêp đến **fluentd-aggregator**
```console
FROM ruby:2.6.6
RUN apt-get update
RUN gem install fluentd -v "~>1.14.0"
RUN mkdir /etc/fluent
RUN apt-get install libcurl4-gnutls-dev -y
ADD ./fluent.conf /etc/fluent/
ENTRYPOINT ["/usr/local/bundle/bin/fluentd", "-c", "/etc/fluent/fluent.conf"]
```
Tệp **fluent.conf** sẽ config như sau. Bạn có thể sao chép chính xác tệp này:
```console
<match *.**>
  @type forward
  send_timeout 60s
  recover_wait 10s
  hard_timeout 60s
  flush_interval 5s
  <server>
    name log_EFK
    host 192.168.5.30
    port 24224
    weight 60
  </server>
</match>

<source>
  @type tail
  path  /var/lib/docker/containers/0c4d9eb7311717f166289342b743391d1a499a1e0f9b1364f5558e3e5a6bf9c6/0c4d9eb7311717f166289342b743391d1a499a1e0f9b1364f5558e3e5a6bf9c6-json.log
  pos_file  /var/log/docker-nginx.pos
  tag docker.nginx
  <parse>
    @type json
    localtime true
    time_type string
    unmatched_lines
  </parse>
</source>

<filter *.**>
  @type stdout
</filter>
```
**-> Phân tích cú pháp cấu hình:**

**match**
- **@type** sử dụng **forward** để gửi logs
- **send_timeout**: Thời gian chờ khi gửi logs
- **recover_wait**: Thời gian đợi trước khi chấp nhận khôi phục lỗi từ phía server
- **hard_timeout**: Thời gian chờ này sử dụng để phát hiện lỗi từ phía server. Giá trị mặc định bằng tham số **send_timeout**
- Phần trong **< server >** để cho Fluentd biết nó gửi logs đến đâu

**source**
- **@type** bắt buộc phải là **tail**, cho phép Fluentd đọc các sự kiện từ phần đuôi của tệp lưu logs
- **path** đường dẫn để đọc logs. Nếu có nhiều dịch vụ, mỗi dịch vụ 1 đường dẫn thì dùng dấu phẩy **','** để ngăn cách
- **pos_file** nên sử dụng vì Fluentd sẽ ghi lại **vị trí** mà nó đọc logs lần cuối từ tệp này
- **tag**: Tên của index khi hiện thị trên Kibana
- Phần trong **< parse >** để kích hoạt cho các plugin hỗ trợ phân tích cú pháp của file logs. Cụ thể trong hướng dẫn này logs là file **json**. Tiếp đến **unmatched_lines** là 1 plugin phân tích file **json**

**filter** dùng filter stdout in ra các sự kiện đầu ra tiêu chuẩn

Xây dựng image **Fluentd Docker** của bạn, đặt tên image là **fluentd-forwarder**:
```console
docker build -t fluentd-forwarder .
```
Quá trình này sẽ mất vài phút để hoàn thành. Kiểm tra xem bạn đã tạo thành công các image chưa:
```console
docker image ls
```
Bạn sẽ thấy đầu ra như thế này:

    REPOSITORY          TAG       IMAGE ID       CREATED         SIZE
    fluentd-forwarder   latest    6fabc838ed11   2 hours ago     893MB
    nginx               latest    fa5269854a5e   13 days ago     142MB
    ruby                2.6.6     6d86b0beade7   13 months ago   840MB

Khởi động **container fluentd-forwarder** để gửi logs đến server EFK. Ánh xạ thư mục lưu logs của container trên server vào container fluentd-forwarder,...Dùng lệnh sau:
```console
docker run -d --name fluentd-forwarder -v /var/lib/docker/containers:/var/lib/docker/containers -p 24224:24224/tcp -p 24224:24224/udp -e "TZ=Asia/Ho_Chi_Minh" fluentd-forwarder
```

**->** Kiểm tra xem container **Fluentd** và **Nginx** có đang chạy hay không:
```console
docker ps
```
Bạn sẽ thấy kết quả tương tự:

    CONTAINER ID   IMAGE               COMMAND                  CREATED       STATUS       PORTS                                                                                          NAMES
    d38149be544e   fluentd-forwarder   "/usr/local/bundle/b…"   2 hours ago   Up 2 hours   0.0.0.0:24224->24224/tcp, 0.0.0.0:24224->24224/udp, :::24224->24224/tcp, :::24224->24224/udp   fluentd-forwarder
    0c4d9eb73117   nginx:latest        "/docker-entrypoint.…"   2 days ago    Up 3 hours   0.0.0.0:80->80/tcp, :::80->80/tcp                                                              nginx

**-> Đến đây chúng ta đã setup và config xong server gửi logs 192.168.5.40**

# Phần III - Test logs trên Kibaba
Tương tự **Phần I - Bước 5** do đó mình sẽ thao tác nhanh

- Tạo index **docker.nginx**

![Kibana](https://user-images.githubusercontent.com/97789851/166665070-e5faeb84-8b19-4c0c-a81e-7029fe50948c.png)

- Sau đó chuyển sang **Discover** để xem thông tin logs của dịch vụ **container Nginx**

![Kibana](https://user-images.githubusercontent.com/97789851/166665785-b604e195-7fea-4115-981f-250fe266f195.png)

**-> Phần về thu thập logs đến đây là kết thúc**

### Để phân tích logs và sử dụng các tính năng khác giúp lọc dữ liệu logs trên Kibana các bạn sẽ phải tự tìm hiểu. Nếu có thời gian mình sẽ tiếp tục 1 series về phân tính logs, regex logs, authentication EK,...

## Chúc các bạn thành công :>
