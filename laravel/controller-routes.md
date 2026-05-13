# Controller & Routes

## Routes

File: `Modules/Baokim/Http/Routes/api.php`

```php
<?php

use Illuminate\Support\Facades\Route;
use Modules\Baokim\Http\Controllers\BaokimController;

Route::prefix('api/baokim')->middleware(['api', 'auth:sanctum'])->group(function () {
    Route::post('verify',   [BaokimController::class, 'verify']);
    Route::post('transfer', [BaokimController::class, 'transfer']);
    Route::get('lookup/{referenceId}', [BaokimController::class, 'lookup']);
    Route::get('balance',   [BaokimController::class, 'balance']);
});
```

## Controller

File: `Modules/Baokim/Http/Controllers/BaokimController.php`

```php
<?php

namespace Modules\Baokim\Http\Controllers;

use App\Http\Controllers\Controller;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Modules\Baokim\Services\BaokimService;
use Modules\Baokim\Jobs\ProcessBaokimTransfer;

class BaokimController extends Controller
{
    public function __construct(private BaokimService $baokim) {}

    // POST /api/baokim/verify
    public function verify(Request $request): JsonResponse
    {
        $request->validate([
            'bank_no'  => 'required|string',
            'acc_no'   => 'required|string',
            'acc_type' => 'sometimes|integer|in:0,1',
        ]);

        $result = $this->baokim->verifyCustomer(
            $request->bank_no,
            $request->acc_no,
            $request->acc_type ?? 0
        );

        return response()->json($result);
    }

    // POST /api/baokim/transfer
    // Dispatch vào Queue để xử lý async (khuyến nghị)
    public function transfer(Request $request): JsonResponse
    {
        $request->validate([
            'reference_id' => 'required|string|max:50|unique:baokim_transactions,reference_id',
            'bank_no'      => 'required|string',
            'acc_no'       => 'required|string',
            'acc_type'     => 'required|integer|in:0,1',
            'amount'       => 'required|integer|min:1000',
            'memo'         => 'sometimes|string|max:100',
        ]);

        // Dispatch job vào queue
        ProcessBaokimTransfer::dispatch($request->validated());

        return response()->json([
            'message'      => 'Transfer queued successfully',
            'reference_id' => $request->reference_id,
        ]);
    }

    // Chuyển tiền đồng bộ (dùng khi cần kết quả ngay)
    public function transferSync(Request $request): JsonResponse
    {
        $request->validate([
            'reference_id' => 'required|string|max:50',
            'bank_no'      => 'required|string',
            'acc_no'       => 'required|string',
            'acc_type'     => 'required|integer|in:0,1',
            'amount'       => 'required|integer|min:1000',
            'memo'         => 'sometimes|string|max:100',
        ]);

        $result = $this->baokim->transfer(
            $request->reference_id,
            $request->bank_no,
            $request->acc_no,
            $request->acc_type,
            $request->amount,
            $request->memo ?? ''
        );

        return response()->json($result);
    }

    // GET /api/baokim/lookup/{referenceId}
    public function lookup(string $referenceId): JsonResponse
    {
        $result = $this->baokim->lookupTransaction($referenceId);
        return response()->json($result);
    }

    // GET /api/baokim/balance
    public function balance(): JsonResponse
    {
        $result = $this->baokim->getBalance();
        return response()->json($result);
    }
}
```
