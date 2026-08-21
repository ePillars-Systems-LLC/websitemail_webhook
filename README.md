# Website Mail Webhook

A lightweight Laravel webhook for receiving website contact-form submissions and sending notifications through:

- **Gmail API** — for sending email notifications
- **Google Chat Incoming Webhook** — for posting notifications to a Google Chat Space

The application is designed for websites that need to submit contact/enquiry forms to a central backend.

---

## Architecture

```text
                    Website
                       │
                       │ HTTPS POST
                       ▼
              ┌──────────────────┐
              │      Nginx       │
              │ TLS / Rate Limit │
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │      Laravel     │
              │  Website Webhook │
              └────────┬─────────┘
                       │
                Validate Request
                       │
              ┌────────┴─────────┐
              │                  │
              ▼                  ▼
        ┌───────────┐    ┌────────────────┐
        │ Gmail API │    │ Google Chat    │
        │           │    │ Incoming       │
        │ OAuth     │    │ Webhook        │
        └─────┬─────┘    └───────┬────────┘
              │                  │
              ▼                  ▼
        Email Notification   Google Chat
                            Space Message
```

---

# 1. How Authentication Works

There are **two separate Google integrations** in this application.

## Gmail API

Gmail API requires Google authentication.

The application uses:

```text
Google OAuth 2.0
       │
       ├── Client ID
       ├── Client Secret
       └── Refresh Token
```

The refresh token is used to obtain new Gmail access tokens when required.

---

## Google Chat Incoming Webhook

Google Chat is different.

This application uses an **Incoming Webhook registered directly in a Google Chat Space**.

It does **not** use:

- Google OAuth
- Google access tokens
- Google refresh tokens
- Google service accounts
- Google Chat API OAuth scopes

The webhook URL itself contains the required credentials:

```text
https://chat.googleapis.com/v1/spaces/SPACE_ID/messages?key=KEY&token=TOKEN
```

Google's official documentation shows external applications sending a normal HTTP `POST` directly to this URL.

### Important

The Chat webhook URL must be treated as a secret.

The `token` in the URL is specifically described by Google as a secret value that must be kept private. Do **not** commit the webhook URL to GitHub or expose it publicly.

---

# 2. Requirements

## Server

Recommended:

- Ubuntu 22.04+
- PHP 8.2+
- PHP-FPM
- Composer
- Nginx
- MySQL/MariaDB or SQLite
- HTTPS

## Google

Required:

- Google Cloud project
- Gmail API enabled
- Google Workspace account
- Google Chat Space
- Incoming webhook configured in the Chat Space

Google Chat incoming webhooks are intended for one-way notifications from external systems to a specific Chat Space.

---

# 3. Create Laravel Application

If creating the project from scratch:

```bash
composer create-project laravel/laravel websitemail_webhook
cd websitemail_webhook
```

Install the Google API client:

```bash
composer require google/apiclient
```

---

# 4. Google Cloud Configuration

Open:

```text
https://console.cloud.google.com/
```

Create or select a Google Cloud project.

Enable:

```text
Gmail API
```

Google Chat does **not** need to be configured as an OAuth API for this implementation because we are using a Chat Incoming Webhook.

---

# 5. Gmail OAuth Configuration

Create OAuth credentials in Google Cloud.

The application needs:

```text
GOOGLE_CLIENT_ID
GOOGLE_CLIENT_SECRET
GOOGLE_REFRESH_TOKEN
```

The Gmail scope required for sending email is:

```text
https://www.googleapis.com/auth/gmail.send
```

### Why use a refresh token?

Gmail access tokens expire.

Therefore, do not configure the application around a manually stored:

```env
GOOGLE_ACCESS_TOKEN=...
```

Instead, store the OAuth refresh token and allow the Google API client to obtain an access token when needed.

---

# 6. Google Chat Incoming Webhook

Open Google Chat.

Go to the Space where notifications should be posted.

Then:

