# Laravel Clean Architecture Project Structure: A Pragmatic Approach

This document outlines a comprehensive project structure for Laravel applications following Clean Architecture principles with a domain-driven design approach, presented in order from simplest to most complex concepts.

## Introduction

This guide presents a spectrum of implementations from pragmatic to purist approaches. You can choose the level of architectural purity that makes sense for your project:

- **Pragmatic Approach**: Directly use Eloquent in use cases for simpler applications or teams new to Clean Architecture
- **Standard Approach**: Use repositories for more decoupling and testability
- **Purist Approach**: Full separation with presenters, DTOs, and strict layer isolation

Start with the approach that fits your current needs, then evolve as your application grows or your team gains experience with the architecture.

## Basic Principles

1. **Domain-Focused Organization**: Top-level directories represent business domains rather than technical concerns
2. **Clean Architecture Layers**: Each domain follows the standard Clean Architecture layers
3. **Co-located Specifications**: Tests reside with the code they verify
4. **Testing Support**: Each domain includes testing support utilities
5. **Explicit IO Layer**: Clear separation of external integrations and frameworks

## Core Structure Overview

```
project-root/
├── app/
│   ├── Foundation/               # Shared kernel, base classes, traits
│   ├── Booking/                  # Booking domain module
│   ├── Payment/                  # Payment domain module
│   └── User/                     # User domain module
```

## Clean Architecture Layers

Each domain is organized into layers following Clean Architecture principles:

```
Booking/
├── Entities/             # Domain entities, models, business rules
├── UseCases/             # Application business logic
├── Adapters/             # Framework-agnostic interface adapters  
├── IO/                   # Frameworks, drivers, external services
├── Specs/                # Behavior specifications
└── Testing/              # Testing utilities
```

### Entities Layer

Contains your domain models and business rules:

```php
// app/Booking/Entities/Booking.php
namespace App\Booking\Entities;

use Illuminate\Database\Eloquent\Model;
use App\User\Entities\User;

class Booking extends Model
{
    protected $fillable = ['user_id', 'start_date', 'end_date', 'status', 'notes'];
    
    // Domain methods (business logic)
    public function cancel()
    {
        $this->status = 'cancelled';
        $this->cancelled_at = now();
        return $this;
    }
    
    public function canBeCancelled()
    {
        return $this->status === 'active' && $this->start_date > now();
    }
    
    // Relationships
    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

For pragmatic reasons, we place Eloquent models directly in the Entities layer, which is a compromise that works well in the Laravel ecosystem.

### Simple Use Case Implementation

The simplest use cases can interact directly with Eloquent models for a pragmatic approach:

```php
// app/Booking/UseCases/CancelBooking.php
namespace App\Booking\UseCases;

use App\Booking\Entities\Booking;
use App\Booking\UseCases\Exceptions\BookingNotFoundException;
use App\Booking\UseCases\Exceptions\BookingCancellationException;

class CancelBooking
{
    public function execute(string $bookingId)
    {
        // Directly use Eloquent for simplicity
        $booking = Booking::find($bookingId);
        
        if (!$booking) {
            throw new BookingNotFoundException("Booking not found: {$bookingId}");
        }
        
        if (!$booking->canBeCancelled()) {
            throw new BookingCancellationException('Booking cannot be cancelled');
        }
        
        $booking->cancel();
        $booking->save();
        
        return $booking;
    }
}
```

This can be used directly in a controller:

```php
// app/Booking/IO/Http/BookingController.php
namespace App\Booking\IO\Http;

use App\Booking\UseCases\CancelBooking;
use App\Booking\UseCases\Exceptions\BookingNotFoundException;
use App\Booking\UseCases\Exceptions\BookingCancellationException;
use Illuminate\Http\JsonResponse;

class BookingController
{
    private $cancelBooking;
    
    public function __construct(CancelBooking $cancelBooking)
    {
        $this->cancelBooking = $cancelBooking;
    }
    
