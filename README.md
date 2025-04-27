# Laravel Clean Architecture: A Practical Implementation Guide

## Introduction

This guide presents a pragmatic approach to implementing Clean Architecture principles within Laravel applications using a domain-driven design. It offers a balanced solution that respects both Laravel's conventions and Clean Architecture's separation of concerns, resulting in a codebase that is both maintainable and aligned with business domains.

## Core Principles

The architecture described here is built on several key principles:

1. **Domain-Focused Organization** - Applications are structured around business domains rather than technical concerns, making the codebase's purpose immediately clear.

2. **Clean Architecture Layers** - Each domain follows the standard Clean Architecture layers, ensuring proper separation of concerns and dependencies pointing inward.

3. **Co-located Specifications** - Tests reside with the code they verify, making it easier to understand component behavior and maintain tests alongside code changes.

4. **Testing Support** - Each domain includes testing support utilities, encouraging comprehensive testing practices.

5. **Explicit IO Layer** - External integrations and framework-specific code are clearly separated from business logic, making the system more adaptable to change.

## Directory Structure Overview

The structure organizes code by business domains at the top level, with each domain containing the four Clean Architecture layers:

```
project-root/
├── app/
│   ├── Foundation/               # Shared kernel, base classes
│   ├── Booking/                  # Booking domain module
│   ├── Payment/                  # Payment domain module
│   └── User/                     # User domain module
```

Each domain module follows a consistent structure with Clean Architecture layers:

```
app/Booking/
├── Entities/             # Domain entities and business rules
├── UseCases/            # Application business logic
│   ├── Repositories/    # Repository interfaces
├── Adapters/            # Framework-agnostic interface adapters
├── IO/                  # Frameworks, drivers, external services
│   ├── Database/        # Database-specific components
│   ├── Http/            # HTTP interfaces
│   ├── Web/             # UI components
│   ├── GraphQL/         # GraphQL components
│   └── ExternalServices/ # External service integrations
├── Specs/               # Domain behavior specifications
└── Testing/             # Domain-specific testing utilities
```

## Clean Architecture Layers in Detail

### 1. Entities Layer

The Entities layer contains the enterprise-wide business rules and core business models:

- Domain entities (Eloquent models)
- Value objects
- Domain events
- Business rule validators

**Example: Booking Entity**

```php
// app/Booking/Entities/Booking.php
namespace App\Booking\Entities;

use Illuminate\Database\Eloquent\Model;
use App\User\Entities\User;

class Booking extends Model
{
    protected $fillable = ['user_id', 'start_date', 'end_date', 'status'];

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

> **Note**: While pure Clean Architecture would keep this layer framework-agnostic, this pragmatic approach uses Eloquent models to balance architectural purity with development efficiency.

### 2. UseCases Layer

The UseCases layer contains application-specific business rules:

- Interactors/services
- Command handlers
- Repository interfaces
- DTOs (Data Transfer Objects)

This layer orchestrates the flow of data through the application and should be independent of frameworks and external tools.

**Example: Creating DTOs with spatie/laravel-data**

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

**Example: Use Case implementation**

```php
// app/Booking/UseCases/CreateBooking.php
namespace App\Booking\UseCases;

use App\Booking\Entities\Booking;
use App\Booking\UseCases\CreateBooking\CreateBookingRequest;
use App\Booking\UseCases\Repositories\BookingRepositoryInterface;
use App\Booking\UseCases\Exceptions\BookingConflictException;

class CreateBooking
{
    private $bookingRepository;

    public function __construct(BookingRepositoryInterface $bookingRepository)
    {
        $this->bookingRepository = $bookingRepository;
    }

    public function execute(CreateBookingRequest $request): Booking
    {
        // Check for conflicts
        $existingBookings = $this->bookingRepository->findActiveByDateRange(
            $request->userId,
            new \DateTime($request->startDate),
            new \DateTime($request->endDate)
        );

        if (count($existingBookings) > 0) {
            throw new BookingConflictException('User already has a booking in this date range');
        }

        // Create new booking
        $booking = new Booking();
        $booking->user_id = $request->userId;
        $booking->start_date = $request->startDate;
        $booking->end_date = $request->endDate;
        $booking->notes = $request->notes;
        $booking->status = 'pending';

        // Persist
        return $this->bookingRepository->save($booking);
    }
}
```

#### Response Handling Strategies

When implementing use cases, you have two options for handling responses:

**Option 1: Return entities directly** (simpler approach)
```php
public function execute(CreateBookingRequest $request): Booking
{
    // Implementation logic
    return $booking;
}
```

**Option 2: Use response DTOs** (when transformation is needed)
```php
// app/Booking/UseCases/CreateBooking/CreateBookingResponse.php
namespace App\Booking\UseCases\CreateBooking;