```text
Space
  ↓
Apps & integrations
  ↓
Add webhooks
```

Create the webhook and copy the generated URL.

It will look similar to:

```text
https://chat.googleapis.com/v1/spaces/AAAA.../messages?key=XXX&token=YYY
```

Google's documented webhook implementation simply sends:

```http
POST
Content-Type: application/json
```

with:

```json
{
    "text": "Hello from the website!"
}
```

to that URL.

---

# 7. Environment Configuration

Create:

```bash
cp .env.example .env
```

Example:

```env
APP_NAME="Website Mail Webhook"
APP_ENV=production
APP_DEBUG=false
APP_URL=https://mail.example.com

LOG_CHANNEL=stack
LOG_LEVEL=info

# --------------------------------------------------
# Website Webhook
# --------------------------------------------------

WEBSITE_WEBHOOK_SECRET=CHANGE_THIS_TO_A_LONG_RANDOM_SECRET

# --------------------------------------------------
# Gmail API
# --------------------------------------------------

GOOGLE_CLIENT_ID=xxxxxxxxxxxxxxxx.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=xxxxxxxxxxxxxxxx
GOOGLE_REFRESH_TOKEN=xxxxxxxxxxxxxxxx

GOOGLE_GMAIL_FROM_EMAIL=notifications@example.com
GOOGLE_GMAIL_FROM_NAME="Website Mail"

WEBSITE_MAIL_TO=admin@example.com

# --------------------------------------------------
# Google Chat Incoming Webhook
# --------------------------------------------------

GOOGLE_CHAT_WEBHOOK_URL="https://chat.googleapis.com/v1/spaces/XXX/messages?key=XXX&token=XXX"

# --------------------------------------------------
# Queue
# --------------------------------------------------

QUEUE_CONNECTION=database
```

---

# 8. Important Security Rule

Never commit `.env`.

The following should never be stored in Git:

```text
.env
Google Client Secret
Google Refresh Token
Google Chat Webhook URL
Website Webhook Secret
```

Add to `.gitignore`:

```gitignore
.env
.env.*
!.env.example
```

The Google Chat webhook URL is particularly sensitive because possession of that URL allows an external party to post messages to the Space associated with the webhook. Google explicitly warns not to share the URL publicly.

---

# 9. Laravel Services Configuration

Edit:

```text
config/services.php
```

Add:

```php
'google' => [

    'client_id' => env('GOOGLE_CLIENT_ID'),

    'client_secret' => env('GOOGLE_CLIENT_SECRET'),

    'refresh_token' => env('GOOGLE_REFRESH_TOKEN'),

    'gmail' => [
        'from_email' => env('GOOGLE_GMAIL_FROM_EMAIL'),
        'from_name' => env(
            'GOOGLE_GMAIL_FROM_NAME',
            'Website Mail'
        ),
        'to_email' => env('WEBSITE_MAIL_TO'),
    ],

    /*
     * Google Chat Incoming Webhook.
     *
     * No OAuth credentials are required here.
     * The webhook URL contains the authentication
     * information provided by Google Chat.
     */
    'chat' => [
        'webhook_url' => env(
            'GOOGLE_CHAT_WEBHOOK_URL'
        ),
    ],
],

'website_webhook' => [

    /*
     * Secret used by the website when calling
     * our Laravel webhook.
     */
    'secret' => env(
        'WEBSITE_WEBHOOK_SECRET'
    ),
],
```

---

# 10. Gmail Service

Create:

```text
app/Services/GmailService.php
```

