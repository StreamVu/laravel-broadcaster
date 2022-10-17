# Ably Broadcaster for Laravel

This is a service provider package that allows using the Ably Broadcaster in Laravel.


# Requirements
1. PHP version >= 7.2.0
2. Laravel version >= 6.0.0

# Installation

The service provider is available as a composer package on packagist. If you don't have composer already installed, you can get it from https://getcomposer.org/.

Install Ably Broadcaster from the shell with:
```
composer require ably/laravel-broadcaster
```

# Setup

1. In your `.env` file, set `BROADCAST_DRIVER` and `ABLY_KEY`.
```dotenv
BROADCAST_DRIVER=ably

ABLY_KEY=ROOT_API_KEY_COPIED_FROM_ABLY_WEB_DASHBOARD
```

2. The following `.env` variables are optional, but can be added to further modify the behavior of Ably Broadcaster.
```dotenv
ABLY_DISABLE_PUBLIC_CHANNELS=false
ABLY_TOKEN_EXPIRY=3600
```

3. Uncomment `BroadcastServiceProvider` in `config/app.php`, as follows:
<pre>
        /*
         * Application Service Providers...
         */
        App\Providers\AppServiceProvider::class,
        App\Providers\AuthServiceProvider::class,
        <b>App\Providers\BroadcastServiceProvider::class,</b>
        App\Providers\EventServiceProvider::class,
        App\Providers\RouteServiceProvider::class,
</pre>

4. If running Laravel 8 or older, add the following lines in `config/broadcaster.php` to the `connections` array
```php
        'ably' => [
            'driver' => 'ably',
            'key' => env('ABLY_KEY'),
            'disable_public_channels' => env('ABLY_DISABLE_PUBLIC_CHANNELS', false),
            'token_expiry' => env('ABLY_TOKEN_EXPIRY', 3600)
        ],
```

5. For Laravel 9, you can optionally add the following lines in `config/broadcaster.php` to the `ably` array:
```php
            'disable_public_channels' => env('ABLY_DISABLE_PUBLIC_CHANNELS', false),
            'token_expiry' => env('ABLY_TOKEN_EXPIRY', 3600)
```

# Example code

## Registering channels

You can define channel capabilities for private and presence channels in `routes/channels.php`.

**Private chanel**

For private channels, access is allowed for truthy values and denied for falsy values.
```php
Broadcast::channel('channel1', function ($user) {
    return ['capability' => ["subscribe", "history"]];
});
```

**Presence channel**

For presence channels, we can also return data about the user. ([read more](https://laravel.com/docs/9.x/broadcasting#authorizing-presence-channels))
```php
Broadcast::channel('channel2', function ($user) {
    return ['id' => $user->id, 'name' => $user->name, 'capability' => ["subscribe", "presence"]];
});
```

## Using Laravel Echo on client-side

Laravel Echo is a JavaScript library that makes it painless to subscribe to channels and listen for events broadcast by your server-side broadcasting driver.

1. Install Laravel Echo and Ably:
```
npm install --save-dev laravel-echo ably
```
2. Add to `resources/js/bootstrap.js`:
```js
import Echo from 'laravel-echo';
import * as Ably from 'ably';

window.Ably = Ably;
window.Echo = new Echo({
    broadcaster: 'ably',
});

window.Echo.connector.ably.connection.on(stateChange => {
    if (stateChange.current === 'connected') {
        console.log('connected to ably server');
    }
});
```
3. Recompile Laravel assets:
```
npm run dev
```

## Broadcasting messages from server-side

Laravel supports [defining events](https://laravel.com/docs/events#defining-events) on server-side, and firing them at any time, to be received by the event listeners. Below is guide on how to send a public message notification that can be received via Laravel Echo on frontend.

1. Create `app/Events/PublicMessageNotification.php` with the following content:
```php
<?php

namespace App\Events;

use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class PublicMessageNotification implements ShouldBroadcast
{
    public $channel;
    public $message;

    use Dispatchable, InteractsWithSockets, SerializesModels;

    /**
     * Create a new event instance.
     *
     * @return void
     */
    public function __construct($channel, $message)
    {
        $this->channel = $channel;
        $this->message = $message;
    }

    /**
     * Get the channels the event should broadcast on.
     *
     * @return \Illuminate\Broadcasting\Channel|array
     */
    public function broadcastOn()
    {
        return new Channel($this->channel);
    }
}
```

2. Fire the event from anywhere within your application:
```php
event(new PublicMessageNotification($channel, $message));
```
The above event will be sent to all participants of the specified channel.

3. Receive the `PublicMessageNotification` event on frontend:
```js
Echo.channel(channel)
    .listen('PublicMessageNotification', (data) => {
        console.log(data);
        // Sample data: {"channel": "channelName", "message": "messageContent", "socket": null}
    })
```

# Contributing
