# BaokimService — Service chính

File: `Modules/Baokim/Services/BaokimService.php`

```php
<?php

namespace Modules\Baokim\Services;

use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Log;
use Modules\Baokim\Models\BaokimTransaction;

class BaokimService
{
    private string $partnerCode;
    private string $basicAuth;
    private string $privateKeyPath;
    private string $publicKeyPath;
    private string $url;
    private int $timeout;

    public function __construct(array $config)
    {
        $this->partnerCode    = $config['partner_code'];
        $this->basicAuth      = $config['basic_auth'];
        $this->privateKeyPath = $config['private_key_path'];
        $this->publicKeyPath  = $config['public_key_path'];
        $this->url            = $config['url'];
        $this->timeout        = $config['timeout'] ?? 30;
    }

    // -------------------------------------------------------------------------
    // Operation 9001 — Xác thực thông tin khách hàng
    // -------------------------------------------------------------------------
    public function verifyCustomer(string $bankNo, string $accNo, int $accType = 0): array
    {
        $requestId   = $this->generateRequestId();
        $requestTime = $this->requestTime();

        $dataToSign = implode('|', [$requestId, $requestTime, $this->partnerCode, 9001, $bankNo, $accNo, $accType]);

        return $this->post([
            'RequestId'   => $requestId,
            'RequestTime' => $requestTime,
            'PartnerCode' => $this->partnerCode,
            'Operation'   => 9001,
            'BankNo'      => $bankNo,
            'AccNo'       => $accNo,
            'AccType'     => $accType,
            'Signature'   => $this->sign($dataToSign),
        ]);
    }

    // -------------------------------------------------------------------------
    // Operation 9002 — Chuyển tiền
    // -------------------------------------------------------------------------
    public function transfer(
        string $referenceId,
        string $bankNo,
        string $accNo,
        int    $accType,
        int    $amount,
        string $memo = ''
    ): array {
        $requestId   = $this->generateRequestId();
        $requestTime = $this->requestTime();

        $dataToSign = implode('|', [
            $requestId, $requestTime, $this->partnerCode,
            9002, $referenceId, $bankNo, $accNo, $accType, $amount, $memo,
        ]);

        $payload = [
            'RequestId'     => $requestId,
            'RequestTime'   => $requestTime,
            'PartnerCode'   => $this->partnerCode,
            'Operation'     => 9002,
            'ReferenceId'   => $referenceId,
            'BankNo'        => $bankNo,
            'AccNo'         => $accNo,
            'AccType'       => $accType,
            'RequestAmount' => $amount,
            'Memo'          => $memo,
            'Signature'     => $this->sign($dataToSign),
        ];

        $response = $this->post($payload);

        // Lưu transaction vào DB
        $this->saveTransaction($referenceId, $requestId, $payload, $response);

        // Nếu timeout → tra cứu lại
        if (($response['ResponseCode'] ?? null) === 99) {
            sleep(3);
            return $this->lookupTransaction($referenceId);
        }

        return $response;
    }

    // -------------------------------------------------------------------------
    // Operation 9003 — Tra cứu giao dịch
    // -------------------------------------------------------------------------
    public function lookupTransaction(string $referenceId): array
    {
        $requestId   = $this->generateRequestId();
        $requestTime = $this->requestTime();

        $dataToSign = implode('|', [$requestId, $requestTime, $this->partnerCode, 9003, $referenceId]);

        return $this->post([
            'RequestId'   => $requestId,
            'RequestTime' => $requestTime,
            'PartnerCode' => $this->partnerCode,
            'Operation'   => 9003,
            'ReferenceId' => $referenceId,
            'Signature'   => $this->sign($dataToSign),
        ]);
    }

    // -------------------------------------------------------------------------
    // Operation 9004 — Tra cứu số dư
    // -------------------------------------------------------------------------
    public function getBalance(): array
    {
        $requestId   = $this->generateRequestId();
        $requestTime = $this->requestTime();

        $dataToSign = implode('|', [$requestId, $requestTime, $this->partnerCode, 9004]);

        return $this->post([
            'RequestId'   => $requestId,
            'RequestTime' => $requestTime,
            'PartnerCode' => $this->partnerCode,
            'Operation'   => 9004,
            'Signature'   => $this->sign($dataToSign),
        ]);
    }

    // -------------------------------------------------------------------------
    // Xác thực chữ ký response từ Baokim
    // -------------------------------------------------------------------------
    public function verifyResponseSignature(string $data, string $signature): bool
    {
        $publicKey = file_get_contents($this->publicKeyPath);
        return (bool) openssl_verify($data, base64_decode($signature), $publicKey, OPENSSL_ALGO_SHA1);
    }

    // -------------------------------------------------------------------------
    // Helpers
    // -------------------------------------------------------------------------
    private function post(array $payload): array
    {
        try {
            $response = Http::withHeaders([
                'Authorization' => 'Basic ' . $this->basicAuth,
                'Content-Type'  => 'application/json',
            ])
            ->timeout($this->timeout)
            ->post($this->url, $payload);

            return $response->json() ?? [];
        } catch (\Exception $e) {
            Log::error('BaokimService error', ['error' => $e->getMessage(), 'payload' => $payload]);
            throw $e;
        }
    }

    private function sign(string $data): string
    {
        $privateKey = file_get_contents($this->privateKeyPath);
        openssl_sign($data, $signature, $privateKey, OPENSSL_ALGO_SHA1);
        return base64_encode($signature);
    }

    private function generateRequestId(): string
    {
        return $this->partnerCode . 'BK' . now()->format('Ymd') . substr((string) time(), -5);
    }

    private function requestTime(): string
    {
        return now()->format('Y-m-d H:i:s');
    }

    private function saveTransaction(string $referenceId, string $requestId, array $payload, array $response): void
    {
        BaokimTransaction::updateOrCreate(
            ['reference_id' => $referenceId],
            [
                'request_id'      => $requestId,
                'bank_no'         => $payload['BankNo'] ?? null,
                'acc_no'          => $payload['AccNo'] ?? null,
                'acc_type'        => $payload['AccType'] ?? null,
                'request_amount'  => $payload['RequestAmount'] ?? null,
                'memo'            => $payload['Memo'] ?? null,
                'response_code'   => $response['ResponseCode'] ?? null,
                'transaction_id'  => $response['TransactionId'] ?? null,
                'transfer_amount' => $response['TransferAmount'] ?? null,
                'status'          => $this->mapStatus($response['ResponseCode'] ?? null),
                'raw_response'    => json_encode($response),
            ]
        );
    }

    private function mapStatus(?int $code): string
    {
        return match ($code) {
            200     => 'success',
            99      => 'timeout',
            11      => 'failed',
            default => 'pending',
        };
    }
}
```