    public function cancel(string $id): JsonResponse
    {
        try {
            $booking = $this->cancelBooking->execute($id);
            return response()->json([
                'success' => true,
                'data' => $booking
            ]);
        } catch (BookingNotFoundException $e) {
            return response()->json([
                'success' => false,
                'error' => $e->getMessage()
            ], 404);
        } catch (BookingCancellationException $e) {
            return response()->json([
                'success' => false,
                'error' => $e->getMessage()
            ], 400);
        }
    }
}
```

This pragmatic approach allows you to get many of the benefits of Clean Architecture while leveraging Laravel's Eloquent ORM directly. As your application grows, you can gradually migrate to more decoupled implementations using repositories.

### More Complex Use Cases with Repository Pattern

As your application grows, you might want to add more decoupling by using the repository pattern:

```php
// app/Booking/UseCases/Repositories/BookingRepositoryInterface.php
namespace App\Booking\UseCases\Repositories;

use App\Booking\Entities\Booking;
use DateTime;

interface BookingRepositoryInterface
{
    public function findById(string $id): ?Booking;
    public function findActiveByDateRange(string $userId, DateTime $startDate, DateTime $endDate): array;
    public function save(Booking $booking): Booking;
    public function delete(string $id): bool;
}

// app/Booking/UseCases/CancelBooking.php
namespace App\Booking\UseCases;

use App\Booking\UseCases\Repositories\BookingRepositoryInterface;
use App\Booking\Entities\Booking;

class CancelBooking
{
    private $bookingRepository;
    
    public function __construct(BookingRepositoryInterface $bookingRepository)
    {
        $this->bookingRepository = $bookingRepository;
    }
    
    public function execute(string $bookingId)
    {
        $booking = $this->bookingRepository->findById($bookingId);
        
        if (!$booking) {
            throw new BookingNotFoundException("Booking not found: {$bookingId}");
        }
        
        if (!$booking->canBeCancelled()) {
            throw new BookingCancellationException('Booking cannot be cancelled');
        }
        
        $booking->cancel();
        return $this->bookingRepository->save($booking);
    }
}
```

## Complete Directory Structure

```
project-root/
├── app/
│   ├── Foundation/               # Shared kernel, base classes, traits
│   │   ├── Entities/             # Base entities, models, value objects, shared domain logic
│   │   ├── UseCases/             # Base interactors, shared application services
│   │       ├── Repositories/     # Repository interfaces for shared entities
│   │   ├── Adapters/             # Base presenters, view models, framework-agnostic components
│   │   ├── IO/                   # Shared IO components
│   │       ├── Database/         # Common database concerns, repository implementations
│   │       ├── Http/             # Shared controllers, middleware, API resources
│   │       ├── Web/              # Common layouts, shared components, base templates
│   │       ├── GraphQL/          # Shared GraphQL components
│   │       └── ExternalServices/ # Shared service clients, integrations
│   │       ├── FoundationServiceProvider.php  # Core app-wide service provider
│   │   ├── Specs/                # Foundation specifications
│   │   └── Testing/              # Foundation testing support API
│   │
│   ├── Booking/                  # Booking domain module
│   │   ├── Entities/             # Domain entities, models, and business rules
│   │   ├── UseCases/             # Application business logic
│   │       ├── Repositories/     # Repository interfaces
│   │   ├── Adapters/             # Framework-agnostic interface adapters
│   │   ├── IO/                   # Frameworks, drivers, external services
│   │       ├── Database/         # Booking-specific database migrations, seeders, repositories
│   │       ├── Http/             # Booking-specific HTTP interfaces, controllers, resources
│   │       ├── Web/              # Booking-specific UI elements
│   │       ├── GraphQL/          # Booking-specific GraphQL components
│   │       └── ExternalServices/ # Booking-specific external services
│   │       ├── BookingServiceProvider.php      # Booking domain service provider
│   │   ├── Specs/                # Booking behavior specifications
│   │   └── Testing/              # Booking-specific testing utilities
│   │
│   ├── Payment/                  # Payment domain module (same structure)
│   └── User/                     # User domain module (same structure)
```

## More Advanced Use Case Patterns

### Request/Response Models with spatie/laravel-data

For more complex use cases, the `spatie/laravel-data` package provides strongly-typed DTOs:

```php
// app/Booking/UseCases/CreateBooking/CreateBookingRequest.php
namespace App\Booking\UseCases\CreateBooking;

