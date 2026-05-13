# API: Tra cứu số dư (Operation 9004)

Kiểm tra số dư khả dụng và số tiền đang giữ của partner.

## Request

**POST** `https://devtest.baokim.vn/Sandbox/FirmBanking`

```json
{
  "RequestId": "PARTNERBK2024010100004",
  "RequestTime": "2024-01-01 10:00:00",
  "PartnerCode": "PARTNER",
  "Operation": 9004,
  "Signature": "<base64 RSA-SHA1>"
}
```

### Tham số

| # | Tên | Kiểu | Bắt buộc | Mô tả |
|---|-----|------|----------|-------|
| 1 | RequestId | String(50) | ✓ | Mã unique |
| 2 | RequestTime | String(19) | ✓ | `YYYY-MM-DD HH:MM:SS` |
| 3 | PartnerCode | String(20) | ✓ | Mã partner |
| 4 | Operation | Int | ✓ | Fix = `9004` |
| 5 | Signature | String(500) | ✓ | RSA-SHA1 của: `RequestId\|RequestTime\|PartnerCode\|Operation` |

## Response

```json
{
  "ResponseCode": 200,
  "ResponseMessage": "Successful",
  "RequestId": "PARTNERBK2024010100004",
  "PartnerCode": "PARTNER",
  "Available": 150000000,
  "Holding": 2500000,
  "Signature": "<base64 RSA-SHA1 từ Baokim>"
}
```

### Tham số response

| Tên | Mô tả |
|-----|-------|
| Available | Số dư khả dụng (VNĐ) |
| Holding | Số tiền đang tạm giữ/pending (VNĐ) |
| Signature | Baokim ký: `ResponseCode\|ResponseMessage\|RequestId\|PartnerCode\|Available\|Holding` |
