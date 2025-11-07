# คู่มือแก้ไขปัญหา Docker Port 80 Conflict

## ปัญหาที่พบ

เมื่อรัน `docker compose up` บน EC2 instance เกิด error ดังนี้:

```
Error response from daemon: failed to set up container networking: driver failed programming external connectivity on endpoint nginx-proxy: failed to bind host port for 0.0.0.0:80:172.18.0.5:80/tcp: address already in use
```

## สาเหตุของปัญหา

Port 80 บน EC2 instance ถูกใช้งานอยู่แล้วโดยโปรแกรมอื่น ซึ่งมักจะเป็น:
- Apache web server
- Nginx ที่ติดตั้งไว้แล้วบน host machine
- โปรแกรมอื่นๆ ที่ใช้ port 80

Docker ไม่สามารถ bind port 80 ให้กับ nginx-proxy container ได้เพราะ port นี้ถูกใช้งานอยู่

## วิธีตรวจสอบว่าโปรแกรมไหนใช้ Port 80

รันคำสั่งใดคำสั่งหนึ่งต่อไปนี้บน EC2 instance:

```bash
# วิธีที่ 1: ใช้ lsof
sudo lsof -i :80

# วิธีที่ 2: ใช้ ss
sudo ss -tulpn | grep :80

# วิธีที่ 3: ใช้ netstat
sudo netstat -tulpn | grep :80
```

## วิธีแก้ไข

### วิธีที่ 1: หยุดและปิด Service ที่ใช้ Port 80 (แนะนำ)

หาก port 80 ถูกใช้โดย Apache หรือ Nginx:

**สำหรับ Apache:**
```bash
# หยุด Apache
sudo systemctl stop apache2

# ปิดการทำงานอัตโนมัติเมื่อบูตเครื่อง
sudo systemctl disable apache2

# ตรวจสอบสถานะ
sudo systemctl status apache2
```

**สำหรับ Nginx:**
```bash
# หยุด Nginx
sudo systemctl stop nginx

# ปิดการทำงานอัตโนมัติเมื่อบูตเครื่อง
sudo systemctl disable nginx

# ตรวจสอบสถานะ
sudo systemctl status nginx
```

หลังจากนั้นรัน Docker Compose อีกครั้ง:
```bash
docker compose up -d
```

### วิธีที่ 2: เปลี่ยน Port ของ Docker Compose

หากไม่ต้องการหยุด service ที่มีอยู่ สามารถเปลี่ยน port ของ nginx-proxy ได้

แก้ไขไฟล์ `docker-compose.yml` บรรทัดที่ 54:

```yaml
nginx:
  image: nginx:alpine
  container_name: nginx-proxy
  restart: always
  ports:
    - "8080:80"  # เปลี่ยนจาก "80:80" เป็น "8080:80"
  volumes:
    - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
  depends_on:
    - frontend
    - backend
  networks:
    - mern-network
```

จากนั้นรัน:
```bash
docker compose up -d
```

เว็บไซต์จะเข้าถึงได้ที่: `http://<EC2-IP>:8080`

**หมายเหตุ:** ต้องเปิด port 8080 ใน EC2 Security Group ด้วย

### วิธีที่ 3: Kill Process ที่ใช้ Port 80 โดยตรง (ไม่แนะนำ)

หา PID ของ process ที่ใช้ port 80:
```bash
sudo lsof -i :80 | grep LISTEN
```

Kill process (แทน PID ด้วยเลข process ID ที่ได้):
```bash
sudo kill -9 <PID>
```

จากนั้นรัน Docker Compose อีกครั้ง:
```bash
docker compose up -d
```

## การตรวจสอบหลังแก้ไข

หลังจากแก้ไขแล้ว ให้ตรวจสอบว่า containers ทำงานปกติ:

```bash
# ดู status ของ containers
docker compose ps

# ดู logs
docker compose logs -f

# ทดสอบเข้า health endpoint
curl http://localhost/api/health
# หรือถ้าใช้ port 8080
curl http://localhost:8080/api/health
```

## การเปิด Port ใน EC2 Security Group

หากใช้ port อื่นที่ไม่ใช่ 80 ต้องเพิ่ม Inbound Rule ใน Security Group:

1. เข้า EC2 Console
2. เลือก Security Groups
3. เลือก Security Group ของ instance
4. คลิก "Edit inbound rules"
5. เพิ่ม rule:
   - Type: Custom TCP
   - Port range: 8080 (หรือ port ที่เลือกใช้)
   - Source: 0.0.0.0/0 (สำหรับ public access)
6. บันทึก

## คำแนะนำ

- **วิธีที่ 1 เหมาะสมที่สุด** หากต้องการใช้ MERN app เป็น web server หลัก
- **วิธีที่ 2 เหมาะสำหรับการทดสอบ** หรือต้องการรัน multiple web applications
- **วิธีที่ 3 ไม่แนะนำ** เพราะ process อาจกลับมาทำงานอีกหลัง reboot

## ปัญหาเพิ่มเติมที่อาจพบ

### ถ้า containers ยังไม่ทำงาน

ตรวจสอบ logs ของแต่ละ service:
```bash
docker compose logs mongodb
docker compose logs backend
docker compose logs frontend
docker compose logs nginx
```

### ถ้า MongoDB ไม่ healthy

```bash
# เข้าไปใน MongoDB container
docker exec -it mongodb mongosh

# ทดสอบการเชื่อมต่อ
db.runCommand({ping: 1})
```

### ถ้า Backend ไม่สามารถเชื่อมต่อ MongoDB

ตรวจสอบว่า MongoDB container ทำงานและ healthy:
```bash
docker compose ps
```

Restart services:
```bash
docker compose restart backend
```
