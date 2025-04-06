# Các bước cài Nginx làm reverse proxy cho apache (Thực hành với 2 source code wordpress và laravel)
## Cài đặt apache2 
    sudo apt update
    sudo apt install apache2
## Cài đặt PHP7.4
    sudo apt -y install software-properties-common
    sudo add-apt-repository ppa:ondrej/php
    sudo apt-get update
    sudo apt -y install php7.4
    sudo apt-get install -y php7.4-cli php7.4-json php7.4-common php7.4-mysql php7.4-zip php7.4-gd php7.4-mbstring php7.4-curl php7.4-xml php7.4-bcmath
    php -v

## Upload 2 Source code lên vps 
    scp -r Downloads/SourceLaravel/laravel_source root@14.225.204.253:/var/www/luc.laravel.vietnix.tech
**Có thể đứng ở máy chứa source sử sụng `scp` như ví dụ trên để tải source lên tùy theo đường dẫn mà sửa lại cho đúng**
### Sau khi upload xong cấp quyền cho webserver
    sudo chown -R www-data:www-data /var/www/luc.wp.vietnix.tech
    sudo chown -R www-data:www-data /var/www/luc.laravel.vietnix.tech
    sudo chmod -R 755 /var/www/

## Chỉnh sửa port cho apache để nó listen ở port 8080 và 8081 trong file /etc/apache2/ports.conf:
    vi /etc/apache2/ports.conf
![text](./Imagers%20/Screenshot%20from%202025-04-03%2016-22-19.png)

## Cấu hình Virtual Hosts
### Website Wordpress 
#### Tạo file config cho wordpress 
    vi /etc/apache2/sites-available/luc.wp.vietnix.tech.conf
#### Nội dung file .conf 
    <VirtualHost *:8080>
    ServerName luc.wp.vietnix.tech
    DocumentRoot /var/www/luc.wp.vietnix.tech

    <Directory /var/www/luc.wp.vietnix.tech>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/wp_error.log
    CustomLog ${APACHE_LOG_DIR}/wp_access.log combined
    </VirtualHost>
### Website Laravel 
#### Tạo file config cho Laravel 
    vi /etc/apache2/sites-available/luc.laravel.vietnix.tech.conf
#### Nội dung file .conf 
    <VirtualHost *:8081>
    ServerName luc.laravel.vietnix.tech
    DocumentRoot /var/www/luc.laravel.vietnix.tech/public

    <Directory /var/www/luc.laravel.vietnix.tech/public>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/laravel_error.log
    CustomLog ${APACHE_LOG_DIR}/laravel_access.log combined
    </VirtualHost>

### Bật các Vitual Host và khởi động lại Apache 
#### Chạy các lệnh sau để kích hoạt 
    sudo a2ensite luc.wp.vietnix.tech.conf
    sudo a2ensite luc.laravel.vietnix.tech.conf
    sudo systemctl restart apache2
#### Kiểm tra trạng thái xem Apache có listen trên port 8080 và 8081 
    ss -tulnp | grep apache
![text](./Imagers%20/Screenshot%20from%202025-04-03%2016-27-43.png)

## Cài đặt mysql 
    sudo apt update
    sudo apt install mysql-server
    sudo systemctl start mysql.service
## Tạo Database vào user cho wordpress 
    mysql -u root -p // Vào mysql
    create database WORDPRESS; // Tạo DB WORDPRESS
    create user 'LucWP'@'localhost' IDENTIFIED WITH mysql_native_password BY '123456'; //Tạo user LucWP với pass 123456
    GRANT ALL ON `WORDPRESS`.* TO 'LucWP'@'localhost'; //gán full quyền cho user
    flush privileges;
## Tạo Database và user cho laravel 
    mysql -u root -p
    create database LARAVEL;
    create user 'LucLaravel'@'localhost' IDENTIFIED WITH mysql_native_password BY '123456';
    GRANT ALL ON `LARAVEL`.* TO 'LucLaravel'@'localhost';
    flush privileges;
#### `show databases;` để xem 2 DB vừa tạo 
![text](./Imagers%20/Screenshot%20from%202025-04-03%2016-48-04.png)
## Cài đặt Ngnix 
    sudo apt update
    sudo apt install nginx