```php
<?php

namespace App\Services;

use Google\Client;
use Google\Service\Gmail;
use Google\Service\Gmail\Message;
use RuntimeException;

class GmailService
{
    protected function getClient(): Client
    {
        $client = new Client();

        $client->setClientId(
            config('services.google.client_id')
        );

        $client->setClientSecret(
            config('services.google.client_secret')
        );

        $client->setScopes([
            Gmail::GMAIL_SEND,
        ]);

        $refreshToken = config(
            'services.google.refresh_token'
        );

        if (!$refreshToken) {
            throw new RuntimeException(
                'Google refresh token is not configured.'
            );
        }

        $token = $client->fetchAccessTokenWithRefreshToken(
            $refreshToken
        );

        if (isset($token['error'])) {
            throw new RuntimeException(
                'Unable to refresh Google access token: '
                . json_encode($token)
            );
        }

        return $client;
    }

    public function send(
        string $to,
        string $subject,
        string $htmlBody,
        ?string $replyTo = null
    ): void {
        $client = $this->getClient();

        $gmail = new Gmail($client);

        $fromEmail = config(
            'services.google.gmail.from_email'
        );

        $fromName = config(
            'services.google.gmail.from_name'
        );

        $headers = [];

        $headers[] =
            'From: '
            . $fromName
            . ' <'
            . $fromEmail
            . '>';

        $headers[] = 'To: ' . $to;

        $headers[] = 'Subject: ' . $subject;

        if ($replyTo) {
            $headers[] = 'Reply-To: ' . $replyTo;
        }

        $headers[] = 'MIME-Version: 1.0';

        $headers[] =
            'Content-Type: text/html; charset=UTF-8';

        $rawMessage =
            implode("\r\n", $headers)
            . "\r\n\r\n"
            . $htmlBody;

        $encodedMessage = rtrim(
            strtr(
                base64_encode($rawMessage),
                '+/',
                '-_'
            ),
            '='
        );

        $message = new Message();

        $message->setRaw(
            $encodedMessage
        );

        $gmail->users_messages->send(
            'me',
            $message
        );
    }
}
```

---

# 11. Google Chat Service

Create:

```text
app/Services/GoogleChatService.php
```

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\Http;
use RuntimeException;

class GoogleChatService
{
    public function send(string $message): void
    {
        $webhookUrl = config(
            'services.google.chat.webhook_url'
        );

        if (!$webhookUrl) {
            throw new RuntimeException(
                'Google Chat webhook URL is not configured.'
            );
        }

        $response = Http::timeout(15)
            ->post(
                $webhookUrl,
                [
                    'text' => $message,
                ]
            );

        if (!$response->successful()) {
            throw new RuntimeException(
                'Google Chat notification failed: '
                . $response->body()
            );
        }
    }
}
```

There is intentionally **no OAuth code** in this service.

The authentication/authorization information is contained in:

```text
GOOGLE_CHAT_WEBHOOK_URL
```

---

# 12. Website Mail Controller

Create:

```text
app/Http/Controllers/WebsiteMailController.php
```

```php
<?php

namespace App\Http\Controllers;

use App\Services\GmailService;
use App\Services\GoogleChatService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Validator;
use Throwable;

class WebsiteMailController extends Controller
{
    public function __construct(
        protected GmailService $gmail,
        protected GoogleChatService $chat
    ) {
    }