use Spatie\LaravelData\Data;

class CreateBookingResponse extends Data
{
    public string $id;
    public string $userId;
    public string $startDate;
    public string $endDate;
    public string $status;
    public ?string $notes;
}

// In the use case:
public function execute(CreateBookingRequest $request): CreateBookingResponse
{
    // Implementation logic
    return CreateBookingResponse::from($booking);
}
```

Use response DTOs when:
- You need to transform data before returning (renaming fields, changing formats)
- You want to hide certain properties or add computed values
- You need to combine data from multiple entities
- You want explicit control over the response structure

Return entities directly when:
- The entity already contains all needed data in the right format
- You want to keep the code simpler
- The entity structure matches the expected response structure

### 3. Adapters Layer

The Adapters layer contains interface adapters that convert data between the use cases and external formats. This layer serves as a bridge between the application's core business logic and how it interacts with the outside world.

Key responsibilities include:
- Data transformation
- Interface adaptation
- Presentation formatting

**Example: Framework-Independent Presenter**

First, define the presenter interface in the Use Cases layer:

```php
// app/Booking/UseCases/ViewBooking/BookingPresenterInterface.php
namespace App\Booking\UseCases\ViewBooking;

use App\Booking\Entities\Booking;

interface BookingPresenterInterface
{
    public function presentSuccess(Booking $booking): mixed;
    public function presentFailure(string $message): mixed;
}
```

Then implement the use case that uses this presenter:

```php
// app/Booking/UseCases/ViewBooking.php
namespace App\Booking\UseCases;

use App\Booking\Entities\Booking;
use App\Booking\UseCases\Repositories\BookingRepositoryInterface;
use App\Booking\UseCases\ViewBooking\BookingPresenterInterface;

class ViewBooking
{
    private $bookingRepository;

    public function __construct(BookingRepositoryInterface $bookingRepository)
    {
        $this->bookingRepository = $bookingRepository;
    }

    public function execute(string $bookingId, BookingPresenterInterface $presenter)
    {
        $booking = $this->bookingRepository->findById($bookingId);

        if (!$booking) {
            return $presenter->presentFailure("Booking not found: {$bookingId}");
        }

        return $presenter->presentSuccess($booking);
    }
}
```

Finally, implement the presenter in the Adapters layer:

```php
// app/Booking/Adapters/Presenters/BookingJsonPresenter.php
namespace App\Booking\Adapters\Presenters;

use App\Booking\Entities\Booking;
use App\Booking\UseCases\ViewBooking\BookingPresenterInterface;

class BookingJsonPresenter implements BookingPresenterInterface
{
    public function presentSuccess(Booking $booking): array
    {
        return [
            'success' => true,
            'data' => [
                'id' => $booking->id,
                'user' => [
                    'id' => $booking->user_id,
                    'name' => $booking->user->name ?? 'Unknown',
                ],
                'period' => [
                    'start' => $booking->start_date->format('Y-m-d'),
                    'end' => $booking->end_date->format('Y-m-d'),
                ],
                'status' => $booking->status,
                'is_cancellable' => $booking->canBeCancelled(),
                'created_at' => $booking->created_at->format('Y-m-d H:i:s'),
            ]
        ];
    }

    public function presentFailure(string $message): array
    {
        return [
            'success' => false,
            'error' => [
                'message' => $message
            ]
        ];
    }
}
```

### 4. IO Layer

The IO layer contains frameworks, drivers, and external services:

- Controllers and HTTP resources
- Database implementations
- Web interfaces
- External API integrations
- Framework-specific code

This is the outermost layer and can depend on all inner layers. Code in this layer is typically tightly coupled to a specific framework or external system.

**Example: Controller using the Presenter**

```php
// app/Booking/IO/Http/BookingController.php
namespace App\Booking\IO\Http;

