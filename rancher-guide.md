<p align="center">
  <img src="https://raw.githubusercontent.com/rancher/rancher/master/assets/logo.png" width="220" alt="Rancher Logo" />
</p>

# คู่มือสรุปการติดตั้ง Helm และ Rancher Server บน Kubernetes จำลองด้วย K3d (Step-by-Step)

คู่มือฉบับนี้สรุปขั้นตอนการดาวน์โหลดและติดตั้งเครื่องมือจัดการแพ็กเกจ Kubernetes (Helm) การสร้าง Cluster จำลองโดยใช้ K3d การติดตั้ง Cert-Manager และการ Deploy Rancher Server สำหรับใช้จัดการหน้า GUI ของ Kubernetes Cluster ได้อย่างสะดวกและรวดเร็ว โดยตัดรหัสอ้างอิงเวลา (Timestamps) ออกเพื่อความกระชับและพร้อมนำไปใช้ปฏิบัติงานจริง

---

## 📥 ภาคที่ 1: การติดตั้ง Helm (Package Manager สำหรับ Kubernetes)
* **สถานที่ปฏิบัติงาน:** ทำบนเครื่องคอมพิวเตอร์หลัก (Terminal ของระบบ Linux/Ubuntu Server)

### Step 1: ดาวน์โหลดและแตกไฟล์ Helm Binary
ดาวน์โหลด Helm เวอร์ชัน Linux AMD64 และทำการแตกไฟล์บีบอัดเข้าระบบ
```bash
# 1. ดาวน์โหลดไฟล์ Helm บีบอัด
wget https://get.helm.sh/helm-v3.8.0-linux-amd64.tar.gz

# 2. แตกไฟล์บีบอัด
tar -zxvf helm-v3.8.0-linux-amd64.tar.gz
```

### Step 2: ย้ายไฟล์ Helm ไปยัง System Path
ย้ายไฟล์เครื่องมือ Helm ไปไว้ที่โฟลเดอร์ตำแหน่งระบบเพื่อให้สามารถพิมพ์เรียกใช้งานคำสั่ง `helm` ได้จากทุกที่
```bash
sudo mv linux-amd64/helm /usr/local/bin/helm
```

**ตรวจสอบเวอร์ชันเพื่อยืนยันการติดตั้ง:**
```bash
helm version
```

---

## 🐳 ภาคที่ 2: การจำลอง Kubernetes Cluster (ด้วย K3d)
* **สถานที่ปฏิบัติงาน:** Terminal ของระบบ Linux

### Step 3: สร้าง Kubernetes Cluster ใหม่และทำการ Map พอร์ตออกสู่ภายนอก
สั่งสร้างคลัสเตอร์จำลองชื่อ `cluster1` พร้อมทั้งกำหนดให้พอร์ต 80 และ 443 ชี้ออกมายังเครื่องหลักเพื่อรองรับการเปิดหน้าเว็บของ Rancher (ผ่านระบบสถาปัตยกรรมแบบ Load Balancer ภายใน)
```bash
k3d cluster create cluster1 -p "80:80@loadbalancer" -p "443:443@loadbalancer" --agents 1
```

**ตรวจสอบการเชื่อมต่อกับเครื่องมือควบคุม Cluster:**
```bash
kubectl get nodes
```

---

## 📦 ภาคที่ 3: การเพิ่ม Repository และติดตั้ง Cert-Manager
* **สถานที่ปฏิบัติงาน:** Terminal ของระบบ Linux (สั่งรันเข้าสู่ Kubernetes Cluster ที่สร้างไว้)

### Step 4: เพิ่ม Helm Repository ของ Rancher และ Jetstack
เพิ่มแหล่งเก็บซอร์สแพ็กเกจของ Rancher และโปรแกรมจัดการ SSL Certificate เข้าสู่ Helm
```bash
# 1. เพิ่ม Repo ของ Rancher
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable

# 2. เพิ่ม Repo ของ Jetstack (Cert-manager)
helm repo add jetstack https://charts.jetstack.io

# 3. อัปเดตรายชื่อแพ็กเกจล่าสุดของ Helm
helm repo update
```

### Step 5: ติดตั้ง Cert-Manager
สร้าง Name Space แยกเฉพาะและสั่งติดตั้งตัวจัดการและออกใบรับรองความปลอดภัย TLS/SSL (Cert-Manager)
```bash
# 1. ติดตั้ง Custom Resource Definitions (CRDs) ของ Cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.7.1/cert-manager.crds.yaml

# 2. ติดตั้ง Cert-Manager ผ่าน Helm
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.7.1
```

