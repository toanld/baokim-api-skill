---
name: baokim-api
description: >
  Hỗ trợ tích hợp Baokim Disbursement API (chi hộ) cho JavaScript/TypeScript/PHP.
  Dùng skill này bất cứ khi nào người dùng đề cập đến: Baokim, baokim API, chi hộ baokim,
  chuyển tiền ngân hàng tự động, disbursement payment, verify tài khoản ngân hàng,
  transfer money baokim, kiểm tra số dư baokim, tích hợp cổng thanh toán baokim,
  hoặc bất kỳ tác vụ nào liên quan đến Baokim API.
  LUÔN dùng skill này khi hỏi về baokim hoặc chiho.baokim.vn.
---

# Baokim Disbursement API Skill

> 📌 Tài liệu gốc: https://chiho.baokim.vn/docs/api

## Môi trường

| Môi trường | URL |
|-----------|-----|
| Test | `https://devtest.baokim.vn/Sandbox/FirmBanking` |
| Production | Được cung cấp khi ký hợp đồng |

## Authentication

Basic Authentication — header mỗi request:
```
Authorization: Basic <base64(username:password)>
```

## Chữ ký số (Digital Signature)

Baokim dùng **RSA-SHA1**. Chi tiết xem `auth/signature.md`.

## Các API

Khi người dùng hỏi về API cụ thể, đọc file tương ứng:

| Chức năng | Operation | File |
|-----------|-----------|------|
| Xác thực thông tin khách hàng | 9001 | `apis/verify-customer.md` |
| Chuyển tiền | 9002 | `apis/transfer.md` |
| Tra cứu giao dịch | 9003 | `apis/lookup-transaction.md` |
| Tra cứu số dư | 9004 | `apis/get-balance.md` |

## Cấu trúc request chung

Mọi request đều POST JSON với các trường bắt buộc:

```json
{
  "RequestId": "PARTNERBK20240101001",
  "RequestTime": "2024-01-01 10:00:00",
  "PartnerCode": "YOUR_PARTNER_CODE",
  "Operation": 9001,
  "Signature": "<RSA-SHA1 base64>"
}
```

- **RequestId** format: `{PartnerCode}BK{YYYYMMDD}{UniqueId}`
- **RequestTime** format: `YYYY-MM-DD HH:MM:SS`

## Response Code

| Code | Ý nghĩa |
|------|---------|
| 200 | Thành công |
| 99 | Transaction timeout |
| 11 | Thất bại |
| 101 | Lỗi xử lý từ Baokim |
| 102 | RequestId bị trùng |
| 103 | Sai chữ ký |
| 110 | PartnerCode không đúng |
| 111 | PartnerCode đã bị xóa |
| 112 | PartnerCode chưa kích hoạt |
| 113 | Thiếu Operation code |
| 114 | Operation code không đúng |
| 115 | Thiếu BankID |
| 116 | BankID không được hỗ trợ |
| 117 | Số tài khoản/thẻ phải 6-22 ký tự |
| 118 | Số tài khoản/thẻ không hợp lệ |
| 119 | Số tài khoản/thẻ không tồn tại |
| 120 | AccType không đúng |
| 121 | Thiếu ReferenceId |
| 122 | ReferenceId đã tồn tại |
| 123 | Không tìm thấy giao dịch |
| 124 | Thiếu số tiền |
| 125 | Số tiền không hợp lệ |
| 126 | Lỗi xử lý giữa Baokim và ngân hàng |
| 127 | Lỗi kết nối ngân hàng |
| 128 | Lỗi xử lý từ ngân hàng |
| 129 | Vượt hạn mức giải ngân hoặc hết thời gian bảo lãnh |
| 130 | Vượt hạn mức chuyển trong ngày |

## Xử lý Timeout

Xem chi tiết `apis/timeout.md`. Tóm tắt:
- **Case 1**: Baokim trả về code **99** → gọi API tra cứu giao dịch (9003)
- **Case 2**: Mất kết nối → gọi tra cứu, nếu vẫn lỗi → liên hệ tungpt@baokim.vn

## Danh sách ngân hàng

Xem đầy đủ `apis/bank-list.md` — hỗ trợ 61 ngân hàng.

Một số ngân hàng phổ biến:

| BankNo | Ngân hàng |
|--------|-----------|
| 970436 | Vietcombank |
| 970418 | BIDV |
| 970405 | Agribank |
| 970415 | Vietinbank |
| 970403 | Sacombank |
| 970407 | Techcombank |
| 970432 | VPBank |
| 970422 | MB Bank |
| 970423 | TPBank |
| 971025 | Ví MoMo |