use Spatie\LaravelData\Data;

class CreateBookingRequest extends Data
{
    public string $userId;
    public string $startDate;
    public string $endDate;
    public ?string $notes = null;
}
```

Use this with your use cases:

```php
// app/Booking/UseCases/CreateBooking.php
namespace App\Booking\UseCases;

use App\Booking\Entities\Booking;
use App\Booking\UseCases\CreateBooking\CreateBookingRequest;
use App\Booking\UseCases\Repositories\BookingRepositoryInterface;

class CreateBooking
{
    private $bookingRepository;
    
    public function __construct(BookingRepositoryInterface $bookingRepository)
    {
        $this->bookingRepository = $bookingRepository;
    }
    
    public function execute(CreateBookingRequest $request): Booking
    {
        $booking = new Booking();
        $booking->user_id = $request->userId;
        $booking->start_date = $request->startDate;
        $booking->end_date = $request->endDate;
        $booking->notes = $request->notes;
        $booking->status = 'pending';
        
        return $this->bookingRepository->save($booking);
    }
}
```

## Repository Pattern Implementation Choices

When implementing repositories, you have two main approaches to choose from:

### Direct Eloquent Access (Simpler)

```php
// app/Booking/IO/Database/Repositories/EloquentBookingRepository.php
namespace App\Booking\IO\Database\Repositories;

use App\Booking\UseCases\Repositories\BookingRepositoryInterface;
use App\Booking\Entities\Booking;

class EloquentBookingRepository implements BookingRepositoryInterface
{
    public function findById(string $id): ?Booking
    {
        return Booking::find($id);
    }
    
    public function save(Booking $booking): Booking
    {
        $booking->save();
        return $booking;
    }
}
```

**Best for:**
- Smaller applications with straightforward data access
- When you need to leverage Eloquent relationships
- When performance is not the primary concern
- Teams more familiar with Eloquent than raw queries

### Query Builder Approach (More Performant)

```php
// app/Booking/IO/Database/Repositories/QueryBookingRepository.php
namespace App\Booking\IO\Database\Repositories;

use App\Booking\UseCases\Repositories\BookingRepositoryInterface;
use App\Booking\Entities\Booking;
use Illuminate\Support\Facades\DB;

class QueryBookingRepository implements BookingRepositoryInterface
{
    public function findById(string $id): ?Booking
    {
        // Use DB::table for better performance
        $bookingData = DB::table('bookings')->where('id', $id)->first();
        
        if (!$bookingData) {
            return null;
        }
        
        return $this->mapToEntity($bookingData);
    }
    
    public function save(Booking $booking): Booking
    {
        if (empty($booking->id)) {
            // Insert new record
            $id = DB::table('bookings')->insertGetId([
                'user_id' => $booking->user_id,
                'start_date' => $booking->start_date,
                'end_date' => $booking->end_date,
                'status' => $booking->status,
                'notes' => $booking->notes,
                'created_at' => now(),
                'updated_at' => now(),
            ]);
            
            $booking->id = $id;
        } else {
            // Update existing record
            DB::table('bookings')
                ->where('id', $booking->id)
                ->update([
                    'user_id' => $booking->user_id,
                    'start_date' => $booking->start_date,
                    'end_date' => $booking->end_date,
                    'status' => $booking->status,
                    'notes' => $booking->notes,
                    'updated_at' => now(),
                ]);
        }
        
        return $booking;
    }
    
