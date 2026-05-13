# API: Xác thực thông tin khách hàng (Operation 9001)

Kiểm tra số tài khoản/thẻ ngân hàng có tồn tại và lấy tên chủ tài khoản.

## Request

**POST** `https://devtest.baokim.vn/Sandbox/FirmBanking`

```json
{
  "RequestId": "PARTNERBK2024010100001",
  "RequestTime": "2024-01-01 10:00:00",
  "PartnerCode": "PARTNER",
  "Operation": 9001,
  "BankNo": "970436",
  "AccNo": "0021000382448",
  "AccType": 0,
  "Signature": "<base64 RSA-SHA1>"
}
```

### Tham số

| # | Tên | Kiểu | Bắt buộc | Mô tả |
|---|-----|------|----------|-------|
| 1 | RequestId | String(50) | ✓ | Mã unique: `{PartnerCode}BK{YYYYMMDD}{UniqueId}` |
| 2 | RequestTime | String(19) | ✓ | `YYYY-MM-DD HH:MM:SS` |
| 3 | PartnerCode | String(20) | ✓ | Mã partner do Baokim cấp |
| 4 | Operation | Int | ✓ | Fix = `9001` |
| 5 | BankNo | String(20) | ✓ | Mã ngân hàng (xem bank-list.md) |
| 6 | AccNo | String(22) | ✓ | Số tài khoản hoặc số thẻ |
| 7 | AccType | Int | ✓ | `0` = số tài khoản, `1` = số thẻ |
| 8 | Signature | String(500) | ✓ | RSA-SHA1 của: `RequestId\|RequestTime\|PartnerCode\|Operation\|BankNo\|AccNo\|AccType` |

## Response thành công

```json
{
  "ResponseCode": 200,
  "ResponseMessage": "Successful",
  "RequestId": "PARTNERBK2024010100001",
  "BankNo": "970436",
  "AccNo": "0021000382448",
  "AccType": 0,
  "AccName": "NGUYEN VAN A",
  "Signature": "<base64 RSA-SHA1 từ Baokim>"
}
```

### Tham số response

| # | Tên | Mô tả |
|---|-----|-------|
| 1 | ResponseCode | Mã kết quả (200 = thành công) |
| 2 | ResponseMessage | Mô tả kết quả |
| 3 | AccName | Tên chủ tài khoản (nếu hợp lệ) |
| 4 | Signature | Baokim ký: `ResponseCode\|ResponseMessage\|RequestId\|BankNo\|AccNo\|AccType\|AccName` |

## Code mẫu (TypeScript)

```typescript
import crypto from "crypto";
import fs from "fs";

async function verifyCustomer(bankNo: string, accNo: string, accType: 0 | 1) {
  const requestId = `PARTNERBK${new Date().toISOString().slice(0,10).replace(/-/g,"")}${Date.now().toString().slice(-5)}`;
  const requestTime = new Date().toISOString().replace("T"," ").slice(0,19);
  const partnerCode = process.env.BAOKIM_PARTNER_CODE!;
  const operation = 9001;

  const dataToSign = [requestId, requestTime, partnerCode, operation, bankNo, accNo, accType].join("|");
  const privateKey = fs.readFileSync(process.env.BAOKIM_PRIVATE_KEY_PATH!, "utf8");
  const sign = crypto.createSign("SHA1");
  sign.update(dataToSign);
  const signature = sign.sign(privateKey, "base64");

  const response = await fetch("https://devtest.baokim.vn/Sandbox/FirmBanking", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": `Basic ${process.env.BAOKIM_BASIC_AUTH}`,
    },
    body: JSON.stringify({ RequestId: requestId, RequestTime: requestTime, PartnerCode: partnerCode, Operation: operation, BankNo: bankNo, AccNo: accNo, AccType: accType, Signature: signature }),
  });

  return response.json();
}
```
