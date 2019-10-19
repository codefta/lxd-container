# Membuat Simple Wordpress dengan lxd-container
Dokumentasi untuk membuat sebuah simple wordpress site menggunakan LXD Container


1. Install zfsutils-tools
```
sudo install zfsutils-linux
```
2. `lxd init`
3. `lxc launch image:centos/7 server`
4. `lxc launch image:centos/7 database`
    `lxc list` for see lists
    `lxc exec {container} -- {/bin/bash} / {command}` for execute container /using container
5. install wp, nginx, php-fpm in linux di kontainer server
    - Install nginx 
    ```
    nano /etc/yum.repos.d/nginx.repo
    ```
    ```
    [nginx]
    name=nginx repo
    baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
    gpgcheck=0
    enabled=1
    ```
    ```
    yum install nginx
    ```

    - Install php-fpm versi 7.2 

    ```
    yum install epel-release
    rpm -Uvh http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
    rpm -Uvh http://repo.mysql.com/mysql-community-release-el7-7.noarch.rpm
    ```

    ```
    yum --enablerepo=remi-php72 install php-mysql php-xml php-soap php-xmlrpc php-mbstring php-json php-gd php-mcrypt php php-fpm
    ```

    - Install wp
    ```
    cd /tmp/
    ```
    ```
    wget http://wordpress.org/latest.tar.gz
    ```
    
6. Menginstall mariadb di server database
    ```
    yum install mariadb-server
    ```

7. Konfigurasi WP, Webserver, Database
    ### Webserver ###
    - Konfigurasi WP
    ```
    tar -xzvf latest.tar.gz
    ```
    ```
    mv wordpress /usr/share/nginx/html
    ```

    - Konfigurasi Nginx
    ```
    cd /etc/nginx/conf.d
    ```
    ```
    cp default.conf default.conf.bak
    ```

    ```
    server {
        listen   80;
        server_name  {your domain};

        # note that these lines are originally from the "location /" block
        root   /usr/share/nginx/html/wordpress;
        index index.php index.html index.htm;

        location / {
            try_files $uri $uri/ =404;
        }
        error_page 404 /404.html;
        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
            root /usr/share/nginx/html;
        }

        location ~ \.php$ {
            try_files $uri =404;
            fastcgi_pass unix:/var/run/php-fpm/php-fpm.sock;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include fastcgi_params;
        }
    }
    ```
    - Konfigurasi php-fpm
    ```
    cd cd /etc/php-fpm.d/www.conf
    ```

    ```
    user = nginx
    group = nginx

    # uncomment and change
    listen.owner = nginx
    listen.group = nginx

    # Add listen below listen = 127... and comment listen = 127...
    listen = var/run/php-fpm/php-fpm.sock;

    ```

    #### Start and enable them
    ```
    systemctl start nginx php-fpm
    systemctl enable nginx php-fpm
    ```

    ### Konfigurasi database
    ! Jangan lupa untuk keluar kontainer ya kemudian masuk ke kontainer database

    ### Membuat user baru dan membuat database
    ```
    mysql -u root

    CREATE USER 'namauser'@'%' identified by 'paswordmu';
    GRANT ALL PRIVILEGES ON * . *  TO 'namauser'@'%';
    FLUSH PRIVILEGES;

    mysql -u namauser

    CREATE DATABASE nama_database;
    ```

8. Set ip static `/etc/hosts` on linux
    ```
    nano /etc/hosts
    # Add line
    {your ip vm}    {your website domain}

    ```
9. Firewall with iptables
    - redirect TCP / 80 your vm to your web server
    ```
    sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -d {your vm ip address}-j DNAT --to-destination {container ip web server}:80

    sudo iptables -t nat -A POSTROUTING -o {network card vm}  -j MASQUERADE
    ```

    - redirect UDP / 443 your vm to your web server

    ```
    sudo iptables -t nat -A PREROUTING -p tcp --dport 443 -d {your vm ip address}-j DNAT --to-destination {container ip web server}:443

    sudo iptables -t nat -A POSTROUTING -o {network card vm}  -j MASQUERADE
    ```

    - Forwarding IP 80 vm to web server
    ```
    sudo iptables -A FORWARD -i {network card vm} -p tcp --dport 80 -d {your ip container web server} -j ACCEPT

10. Access your website domain
