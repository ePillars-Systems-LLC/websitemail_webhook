# Laravel Google Send API Integration Guide

This comprehensive guide covers how to integrate Google Send APIs with **PHP Laravel**. It details both **Method 1: Gmail API (Email Sending)** and **Method 2: Google Chat Spaces API (Webhook Alerts)**.

---

## Prerequisites & Setup

### 1. Google Cloud Console Configuration
Before writing code, configure your credentials on the [Google Cloud Console](https://console.cloud.google.com/):

1. **Create a Project:** Select or create a project.
2. **Enable APIs:** Enable the **Gmail API** and/or the **Google Chat API** depending on your needs.
3. **Configure Credentials:**
   * **Gmail API:** Set up your OAuth consent screen and generate **OAuth 2.0 Client IDs**. Download the `credentials.json` file.
   * **Google Chat API:** Set up a Space Webhook directly inside your Google Chat application configuration.

### 2. Base Laravel Package Installation
Install the official Google API Client SDK for PHP via Composer:
```bash
composer require google/apiclient
```

Add your credentials directly to your Laravel `.env` file:
```env
# Gmail API Credentials
GOOGLE_CLIENT_ID="your-client-id.apps.googleusercontent.com"
GOOGLE_CLIENT_SECRET="your-client-secret"
GOOGLE_ACCESS_TOKEN="your-stored-oauth-or-service-account-token"

# Google Chat API Configuration
GOOGLE_CHAT_WEBHOOK_URL="https://chat.googleapis.com/v1/spaces/AAAA.../messages?key=XXX&token=YYY"
```

---

## Method 1: Sending Emails via Gmail API

The Gmail API requires your raw email strings to conform strictly to RFC 2822 formatting and then be safely encrypted using base64url encoding.

### Implementation Example

Create a controller (`app/Http/Controllers/GmailController.php`) with the following logic:

```php
<?php

namespace App\Http\Controllers;

use Google\Client;
use Google\Service\Gmail;
use Google\Service\Gmail\Message;
use Illuminate\Http\JsonResponse;

class GmailController extends Controller
{
    public function sendEmail(): JsonResponse
    {
        // 1. Initialize Google Client
        $client = new Client();
        $client->setClientId(config('services.google.client_id', env('GOOGLE_CLIENT_ID')));
        $client->setClientSecret(config('services.google.client_secret', env('GOOGLE_CLIENT_SECRET')));
        $client->setAccessToken(config('services.google.access_token', env('GOOGLE_ACCESS_TOKEN')));

        // Automatically handle token refresh if it has expired
        if ($client->isAccessTokenExpired()) {
            $refreshToken = $client->getRefreshToken();
            if ($refreshToken) {
                $client->fetchAccessTokenWithRefreshToken($refreshToken);
            }
        }

        $gmailService = new Gmail($client);

        // 2. Build the Raw RFC 2822 Email Body
        $boundary = uniqid(rand(), true);
        $rawMessageString = "From: Me <me@gmail.com>\r\n";
        $rawMessageString .= "To: recipient@example.com\r\n";
        $rawMessageString .= "Subject: Hello via Gmail SendAPI!\r\n";
        $rawMessageString .= "MIME-Version: 1.0\r\n";
        $rawMessageString .= "Content-Type: multipart/alternative; boundary=\"$boundary\"\r\n\r\n";
        
        $rawMessageString .= "--$boundary\r\n";
        $rawMessageString .= "Content-Type: text/html; charset=utf-8\r\n\r\n";
        $rawMessageString .= "<h1>This is the email body sent using the Google API!</h1>\r\n";
        $rawMessageString .= "--$boundary--";

        // 3. Convert RFC 2822 structure to safe base64url standard
        $mime = rtrim(strtr(base64_encode($rawMessageString), '+/', '-_'), '=');

        // 4. Map to Gmail Message Instance
        $message = new Message();
        $message->setRaw($mime);

        try {
            // 'me' indicates authenticated user
            $gmailService->users_messages->send('me', $message);
            return response()->json(['status' => 'Email dispatched successfully!']);
        } catch (\Exception $e) {
            return response()->json(['error' => $e->getMessage()], 500);
        }
    }
}
```

---

## Method 2: Sending Messages via Google Chat Spaces API

To send real-time structured application alerts or team workspace tracking cards, utilize Laravel’s built-in, highly optimized `Http` client wrapper.

### Implementation Example

Create a controller or service class (`app/Http/Controllers/GoogleChatController.php`) with the following logic:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\JsonResponse;
use Illuminate\Support\Facades\Http;

class GoogleChatController extends Controller
{
    public function sendChatAlert(): JsonResponse
    {
        $webhookUrl = env('GOOGLE_CHAT_WEBHOOK_URL');

        if (!$webhookUrl) {
            return response()->json(['error' => 'Webhook URL not configured'], 400);
        }

        // Send a structured payload matching the Google Chat JSON schema
        $response = Http::post($webhookUrl, [
            'text' => "Hello Team! 🚀 This is an automated message triggered from **Laravel**."
        ]);

        if ($response->successful()) {
            return response()->json(['status' => 'Notification posted to Google Chat!']);
        }
        
        return response()->json([
            'error' => 'Failed to deliver message to workspace',
            'details' => $response->json()
        ], $response->status());
    }
}
```

---

## Production Recommendations & Ecosystem Tools

Instead of executing everything inline, production environments should leverage native abstraction ecosystems:

* **Queue Implementations:** Always dispatch these controllers or tasks using [Laravel Queues](https://laravel.com/docs/queues) to make sure third-party API delays never hurt user response metrics.
* **Laravel Gmail Package:** Utilize [ad5jp/laravel-gmail](https://packagist.org/packages/ad5jp/laravel-gmail) to use Gmail API smoothly inside Laravel's built-in `Mail::send()` workflows.
* **Notification Channels:** Use [Laravel Notification Channels for Google Chat](https://github.com/laravel-notification-channels/google-chat) to generate production-ready visual cards with buttons natively.
