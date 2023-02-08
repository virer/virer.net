# Apache Guacamole with podman setup
Here is a very quick documentation to configure a simple Guacamole setup.<br>

For this setup :<br>
- I used CentOS 7<br>
- I used podman as docker replacement.<br>
- I removed podman internal (default) network to avoid IP/Subnet overlap.<br>
<br>
<br>

<br>
<br>
First step install podman
```console
yum install -y podman
```
<br>
<br>
Remove podman default network (to avoid IP/Subnet overlap):<br>
```console
podman network rm podman
```
<br>

Pull Apache Guacamole  dockerhub images:<br>
```console
podman pull guacamole/guacamole:1.4.0
podman pull guacamole/guacd:1.4.0
podman pull mysql/mysql-server
```
<br>
<br>
<br>
Start first container:
```console
podman run --rm --network=host --name guac-guacd  -d guacamole/guacd:1.4.0
```
<br>
<br>
Extract the inital MySQL configuration.
```console
podman run --rm --network=host guacamole/guacamole:1.4.0 /opt/guacamole/bin/initdb.sh --mysql > /opt/initdb.sql
```
<br>
<br>
Then, start the MySQL server and check the logs for the first time generated password
```console
podman run --run --network=host --name guac-mysql --mount type=bind,src=/var/lib/mysql,dst=/var/lib/mysql -e MYSQL_RANDOM_ROOT_PASSWORD=yes -e MYSQL_ONETIME_PASSWORD=yes -d mysql/mysql-server:8.0.29

podman logs guac-mysql
```
<br>
<br>

<br>
<br>
<br>
Configure MySQL for guacamole db &amp; user (with the previously recovered password in the logs) and restart it
```console
podman exec -it guac-mysql bash

$ mysql -u root -p
Enter password: &lt;Enter the Generated Root Password In MySQL container logs&gt;

&gt;&gt; ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPassword';
&gt;&gt; CREATE DATABASE guacamole_db;
&gt;&gt; CREATE USER 'guacamole_user'@'%' IDENTIFIED BY 'HeyThisISaGuacamoleDemoPassword';
&gt;&gt; GRANT SELECT,INSERT,UPDATE,DELETE ON guacamole_db.* TO 'guacamole_user'@'%';
&gt;&gt; FLUSH PRIVILEGES;
&gt;&gt; quit

$ ^D

podman stop guac-mysql
podman run --name guac-mysql --network=host --hostname guac-mysql --mount type=bind,src=/var/lib/mysql,dst=/var/lib/mysql -e MYSQL_ONETIME_PASSWORD=no -d mysql/mysql-server:8.0.29
```

<br>
<br>
Then, generate self-signed SSL certificate in "/opt" and configure nginx that will serve as reverse proxy:<br>
So here I create local files, then I map the directory to the "/etc/nginx..." inside the containers<br>

### Note: 
  certificates should not be in that directory , this is maybe a security issue, here it is just for quick demo purpose<br>
```console
mkdir -p /opt/etc/nginx/conf.d
openssl req -x509 -sha256 -nodes -days 365 -subj '/C=BE/CN=guacamole.local' -newkey rsa:4096 -keyout cert.key -out cert.crt
cat &lt;&lt;EOF&gt; /opt/etc/nginx/conf.d/default.conf
server {
    listen       443 ssl;
    ssl_protocols TLSv1.3 TLSv1.2;
    ssl_certificate /etc/nginx/conf.d/cert.crt;
    ssl_certificate_key /etc/nginx/conf.d/cert.key;
    server_name  localhost;
    location / {
        proxy_pass   http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
        # Enable the following to force Header Authentication
        # proxy_set_header X-Remote-User guacadmin;
        client_max_body_size 1g;
    }
}
EOF

podman run --rm --network=host --name guac-nginx -v /opt/etc/nginx/conf.d/:/etc/nginx/conf.d/ -d -p 443:443 nginx:1.21.6
```
<br>
<br>
<br>
Now, you can finaly run the guacamole frontend part:<br>
```console
podman run --rm --network=host --name guac-guacamole -e GUACD_HOSTNAME=127.0.0.1 -e GUACD_PORT=4822 -e MYSQL_HOSTNAME=127.0.0.1 -e MYSQL_DATABASE=guacamole_db -e MYSQL_USER=guacamole_user -e MYSQL_PASSWORD="HeyThisISaGuacamoleDemoPassword" -d guacamole/guacamole:1.4.0
```


if you need Header Authentication add the following podman arguments:<br>
```console
-e HEADER_ENABLED=true -e HTTP_AUTH_HEADER=X-Remote-User
```
<br>
<br>

<br>
<br>
Now just use your browser on https://your-machine-ip/guacamole/<br>
and enjoy :)<br>
<br>
<br>
Tips: Never, ever use "latest" tags in your deployments to avoid using wrong/not tested version, it will avoid some waste of time :)
