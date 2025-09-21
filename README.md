# Docker-Note

## 1) ภาพรวมสั้นๆ
* Dockerfile = “สูตรทำภาพระบบ (Image)” ว่าจะเอาอะไรลงบ้าง ขั้นตอนไหนก่อนหลัง
* docker-compose.yml = “รายการจัดครัว” บอกว่าจะเปิดกี่บริการ (services) แต่ละบริการใช้ภาพไหน ผูกพอร์ต/โฟลเดอร์อะไรเข้าด้วยกัน

## 2) Dockerfile: คำสั่งหลักแบบเข้าใจง่าย
```
# 1) เลือกครัวตั้งต้น (Base image)
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 8080

# 2) ขั้นตอน Build (มี SDK)
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet restore
RUN dotnet publish -c Release -o /out

# 3) เอาของที่อบเสร็จไปใส่ครัวตัวจริง (runtime บางเบา)
FROM base AS final
WORKDIR /app
COPY --from=build /out .
# 4) ตั้งคำสั่งเริ่มต้นเมื่อ container รัน
ENTRYPOINT ["dotnet", "YourApp.dll"]
```
### คำสั่งสำคัญ (เปรียบเทียบให้เห็นภาพ)

โครงสร้างตัวอย่าง (สำหรับ .NET 8 Web API)

`FROM` <image> – “เลือกเตา/ครัวพื้นฐาน” ที่มีของบางอย่างพร้อมอยู่แล้ว เช่น Ubuntu/Alpine/.NET Runtime/Node

`WORKDIR /path` – “ย้ายเข้าโซนทำงาน” (เหมือน cd) เพื่อให้คำสั่งต่อๆ ไปทำในโฟลเดอร์นี้

`COPY <src> <dest>` – “ยกวัตถุดิบเข้าครัว” (คัดลอกไฟล์จากเครื่องเราเข้า image)

`RUN <command>` – “ลงมือทำระหว่างอบภาพ” เช่น RUN dotnet restore หรือ RUN apt-get update

`EXPOSE <port>` – “แจ้งพอร์ตที่บริการนี้ฟังอยู่” (ไม่ได้เปิดไฟร์วอลล์ แค่ประกาศให้รู้)

`ENV KEY=value` – “ตั้งค่าตัวแปรแวดล้อม” เหมือนป้ายกำกับในครัว

`ARG KEY=value` – “ตัวแปรช่วง Build” (ใช้ตอน docker build --build-arg)

`ENTRYPOINT ["cmd", "arg"]` – “คำสั่งหลักเวลา container เริ่มทำงาน” (เปรียบเหมือนเปิดร้านแล้วทำเมนูนี้ทันที)

`CMD ["arg1","arg2"]` – “ค่าเริ่มต้นสำหรับคำสั่งหลัก” (ถูกแทนได้ตอน docker run ...)

`VOLUME /data` – “ประกาศพื้นที่เก็บข้อมูลถาวร” (ไม่หายเมื่อคอนเทนเนอร์ดับ)

`USER appuser` – “สลับผู้ใช้ที่รัน” (ลดสิทธิ์ root เพื่อความปลอดภัย)

`HEALTHCHECK` – “เช็คสุขภาพ” แจ้ง Docker ว่าบริการยังดีอยู่ไหม

### เปรียบเทียบ ENTRYPOINT vs CMD:

* ENTRYPOINT คือ “เมนูหลักของร้าน”
* CMD คือ “ตัวเลือกเริ่มต้นของเมนูหลัก” (ลูกค้าใส่คำสั่งใหม่มาทับได้)

## 3) docker-compose.yml: รวมหลายบริการให้ทำงานร่วมกัน
ตัวอย่าง: เว็บ .NET + PostgreSQL + Redis (แสดงภาพกว้างๆ เข้าใจง่าย)
```
version: "3.9"
services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"       # host:container
    environment:
      - ASPNETCORE_URLS=http://+:8080
      - ConnectionStrings__Default=Host=db;Port=5432;Database=app;Username=postgres;Password=postgres
      - Cache__Host=cache
    depends_on:
      - db
      - cache

  db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=app
    volumes:
      - dbdata:/var/lib/postgresql/data

  cache:
    image: redis:7-alpine

volumes:
  dbdata:

```
### คำสำคัญใน compose
`services` – รายชื่อบริการ (web, db, cache)

`image` – ใช้ภาพสำเร็จรูปจาก registry (ไม่ต้อง build เอง)

