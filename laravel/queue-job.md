# Queue Job — Xử lý chuyển tiền async

Dùng Queue để tránh timeout HTTP khi Baokim xử lý chậm.

## Tạo Job

```bash
php artisan make:job ProcessBaokimTransfer
```

File: `Modules/Baokim/Jobs/ProcessBaokimTransfer.php`

```php
<?php

namespace Modules\Baokim\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;
use Modules\Baokim\Models\BaokimTransaction;
use Modules\Baokim\Services\BaokimService;

class ProcessBaokimTransfer implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $timeout = 60;
    public int $backoff = 10; // giây chờ giữa các lần retry

    public function __construct(private array $data) {}

    public function handle(BaokimService $baokim): void
    {
        try {
            $result = $baokim->transfer(
                referenceId: $this->data['reference_id'],
                bankNo:      $this->data['bank_no'],
                accNo:       $this->data['acc_no'],
                accType:     $this->data['acc_type'],
                amount:      $this->data['amount'],
                memo:        $this->data['memo'] ?? '',
            );

            Log::info('BaokimTransfer done', [
                'reference_id' => $this->data['reference_id'],
                'response'     => $result,
            ]);

            // Nếu có event, fire tại đây
            // event(new BaokimTransferCompleted($result));

        } catch (\Exception $e) {
            Log::error('BaokimTransfer failed', [
                'reference_id' => $this->data['reference_id'],
                'error'        => $e->getMessage(),
            ]);

            // Nếu hết lần retry, đánh dấu failed
            if ($this->attempts() >= $this->tries) {
                BaokimTransaction::where('reference_id', $this->data['reference_id'])
                    ->update(['status' => 'failed']);
            }

            throw $e;
        }
    }

    // Kiểm tra duplicate trước khi xử lý
    public function middleware(): array
    {
        return [new \Illuminate\Queue\Middleware\WithoutOverlapping($this->data['reference_id'])];
    }
}
```

## Dispatch từ bất kỳ đâu

```php
use Modules\Baokim\Jobs\ProcessBaokimTransfer;

// Dispatch ngay
ProcessBaokimTransfer::dispatch([
    'reference_id' => 'ORDER_BK' . time(),
    'bank_no'      => '970436',
    'acc_no'       => '0021000382448',
    'acc_type'     => 0,
    'amount'       => 500000,
    'memo'         => 'Thanh toan don hang #456',
]);

// Dispatch sau 5 phút
ProcessBaokimTransfer::dispatch($data)->delay(now()->addMinutes(5));

// Dispatch vào queue cụ thể
ProcessBaokimTransfer::dispatch($data)->onQueue('payments');
```

## Cấu hình `.env` cho Queue

```env
QUEUE_CONNECTION=database   # hoặc redis

# Chạy worker
# php artisan queue:work --queue=payments,default --tries=3
```

## Chạy queue worker

```bash
# Development
php artisan queue:work --queue=payments,default

# Production (dùng Supervisor)
# /etc/supervisor/conf.d/baokim-worker.conf
[program:baokim-worker]
command=php /var/www/html/artisan queue:work --queue=payments,default --sleep=3 --tries=3
autostart=true
autorestart=true
```

## Kiểm tra trạng thái giao dịch sau khi dispatch

```php
use Modules\Baokim\Models\BaokimTransaction;

$tx = BaokimTransaction::where('reference_id', 'ORDER_BK12345')->first();

if ($tx?->isSuccess()) {
    // Đã chuyển thành công
} elseif ($tx?->isPending()) {
    // Đang xử lý
} else {
    // Thất bại hoặc chưa có
}
```
