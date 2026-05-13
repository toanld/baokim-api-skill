# Xử lý Timeout

## Case 1: Baokim trả về ResponseCode = 99

Baokim chủ động trả về timeout khi không xử lý kịp trong thời gian quy định.

**Cách xử lý:**
```typescript
const result = await transferMoney(...);

if (result.ResponseCode === 99) {
  // Chờ vài giây rồi tra cứu trạng thái
  await sleep(3000);
  const status = await lookupTransaction(referenceId);
  // Xử lý theo ResponseCode của lookupTransaction
}
```

## Case 2: Mất kết nối, không nhận được response

Không biết giao dịch có được Baokim tiếp nhận hay chưa.

**Cách xử lý:**

```typescript
async function transferWithRetry(params: TransferParams, maxRetries = 3) {
  let result;
  
  try {
    result = await transferMoney(params);
  } catch (networkError) {
    // Mất kết nối — tra cứu trước khi retry
    const lookup = await lookupTransaction(params.referenceId);
    
    if (lookup.ResponseCode === 200) {
      return lookup; // Giao dịch đã thành công
    }
    
    if (lookup.ResponseCode === 123) {
      // Giao dịch không tồn tại → có thể retry với ReferenceId mới
      throw new Error("Transaction not found, safe to retry with new ReferenceId");
    }
    
    // Vẫn không rõ → liên hệ Baokim: tungpt@baokim.vn
    throw new Error("Cannot determine transaction status, contact Baokim support");
  }
  
  return result;
}
```

## Lưu ý

- **Không dùng lại ReferenceId** khi retry — nếu giao dịch đã được nhận sẽ bị lỗi 122 (duplicate)
- Nếu tra cứu nhiều lần vẫn timeout → email `tungpt@baokim.vn` kèm ReferenceId và RequestId để hỗ trợ thủ công
