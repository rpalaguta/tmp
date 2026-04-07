# PHP Developer Exercise for Boozt

## Prerequisites
Here at Boozt we ship hundreds of packages per one month to Scandinavia and other parts of Europe. In order to provide the best service for our customers we are constantly expanding the pallet of Shipping Providers (currently there are more than 10). That is why is very important to have simple and reliable way to add more shipping providers.

Exercise provides a basic framework with Order entity and service, so you could spend less time bootstrapping the project, however feel free to taylor it according to your needs

### Requirements
- We expect this to be Unit tested. It is not a requirement to have 100% coverage, but basic functionality should be tested.
- APIs should be mocked, returning hard-coded results.
- Even though exercise contains 3 providers only, design your code to be extensible and as flexible as possible. Implementing new providers should be very straightforward. If you want you can do only 2 shipping providers.
- Do not implement any persistence layer or ORM, entity should be constructed with mock data.
- Create a Merge Request, so we can evaluate the code afterwards and give you feedback.

### Problem
Please implement a console command which would register a shipment for given shipping provider key. Provider key could be passed as an argument from STDIN. The rest order data could be mocked.

Each shipping provider could deliver the Order (`\App\Entity\Order`), however in the future we might add validation to limit supported providers. Provider is chosen by `\App\Entity\Order::getShippingProviderKey` method which returns provider key: __ups__, __omniva__ or __dhl__.
Shipment is registered by calling `\App\Service\Order::registerShipping` method, which should ensure a chosen provider is notified about the new shipment.

Command should exit if shipment has been registered successfully.

### Provider specifications
- **UPS**, send by api to `upsfake.com/register` -> `order_id`, `country`, `street`, `city`, `post_code`
- **OMNIVA** - get pick up point id by calling the api `omnivafake.com/pickup/find` : `country`, `post_code`, then send registration to `omnivafake.com/register` using `pickup_point_id` and `order_id`
- **DHL**, send by api to `dhlfake.com/register` -> `order_id`, `country`, `address`, `town`, `zip_code` 

### Evaluation Criteria
We will evaluate code based on these criteria:
- Code functions as specified in the Problem
- Whether tests pass (`docker compose exec app bin/phpunit`)
- Code readability and quality
- System flexibility and extensibility


# How to test application:
In the root directory:
1. Run `docker compose up -d`
2. Run `docker compose exec app composer install`
3. Run `docker compose exec app bin/phpunit`

---

# Implementation Notes

## What was implemented

### Architecture

| Component | File | Description |
|---|---|---|
| Console command | `src/Command/RegisterShipmentCommand.php` | Accepts provider key as argument, constructs a mock `Order`, calls `registerShipping()` |
| Order service | `src/Service/Order.php` | Delegates to the correct provider via `ShippingProviderRegistry` |
| Provider interface | `src/Service/ShippingProvider/ShippingProviderInterface.php` | Contract every provider must implement (`getProviderKey()`, `register()`) |
| Provider registry | `src/Service/ShippingProvider/ShippingProviderRegistry.php` | Holds all providers indexed by key; resolves the correct one at runtime |
| UPS provider | `src/Service/ShippingProvider/UpsShippingProvider.php` | `POST upsfake.com/register` with `order_id`, `country`, `street`, `city`, `post_code` |
| Omniva provider | `src/Service/ShippingProvider/OmnivaShippingProvider.php` | `GET omnivafake.com/pickup/find` → `POST omnivafake.com/register` |
| DHL provider | `src/Service/ShippingProvider/DhlShippingProvider.php` | `POST dhlfake.com/register` with `order_id`, `country`, `address`, `town`, `zip_code` |
| Mock HTTP client | `src/Http/MockHttpClient.php` | Implements `GuzzleHttp\ClientInterface`; returns hard-coded responses — no real HTTP calls |

### Dependency Injection wiring (`config/services.yaml`)

- All classes under `src/` are auto-registered via autowire + autoconfigure.
- Any class implementing `ShippingProviderInterface` is automatically tagged `app.shipping_provider` via `_instanceof`.
- `ShippingProviderRegistry` receives all tagged providers via `!tagged_iterator`.
- `GuzzleHttp\ClientInterface` is aliased to `App\Http\MockHttpClient` so no real HTTP calls are made.

### Tests (`tests/Unit/`)

- `Entity/OrderTest.php` — verifies `Order` fields and default provider key.
- `Service/OrderTest.php` — verifies `registerShipping()` delegates to the correct provider.
- `Service/ShippingProvider/UpsShippingProviderTest.php` — verifies correct HTTP method, URL, and field names.
- `Service/ShippingProvider/OmnivaShippingProviderTest.php` — verifies two-step call (GET pickup point → POST register).
- `Service/ShippingProvider/DhlShippingProviderTest.php` — verifies DHL-specific field mapping (`address`, `town`, `zip_code`).
- `Service/ShippingProvider/ShippingProviderRegistryTest.php` — verifies provider resolution by key and unknown key exception.

### Running the command

```bash
docker compose exec app bin/console app:register-shipment ups
docker compose exec app bin/console app:register-shipment omniva
docker compose exec app bin/console app:register-shipment dhl
```

---

## How to add a new shipping provider

Only **one new file** is needed. No existing files require modification.

**1. Create the provider class** in `src/Service/ShippingProvider/`:

```php
<?php

declare(strict_types=1);

namespace App\Service\ShippingProvider;

use GuzzleHttp\ClientInterface;

class FedexShippingProvider implements ShippingProviderInterface
{
    public function __construct(
        private readonly ClientInterface $client,
    ) {
    }

    public function getProviderKey(): string
    {
        return 'fedex';
    }

    public function register(\App\Entity\Order $order): void
    {
        $this->client->request('POST', 'http://fedexfake.com/register', [
            'json' => [
                'order_id' => $order->getId(),
                'country'  => $order->getCountry(),
                // map any other fields the provider API requires
            ],
        ]);
    }
}
```

**2. That's it for production code.** Symfony's autowiring + the `_instanceof` tag in `services.yaml` will automatically:
- register the class as a service
- tag it as `app.shipping_provider`
- inject it into `ShippingProviderRegistry`
- make it available via `bin/console app:register-shipment fedex`

The help text (`--help`) will also automatically list `fedex` as a valid key.

**3. Add a test** in `tests/Unit/Service/ShippingProvider/FedexShippingProviderTest.php` following the pattern of the existing provider tests.

**4. If the new provider API returns data** (like Omniva's pickup point), add a matching hard-coded response to `MockHttpClient::buildResponse()`:

```php
str_contains($uri, 'fedexfake.com/some/endpoint') => '{"some_field": "value"}',
```

