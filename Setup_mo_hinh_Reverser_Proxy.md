# Xây dựng mô hình reverse proxy kết hợp giữa 2 webserver là nginx và apache, php 7.4, mysql và phpmyadmin, tìm hiểu vì sao nginx đứng trước apache.
## Xây dựng mô hình reverse proxy kết hợp giữa 2 webserver 
- Mô hình này kết hợp giữa Nginx và Apache theo kiến trúc Reverse Proxy, trong đó Nginx đóng vai trò là proxy đứng trước, nhận request từ client rồi chuyển tiếp đến Apache để xử lý PHP. Lúc này Ngnix đóng vai trò làm front end xử lý các file tĩnh còn Apache làm back end để xử lý các dynamic content.
#### 
![text](./Imagers%20/Screenshot%20from%202025-04-02%2018-13-34.png)
- Giải thích mô hình 
  + Ngnix : Lắng nghe các request từ internet, xử lý cache, các static file (ảnh, css, javascript,...) và chuyển tiếp request tới Apache.
  + Apache : Chạy module PHP (PHP-FPM hoặc mod_php) để xử lý các request php.
  + MySQL : Cơ sở dữ liệu website 
  + ZFS :  Hệ thống quản lý file 
## Vì sao nginx đứng trước apache
- Hiệu suất cao hơn: Nginx sử dụng mô hình event-driven, giúp xử lý nhiều request đồng thời mà không tiêu tốn quá nhiều tài nguyên, trong khi Apache sử dụng thread-based nên dễ bị quá tải.
- Tối ưu xử lý file tĩnh: Nginx phục vụ file tĩnh nhanh hơn nhiều so với Apache.
- Load Balancing & Reverse Proxy: Nginx có thể điều phối request đến nhiều backend (có thể là nhiều Apache hoặc nhiều dịch vụ khác).
- Giảm tải cho Apache: Apache chỉ cần tập trung xử lý các request PHP, giúp hệ thống tối ưu hơn.
- Bảo mật: Nginx có thể đóng vai trò là firewall ứng dụng web (WAF), giúp bảo vệ backend Apache khỏi tấn công.
## Cài đặt mô hình 
### Bước 1 : Cài đặt Apache và PHP-FPM
    sudo apt update // Cập nhật 
    sudo apt install apache2 php-fpm // Tiếp theo cài đặt gói Apache và PHP-FPM
#### Module FastCGI Apache sẽ không có sẵn trong kho lưu trữ Ubuntu nên phải tải từ kernel.org và cài đặt bằng câu lệnh dpkg
    wget https://mirrors.edge.kernel.org/ubuntu/pool/multiverse/liba/libapache-mod-fastcgi/libapache2-mod-fastcgi_2.4.7~0910052141-1.2_amd64.deb
    sudo dpkg -i libapache2-mod-fastcgi_2.4.7~0910052141-1.2_amd64.deb
### Bước 2 : Cấu hình Apache và PHP-FPM
#### Đổi cổng Apache sang 8080 và cấu hình để làm việc với PHP-FPM bằng cách sửu dụng module mod_fastcgi. Đổi tên file cấu hình ports.conf của Apache bằng:
    sudo mv /etc/apache2/ports.conf /etc/apache2/ports.conf.default
#### Tạo mới một file ports.conf với cổng 8080:
    echo "Listen 8080" | sudo tee /etc/apache2/ports.conf
#### Tiếp theo tạo một file virtual server cho Apache, chỉ thị <VirtualHost> trong tệp này sẽ được đặt để chỉ phục vụ các trang web trên cổng 8080.
#### Vô hiệu hoá virtual server mặc định:
    sudo a2dissite 000-default
#### Tiếp theo, tạo một file virtual server mới:
    sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/001-default.confsudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/001-default.conf
#### mở file cấu hình và sửa port thành 8080
    sudo vi /etc/apache2/sites-available/001-default.conf
![text](./Imagers%20/Screenshot%20from%202025-04-03%2014-54-17.png)
#### Kích hoạt file 001-default.conf 
    sudo a2ensite 001-default
#### Reload Apache 
    sudo systemctl reload apache2
#### Kiểm tra apache đang listening trên port 8080 
    netstat -tlpn 
![text](./Imagers%20/Screenshot%20from%202025-04-03%2010-56-33.png)
### Cấu hình Apache để sử dụng mod_fastcgi
#### mod_fastcgi phụ thuộc vào mod_action(mặc định bị vô hiệu hóa) 
#### Nên đầu tiên cần bâtj mod_action 
    sudo a2enmod actions
#### Kiếm tra nếu có file fastcgi.conf trong /etc/apache2/mods-enable thì đổi tên thành fastcgi.conf.default 
    sudo mv /etc/apache2/mods-enabled/fastcgi.conf /etc/apache2/mods-enabled/fastcgi.conf.default
#### Nếu không có file đó thì tạo file 
    vi /etc/apache2/mods-enabled/fastcgi.conf
#### Thêm các lệnh sau vào file để chuyển tiếp các yêu cầu cho file .php sang socket UNIX PHP-FPM.
    <IfModule mod_fastcgi.c>
     AddHandler fastcgi-script .fcgi
     FastCgiIpcDir /var/lib/apache2/fastcgi
     AddType application/x-httpd-fastphp .php
     Action application/x-httpd-fastphp /php-fcgi
     Alias /php-fcgi /usr/lib/cgi-bin/php-fcgi
     FastCgiExternalServer /usr/lib/cgi-bin/php-fcgi -socket /run/php/php7.4-fpm.sock -pass-header Authorization
      <Directory /usr/lib/cgi-bin>
        Require all granted
      </Directory>
    </IfModule>
#### Lưu các thay đổi và chạy lệnh kiểm tra 
    sudo apachectl -t
##### Trạng thái : Syntax OK 
#### Reload apache 
    sudo systemctl reload apache2
### Bước 4 : Xác minh lại chức năng php 
#### Đảm bảo rằng PHP hoạt động bằng cách tạo tệp phpinfo() và truy cập file đó từ trình duyệt web. Tạo một tệp /var/www/html/info.php, nó sẽ chứa lệnh gọi hàm phpinfo:
    echo "<?php phpinfo(); ?>" | sudo tee /var/www/html/info.php
#### Cho phép cổng 8080 thông qua firewall:
    sudo ufw allow 8080
#### Chấp nhận Apache Full kiểm soát lưu lượng truy cập trên cổng 80 và 443:
    sudo ufw allow "Apache Full"
#### Kiểm tra lại trạng thái firewall:
    sudo ufw status
![text](./Imagers%20/Screenshot%20from%202025-04-03%2015-10-43.png)
#### Kiểm tra trang info.php trên trình duyệt, hãy đi đến http://14.225.204.253:8080/info.php. Nó sẽ hiển thị list các hiệu chỉnh cài đặt PHP đang sử dụng (Server API là Apache2 ):
![text](./Imagers%20/Screenshot%20from%202025-04-03%2015-22-37.png)
