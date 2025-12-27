
# ST Public API

## 1. Overview
เอกสารนี้อธิบายวิธีการเชื่อมต่อและใช้งาน ST Public API สำหรับ Partner และ Developer  
รายละเอียดของ Endpoint ทั้งหมดสามารถดูเพิ่มเติมได้จาก Swagger UI

---

## 2. Authentication & Request Security

ทุกคำขอไปยัง Public API ต้องผ่านการยืนยันตัวตนและความถูกต้องของคำขอ ตามหัวข้อด้านล่าง

---

### 2.1 API Key

ผู้ใช้งานต้องส่ง API Key ผ่าน HTTP Header ในทุก request

```http
X-API-KEY: <your_api_key>
````

---

### 2.2 Signature Header

Signature Header ใช้สำหรับยืนยันว่าคำขอถูกส่งจากผู้ที่ถือ API Secret และข้อมูลในคำขอไม่ได้ถูกแก้ไขระหว่างทาง
ผู้ใช้งานต้องสร้าง Signature จากข้อมูลของ request และส่งมาใน Header ทุกครั้ง

```http
X-SIGNATURE: <HMAC_SHA256>
```

#### Signature Payload Format

Signature ถูกสร้างจากการนำค่าต่อไปนี้มาต่อกันเป็น string ตามลำดับ

```
{IDEMPOTENCY_KEY}{HTTP_METHOD}{REQUEST_PATH}{REQUEST_BODY}
```

* `HTTP_METHOD` ต้องเป็นตัวพิมพ์ใหญ่
* หากไม่มี request body ให้ใช้ empty string
* ข้อมูลที่นำไป sign ต้องตรงกับ request จริงทุกตัวอักษร

---

#### generateSignature (Node.js)

ใช้สำหรับสร้างค่า Signature เพื่อส่งใน Header ของทุกคำขอ

```ts
import crypto from "crypto";

function generateSignature({ idempotencyKey, method, path, body, apiSecret }) {
  const payload =
    idempotencyKey +
    method.toUpperCase() +
    path +
    JSON.stringify(body);

  return crypto
    .createHmac("sha256", apiSecret)
    .update(payload)
    .digest("hex");
}
```

---

#### ตัวอย่างการใช้งาน (Node.js + fetch)

```ts
import fetch from "node-fetch";
import { v4 as uuidv4 } from "uuid";

const API_URL = "https://api.example.com";
const API_KEY = "your_api_key";
const API_SECRET = "your_api_secret";

const method = "POST";
const path = "/v1/buys";
const idempotencyKey = uuidv4();

const body = {
  amount: "1000",
  network: "JFIN",
  address: "0x000000000000000000000000000000000000dead"
};

const signature = generateSignature({
  idempotencyKey,
  method,
  path,
  body,
  apiSecret: API_SECRET
});

fetch(`${API_URL}${path}`, {
  method,
  headers: {
    "Content-Type": "application/json",
    "X-API-KEY": API_KEY,
    "Idempotency-Key": idempotencyKey,
    "X-SIGNATURE": signature
  },
  body: JSON.stringify(body)
})
  .then(res => res.json())
  .then(result => console.log(result))
  .catch(err => console.error(err));
```

---

### 2.3 Idempotency-Key

Idempotency-Key ใช้เพื่อป้องกันการสร้างคำสั่งซ้ำในกรณีที่ client มีการ retry request
ทุก request ที่สร้าง resource สำคัญ **ต้องส่ง Idempotency-Key เสมอ** เช่น

* สร้างคำสั่งซื้อ USDT
* สร้างคำสั่งขาย USDT

```http
Idempotency-Key: <uuid_v4 หรือ unique string>
```

#### กติกาการใช้งาน

* ชุดค่า `Idempotency-Key` + `path` + `method` ต้องไม่ซ้ำกันภายในช่วงเวลา TTL (เช่น 24 ชั่วโมง)

* หากมีการส่ง request ซ้ำด้วย `Idempotency-Key` เดิม:

  * หากคำขอเดิมสำเร็จแล้ว → ระบบจะคืนผลลัพธ์เดิม (status code และ response body เดิม)
  * หากคำขอยังอยู่ระหว่างการประมวลผล → ระบบอาจตอบกลับ `202 Accepted` หรือรอจนประมวลผลเสร็จ

* หากตรวจพบ `Idempotency-Key` เดียวกัน แต่ payload แตกต่างกัน
  → ระบบจะตอบกลับ `409 Conflict` พร้อม error code `IDEMPOTENCY_KEY_CONFLICT`

---

## 3. Environments

### Base URL (Production)

```
-
```

---

## 4. Response Structure

### 4.1 Success (2xx)

```json
{
  "success": true,
  "data": {
    // payload ของ endpoint นั้นๆ
  }
}
```

### 4.2 Error (4xx / 5xx)

```json
{
  "success": false,
  "error": {
    "code": "ORDER_NOT_FOUND",
    "message": "Order not found"
  }
}
```

---

## 5. API Endpoints

รายละเอียดของ API Endpoint ทั้งหมด รวมถึงพารามิเตอร์ รูปแบบข้อมูล Request และ Response
ถูกรวบรวมไว้ใน Scalar UI

กรุณาใช้งาน Scalar UI เพื่อ: 
👉 [Scalar UI](https://registry.scalar.com/@default-team-vq802/apis/st-public-apis/latest)

---

## 6. Error Handling

รายละเอียด Error Code และความหมายของแต่ละ Error จะถูกระบุไว้ใน Scalar UI

---

## 7. Rate Limiting

ระบบอาจมีการจำกัดจำนวน request ต่อ API Key ตามนโยบายที่กำหนด
หากเกินข้อจำกัด ระบบจะตอบกลับด้วย HTTP 429


