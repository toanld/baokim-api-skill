# Chữ ký số RSA-SHA1

Baokim dùng RSA-SHA1 để xác thực mọi request/response.

## Tạo RSA Key Pair

**Dùng OpenSSL (Linux/Mac):**
```bash
# Tạo private key
openssl genrsa -out partner_private.pem 2048

# Tạo public key từ private key
openssl rsa -in partner_private.pem -pubout -out partner_public.pem
```

Sau đó gửi `partner_public.pem` cho Baokim để họ xác thực chữ ký của bạn.

## Cách ký (Node.js)

```typescript
import crypto from "crypto";
import fs from "fs";

function signData(data: string, privateKeyPath: string): string {
  const privateKey = fs.readFileSync(privateKeyPath, "utf8");
  const sign = crypto.createSign("SHA1");
  sign.update(data);
  return sign.sign(privateKey, "base64");
}

// Ví dụ ký cho Verify Customer (Operation 9001)
const dataToSign = [
  requestId,
  requestTime,
  partnerCode,
  operation,
  bankNo,
  accNo,
  accType,
].join("|");

const signature = signData(dataToSign, "./partner_private.pem");
```

## Cấu trúc data ký theo từng Operation

| Operation | Data string để ký |
|-----------|-------------------|
| 9001 (Verify) | `RequestId\|RequestTime\|PartnerCode\|Operation\|BankNo\|AccNo\|AccType` |
| 9002 (Transfer) | `RequestId\|RequestTime\|PartnerCode\|Operation\|ReferenceId\|BankNo\|AccNo\|AccType\|RequestAmount\|Memo` |
| 9003 (Lookup) | `RequestId\|RequestTime\|PartnerCode\|Operation\|ReferenceId` |
| 9004 (Balance) | `RequestId\|RequestTime\|PartnerCode\|Operation` |

## Xác thực chữ ký response từ Baokim

```typescript
function verifySignature(data: string, signature: string, baoKimPublicKeyPath: string): boolean {
  const publicKey = fs.readFileSync(baoKimPublicKeyPath, "utf8");
  const verify = crypto.createVerify("SHA1");
  verify.update(data);
  return verify.verify(publicKey, signature, "base64");
}
```

## Helper: Tạo RequestId

```typescript
function generateRequestId(partnerCode: string): string {
  const date = new Date().toISOString().slice(0, 10).replace(/-/g, "");
  const unique = Date.now().toString().slice(-6);
  return `${partnerCode}BK${date}${unique}`;
}

function getRequestTime(): string {
  return new Date().toISOString().replace("T", " ").slice(0, 19);
}
```
