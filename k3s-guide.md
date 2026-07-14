<p align="center">
  <img src="https://raw.githubusercontent.com/cncf/artwork/main/projects/k3s/icon/color/k3s-icon-color.png" width="150" alt="K3s Logo" />
</p>

# คู่มือสรุปการติดตั้งและใช้งาน K3s (Kubernetes) บน VPS Ubuntu (Step-by-Step)

คู่มือฉบับนี้สรุปขั้นตอนการติดตั้งและตั้งค่าระบบ Kubernetes ย่อส่วน (Lightweight Kubernetes - K3s) บนเซิร์ฟเวอร์ Ubuntu เพื่อสร้าง Cluster สำหรับจำลองการทำงานของ Master Node และ Worker Node พร้อมขั้นตอนการ Deploy แอปพลิเคชัน Nginx การเปิดพอร์ต (NodePort) และการติดตั้ง Kubernetes Dashboard 

---

## 🌐 ภาคที่ 1: การติดตั้งและตั้งค่า Master Node (Server)
* **สถานที่ปฏิบัติงาน:** ทำที่เครื่องเซิร์ฟเวอร์หลักที่จะใช้เป็น Master Node

### Step 1: ดาวน์โหลดและติดตั้ง K3s Master
ใช้คำสั่ง Conveniences Script ของ K3s เพื่อดาวน์โหลดและติดตั้งระบบแบบอัตโนมัติ 
```bash
curl -sfL https://get.k3s.io | sh -
```
> [!NOTE]  
> ระบบจะดาวน์โหลด K3s และติดตั้งเครื่องมือพื้นฐานที่จำเป็นในการควบคุม Cluster เช่น `kubectl` ให้โดยอัตโนมัติ

### Step 2: ตรวจสอบสถานะ Service
ตรวจสอบว่าบริการ k3s ทำงานและอยู่ในสถานะ Active หรือไม่
```bash
sudo systemctl status k3s
```

### Step 3: แก้ไขสิทธิ์การเข้าถึงไฟล์คอนฟิก (Kubernetes Config)
เนื่องจากไฟล์คอนฟิกของ K3s จะอนุญาตให้เฉพาะสิทธิ์ `root` อ่านได้เท่านั้น จึงต้องเปลี่ยนสิทธิ์เพื่อให้ User ปกติเรียกใช้งาน `kubectl` ได้
```bash
sudo chmod g+r /etc/rancher/k3s/k3s.yaml
```

**ทดสอบเรียกใช้คำสั่งเบื้องต้น:**
```bash
kubectl cluster-info
# หรือ
kubectl get nodes
```

### Step 4: เปิดใช้งานระบบพิมพ์คำสั่งอัตโนมัติ (Bash Autocompletion)
ตั้งค่าให้ระบบ Terminal สามารถกดปุ่ม `Tab` เพื่อช่วยพิมพ์คำสั่ง `kubectl` ได้รวดเร็วขึ้น
```bash
sudo kubectl completion bash > /etc/bash_completion.d/kubectl
```
> [!TIP]  
> เมื่อรันคำสั่งเสร็จสิ้น ให้เปิด-ปิดหน้าต่าง Terminal ใหม่ หรือทำการ Logout แล้ว Login เข้ามาใหม่ เพื่อให้การตั้งค่าการเติมคำสั่งเริ่มทำงาน

### Step 5: ค้นหา Token ของ Master Node เพื่อใช้เชื่อมต่อกับ Node อื่น
ดึงรหัส Token สำคัญที่จะใช้ในการตรวจสอบสิทธิ์เพื่อให้เครื่อง Worker Node อื่นๆ สามารถนำมาใช้ต่อเข้ากับ Cluster ได้
```bash
sudo cat /var/lib/rancher/k3s/server/token
```
> [!IMPORTANT]  
> คัดลอกและบันทึกรหัส Token ยาวๆ ที่แสดงบนหน้าจอเก็บไว้ให้ดี เพื่อนำไปใช้ในขั้นตอนของเครื่อง Worker Node

---

## 🚜 ภาคที่ 2: การจอย Worker Node (Agent) เข้าระบบ
* **สถานที่ปฏิบัติงาน:** ทำที่เครื่องอื่นๆ ที่ต้องการเข้าร่วม Cluster เป็น Worker Node (Agent)