    public function store(
        Request $request
    ): JsonResponse {
        /*
         * Authenticate the website calling our
         * Laravel webhook.
         */
        $providedSecret = $request->header(
            'X-Webhook-Secret'
        );

        $expectedSecret = config(
            'services.website_webhook.secret'
        );

        if (
            !$providedSecret ||
            !$expectedSecret ||
            !hash_equals(
                $expectedSecret,
                $providedSecret
            )
        ) {
            return response()->json([
                'status' => 'error',
                'message' => 'Unauthorized',
            ], 401);
        }

        /*
         * Validate incoming request.
         */
        $validator = Validator::make(
            $request->all(),
            [
                'name' => [
                    'required',
                    'string',
                    'max:150',
                ],

                'email' => [
                    'required',
                    'email',
                    'max:255',
                ],

                'subject' => [
                    'required',
                    'string',
                    'max:255',
                ],

                'message' => [
                    'required',
                    'string',
                    'max:10000',
                ],
            ]
        );

        if ($validator->fails()) {
            return response()->json([
                'status' => 'error',
                'message' => 'Validation failed.',
                'errors' => $validator->errors(),
            ], 422);
        }

        $data = $validator->validated();

        try {
            /*
             * Escape user input before inserting into HTML.
             */
            $name = e($data['name']);
            $email = e($data['email']);
            $subject = e($data['subject']);
            $message = nl2br(
                e($data['message'])
            );

            /*
             * Send Gmail notification.
             */
            $html = <<<HTML
<h2>New Website Enquiry</h2>

<table cellpadding="6" cellspacing="0">
    <tr>
        <td><strong>Name</strong></td>
        <td>{$name}</td>
    </tr>

    <tr>
        <td><strong>Email</strong></td>
        <td>{$email}</td>
    </tr>

    <tr>
        <td><strong>Subject</strong></td>
        <td>{$subject}</td>
    </tr>
</table>

<hr>

<h3>Message</h3>

<p>{$message}</p>
HTML;

            $this->gmail->send(
                config(
                    'services.google.gmail.to_email'
                ),
                '[Website] ' . $data['subject'],
                $html,
                $data['email']
            );

            /*
             * Send Google Chat notification.
             *
             * This uses the Google Chat Incoming
             * Webhook URL. No OAuth is required.
             */
            $chatMessage = <<<TEXT
📩 New website enquiry

Name: {$data['name']}
Email: {$data['email']}
Subject: {$data['subject']}

Message:
{$data['message']}
TEXT;

            $this->chat->send(
                $chatMessage
            );

            return response()->json([
                'status' => 'success',
                'message' =>
                    'Message received successfully.',
            ]);
        } catch (Throwable $e) {

            Log::error(
                'Website mail webhook failed.',
                [
                    'error' => $e->getMessage(),
                    'email' =>
                        $data['email'] ?? null,
                ]
            );

            return response()->json([
                'status' => 'error',
                'message' =>
                    'Unable to process message.',
            ], 500);
        }
    }
}
```

---

# 13. API Route

Edit:

```text
routes/api.php
```

Add:

```php
<?php

use App\Http\Controllers\WebsiteMailController;
use Illuminate\Support\Facades\Route;

Route::middleware('throttle:30,1')
    ->post(
        '/website-mail',
        [
            WebsiteMailController::class,
            'store',
        ]
    );
```

The endpoint becomes:

```text
POST https://mail.example.com/api/website-mail
```

---

# 14. Website Request

The website sends:

```http
POST /api/website-mail
Content-Type: application/json
X-Webhook-Secret: YOUR_SECRET
```

Body:

```json
{
    "name": "John Smith",
    "email": "john@example.com",
    "subject": "Website enquiry",
    "message": "Please contact me regarding your services."
}
```

---

# 15. Test Using cURL

```bash
curl -X POST \
  https://mail.example.com/api/website-mail \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: YOUR_SECRET" \
  -d '{
    "name": "John Smith",
    "email": "john@example.com",
    "subject": "Test Message",
    "message": "This is a test message."
  }'
```

Expected:

```json
{
    "status": "success",
    "message": "Message received successfully."
}
```

---

# 16. Test Google Chat Directly

Before testing the Laravel application, you can verify that the Google Chat webhook itself works.

```bash
curl -X POST \
  "https://chat.googleapis.com/v1/spaces/SPACE_ID/messages?key=KEY&token=TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Test message from webhook"
  }'
```

If the message appears in the configured Space, the Google Chat webhook is working.

This is the same basic HTTP POST architecture documented by Google.

---

# 17. Security Model

There are two credentials involved, but they serve different purposes.

## Website → Laravel

The Laravel application should authenticate the website.

Recommended:

```http
X-Webhook-Secret: YOUR_SECRET
```

This prevents arbitrary Internet users from submitting messages to your application.

---

## Laravel → Gmail

Gmail uses:

```text
OAuth Client ID
OAuth Client Secret
OAuth Refresh Token
```

---

## Laravel → Google Chat

Google Chat Incoming Webhook uses:

```text
GOOGLE_CHAT_WEBHOOK_URL
```

No OAuth token is added to the HTTP request.

The URL itself contains the webhook's authentication information.

Google specifically states that the webhook URL contains a `token` that must be kept secret.

---

# 18. Do Not Confuse These Two Concepts

This project does **not** implement a full Google Chat application.

It uses an **Incoming Webhook**.

### Incoming Webhook

```text
Laravel
   │
   │ POST
   │
   ▼