**ตรวจสอบสถานะการสร้างและรัน Pods ของระบบใบรับรอง:**
```bash
kubectl get pods --namespace cert-manager
```

---

## 🚀 ภาคที่ 4: การติดตั้ง Rancher Server
* **สถานที่ปฏิบัติงาน:** Terminal ของระบบ Linux

### Step 6: สร้าง Name Space สำหรับ Rancher
สร้าง Name Space ชื่อ `cattle-system` สำหรับเป็นพื้นที่ทำงานของระบบ Rancher
```bash
kubectl create namespace cattle-system
```

### Step 7: ผูกข้อมูลเครื่องระบุ Local Hostname (สำคัญสำหรับการเข้าหน้าเว็บทดสอบ)
ก่อนสั่งติดตั้ง เราจำเป็นต้องมีชื่อโดเมนในการเข้าถึง ในกรณีนี้จะใช้วิธี Local DNS โดยการเพิ่มชื่อโดเมนสมมุติชี้ไปที่ IP ของตัวเอง

* **ฝั่ง Linux (เครื่องเซิร์ฟเวอร์):**
  พิมพ์คำสั่งเปิดไฟล์โฮสต์:
  ```bash
  sudo nano /etc/hosts
  ```
  เพิ่มบรรทัดดังต่อไปนี้ (แทนที่ `<เลข_IP_เครื่อง_Linux>` ด้วย IP จริงของเครื่องเซิร์ฟเวอร์):
  ```text
  <เลข_IP_เครื่อง_Linux> rancher.local
  ```

* **ฝั่ง Windows (เครื่องที่เราใช้เปิดเว็บเบราว์เซอร์เพื่อเข้าชม):**
  1. เปิดโปรแกรม **Notepad** ด้วยสิทธิ์ **Run as administrator**
  2. เปิดไปที่ไฟล์ตำแหน่ง: `C:\Windows\System32\drivers\etc\hosts`
  3. เพิ่มบรรทัดเดียวกับฝั่ง Linux ลงไปที่ท้ายไฟล์:
     ```text
     <เลข_IP_เครื่อง_Linux> rancher.local
     ```

### Step 8: สั่งติดตั้ง Rancher ผ่าน Helm
สั่งรันคำสั่งติดตั้งด้วย Helm โดยส่งพารามิเตอร์กำหนด Hostname และปรับแก้จำนวน Replicas (จำนวน Container ที่รัน) ให้เหลือ 1 ตามข้อจำกัดสเปคเครื่องทดสอบ
```bash
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.local \
  --set replicas=1
```

### Step 9: ติดตามตรวจสอบสถานะการ Deploy
ตรวจสอบความคืบหน้าการสร้างและ Rollout ตัวระบบของ Rancher จนกว่าจะขึ้นข้อความระบุว่าสำเร็จเสร็จสิ้นทั้งหมด
```bash
kubectl -n cattle-system rollout status deployment/rancher
```

---

## 💻 ภาคที่ 5: การเข้าสู่หน้าเว็บและการหารหัสผ่านครั้งแรก (Bootstrap Password)
* **สถานที่ปฏิบัติงาน:** บนเว็บเบราว์เซอร์ และ Terminal เครื่อง Linux

### Step 10: เปิดเว็บเบราว์เซอร์และดึงรหัสผ่านครั้งแรก
1. เปิดหน้าเว็บเบราว์เซอร์ไปที่: `https://rancher.local`  
   *(กดยอมรับความเสี่ยงเรื่องใบรับรอง SSL ชั่วคราวบนเว็บเบราว์เซอร์เนื่องจากเป็นใบรับรองออกเองแบบ Local)*
2. หน้าเว็บจะระบุคำสั่งในการดึงรหัสผ่านใช้งานครั้งแรก ให้เราคัดลอกคำสั่งดังกล่าวกลับมารันที่หน้าจอ Terminal ของเครื่อง Linux:
   ```bash
   kubectl get secret --namespace cattle-system bootstrap-secret -o jsonpath="{.data.bootstrapPassword}" | base64 --decode --ignore-garbage
   ```
3. นำรหัสยาวๆ (Bootstrap Password) ที่ได้จากผลลัพธ์บนจอ Terminal ไปวางลงในหน้าเว็บเบราว์เซอร์ จากนั้นตั้งรหัสผ่านใหม่ที่มีความยาวอย่างน้อย 12 ตัวอักษรขึ้นไปเพื่อเข้าสู่หน้า UI จัดการอย่างเป็นทางการครับ