    /**
     * Map a database record to a domain entity
     */
    private function mapToEntity(object $data): Booking
    {
        $booking = new Booking();
        $booking->exists = true;
        $booking->id = $data->id;
        $booking->user_id = $data->user_id;
        $booking->start_date = $data->start_date;
        $booking->end_date = $data->end_date;
        $booking->status = $data->status;
        $booking->notes = $data->notes;
        $booking->created_at = $data->created_at;
        $booking->updated_at = $data->updated_at;
        
        return $booking;
    }
}
```

**Best for:**
- Applications with high performance requirements
- Complex queries with many joins or conditions
- When you need explicit control over the SQL being generated
- Larger datasets where Eloquent's overhead becomes noticeable

## Service Providers

Each domain has its own service provider to register dependencies:

```php
// app/Booking/IO/BookingServiceProvider.php
namespace App\Booking\IO;

use App\Booking\UseCases\Repositories\BookingRepositoryInterface;
use App\Booking\IO\Database\Repositories\EloquentBookingRepository;
use Illuminate\Support\ServiceProvider;

class BookingServiceProvider extends ServiceProvider
{
    public function register()
    {
        // Register repository implementations
        $this->app->bind(
            BookingRepositoryInterface::class,
            EloquentBookingRepository::class
        );
        
        // Register other bindings...
    }
    
    public function boot()
    {
        // Load routes, views, etc.
    }
}
```

## Consistent Error Handling Across Interfaces

When your application has multiple entry points (HTTP, CLI, etc.), handling exceptions consistently becomes challenging. The presenter pattern solves this problem:

```php
// app/Booking/UseCases/CancelBooking/CancelBookingPresenterInterface.php
namespace App\Booking\UseCases\CancelBooking;

use App\Booking\Entities\Booking;

interface CancelBookingPresenterInterface
{
    public function presentSuccess(Booking $booking): mixed;
    public function presentNotFound(string $bookingId): mixed;
    public function presentCannotCancel(string $reason): mixed;
}

// app/Booking/UseCases/CancelBooking.php
namespace App\Booking\UseCases;

use App\Booking\Entities\Booking;
use App\Booking\UseCases\Repositories\BookingRepositoryInterface;
use App\Booking\UseCases\CancelBooking\CancelBookingPresenterInterface;

class CancelBooking
{
    private $bookingRepository;
    
    public function __construct(BookingRepositoryInterface $bookingRepository)
    {
        $this->bookingRepository = $bookingRepository;
    }
    
    public function execute(string $bookingId, CancelBookingPresenterInterface $presenter)
    {
        $booking = $this->bookingRepository->findById($bookingId);
        
        if (!$booking) {
            return $presenter->presentNotFound($bookingId);
        }
        
        if (!$booking->canBeCancelled()) {
            return $presenter->presentCannotCancel('Booking is not in an active state or has already started');
        }
        
        $booking->cancel();
        $booking = $this->bookingRepository->save($booking);
        
        return $presenter->presentSuccess($booking);
    }
}
```

Now you can implement different presenters for different interfaces:

```php
// app/Booking/Adapters/Presenters/CancelBookingJsonPresenter.php
namespace App\Booking\Adapters\Presenters;

use App\Booking\Entities\Booking;
use App\Booking\UseCases\CancelBooking\CancelBookingPresenterInterface;

class CancelBookingJsonPresenter implements CancelBookingPresenterInterface
{
    public function presentSuccess(Booking $booking): array
    {
        return [
            'success' => true,
            'data' => [
                'id' => $booking->id,
                'status' => 'cancelled',
                'cancelled_at' => $booking->cancelled_at->format('Y-m-d H:i:s'),
            ]
        ];
    }
    
    public function presentNotFound(string $bookingId): array
    {
        return [
            'success' => false,
            'error' => [
                'code' => 'booking_not_found',
                'message' => "Booking {$bookingId} not found"
            ]
        ];
    }
    
    public function presentCannotCancel(string $reason): array
    {
        return [
            'success' => false,
            'error' => [
                'code' => 'booking_cannot_cancel',
                'message' => $reason
            ]
        ];
    }
}

