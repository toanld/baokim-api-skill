# Migration & Model

## Migration

File: `Modules/Baokim/Database/Migrations/xxxx_create_baokim_transactions_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('baokim_transactions', function (Blueprint $table) {
            $table->id();
            $table->string('reference_id', 50)->unique()->comment('Mã giao dịch do partner tạo');
            $table->string('request_id', 50)->comment('RequestId gửi Baokim');
            $table->string('transaction_id', 50)->nullable()->comment('Mã giao dịch phía Baokim');
            $table->string('bank_no', 20)->nullable();
            $table->string('acc_no', 22)->nullable();
            $table->string('acc_name', 100)->nullable();
            $table->tinyInteger('acc_type')->default(0)->comment('0=tài khoản, 1=thẻ');
            $table->bigInteger('request_amount')->default(0);
            $table->bigInteger('transfer_amount')->nullable();
            $table->bigInteger('after_balance')->nullable();
            $table->string('memo', 100)->nullable();
            $table->string('status', 20)->default('pending')
                ->comment('pending, success, failed, timeout');
            $table->integer('response_code')->nullable();
            $table->string('response_message', 200)->nullable();
            $table->json('raw_response')->nullable();
            $table->timestamp('transaction_time')->nullable();
            $table->nullableMorphs('transactionable'); // liên kết với model khác (order, payout...)
            $table->timestamps();

            $table->index('status');
            $table->index('transaction_id');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('baokim_transactions');
    }
};
```

## Model

File: `Modules/Baokim/Models/BaokimTransaction.php`

```php
<?php

namespace Modules\Baokim\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphTo;

class BaokimTransaction extends Model
{
    protected $table = 'baokim_transactions';

    protected $fillable = [
        'reference_id', 'request_id', 'transaction_id',
        'bank_no', 'acc_no', 'acc_name', 'acc_type',
        'request_amount', 'transfer_amount', 'after_balance',
        'memo', 'status', 'response_code', 'response_message',
        'raw_response', 'transaction_time',
        'transactionable_type', 'transactionable_id',
    ];

    protected $casts = [
        'raw_response'     => 'array',
        'transaction_time' => 'datetime',
        'acc_type'         => 'integer',
        'request_amount'   => 'integer',
        'transfer_amount'  => 'integer',
        'after_balance'    => 'integer',
        'response_code'    => 'integer',
    ];

    // Liên kết polymorphic với bất kỳ model nào (Order, Payout, Withdrawal...)
    public function transactionable(): MorphTo
    {
        return $this->morphTo();
    }

    // Scopes tiện ích
    public function scopeSuccess($query)
    {
        return $query->where('status', 'success');
    }

    public function scopePending($query)
    {
        return $query->where('status', 'pending');
    }

    public function scopeTimeout($query)
    {
        return $query->where('status', 'timeout');
    }

    public function isSuccess(): bool
    {
        return $this->status === 'success';
    }

    public function isPending(): bool
    {
        return $this->status === 'pending';
    }
}
```

## Dùng polymorphic để liên kết với model khác

```php
// Trong model Order hoặc Payout:
use Modules\Baokim\Models\BaokimTransaction;

public function baokimTransaction()
{
    return $this->morphOne(BaokimTransaction::class, 'transactionable');
}

// Lấy transaction của một order:
$order->baokimTransaction; // → BaokimTransaction
```