`build` – บอกว่าจะสร้างภาพจาก Dockerfile ในโปรเจกต์เรา

`ports` – ผูกพอร์ตจากเครื่องเราไปคอนเทนเนอร์

`environment` – ตั้งค่า ENV ให้บริการนั้น

`volumes` – ที่เก็บข้อมูลถาวร (เช่นฐานข้อมูล)

`depends_on` – ลำดับการสตาร์ท (ไม่ได้รอจน “พร้อมใช้งาน” แค่เริ่มก่อน)

> เปรียบเทียบ: Dockerfile = ทำภาพหนึ่งก้อน, Compose = ยกทั้งกองทัพขึ้นมาพร้อมกัน และกำหนดวิธีคุยกันในเครือข่ายเดียวกัน (service name คือ hostname)

## 4) วิธีใช้งานแบบ step-by-step
สร้าง/อัปเดตภาพ
```
docker build -t myapp:latest .
```
รันทีละคอนเทนเนอร์ (ไม่ใช้ compose)
```
docker run -p 8080:8080 myapp:latest
```
รันทั้งชุดด้วย compose
```
docker compose up --build
# กด Ctrl+C เพื่อหยุด
# ถ้าจะรันฉากหลัง:
docker compose up -d --build
# ปิดและลบคอนเทนเนอร์ (ไม่ลบ volume)
docker compose down
```
## 5) โครงสร้างโปรเจกต์ .NET ที่แนะนำ (ง่ายและสะอาด)
```
/YourApp
  ├─ src/
  │   └─ YourApp.csproj
  ├─ Dockerfile
  ├─ docker-compose.yml
  └─ .dockerignore

```
ตัวอย่าง .dockerignore (ช่วยให้ build ไวขึ้น)
```
bin/
obj/
.git/
.gitignore
**/*.user
**/*.swp
```
## 6) ข้อผิดพลาดที่พบบ่อย (พร้อมวิธีคิดเชิงเหตุผล)
* COPY ผิด path → ตรวจ WORKDIR เสมอ ว่าไฟล์ถูกคัดลอกไปที่ไหน
* พอร์ตไม่ตรง → ใน Dockerfile EXPOSE 8080 แต่ใน compose ต้อง ports: "8080:8080" ให้ตรงพอร์ตที่แอปฟังจริง
* ลืม multi-stage build → ภาพจะ “อ้วน” และช้า แนะนำ build ด้วย SDK แล้วคัดเฉพาะผลลัพธ์ไป runtime
* สิทธิ์ไฟล์/ผู้ใช้ → ถ้าไม่จำเป็นอย่ารันเป็น root (ใช้ USER) เพื่อความปลอดภัย
* depends_on ไม่ได้รอ DB พร้อม → ถ้าต้องการรอจริง ให้ทำ healthcheck หรือรอรีไทรในโค้ด/entrypoint script

## 7) สูตรลัดจำง่าย (Cheat Sheet)
Dockerfile
* เริ่ม: `FROM` → ตั้งโฟลเดอร์งาน: `WORKDIR`
* เอาไฟล์เข้า: `COPY` → ติดตั้ง/คอมไพล์: `RUN`
* บอกพอร์ต: `EXPOSE` → รันจริง: `ENTRYPOINT` (พร้อม `CMD` ถ้าต้อง)
Compose
* `services: web/db/cache`
* `build` หรือ image
* `ports` แมป host:container
* `environment` ตั้งค่าคอนฟิก
* `volumes` เก็บข้อมูลถาวร

## 8) ถ้าคุณใช้ MySQL/MongoDB อยู่แล้ว (ตัวอย่าง Compose สั้นๆ)
```
services:
  web:
    build: .
    ports: ["8080:8080"]
    environment:
      - ConnectionStrings__Sql=Server=mysql;Database=app;User=root;Password=secret
      - Mongo__Connection=mongodb://mongo:27017/app
    depends_on: [mysql, mongo]

  mysql:
    image: mysql:8
    environment:
      - MYSQL_ROOT_PASSWORD=secret
      - MYSQL_DATABASE=app
    volumes:
      - mysqldata:/var/lib/mysql

  mongo:
    image: mongo:7
    volumes:
      - mongodata:/data/db

volumes:
  mysqldata:
  mongodata:
```
> ในโค้ด .NET ให้ใช้ host เป็นชื่อ service (mysql, mongo) แทน localhost เพราะอยู่ใน network เดียวกันของ compose