#### Xóa file default trong /etc/nginx/site-available và symlink của nó bên enabled
#### Tạo 2 file /etc/nginx/sites-available/wordpress và /etc/nginx/sites-available/nginx:
    server {
        server_name luc.wp.vietnix.tech;

        location / {
            proxy_pass http://localhost:8080;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
        location ~* \.(?:ico|css|js|gif|jpe?g|png|woff2?|eot|ttf|otf|svg|mp4|webp)$ {
            expires 7d;
            log_not_found off;
            access_log off;
        }
    }
####
    server {
        server_name luc.laravel.vietnix.tech;

        location / {
            proxy_pass http://localhost:8081;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
        location ~* \.(?:ico|css|js|gif|jpe?g|png|woff2?|eot|ttf|otf|svg|mp4|webp)$ {
            expires 7d;
            log_not_found off;
            access_log off;
        }
    }
#### Tạo symlink đến /etc/nginx/site-enabled/
    ln -s /etc/nginx/sites-available/wordpress /etc/nginx/sites-enabled/
####
    ln -s /etc/nginx/sites-available/laravel /etc/nginx/sites-enabled/

## Cấu hình database phù hợp vào file wp-config.php của wordpress và file .env của laravel (php artisan migrate - để đồng bộ database cho laravel)
#### Trước khi chạy migration cho DB thì ta phải import data từ file .sql 
#### Ta sử dụng cú pháp lệnh scp path/file.sql root@ip:/path/folder_destionation để tải file .sql lên vps 
#### Sau đó ta đăng nhập vào từng user để import data 
    mysql -u lucWP -p
    USE WORDPRESS;
    source path_to_destionation/file.sql 
- làm tương tự cho user LucLaravel với database LARAVEL 
#### Sau khi kiểm tra database đã có dữ liệu, cấu hình file wp-config.php cho wordpress và .env cho laravel 
#### Đối với laravel ta cd vào luc.laravel.vietnix.tech chạy `php artisan migrate`
---
## Vấn đề tiếp theo là khi đã upload source và import dâta rồi thì ta cần cài các thư viện cũng như framework cho wordpress và laravel 
### Với laravel 
#### Bước 1 : Vào cd/var/www/luc.laravel.vietnix.tech cài các dependency 
    composer install --no-dev --optimize-autoloader
#### Nếu chưa có composer thì cài 
    sudo apt install composer
#### Bước 2 : Phân quyền cho /storage và /bootstrap/cache 
    sudo chown -R www-data:www-data storage bootstrap/cache
    sudo chmod -R 775 storage bootstrap/cache
#### Bước 3 : Tạo key cho laravel 
    php artisan key:generate
#### Bước 4 : Kiểm tra lại các thông tin cấu hình database trong file .env 
#### Bước 5 :; Chạy migration database 
    php artisan migrate
#### Bước 6 : Restart lại apache và nginx
    sudo systemctl restart apache2
    sudo systemctl restart nginx

### Với wordpress 
#### Bước 1 : cấu hình các thông số trong wp-config.php để connect database 
#### Bước 2 : Phân quyền 
    sudo chown -R www-data:www-data /var/www/luc.wp.vietnix.tech
    sudo find /var/www/luc.wp.vietnix.tech -type d -exec chmod 755 {} \;
    sudo find /var/www/luc.wp.vietnix.tech -type f -exec chmod 644 {} \;
#### Bước 3 : Restart lại apache và nginx
---
### Đến đây thì cấu hình đã xong có thể xuất hiện một số lỗi như cấu hình xong như truy cập domain chưa được 
#### Một số tip kiểm tra 
#### 1. Kiểm tra Nginx reverse proxy có hoạt động không
    sudo nginx -t
#### Nếu OK là chạy 
    sudo systemctl restart nginx
#### 2. Kiểm tra domain trỏ đúng hay chưa 
    ping luc.laravel.vietnix.tech
#### 3. Kiểm tra port 80 có mở chưa (trên vps)
    sudo ufw status
####
    sudo ufw allow 80
    sudo ufw allow 443
    sudo ufw reload
#### 4. Kiểm tra file Nginx reverse proxy (đã khai báo trong /etc/nginx/sites-enabled/laravel chưa?)
    ls -l /etc/nginx/sites-enabled/
#### Kết quả phải thấy 
    laravel -> /etc/nginx/sites-available/laravel
#### 5. Kiểm tra xem Nginx có đang nghe port 80 không
    sudo netstat -tulpn | grep :80
#### Kết quả phải thấy 
    tcp   ...   0.0.0.0:80     ...  nginx
#### 6. Clear cache cũ 
    cd /var/www/luc.laravel.vietnix.tech
    php artisan config:clear
    php artisan cache:clear
    php artisan route:clear
    php artisan view:clear
#### 7. Gặp lỗi quyền truy cập có thể là do laravel chưa đọc đúng file .env 
#### 7.1. Kiểm tra lại thông tin cấu hình đúng chưa nếu đúng rồi thì kiếm tra bước sau 
#### 7.2. Kiểm tra quyền trên mysql 
    mysql -u root -p
    SHOW GRANTS FOR 'LucLaravel'@'localhost';
#### Kết quả 
    GRANT ALL PRIVILEGES ON `LARAVEL`.* TO 'LucLaravel'@'localhost'
#### 7.3. Kiểm tra laravel đang đọc file nào
    php artisan tinker
    >>> env('DB_USERNAME')
#### 
    php artisan tinker
    >>> config('database.connections.mysql.username')
#### kết quả 
    => "LucLaravel"
#### ở đây kết quả phải ra username mà ta đã cấu hình nếu ra khác thì xem lại có thể backup file đó lại rồi tạo file mới 
#### 8. Kiểm tra quyền file .env 
    ls -al /var/www/luc.laravel.vietnix.tech | grep env
#### Kết quả 
    -rw-r--r--  1 www-data www-data   ... .env
#### Phân quyền cho đúng file .env 
    sudo chown www-data:www-data /var/www/luc.laravel.vietnix.tech/.env
    sudo chmod 644 /var/www/luc.laravel.vietnix.tech/.env
#### 9. lỗi phiên bản php 
#### Bước 1 : Kiểm tra phiên bản php `php -v`
#### Bước 2 : `sudo apachectl -M | grep php` để kiểm tra apache đang dùng phiên bản php nào nếu kết quả không có gì hoặc ra php7_module thì là vấn đề 
#### Bước 3 : Disable mod_php cũ (php7.x) `sudo a2dismod php7.4`
#### Bước 4 : Enable proxy module và php8.4-fpm 
    sudo a2enmod proxy_fcgi setenvif
    sudo a2enconf php8.4-fpm
#### Bước 5 : Mở file /etc/apache2/sites-available/luc.laravel.vietnix.tech.conf và xác nhận socket đúng:
    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php8.4-fpm.sock|fcgi://localhost"
    </FilesMatch>
#### Bước 6 : Khởi động lại Apache & PHP-FPM
    sudo systemctl restart php8.4-fpm
    sudo systemctl restart apache2
#### Bước 7 : Kiểm tra lại `sudo apachectl -M | grep php`

#### Gặp lỗi 
    SQLSTATE[42S02]: Base table or view not found: 1146 Table 'LARAVEL.sessions' doesn't exist (SQL: select * from sessions where id = sKsSyFsUnPT3nixF8njV23E8c4PVGAP94vWcl4GF limit 1)
#### Lỗi này là do Laravel đang dùng database để lưu session, nhưng bảng sessions chưa tồn tại trong database LARAVEL.
#### chỉ cần chạy lệnh migrate để tạo bảng sessions:
    php artisan session:table
    php artisan migrate

---
## Cài ssl cho 2 domain 
### Cài cho luc.laravel.vietnix.tech 
#### Bước 1 : Cài Certbot 
    sudo apt update 
    sudo apt install certbot python3-certbot-nginx -y
#### Bước 2 : Tạo SSL
    sudo certbot --nginx -d luc.laravel.vietnix.tech
#### Chọn email xác thực 
![text](./Imagers%20/Screenshot%20from%202025-04-05%2007-20-59.png)
#### Chon Y 
![text](./Imagers%20/Screenshot%20from%202025-04-05%2007-22-09.png)
#### Chọn N nếu không muốn share và kết quả thành công
![text](./Imagers%20/Screenshot%20from%202025-04-05%2007-23-20.png)
#### Sau khi cài xong ta kiểm tra trong /etc/nginx/sites-avaiable/laravel 
![text](./Imagers%20/Screenshot%20from%202025-04-05%2008-24-34.png)
#### Bước 3 : Test 
    curl -I https://luc.laravel.vietnix.tech
#### Bước 4 : Ta có thể cài thêm tự động gia hạn cho ssl 
    sudo certbot renew --dry-run
### Cài cho luc.wp.vietnix.tech 
#### Bước 1 : Do đã cài certbot rồi nên ta cài tiếp sll
    sudo certbot --nginx -d luc.wp.vietnix.tech 
#### Ta cũng mà tương tự như các bước cài cho laravel 
![text](./Imagers%20/Screenshot%20from%202025-04-05%2009-31-08.png)

---
### Cấu hình Apache chỉ listen trên một port
#### Bước 1 : Ở lab trước ta cấu hình cho laravel là port 8001 giờ ta cần chỉnh lại 
    vi /etc/apache2/sites-available/luc.laravel.vietnix.tech.conf
![text](./Imagers%20/Screenshot%20from%202025-04-05%2010-24-48.png)
#### Bước 2 : Kiểm tra vitual host của wordpress cũng là port 8080 
    vi /etc/apache2/sites-available/luc.wp.vietnix.tech.conf
![text](./Imagers%20/Screenshot%20from%202025-04-05%2010-27-30.png)
#### Bước 3 : Cập nhật file cấu hình Apache 
    vi /etc/apache2/ports.conf
![text](./Imagers%20/Screenshot%20from%202025-04-05%2010-28-18.png)
#### Bước 4 : Cập nhật Nginx Reverse Proxy cho laravel và wordpress 
    vi /etc/nginx/sites-available/laravel 
    vi /etc/nginx/sites-available/wordpress
![text](./Imagers%20/Screenshot%20from%202025-04-05%2010-30-18.png)
![text](./Imagers%20/Screenshot%20from%202025-04-05%2010-32-47.png)
#### Bước 5 : Restart apache 
    sudo systemctl restart apache2
#### Bước 6 : Reload Nginx
    sudo nginx -t
    sudo systemctl reload nginx


---
# Cài ssl cho cả apache và nginx : https nginx proxypass về https apache - http nginx proxypass về http apache
## 1. Nginx (Reverse Proxy) : 
- Khi truy cập vào các domain qua HTTP (port 80), Nginx sẽ proxy tới Apache cũng chạy HTTP.
- Khi truy cập qua HTTPS (port 443), Nginx sẽ proxy tới Apache chạy HTTPS.
## 2. Apache 
- Apache sẽ phục vụ nội dung của website (WordPress và Laravel) qua cả HTTP và HTTPS.
- Tức là, Nginx sẽ chỉ đóng vai trò là reverse proxy, chuyển hướng traffic tới Apache, nhưng SSL sẽ được xử lý ở Nginx.
## Cách triển khai: 
1. Cấu hình SSL cho Nginx (Reverse Proxy):
   - Nginx sẽ đảm nhận việc xử lý SSL (HTTPS) với Let's Encrypt, sau đó proxy yêu cầu đến Apache qua HTTP hoặc HTTPS tùy thuộc vào kết nối của client.
2. Cấu hình HTTP Proxy từ Nginx sang Apache:
   - Khi client yêu cầu HTTP, Nginx sẽ proxy yêu cầu đó đến Apache qua HTTP (tức là không mã hóa).
3. Cấu hình HTTPS Proxy từ Nginx sang Apache:
   - Khi client yêu cầu HTTPS, Nginx sẽ xử lý SSL và proxy yêu cầu tới Apache qua HTTPS.
## Cấu hình 
### 1 . Trong /etc/nginx/sites-available tạo 4 file 
![text](./Imagers%20/Screenshot%20from%202025-04-06%2023-21-02.png)
### Nội dung trong từng file 
### laravel-http.conf
    server {
    listen 80;
    server_name luc.laravel.vietnix.tech;

    location / {
        return 301 https://$host$request_uri;
        }
    }  
### laravel-https.conf
    server {
    listen 443 ssl;
    server_name luc.laravel.vietnix.tech;

    ssl_certificate /etc/letsencrypt/live/luc.laravel.vietnix.tech/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/luc.laravel.vietnix.tech/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://127.0.0.1:8443;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Port 443;
        proxy_set_header X-Forwarded-Ssl on;
        }
    }
### wordpress-http.conf
    server {
    listen 80;
    server_name luc.wp.vietnix.tech;

    location / {
        return 301 https://$host$request_uri;
        }
    }
### wordpress-https.conf
    server {
    listen 443 ssl;
    server_name luc.wp.vietnix.tech;

    ssl_certificate /etc/letsencrypt/live/luc.wp.vietnix.tech/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/luc.wp.vietnix.tech/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://127.0.0.1:8443;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Port 443;
        proxy_set_header X-Forwarded-Ssl on;
        }
    }