### Step 6: รันคำสั่งเชื่อมต่อ Worker Node เข้าหา Master
พิมพ์คำสั่งดาวน์โหลดสคริปต์ K3s พร้อมกำหนดค่าตัวแปร (Environment Variables) ชี้กลับไปยัง Master Node
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<IP_ADDRESS_MASTER_NODE>:6443 K3S_TOKEN=<TOKEN_จาก_STEP_5> sh -
```

> [!NOTE]  
> * เปลี่ยน `<IP_ADDRESS_MASTER_NODE>` เป็นเลข IP ของเครื่อง Master Node (เช่น `192.168.2.109`)
> * เปลี่ยน `<TOKEN_จาก_STEP_5>` เป็นรหัส Token ที่คัดลอกมาจาก Step 5

**ตรวจสอบความถูกต้องหลังทำเสร็จ:**
* **ที่เครื่อง Worker:** ตรวจสอบสถานะการเชื่อมต่อ
  ```bash
  sudo systemctl status k3s-agent
  ```
* **ที่เครื่อง Master:** ตรวจสอบรายการ Node ใน Cluster (คุณควรเห็นชื่อโหนดของเครื่องที่เป็น Worker เพิ่มเข้ามาใหม่)
  ```bash
  kubectl get nodes
  ```

---

## 🚀 ภาคที่ 3: การ Deploy Application (Nginx) และเปิดพอร์ต
* **สถานที่ปฏิบัติงาน:** ทำที่เครื่อง Master Node

### Step 7: สร้างไฟล์คอนฟิก Deployment ของ Nginx
สร้างไฟล์โครงสร้างแบบ YAML เพื่อนำมาปรับเปลี่ยนค่าการทำงานตามรูปแบบ Best Practice
```bash
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > nginx.yaml
```

### Step 8: แก้ไขและรัน Deployment
1. ใช้ Nano เปิดแก้ไขไฟล์คอนฟิกเพื่อปรับจำนวน Pods:
   ```bash
   nano nginx.yaml
   ```
2. ปรับเปลี่ยนจำนวนการทำงานจำลอง (Replicas) เช่น เปลี่ยนเป็น `replicas: 5` (เพื่อให้กระจายตู้คอนเทนเนอร์ทำงานทั้งหมด 5 ตู้คู่ขนานกัน)
3. สั่งรันโครงการจริงบน Cluster:
   ```bash
   kubectl apply -f nginx.yaml
   ```

**เช็คสถานะการสร้างคอนเทนเนอร์ (Pods):**
```bash
kubectl get pods -o wide
```

### Step 9: เปิดบริการออกสู่ภายนอก (Expose NodePort)
ทำการเชื่อมโยงพอร์ตภายในคอนเทนเนอร์ออกมาให้เครื่องเซิร์ฟเวอร์ภายนอก (ผ่าน IP หลักของตัวโหนด) สามารถเปิดเรียกใช้งานได้
```bash
kubectl expose deployment nginx --port=80 --type=NodePort
```

**เช็คหมายเลขพอร์ตภายนอกที่สุ่มขึ้นมา:**
```bash
kubectl get service
```
> [!NOTE]  
> ให้สังเกตที่หัวข้อ **PORT(S)** ตัวระบบจะสุ่มพอร์ตในช่วง 30000+ ขึ้นมาให้ (เช่น `80:31106/TCP` ซึ่งหมายความว่าพอร์ตภายนอกคือ `31106`)

**การทดสอบเข้าใช้งาน:**
เปิดเว็บเบราว์เซอร์แล้วพิมพ์ URL:
`http://<IP_NODE>:<PORT_NODEPORT>` (เช่น `http://192.168.2.109:31106`) เพื่อดูหน้า Welcome ของ Nginx

---

## 📊 ภาคที่ 4: การติดตั้งและเข้าใช้งาน Kubernetes Dashboard
* **สถานที่ปฏิบัติงาน:** ทำที่เครื่อง Master Node

### Step 10: สั่งติดตั้งแดชบอร์ด
รันคำสั่งดาวน์โหลดไฟล์ Config จาก GitHub เพื่อสร้างสภาพแวดล้อมแดชบอร์ดในระบบ Cluster
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.5.0/aio/deploy/recommended.yaml
```

### Step 11: การเข้าใช้งาน Dashboard ผ่านการ Forward พอร์ต
เนื่องจาก Dashboard จะถูกซ่อนอยู่ภายในระบบเน็ตเวิร์กปิด ในตัวอย่างนี้จึงทำการ Forward พอร์ตออกมาชั่วคราวเพื่อเข้าใช้งานผ่านเครื่องคอมพิวเตอร์ส่วนตัวของคุณ
```bash
kubectl port-forward -n kubernetes-dashboard service/kubernetes-dashboard --address 0.0.0.0 10443:443
```

> [!WARNING]  
> เมื่อรันคำสั่งด้านบนเสร็จสิ้น **ห้ามปิดหน้าต่าง Terminal นี้**  
> จากนั้นเปิดเบราว์เซอร์ไปที่ `https://<IP_MASTER_NODE>:10443` นำ Token สิทธิ์ผู้ใช้งานขั้นสูง (Admin Token) ที่ได้จากระบบ ไปกรอกล็อกอินใช้งานหน้าแดชบอร์ดในรูปแบบ UI กราฟิกที่สวยงาม
