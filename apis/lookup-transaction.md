# API: Tra cứu giao dịch (Operation 9003)

Tra cứu trạng thái và thông tin một giao dịch đã thực hiện.

## Request

**POST** `https://devtest.baokim.vn/Sandbox/FirmBanking`

```json
{
  "RequestId": "PARTNERBK2024010100003",
  "RequestTime": "2024-01-01 10:00:00",
  "PartnerCode": "PARTNER",
  "Operation": 9003,
  "ReferenceId": "5CBCAB920C63CED5E0540010E099E090",
  "Signature": "<base64 RSA-SHA1>"
}
```

### Tham số

| # | Tên | Kiểu | Bắt buộc | Mô tả |
|---|-----|------|----------|-------|
| 1 | RequestId | String(50) | ✓ | Mã unique mới cho request này |
| 2 | RequestTime | String(19) | ✓ | `YYYY-MM-DD HH:MM:SS` |
| 3 | PartnerCode | String(20) | ✓ | Mã partner |
| 4 | Operation | Int | ✓ | Fix = `9003` |
| 5 | ReferenceId | String(50) | ✓ | ReferenceId của giao dịch cần tra cứu |
| 6 | Signature | String(500) | ✓ | RSA-SHA1 của: `RequestId\|RequestTime\|PartnerCode\|Operation\|ReferenceId` |

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
  "Signature": "<base64 RSA-SHA1 từ Baokim>"
}
```

## Khi nào cần gọi API này?

- Nhận ResponseCode = **99** (timeout) từ API chuyển tiền
- Mất kết nối khi gọi API chuyển tiền, không biết giao dịch thành công hay chưa
- Cần đối soát giao dịch
