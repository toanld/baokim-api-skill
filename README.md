# baokim-api

> Agent skill for integrating [Baokim Disbursement API](https://chiho.baokim.vn/docs/api) — automated bank transfers for JavaScript/TypeScript.

## Install skill

```bash
npx skills add toanld/baokim-skill
```

## What this skill does

Gives your AI agent full knowledge of the **Baokim chi hộ (Disbursement) API**, covering:

| Module | Coverage |
|--------|----------|
| 🔐 Auth | Basic Auth, RSA-SHA1 digital signature |
| ✅ Verify | Check bank account/card existence, get account name |
| 💸 Transfer | Send money to any bank account |
| 🔍 Lookup | Check transaction status by ReferenceId |
| 💰 Balance | Query available and holding balance |
| ⏱️ Timeout | Handling timeout and network failure scenarios |
| 🏦 Banks | 61 supported banks including MoMo, Viettel Money |

## Quick example

```typescript
import crypto from "crypto";
import fs from "fs";

// Sign data with RSA-SHA1
function sign(data: string): string {
  const key = fs.readFileSync("./private.pem", "utf8");
  return crypto.createSign("SHA1").update(data).sign(key, "base64");
}

// Verify bank account
const requestId = `PARTNERBK${new Date().toISOString().slice(0,10).replace(/-/g,"")}${Date.now().toString().slice(-5)}`;
const requestTime = new Date().toISOString().replace("T"," ").slice(0,19);
const dataToSign = [requestId, requestTime, "PARTNER", 9001, "970436", "0021000382448", 0].join("|");

const res = await fetch("https://devtest.baokim.vn/Sandbox/FirmBanking", {
  method: "POST",
  headers: { "Content-Type": "application/json", "Authorization": "Basic <token>" },
  body: JSON.stringify({
    RequestId: requestId, RequestTime: requestTime,
    PartnerCode: "PARTNER", Operation: 9001,
    BankNo: "970436", AccNo: "0021000382448", AccType: 0,
    Signature: sign(dataToSign)
  })
});
```

## Skill structure

```
baokim-api/
├── SKILL.md                       # Entry point
├── auth/signature.md              # RSA-SHA1 setup & signing guide
└── apis/
    ├── verify-customer.md         # Operation 9001
    ├── transfer.md                # Operation 9002
    ├── lookup-transaction.md      # Operation 9003
    ├── get-balance.md             # Operation 9004
    ├── timeout.md                 # Timeout handling
    └── bank-list.md               # 61 supported banks
```

## Environments

| | URL |
|--|-----|
| Test | `https://devtest.baokim.vn/Sandbox/FirmBanking` |
| Production | Provided after contract signing |

## Links

- Docs: https://chiho.baokim.vn/docs/api
- Contact: tungpt@baokim.vn