// app/Booking/Adapters/Presenters/CancelBookingCliPresenter.php
namespace App\Booking\Adapters\Presenters;

use App\Booking\Entities\Booking;
use App\Booking\UseCases\CancelBooking\CancelBookingPresenterInterface;

class CancelBookingCliPresenter implements CancelBookingPresenterInterface
{
    public function presentSuccess(Booking $booking): string
    {
        return "✓ Booking {$booking->id} has been successfully cancelled.";
    }
    
    public function presentNotFound(string $bookingId): string
    {
        return "✗ Error: Booking {$bookingId} not found.";
    }
    
    public function presentCannotCancel(string $reason): string
    {
        return "✗ Error: Cannot cancel booking. {$reason}";
    }
}
```

These are used in the appropriate controllers/commands:

```php
// app/Booking/IO/Http/BookingController.php
namespace App\Booking\IO\Http;

use App\Booking\Adapters\Presenters\CancelBookingJsonPresenter;
use App\Booking\UseCases\CancelBooking;
use Illuminate\Http\JsonResponse;

class BookingController
{
    private $cancelBooking;
    
    public function __construct(CancelBooking $cancelBooking)
    {
        $this->cancelBooking = $cancelBooking;
    }
    
    public function cancel(string $id): JsonResponse
    {
        $presenter = new CancelBookingJsonPresenter();
        $result = $this->cancelBooking->execute($id, $presenter);
        
        $statusCode = isset($result['success']) && $result['success'] ? 200 : 
            (isset($result['error']['code']) && $result['error']['code'] === 'booking_not_found' ? 404 : 400);
            
        return response()->json($result, $statusCode);
    }
}

// app/Booking/IO/Console/CancelBookingCommand.php
namespace App\Booking\IO\Console;

use App\Booking\Adapters\Presenters\CancelBookingCliPresenter;
use App\Booking\UseCases\CancelBooking;
use Illuminate\Console\Command;

class CancelBookingCommand extends Command
{
    protected $signature = 'booking:cancel {id}';
    protected $description = 'Cancel a booking';
    
    private $cancelBooking;
    
    public function __construct(CancelBooking $cancelBooking)
    {
        parent::__construct();
        $this->cancelBooking = $cancelBooking;
    }
    
    public function handle()
    {
        $bookingId = $this->argument('id');
        $presenter = new CancelBookingCliPresenter();
        
        $result = $this->cancelBooking->execute($bookingId, $presenter);
        
        $this->line($result);
        
        return str_contains($result, '✓') ? 0 : 1; // Success or error exit code
    }
}
```

This approach ensures consistent handling of all outcomes across different interfaces while keeping the use case interface-agnostic.

## Testing Support

Each domain includes testing utilities:

```php
// app/Booking/Testing/BookingRepositoryMock.php
namespace App\Booking\Testing;

use App\Booking\Entities\Booking;
use App\Booking\UseCases\Repositories\BookingRepositoryInterface;

class BookingRepositoryMock implements BookingRepositoryInterface
{
    private array $bookings = [];
    private ?Booking $lastSavedBooking = null;
    
    public function findById(string $id): ?Booking
    {
        foreach ($this->bookings as $booking) {
            if ($booking->id == $id) {
                return $booking;
            }
        }
        
        return null;
    }
    
    public function save(Booking $booking): Booking
    {
        // Simulate auto-increment ID if not set
        if (empty($booking->id)) {
            $booking->id = count($this->bookings) + 1;
        }
        
        // Store the booking
        $this->lastSavedBooking = $booking;
        $this->bookings[] = $booking;
        
        return $booking;
    }
    
    // Helper methods for tests
    public function addBooking(Booking $booking): self
    {
        $this->bookings[] = $booking;
        return $this;
    }
    
    public function getLastSavedBooking(): ?Booking
    {
        return $this->lastSavedBooking;
    }
}
```

## Advanced Testing Tools

For more complex testing scenarios:

```php
// app/Booking/Testing/BookingTestAssertions.php
namespace App\Booking\Testing;