### Sau khi config xong restart lại nginx
    sudo systemctl restart nginx
### 2. Trong /etc/apache2 config ports.conf listening port 8080 và 8443
![text](./Imagers%20/Screenshot%20from%202025-04-06%2023-33-37.png)
### 3. Trong /etc/apache2/sites-available tạo 4 file 
![text](./Imagers%20/Screenshot%20from%202025-04-06%2023-35-07.png)
#### Dùng `sudo a2dissite default...` để không bị chiếm port 
### luc.laravel.vietnix.tech-ssl.conf
    <VirtualHost *:8443>
        ServerName luc.laravel.vietnix.tech
        DocumentRoot /var/www/luc.laravel.vietnix.tech/public

    #    SSLEngine on
    #    SSLCertificateFile /etc/letsencrypt/live/luc.laravel.vietnix.tech/fullchain.pem
    #    SSLCertificateKeyFile /etc/letsencrypt/live/luc.laravel.vietnix.tech/privkey.pem

        <Directory /var/www/luc.laravel.vietnix.tech/public>
            AllowOverride All
            Require all granted
        </Directory>

        ErrorLog ${APACHE_LOG_DIR}/laravel-ssl-error.log
        CustomLog ${APACHE_LOG_DIR}/laravel-ssl-access.log combined
    </VirtualHost>