Google Chat Space
```

One-way notification.

No OAuth.

No user interaction.

No reading Chat messages.

No listing Spaces.

No Chat API authentication.

Google recommends this architecture for external systems that need to send messages to a specific Chat Space.

### Full Google Chat App

A full Chat application is different:

```text
User
  │
  ▼
Google Chat App
  │
  ▼
Chat API
```

That architecture can require authentication and authorization depending on the operation.

That is **not required for this project**.

---

# 19. Rate Limiting

The public Laravel endpoint should be rate limited.

Example:

```php
Route::middleware('throttle:30,1')
    ->post(
        '/website-mail',
        [WebsiteMailController::class, 'store']
    );
```

For a production website, you may want to use a stricter custom rate limiter.

Cloudflare can also be used to protect the endpoint.

---

# 20. Cloudflare

Recommended architecture:

```text
Internet
   │
   ▼
Cloudflare
   │
   ├── WAF
   ├── Rate Limiting
   └── HTTPS
   │
   ▼
Nginx
   │
   ▼
Laravel
```

Even if Cloudflare is used, keep application-level authentication:

```text
X-Webhook-Secret
```

---

# 21. Queue Processing

For production, it is recommended to process Gmail and Chat notifications asynchronously.

Instead of:

```text
Website
   │
   ▼
Laravel
   │
   ├── Gmail
   ├── Google Chat
   │
   ▼
HTTP Response
```

use:

```text
Website
   │
   ▼
Laravel
   │
   ▼
Queue
   │
   ├── Gmail
   └── Google Chat
```

This allows the website to receive a response quickly even if Google APIs are temporarily slow.

Create a job:

```bash
php artisan make:job ProcessWebsiteMail
```

For database queues:

```bash
php artisan make:queue-table
php artisan migrate
```

Set:

```env
QUEUE_CONNECTION=database
```

Run:

```bash
php artisan queue:work
```

For production, run the worker under Supervisor or systemd.

---

# 22. Error Handling

## Successful request

```http
200 OK
```

```json
{
    "status": "success",
    "message": "Message received successfully."
}
```

## Invalid webhook secret

```http
401 Unauthorized
```

## Validation failure

```http
422 Unprocessable Entity
```

## Google API failure

```http
500 Internal Server Error
```

Do not expose internal Google API errors, tokens, secrets, or stack traces to the website.

---

# 23. Logging

Laravel logs:

```text
storage/logs/laravel.log
```

Monitor:

```bash
tail -f storage/logs/laravel.log
```

Do not log:

```text
Google Refresh Token
Google Client Secret
Google Chat Webhook URL
X-Webhook-Secret
```

---

# 24. Nginx Configuration

Example:

```nginx
server {
    listen 80;

    server_name mail.example.com;

    root /var/www/websitemail_webhook/public;

    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;

        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }

    location ~ /\. {
        deny all;
    }
}
```

Use HTTPS in production.

---

# 25. Laravel Deployment

Example:

```bash
cd /var/www

git clone \
    https://github.com/ePillars-Systems-LLC/websitemail_webhook.git

cd websitemail_webhook

composer install \
    --no-dev \
    --optimize-autoloader
```

Configure:

```bash
cp .env.example .env

nano .env
```

Generate application key:

```bash
php artisan key:generate
```

Run migrations:

```bash
php artisan migrate --force
```

Optimize:

```bash
php artisan optimize
```

Set permissions:

```bash
sudo chown -R www-data:www-data \
    storage \
    bootstrap/cache
