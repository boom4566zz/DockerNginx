![enter image description here](https://testimages1.s3.ap-southeast-1.amazonaws.com/boomgitprofile.png)
# NOTE : Docker,node.js => Nginx 
# การ Deploy Node.js หรือ Docker

> ขั้นตอนในการปรับใช้เซิฟเวอร์ Node.js หรือ docker ของ DigitalOcean โดยใช้ PM2, NGINX เป็น reverse proxy และ SSL จาก LetsEncrypt

## 1. ลงทะเบียน Digital Ocean
หากคุณใช้ลิงก์อ้างอิงด้านล่าง คุณจะได้รับ $10 ฟรี(1 or 2 months)  
https://www.digitalocean.com/    
  

## 2. สร้าง droplet และเข้าสู่ระบบผ่าน ssh
 ฉันจะใช้ผู้ใช้ root แต่แนะนำให้สร้างผู้ใช้ใหม่

## 3. ติดตั้ง Node/NPM
```
curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -

sudo apt install nodejs

node --version
```


### 4. run โปรเจ็ค node ของคุณ
จากตัวอย่างด้านล่างคือการเปิดโปรเจ็ค node.js ของคุณ  //simple port :3000

```
cd yourproject 
npm install
node yourproject.js //port3000
```
>จากตัวอย่างเป็นเพียงการทดลองเปิด port 3000 ของ application จาก node.js ของคุณ  
> ถ้าคุณเปิด port ผ่าน docker ให้ข้ามขั้นตอนถัดไป
## 5. ตั้งค่าตัวจัดการด้วย PM2 เพื่อให้แอปของคุณทำงานต่อไป
> PM2 จะใช้สำหรับ serives ที่เป็น node.js

```
sudo npm i pm2 -g
pm2 start app (หรือชื่อไฟล์อะไรก็ตามของคุณ)

# Other pm2 commands
pm2 show app
pm2 status
pm2 restart app
pm2 stop app
pm2 logs (เเสดง log stream)
pm2 flush (เคลียร์ logs)

# เพื่อให้แน่ใจว่าแอปเริ่มทำงาน pm2 เมื่อรีบูท ubuntu
```
>ตอนนี้คุณควรจะสามารถเข้าถึงแอพของคุณโดยใช้ IP และพอร์ตของคุณ ตอนนี้เราต้องการตั้งค่าไฟร์วอลล์ที่บล็อกพอร์ตนั้นและตั้งค่า NGINX เป็น reverse proxy เพื่อให้เราสามารถเข้าถึงได้โดยตรงโดยใช้พอร์ต 80 (http)

## 6. ติดตั้ง ufw firewall
```
sudo ufw enable
sudo ufw status
sudo ufw allow ssh (Port 22)
sudo ufw allow http (Port 80)
sudo ufw allow https (Port 443)
```

## 7. ติดตั้ง  NGINX เเละ configure
```
sudo apt install nginx

sudo nano /etc/nginx/sites-available/default
```
เพิ่มส่วนต่อไปนี้ไปยัง location path  ของ the server block
> หากคุณใช้ 1 เซิฟเวอร์ต่อ 1 เว็บให่วิธีต่อไปนี้
- มองหาตำเเหน่ง location / เเละทำการเเก้ไขตามตัวอย่างต่อไปนี้
```
    server_name yourdomain.com www.yourdomain.com;

    location / {
        proxy_pass http://localhost:5000; #whatever port your app runs on
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
```
### restart nginx
### เเละทำการ save เเละทำขั้นตอนด้านล่างต่อไป
```
# Check NGINX config
sudo nginx -t

# Restart NGINX
sudo service nginx restart
```
> หากมีข้อความต่อไปนี้จะถือว่าผ่าน ไม่มี  <span style="color:red"> error</span>.
```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

```

> หรือ ถ้าหากคุณมีหลายโดเมน ให้ทำตามขั้นตอนต่อไปนี้ เเทน--
 - ลบไฟล์ default จาก /etc/nginx/sites-available/default
 - ทำการเเก้ไข nginx.conf จาก /etc/nginx/conf.d
 - ทำการปิดคอมเม้นจากตำเเหน่งต่อไปนี้
 ```
    include /etc/nginx/conf.d/*.conf;
	include /etc/nginx/sites-enabled/*;
 ```
 เป็น
  ```
    include /etc/nginx/conf.d/*.conf;
	## include /etc/nginx/sites-enabled/*;
 ```
 > เป็นการคอมเม้นต์

- เเละทำการ seve เเละทำการ [restart nginx](#restart-nginx)

- จากนั้นให้สร้างไฟล์ DomainName.conf
```
sudo nano /etc/nginx/conf.d/{DomainName.conf}
```
> เช่น /etc/nginx/conf.d/catanddog.conf
- เพิ่งโด้ดต่อไปนี้

```
server {

        root /usr/share/nginx/html;
        index index.html index.htm index.nginx-debian.html;

        server_name DomainName.com www.DomainName.shop;

        location / {
                try_files $uri $uri/ =404;
                proxy_pass http://localhost:3000; 
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection 'upgrade';
                proxy_set_header Host $host;
                proxy_cache_bypass $http_upgrade;
        }

   
}
```
>  proxy_pass คือ port ของ app ของคุณ ในที่นี้คือ port : 3000
- เเละทำการ seve เเละทำการ [restart nginx](#restart-nginx)
```
### ตอนนี้คุณควรจะสามารถเยี่ยมชม IP ของคุณโดยไม่มีพอร์ต (พอร์ต 80) และดูแอปของคุณได้ผ่าน browser

## 8. เพิ่มโดเมนใน Digital Ocean

- ใน Digital Ocean ไปที่เครือข่ายและเพิ่มโดเมน

 - เพิ่ม record A สำหรับ @ และสำหรับ www ลงใน droplet ของคุณ


## จดทะเบียนและ/หรือตั้งค่าโดเมนจากผู้รับจดทะเบียน


เลือก "เนมเซิร์ฟเวอร์ที่กำหนดเอง" และเพิ่มสิ่งเหล่านี้

* ns1.digitalocean.com
* ns2.digitalocean.com
* ns3.digitalocean.com

> อาจใช้เวลาเล็กน้อยในการเผยแพร่
```
9. เพิ่ม SSL กับ LetsEncrypt
```
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt install python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```
# ใช้ได้ 90 วันเท่านั้น ทดสอบกระบวนการต่ออายุด้วยคำสั้งด้านล่างนี้
```
certbot renew --dry-run
```