use App\Booking\Adapters\Presenters\BookingJsonPresenter;
use App\Booking\UseCases\ViewBooking;
use Illuminate\Http\JsonResponse;

class BookingController
{
    private $viewBooking;

    public function __construct(ViewBooking $viewBooking)
    {
        $this->viewBooking = $viewBooking;
    }

    public function show(string $id): JsonResponse
    {
        $presenter = new BookingJsonPresenter();
        $result = $this->viewBooking->execute($id, $presenter);

        $statusCode = $result['success'] ? 200 : 404;
        return response()->json($result, $statusCode);
    }
}
```

## Service Providers

Laravel Service Providers are organized according to their domain and purpose:

```
app/Foundation/IO/FoundationServiceProvider.php  # Core app-wide service provider
app/Booking/IO/BookingServiceProvider.php        # Booking domain service provider
app/Payment/IO/PaymentServiceProvider.php        # Payment domain service provider
app/User/IO/UserServiceProvider.php              # User domain service provider
```

Each domain's service provider is responsible for:
- Registering repositories and interfaces
- Loading domain-specific routes
- Setting up event listeners
- Registering domain-specific middleware

**Example: Domain Service Provider**

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
        // Load routes
        $this->loadRoutesFrom(__DIR__ . '/routes.php');
        
        // Register event listeners
        $this->app['events']->listen(
            'App\Booking\Entities\Events\BookingCreated',
            'App\Booking\IO\Listeners\SendBookingConfirmation'
        );
        
        // Other boot operations...
    }
}
```

## Repository Pattern Implementation

In Clean Architecture, repository interfaces are defined by the Use Cases layer, while implementations live in the IO layer.

**Repository Interface (Use Cases Layer)**

```php
// app/Booking/UseCases/Repositories/BookingRepositoryInterface.php
namespace App\Booking\UseCases\Repositories;

use App\Booking\Entities\Booking;

interface BookingRepositoryInterface
{
    public function findById(string $id): ?Booking;
    public function delete(string $id): bool;
}
```

**Repository Implementation (IO Layer)**

```php
// app/Booking/IO/Database/Repositories/EloquentBookingRepository.php
namespace App\Booking\IO\Database\Repositories;

use App\Booking\UseCases\Repositories\BookingRepositoryInterface;
use App\Booking\Entities\Booking;
use Illuminate\Support\Facades\DB;
use DateTime;

class EloquentBookingRepository implements BookingRepositoryInterface
{
    public function findById(string $id): ?Booking
    {
        $bookingData = DB::table('bookings')->where('id', $id)->first();

        return $bookingData ? $this->mapToEntity($bookingData) : null;
    }

    public function delete(string $id): bool
    {
        return DB::table('bookings')->where('id', $id)->delete() > 0;
    }

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

## GraphQL Implementation

GraphQL components are placed in the IO layer of each domain:

```
app/Foundation/IO/GraphQL/        # Shared GraphQL components
app/Booking/IO/GraphQL/           # Booking domain GraphQL components
app/Payment/IO/GraphQL/           # Payment domain GraphQL components
app/User/IO/GraphQL/              # User domain GraphQL components
```

**Example: GraphQL Mutation Implementation**

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

In your Lighthouse schema:
```graphql
type Mutation {
    createBooking(input: CreateBookingInput!): Booking! 
      @field(resolver: "App\\Booking\\IO\\GraphQL\\Mutations\\CreateBookingMutation")
}
```

## Model Factories

There are two approaches to organizing Model Factories:

### Approach 1: Laravel Conventional Structure (Recommended)

```
database/
└── factories/
    ├── Booking/                 # Booking domain factories
    │   └── BookingFactory.php
    ├── Payment/                 # Payment domain factories
    │   └── TransactionFactory.php
    └── User/                    # User domain factories
        └── UserFactory.php
