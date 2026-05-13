# API: Chuyển tiền (Operation 9002)

Chuyển tiền tới tài khoản/thẻ ngân hàng của khách hàng.

## Request

**POST** `https://devtest.baokim.vn/Sandbox/FirmBanking`

```json
{
  "RequestId": "PARTNERBK2024010100002",
  "RequestTime": "2024-01-01 10:00:00",
  "PartnerCode": "PARTNER",
  "Operation": 9002,
  "ReferenceId": "5CBCAB920C63CED5E0540010E099E090",
  "BankNo": "970436",
  "AccNo": "0021000382448",
  "AccType": 0,
  "RequestAmount": 1000000,
  "Memo": "Thanh toan don hang #123",
  "Signature": "<base64 RSA-SHA1>"
}
```

### Tham số

| # | Tên | Kiểu | Bắt buộc | Mô tả |
|---|-----|------|----------|-------|
| 1 | RequestId | String(50) | ✓ | Mã unique: `{PartnerCode}BK{YYYYMMDD}{UniqueId}` |
| 2 | RequestTime | String(19) | ✓ | `YYYY-MM-DD HH:MM:SS` |
| 3 | PartnerCode | String(20) | ✓ | Mã partner do Baokim cấp |
| 4 | Operation | Int | ✓ | Fix = `9002` |
| 5 | ReferenceId | String(50) | ✓ | Mã giao dịch do partner tự tạo (unique) |
| 6 | BankNo | String(20) | ✓ | Mã ngân hàng (xem bank-list.md) |
| 7 | AccNo | String(22) | ✓ | Số tài khoản hoặc số thẻ |
| 8 | AccType | Int | ✓ | `0` = số tài khoản, `1` = số thẻ |
| 9 | RequestAmount | Int | ✓ | Số tiền cần chuyển (VNĐ) |
| 10 | Memo | String(100) | - | Nội dung chuyển khoản |
| 11 | Signature | String(500) | ✓ | RSA-SHA1 của: `RequestId\|RequestTime\|PartnerCode\|Operation\|ReferenceId\|BankNo\|AccNo\|AccType\|RequestAmount\|Memo` |

## Response thành công

```json
{
  "ResponseCode": 200,
  "ResponseMessage": "Successful",
  "ReferenceId": "5CBCAB920C63CED5E0540010E099E090",
  "TransactionId": "BK5CF8D68AE3CF8JY",
  "TransactionTime": "2024-01-01",
  "BankNo": "970436",
  "AccNo": "0021000382448",
  "AccName": "NGUYEN VAN A",
  "AccType": 0,
  "RequestAmount": 1000000,
  "TransferAmount": 1000000,
  "AffterBalance": 100000000,
  "AfterDisbursementDay": 100000000,
  "Signature": "<base64 RSA-SHA1 từ Baokim>"
}
```

### Tham số response

| # | Tên | Mô tả |
|---|-----|-------|
| 1 | ResponseCode | Mã kết quả |
| 2 | TransactionId | Mã giao dịch phía Baokim |
| 3 | TransactionTime | Thời gian hoàn thành (YYYY-MM-DD) |
| 4 | TransferAmount | Số tiền thực tế chuyển đến (có thể nhỏ hơn nếu có phí) |
| 5 | AffterBalance | Số dư hiện tại của partner sau giao dịch |
| 6 | AfterDisbursementDay | Hạn mức giải ngân còn lại trong ngày |
| 7 | Signature | Baokim ký: `ResponseCode\|ResponseMessage\|ReferenceId\|TransactionId\|TransactionTime\|BankNo\|AccNo\|AccName\|AccType\|RequestAmount\|TransferAmount` |

## Lưu ý quan trọng

- **ReferenceId phải unique** — nếu trùng sẽ nhận lỗi code 122
- Nếu nhận ResponseCode = **99** (timeout) → gọi API tra cứu (9003) để kiểm tra trạng thái
- Luôn lưu `TransactionId` phía Baokim để đối soát

## Code mẫu (TypeScript)

```typescript
async function transferMoney(params: {
  referenceId: string;
  bankNo: string;
  accNo: string;
  accType: 0 | 1;
  amount: number;
  memo?: string;
}) {
  const { referenceId, bankNo, accNo, accType, amount, memo = "" } = params;
  const requestId = `PARTNERBK${new Date().toISOString().slice(0,10).replace(/-/g,"")}${Date.now().toString().slice(-5)}`;
  const requestTime = new Date().toISOString().replace("T"," ").slice(0,19);
  const partnerCode = process.env.BAOKIM_PARTNER_CODE!;
  const operation = 9002;

  const dataToSign = [requestId, requestTime, partnerCode, operation, referenceId, bankNo, accNo, accType, amount, memo].join("|");
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
    body: JSON.stringify({ RequestId: requestId, RequestTime: requestTime, PartnerCode: partnerCode, Operation: operation, ReferenceId: referenceId, BankNo: bankNo, AccNo: accNo, AccType: accType, RequestAmount: amount, Memo: memo, Signature: signature }),
  });

  return response.json();
}
```
