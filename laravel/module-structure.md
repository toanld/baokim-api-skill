# Laravel Module Baokim — Cấu trúc & Setup

Dùng [nwidart/laravel-modules](https://nwidart.com/laravel-modules) để tổ chức code theo module.

## Cài đặt

```bash
composer require nwidart/laravel-modules
php artisan module:make Baokim
```

## Cấu trúc module

```
Modules/Baokim/
├── Config/
│   └── baokim.php              ← cấu hình partner code, key, URL
├── Database/
│   └── Migrations/
│       └── create_baokim_transactions_table.php
├── Http/
│   ├── Controllers/
│   │   └── BaokimController.php
│   └── Routes/
│       └── api.php
├── Jobs/
│   └── ProcessBaokimTransfer.php   ← xử lý chuyển tiền async
├── Models/
│   └── BaokimTransaction.php
├── Services/
│   └── BaokimService.php           ← service chính
├── Providers/
│   └── BaokimServiceProvider.php
└── module.json
```

## Config `Modules/Baokim/Config/baokim.php`

```php
<?php

return [
    'partner_code'    => env('BAOKIM_PARTNER_CODE', ''),
    'basic_auth'      => env('BAOKIM_BASIC_AUTH', ''),   // base64(user:pass)
    'private_key_path'=> env('BAOKIM_PRIVATE_KEY_PATH', storage_path('keys/baokim_private.pem')),
    'public_key_path' => env('BAOKIM_PUBLIC_KEY_PATH', storage_path('keys/baokim_public.pem')),
    'url'             => env('BAOKIM_URL', 'https://devtest.baokim.vn/Sandbox/FirmBanking'),
    'timeout'         => env('BAOKIM_TIMEOUT', 30),
];
```

## `.env` cần thêm

```env
BAOKIM_PARTNER_CODE=YOUR_PARTNER_CODE
BAOKIM_BASIC_AUTH=base64encodedstring
BAOKIM_PRIVATE_KEY_PATH=/path/to/storage/keys/baokim_private.pem
BAOKIM_PUBLIC_KEY_PATH=/path/to/storage/keys/baokim_public.pem
BAOKIM_URL=https://devtest.baokim.vn/Sandbox/FirmBanking
```

## ServiceProvider `Modules/Baokim/Providers/BaokimServiceProvider.php`

```php
<?php

namespace Modules\Baokim\Providers;

use Illuminate\Support\ServiceProvider;
use Modules\Baokim\Services\BaokimService;

class BaokimServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->singleton(BaokimService::class, function ($app) {
            return new BaokimService(config('baokim'));
        });

        $this->mergeConfigFrom(__DIR__ . '/../Config/baokim.php', 'baokim');
    }

    public function boot(): void
    {
        $this->loadMigrationsFrom(__DIR__ . '/../Database/Migrations');
        $this->loadRoutesFrom(__DIR__ . '/../Http/Routes/api.php');
    }
}
```

## Bind Facade (tuỳ chọn)

Dùng `app(BaokimService::class)` hoặc inject qua constructor là đủ — không cần Facade.

```php
// Inject vào Controller
public function __construct(private BaokimService $baokim) {}

// Hoặc resolve từ container
$baokim = app(BaokimService::class);
```