```

**Example implementation:**

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

### Approach 2: Domain-Centric Model Factories

```
app/
├── Booking/
│   ├── Entities/
│   │   ├── Models/             # Domain models
│   │   │   ├── Booking.php
│   │   │   └── Factories/      # Model factories
│   │   │       └── BookingFactory.php
```

This approach creates tighter integration between models and factories but introduces a bidirectional dependency. The Laravel conventional approach is generally preferred in Clean Architecture to maintain a clearer separation of concerns.

## Testing Strategy

Each domain includes its own testing support and specifications:

### Specifications (Specs)

```
app/Booking/Specs/                # Booking behavior specifications
app/Payment/Specs/                # Payment behavior specifications
app/User/Specs/                   # User behavior specifications
```

These specifications serve as both tests and living documentation, describing the expected behavior of components.

### Testing Support

```
app/Booking/Testing/              # Booking-specific testing utilities
app/Payment/Testing/              # Payment-specific testing utilities
app/User/Testing/                 # User-specific testing utilities
```

**Example: Repository Mock**

```php
// app/Booking/Testing/BookingRepositoryMock.php
namespace App\Booking\Testing;

use App\Booking\Entities\Booking;
use App\Booking\UseCases\Repositories\BookingRepositoryInterface;
use DateTime;

class BookingRepositoryMock implements BookingRepositoryInterface
{
    private array $bookings = [];
    
    public function findById(string $id): ?Booking
    {
        foreach ($this->bookings as $booking) {
            if ($booking->id == $id) {
                return $booking;
            }
        }
        return null;
    }
    
    // Helper method for tests
    public function addBooking(Booking $booking): self
    {
        $this->bookings[] = $booking;
        return $this;
    }
    
    // Implement other required methods...
}
```

**Example: Testing a Use Case**

```php
// app/Booking/Specs/CreateBookingSpec.php
namespace App\Booking\Specs;

use App\Booking\Entities\Booking;
use App\Booking\Testing\BookingRepositoryMock;
use App\Booking\UseCases\CreateBooking;
use App\Booking\UseCases\CreateBooking\CreateBookingRequest;
use Tests\TestCase;

class CreateBookingSpec extends TestCase
{
    private BookingRepositoryMock $repositoryMock;

    protected function setUp(): void
    {
        parent::setUp();
        $this->repositoryMock = new BookingRepositoryMock();
    }

    /** @test */
    public function it_creates_a_new_booking()
    {
        // Arrange
        $useCase = new CreateBooking($this->repositoryMock);

        // Create a request DTO
        $request = new CreateBookingRequest();
        $request->userId = '456';
        $request->startDate = '2025-05-01';
        $request->endDate = '2025-05-03';
        $request->notes = 'Business trip';

        // Act
        $result = $useCase->execute($request);

        // Assert
        $this->assertBookingCreated($result, $request);
    }

    private function assertBookingCreated(Booking $booking, CreateBookingRequest $request): void
    {
        $this->assertInstanceOf(Booking::class, $booking);
        $this->assertEquals($request->userId, $booking->user_id);
        $this->assertEquals($request->startDate, $booking->start_date);
        $this->assertEquals($request->endDate, $booking->end_date);
        $this->assertEquals($request->notes, $booking->notes);
        $this->assertEquals('pending', $booking->status);
    }
}
```

## Benefits of This Architecture

1. **Clear Domain Focus**: The structure clearly communicates what the application does by organizing around business domains.

2. **Maintainability**: Co-located specifications make it easy to find and update tests alongside the code they verify.

3. **Autonomy**: Each domain can evolve independently, allowing for specialized teams and parallel development.

4. **Scalability**: New domains can be added without affecting existing ones as the application grows.

5. **Clean Dependencies**: Dependencies flow inward, maintaining architectural integrity and making the system more resistant to change.

6. **Testability**: The separation of concerns and clear interfaces make the system highly testable at all levels.

7. **Business Alignment**: The structure reflects the business domains, making it easier for non-technical stakeholders to understand the system's organization.

## Conclusion

This Laravel Clean Architecture approach combines the best of both worlds: the productivity of Laravel with the maintainability of Clean Architecture. By organizing code around business domains rather than technical concerns, the architecture creates a clear mental model of the application while still benefiting from Laravel's extensive ecosystem.

The pragmatic adaptations, such as using Eloquent models in the Entities layer, strike a balance between architectural purity and development efficiency, making this approach suitable for real-world projects of various sizes and complexities.
