# 🚀 RevenueCat Webhook Integration Guide

---

## Table of Contents
- [What is a Webhook in RevenueCat?](#1-what-is-a-webhook-in-revenuecat)
- [RevenueCat Webhook Setup](#2-revenuecat-webhook-setup)
- [Common Webhook Events](#3-common-webhook-events)
- [Laravel Example: Backend Webhook Endpoint](#4-laravel-example-backend-webhook-endpoint)
  - [Step 1: Define Routes](#step-1-define-routes)
  - [Step 2: Create Controller](#step-2-create-controller)
  - [Step 3: Controller Code to Handle Webhook](#step-3-controller-code-to-handle-webhook)
- [Update `services.php` and `.env`](#5-update-servicesphp-and-env)
- [API Endpoint to Check Subscription Status](#6-api-endpoint-to-check-subscription-status)
- [Sample JSON Output](#7-sample-json-output)

---

## 1. What is a Webhook in RevenueCat?

A **webhook** is a way for RevenueCat to notify your server **in real-time** when a subscription event occurs, such as:

- A purchase
- A cancellation
- A renewal
- Other subscription lifecycle events

This allows your backend to react instantly and update user subscription status accordingly.

---

## 2. RevenueCat Webhook Setup

1. Login to your **RevenueCat Dashboard**  
2. Navigate to **Project Settings → Webhooks**  
3. Click **Add Webhook**  
4. Enter your server URL where you want RevenueCat to POST subscription event data  
5. Optionally, set a **secret token** to verify webhook authenticity  

---

## 3. Common Webhook Events

| Event Type        | Description                           |
| ----------------- | ----------------------------------- |
| `INITIAL_PURCHASE` | User made their first subscription  |
| `RENEWAL`         | Subscription was renewed             |
| `CANCELLATION`    | User cancelled the subscription     |
| `UNCANCELLATION`  | User reactivated the subscription   |
| `PRODUCT_CHANGE`  | User changed subscription product   |
| `EXPIRATION`      | Subscription expired                 |

---

## 4. Laravel Example: Backend Webhook Endpoint

### Step 1: Define Routes

Add these routes to your `routes/api.php` or `routes/web.php`:

```php
Route::any('/revenuecat/webhook', [RevenueCatWebhookController::class, 'handleWebhook']);
Route::get('/subscription/check/{userId}', [RevenueCatWebhookController::class, 'checkSubscription']);
```
---
Step 2: Create Controller
---
Run artisan command to create the controller:
```php
php artisan make:controller RevenueCatWebhookController
```
---
Step 3: Controller Code to Handle Webhook
---
Replace the content of app/Http/Controllers/RevenueCatWebhookController.php with:
```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use App\Models\User;

class RevenueCatWebhookController extends Controller
{
    public function handleWebhook(Request $request)
    {
        try {
            // Log webhook payload for debugging
            Log::info('RevenueCat Webhook Received:', $request->all());

            // Get Authorization header
            $authHeader = $request->header('Authorization');
            $expectedSecret = config('services.revenuecat.webhook_secret');

            // Remove 'Bearer ' prefix if present
            if ($authHeader && str_starts_with($authHeader, 'Bearer ')) {
                $authHeader = substr($authHeader, 7);
            }

            // Check authorization header
            if (!$authHeader || !hash_equals($expectedSecret, $authHeader)) {
                Log::error('Unauthorized RevenueCat webhook access attempt.');
                return response()->json(['error' => 'Unauthorized'], 401);
            }

            // Extract event type and user ID
            $eventType = $request->input('event');
            $appUserId = $request->input('app_user_id');

            if (!$eventType || !$appUserId) {
                Log::warning('Invalid webhook payload: missing event or app_user_id.');
                return response()->json(['error' => 'Invalid payload'], 400);
            }

            // Extract numeric user ID from app_user_id string
            $userId = preg_replace('/\D/', '', $appUserId);
            $user = User::find($userId);

            if (!$user) {
                Log::warning("Webhook received for non-existent user ID: {$userId}");
                return response()->json(['error' => 'User not found'], 404);
            }

            // Handle webhook events
            match ($eventType) {
                'INITIAL_PURCHASE' => $this->handleInitialPurchase($user),
                'RENEWAL'          => $this->handleRenewal($user),
                'CANCELLATION'     => $this->handleCancellation($user),
                'EXPIRATION'       => $this->handleExpiration($user),
                default            => Log::info("Unhandled RevenueCat event type: {$eventType}"),
            };

            // Respond success
            return response()->json(['status' => 'ok']);
        } catch (\Throwable $e) {
            Log::error('RevenueCat webhook processing failed: ' . $e->getMessage());
            return response()->json(['error' => 'Server error'], 500);
        }
    }

    private function handleInitialPurchase(User $user): void
    {
        if ($user->has_trial) {
            $user->has_trial = false;
        }
        $user->is_subscribed = true;
        $user->save();

        Log::info("INITIAL_PURCHASE: User ID {$user->id} trial ended, subscription started.");
    }

    private function handleRenewal(User $user): void
    {
        $user->is_subscribed = true;
        $user->save();

        Log::info("RENEWAL: Subscription renewed for user ID {$user->id}.");
    }

    private function handleCancellation(User $user): void
    {
        Log::info("CANCELLATION: User ID {$user->id} cancelled subscription; access remains until expiration.");
    }

    private function handleExpiration(User $user): void
    {
        $user->is_subscribed = false;
        $user->save();

        Log::info("EXPIRATION: Subscription expired for user ID {$user->id}.");
    }

    // API endpoint to check subscription status
    public function checkSubscription(Request $request, $userId)
    {
        $user = User::find($userId);

        if (!$user) {
            return response()->json(['error' => 'User not found'], 404);
        }

        return response()->json([
            'user_id' => $user->id,
            'has_trial' => (bool) $user->has_trial,
            'is_subscribed' => (bool) $user->is_subscribed,
        ]);
    }
}
```
5. Update services.php and .env

Add your RevenueCat webhook secret configuration to connect your Laravel app securely.

Add RevenueCat webhook secret configuration: ` config/services.php `
  
```
'revenuecat' => [
    'webhook_secret' => env('REVENUECAT_WEBHOOK_SECRET'),
],
```
Adding Env File Credential ` .env `
  
```
REVENUECAT_WEBHOOK_SECRET=your_webhook_secret_here
```
</details>

💡 Tip:
Replace your_webhook_secret_here with the secret token you set in the RevenueCat webhook settings.
Keep this secret safe and never commit your .env file to version control!

6. API Endpoint to Check Subscription Status
Use this API route to retrieve a user’s subscription status by their user ID:

```php
GET /subscription/check/{userId}
```
### Example Response
```php
{
  "user_id": 12,
  "has_trial": false,
  "is_subscribed": true
}
```
This endpoint returns the current subscription and trial status, so you can display or process it as needed.