use App\Booking\Entities\Booking;
use PHPUnit\Framework\Assert;

class BookingTestAssertions
{
    /**
     * Assert that a booking has expected basic properties
     */
    public static function assertBookingHasProperties(
        Booking $booking, 
        string $userId, 
        string $startDate, 
        string $endDate, 
        string $status,
        ?string $notes = null
    ): void {
        Assert::assertEquals($userId, $booking->user_id, 'Booking has incorrect user ID');
        Assert::assertEquals($startDate, $booking->start_date, 'Booking has incorrect start date');
        Assert::assertEquals($endDate, $booking->end_date, 'Booking has incorrect end date');
        Assert::assertEquals($status, $booking->status, 'Booking has incorrect status');
        
        if ($notes !== null) {
            Assert::assertEquals($notes, $booking->notes, 'Booking has incorrect notes');
        }
    }

    /**
     * Assert that a booking was saved in the repository
     */
    public static function assertBookingWasSaved(
        BookingRepositoryMock $repository, 
        Booking $expectedBooking
    ): void {
        $savedBooking = $repository->getLastSavedBooking();
        Assert::assertNotNull($savedBooking, 'No booking was saved to the repository');
        Assert::assertSame($expectedBooking, $savedBooking, 'Different booking was saved than expected');
    }
}
```

## Advanced Integration: GraphQL Components

For applications using GraphQL:

```php
// app/Booking/IO/GraphQL/Mutations/CreateBookingMutation.php
namespace App\Booking\IO\GraphQL\Mutations;

use App\Booking\UseCases\CreateBooking;
use App\Booking\UseCases\CreateBooking\CreateBookingRequest;
use GraphQL\Type\Definition\ResolveInfo;
use Nuwave\Lighthouse\Support\Contracts\GraphQLContext;

class CreateBookingMutation
{
    private $createBooking;
    
    public function __construct(CreateBooking $createBooking)
    {
        $this->createBooking = $createBooking;
    }
    
    public function __invoke($rootValue, array $args, GraphQLContext $context, ResolveInfo $resolveInfo)
    {
        // Transform GraphQL input to use case input
        $input = CreateBookingRequest::from([
            'userId' => $args['input']['user_id'],
            'startDate' => $args['input']['start_date'],
            'endDate' => $args['input']['end_date'],
            'notes' => $args['input']['notes'] ?? null,
        ]);
        
        // Execute use case
        $booking = $this->createBooking->execute($input);
        
        // Return result (will be automatically transformed to GraphQL type)
        return $booking;
    }
}
```

## Model Factories (Advanced)

For test data generation:

```php
// database/factories/Booking/BookingFactory.php
namespace Database\Factories\Booking;

use App\Booking\Entities\Booking;
use Illuminate\Database\Eloquent\Factories\Factory;

class BookingFactory extends Factory
{
    protected $model = Booking::class;
    
    public function definition()
    {
        return [
            'user_id' => \Database\Factories\User\UserFactory::new(),
            'start_date' => $this->faker->dateTimeBetween('+1 week', '+2 weeks'),
            'end_date' => $this->faker->dateTimeBetween('+3 weeks', '+4 weeks'),
            'status' => $this->faker->randomElement(['pending', 'confirmed', 'cancelled']),
        ];
    }
    
    // Factory states
    public function confirmed()
    {
        return $this->state(['status' => 'confirmed']);
    }
}
```

## Benefits

1. **Clear Domain Focus**: The structure clearly communicates what the application does
2. **Maintainability**: Co-located specs make it easy to find and update tests
3. **Autonomy**: Each domain can evolve independently
4. **Scalability**: New domains can be added without affecting existing ones
5. **Clean Dependencies**: Dependencies flow inward, maintaining architectural integrity

This structure is designed to support large, complex applications while keeping the codebase maintainable, testable, and aligned with business domains.