```

Restart PHP-FPM:

```bash
sudo systemctl restart php8.3-fpm
```

---

# 26. Health Check

Add to `routes/api.php`:

```php
Route::get('/health', function () {
    return response()->json([
        'status' => 'ok',
        'application' => 'website-mail-webhook',
    ]);
});
```

Test:

```bash
curl https://mail.example.com/api/health
```

Expected:

```json
{
    "status": "ok",
    "application": "website-mail-webhook"
}
```

---

# 27. Project Structure

Recommended structure:

```text
websitemail_webhook/
│
├── app/
│   ├── Http/
│   │   └── Controllers/
│   │       └── WebsiteMailController.php
│   │
│   └── Services/
│       ├── GmailService.php
│       └── GoogleChatService.php
│
├── config/
│   └── services.php
│
├── routes/
│   └── api.php
│
├── storage/
│
├── .env
├── .env.example
├── .gitignore
├── artisan
├── composer.json
└── README.md
```

---

# 28. Production Checklist

- [ ] Laravel installed
- [ ] PHP configured
- [ ] Composer configured
- [ ] HTTPS configured
- [ ] Gmail API enabled
- [ ] Gmail OAuth Client created
- [ ] Gmail refresh token generated
- [ ] Gmail sender configured
- [ ] Google Chat Space created
- [ ] Google Chat Incoming Webhook created
- [ ] Google Chat webhook URL stored in `.env`
- [ ] Website webhook secret generated
- [ ] API route configured
- [ ] Gmail service configured
- [ ] Google Chat service configured
- [ ] Rate limiting enabled
- [ ] `.env` excluded from Git
- [ ] cURL test successful
- [ ] Gmail test successful
- [ ] Google Chat test successful
- [ ] Invalid webhook secret rejected
- [ ] Queue configured
- [ ] Queue worker configured
- [ ] Laravel logs monitored
- [ ] Cloudflare/WAF configured if required

---

# 29. Security Summary

The application has three separate security boundaries:

```text
┌───────────────────────────────────────────────┐
│              Website → Laravel                │
│                                               │
│       X-Webhook-Secret / Rate Limit           │
└───────────────────────┬───────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────┐
│               Laravel → Gmail                 │
│                                               │
│        Google OAuth + Refresh Token           │
└───────────────────────┬───────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────┐
│            Laravel → Google Chat              │
│                                               │
│       Incoming Webhook URL (secret URL)       │
│                 No OAuth                      │
└───────────────────────────────────────────────┘
```

The Google Chat webhook is intentionally simple: Laravel makes an HTTPS POST to the webhook URL. Google handles the webhook credential contained in the URL.

---

# 30. Final Request Flow

A typical website enquiry will work as follows:

```text
1. User submits website contact form

              │
              ▼

2. Website sends POST request

   /api/website-mail

              │
              ▼

3. Laravel validates:
   - Webhook secret
   - Name
   - Email
   - Subject
   - Message

              │
              ▼

4. Laravel sends email through Gmail API

              │
              ▼

5. Laravel sends notification to
   Google Chat Incoming Webhook

              │
              ▼

6. Team receives Chat notification

              │
              ▼

7. Website receives success response
```

---

# 31. Key Design Decision

This project intentionally uses **two different Google mechanisms**:

| Function                      | Technology            |     OAuth Required |
| ----------------------------- | --------------------- | -----------------: |
| Send email                    | Gmail API             |                Yes |
| Send Google Chat notification | Chat Incoming Webhook |             **No** |
| Receive website request       | Laravel API           | Application secret |
| Protect public endpoint       | Laravel/Cloudflare    |        Recommended |

This keeps the implementation simple and avoids unnecessary Google Chat OAuth configuration.

**Gmail requires OAuth because the application is accessing Gmail. Google Chat does not require OAuth when using the Space's Incoming Webhook URL.**