### luc.laravel.vietnix.tech.conf
    <VirtualHost *:8080>
        ServerName luc.laravel.vietnix.tech
        DocumentRoot /var/www/luc.laravel.vietnix.tech/public

        <Directory /var/www/luc.laravel.vietnix.tech/public>
            AllowOverride All
            Require all granted
        </Directory>

        ErrorLog ${APACHE_LOG_DIR}/laravel_error.log
        CustomLog ${APACHE_LOG_DIR}/laravel_access.log combined

        <FilesMatch \.php$>
            SetHandler "proxy:unix:/run/php/php8.4-fpm.sock|fcgi://localhost"
        /FilesMatch>

    </VirtualHost>
### luc.wp.vietnix.tech-ssl.conf
    <VirtualHost *:8443>
        ServerName luc.wp.vietnix.tech
        DocumentRoot /var/www/luc.wp.vietnix.tech

    #    SSLEngine on
    #    SSLCertificateFile /etc/letsencrypt/live/luc.wp.vietnix.tech/fullchain.pem
    #    SSLCertificateKeyFile /etc/letsencrypt/live/luc.wp.vietnix.tech/privkey.pem

        <Directory /var/www/luc.wp.vietnix.tech>
            Options Indexes FollowSymLinks
            AllowOverride All
            Require all granted
        </Directory>

        ErrorLog ${APACHE_LOG_DIR}/wp-ssl-error.log
        CustomLog ${APACHE_LOG_DIR}/wp-ssl-access.log combined
    </VirtualHost>
### luc.wp.vietnix.tech.conf
    <VirtualHost *:8080>
        ServerName luc.wp.vietnix.tech
        DocumentRoot /var/www/luc.wp.vietnix.tech

        <Directory /var/www/luc.wp.vietnix.tech>
            AllowOverride All
            Require all granted
        </Directory>

        ErrorLog ${APACHE_LOG_DIR}/wp_error.log
        CustomLog ${APACHE_LOG_DIR}/wp_access.log combined

        <FilesMatch \.php$>
            SetHandler "proxy:unix:/run/php/php7.4-fpm.sock|fcgi://localhost"
        </FilesMatch>

    </VirtualHost>
#### Sau khi config xong restart lại apache 
    sudo apachectl configtest
    sudo systemctl restart apache2
#### Nếu WP có lỗi có thể xem log 
    sudo tail -n 50 /var/log/apache2/wp-ssl-error.log





