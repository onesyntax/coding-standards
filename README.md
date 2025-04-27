# Laravel Clean Architecture Project Structure: A Pragmatic Approach

This document outlines a comprehensive project structure for Laravel applications following Clean Architecture principles with a domain-driven design approach.

## Key Principles

1. **Domain-Focused Organization**: Top-level directories represent business domains rather than technical concerns
2. **Clean Architecture Layers**: Each domain follows the standard Clean Architecture layers
3. **Co-located Specifications**: Tests reside with the code they verify
4. **Testing Support**: Each domain includes testing support utilities
5. **Explicit IO Layer**: Clear separation of external integrations and frameworks

## Directory Structure

```
project-root/
├── app/
│   ├── Foundation/               # Shared kernel, base classes, traits
│   │   ├── Entities/             # Base entities, models, value objects, shared domain logic
│   │   ├── UseCases/             # Base interactors, shared application services
│   │   ├── Adapters/             # Base presenters, view models, framework-agnostic components
│   │   ├── IO/                   # Shared IO components
│   │       ├── Database/         # Common database concerns, base repositories
│   │       ├── Http/             # Shared controllers, middleware, base API resources
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
│   │       ├── Database/         # Booking-specific database migrations and seeders
│   │       ├── Http/             # Booking-specific HTTP interfaces, controllers
│   │       ├── Web/              # Booking-specific UI elements
│   │       ├── GraphQL/          # Booking-specific GraphQL components
│   │       └── ExternalServices/ # Booking-specific external services
│   │       ├── BookingServiceProvider.php      # Booking domain service provider
│   │   ├── Specs/                # Booking behavior specifications
│   │   └── Testing/              # Booking-specific testing utilities
│   │
│   ├── Payment/                  # Payment domain module
│   │   ├── Entities/             # Payment-specific domain entities and models
│   │   ├── UseCases/             # Payment-specific application logic
│   │       ├── Repositories/     # Repository interfaces
│   │   ├── Adapters/             # Framework-agnostic interface adapters
│   │   ├── IO/                   # Payment-specific frameworks and drivers
│   │       ├── Database/         # Payment-specific database migrations and seeders
│   │       ├── Http/             # Payment-specific HTTP interfaces, controllers
│   │       ├── Web/              # Payment-specific UI elements
│   │       ├── GraphQL/          # Payment-specific GraphQL components
│   │       └── ExternalServices/ # Payment gateways, providers
│   │       ├── PaymentServiceProvider.php      # Payment domain service provider
│   │   ├── Specs/                # Payment behavior specifications
│   │   └── Testing/              # Payment-specific testing utilities
│   │
│   └── User/                     # User domain module
│       ├── Entities/             # User-specific domain entities and models
│       ├── UseCases/             # User-specific application logic
│       │   ├── Repositories/     # Repository interfaces
│       ├── Adapters/             # Framework-agnostic interface adapters
│       ├── IO/                   # User-specific frameworks and drivers
│           ├── Database/         # User-specific database migrations and seeders
│           ├── Http/             # User-specific HTTP interfaces, controllers
│           ├── Web/              # User-specific UI elements
│           ├── GraphQL/          # User-specific GraphQL components
│           └── ExternalServices/ # Authentication services, etc.
│           ├── UserServiceProvider.php         # User domain service provider
│       ├── Specs/                # User behavior specifications
│       └── Testing/              # User-specific testing utilities
```

## Layer Descriptions

### Entities Layer

Contains the enterprise-wide business rules and entities:
- Domain entities (Eloquent models)
- Value objects
- Domain events
- Business rule validators

While this is a pragmatic departure from pure clean architecture, placing Eloquent models in the Entities layer allows for a more practical implementation while maintaining domain organization. Models should encapsulate both data structure and domain behavior.

### UseCases Layer

Contains application-specific business rules:
- Interactors/services
- Command handlers
- Repository interfaces
- DTOs (Data Transfer Objects)

This layer can depend on the Entities layer but not on outer layers.

#### Request/Response Models with spatie/laravel-data

For input and output models in the UseCases layer, the `spatie/laravel-data` package provides an excellent solution for creating strongly-typed DTOs:

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

You have two options for handling responses from use cases:

**Option 1: Return entities directly (simpler approach)**

```php
// app/Booking/UseCases/CreateBooking.php
namespace App\Booking\UseCases;

use App\Booking\Entities\Booking;
use App\Booking\UseCases\CreateBooking\CreateBookingRequest;

class CreateBooking
{
    public function execute(CreateBookingRequest $request): Booking
    {
        $booking = new Booking();
        $booking->user_id = $request->userId;
        $booking->start_date = $request->startDate;
        $booking->end_date = $request->endDate;
        $booking->notes = $request->notes;
        $booking->status = 'pending';
        $booking->save();
        
        return $booking;
    }
}
```

**Option 2: Use response DTOs (when transformation is needed)**

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

// app/Booking/UseCases/CreateBooking.php
namespace App\Booking\UseCases;

use App\Booking\Entities\Booking;
use App\Booking\UseCases\CreateBooking\CreateBookingRequest;
use App\Booking\UseCases\CreateBooking\CreateBookingResponse;

class CreateBooking
{
    public function execute(CreateBookingRequest $request): CreateBookingResponse
    {
        $booking = new Booking();
        $booking->user_id = $request->userId;
        $booking->start_date = $request->startDate;
        $booking->end_date = $request->endDate;
        $booking->notes = $request->notes;
        $booking->status = 'pending';
        $booking->save();
        
        return CreateBookingResponse::from($booking);
    }
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

### Adapters Layer

Contains interface adapters that convert data between the use cases and external formats:
- Presenters
- View Models
- Data transformers 
- Gateway interfaces (when needed beyond what's in the Use Cases layer)

This layer can depend on UseCases but not on the IO layer.

#### Example: Framework-Independent Presenters

The Adapters layer should contain components that are framework-agnostic and serve as a bridge between use cases and external interfaces.

**1. Presenter Interface in Use Cases Layer:**

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

**2. Use Case with Presenter:**

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

**3. Presenter Implementation in Adapters Layer:**

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

**4. Controller in IO Layer Using the Presenter:**

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
    
    // Other controller methods...
}
```

In this implementation, the Adapters layer contains only framework-agnostic components (like presenters), while framework-specific components (controllers, resources) are properly placed in the IO layer. This maintains the clean architecture principle of keeping the inner layers independent of frameworks and external tools.

### IO Layer

Contains frameworks, drivers, and external services:
- Controllers and HTTP resources
- Database implementations
- Web interfaces
- External API integrations (REST, GraphQL)
- Framework-specific code

This is the outermost layer and can depend on all inner layers.

## Service Providers

Laravel Service Providers are organized according to their domain and purpose:

```
project-root/
├── app/
│   ├── Foundation/               
│   │   ├── ...
│   │   ├── IO/                   
│   │       ├── FoundationServiceProvider.php  # Core app-wide service provider
│   │       ├── Http/             
│   │           ├── RouteServiceProvider.php    # Framework-level route provider
│   │
│   ├── Booking/                  
│   │   ├── ...
│   │   ├── IO/                   
│   │       ├── BookingServiceProvider.php      # Booking domain service provider
│   │       ├── ...
│   │
│   ├── Payment/                  
│   │   ├── ...
│   │   ├── IO/                   
│   │       ├── PaymentServiceProvider.php      # Payment domain service provider
│   │       ├── ...
│   │
│   └── User/                     
│       ├── ...
│       ├── IO/                   
│           ├── UserServiceProvider.php         # User domain service provider
│           ├── ...
```

Each domain's service provider is responsible for:
- Registering repositories and interfaces
- Loading domain-specific routes
- Setting up event listeners
- Registering domain-specific middleware

The `FoundationServiceProvider` handles application-wide concerns and bootstraps shared components.

These providers must then be registered in the `config/app.php` providers array.

## Repository Pattern

In Clean Architecture, the repository interfaces should be defined by the Use Cases layer, as they represent the "ports" through which the application core communicates with the outside world.

**1. Repository Interface in Use Cases Layer:**

```php
// app/Booking/UseCases/Repositories/BookingRepositoryInterface.php
namespace App\Booking\UseCases\Repositories;

use App\Booking\Entities\Booking;

interface BookingRepositoryInterface
{
    public function findById(string $id): ?Booking;
    public function findActiveByDateRange(string $userId, \DateTime $startDate, \DateTime $endDate): array;
    public function save(Booking $booking): Booking;
    public function delete(string $id): bool;
}
```

**2. Use Case Using Repository:**

```php
// app/Booking/UseCases/CreateBooking.php
namespace App\Booking\UseCases;

use App\Booking\UseCases\Repositories\BookingRepositoryInterface;
use App\Booking\Entities\Booking;
use App\Booking\UseCases\CreateBooking\CreateBookingRequest;
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

**3. Repository Implementation in IO Layer:**

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
        // Use DB::table for consistent approach across repository methods
        $bookingData = DB::table('bookings')->where('id', $id)->first();
        
        if (!$bookingData) {
            return null;
        }
        
        return $this->mapToEntity($bookingData);
    }
    
    public function findActiveByDateRange(string $userId, DateTime $startDate, DateTime $endDate): array
    {
        // For complex queries, DB::table offers better performance
        $bookingsData = DB::table('bookings')
            ->where('user_id', $userId)
            ->where('status', 'active')
            ->where(function ($query) use ($startDate, $endDate) {
                $query->whereBetween('start_date', [$startDate->format('Y-m-d'), $endDate->format('Y-m-d')])
                    ->orWhereBetween('end_date', [$startDate->format('Y-m-d'), $endDate->format('Y-m-d')])
                    ->orWhere(function ($query) use ($startDate, $endDate) {
                        $query->where('start_date', '<=', $startDate->format('Y-m-d'))
                              ->where('end_date', '>=', $endDate->format('Y-m-d'));
                    });
            })
            ->get();
            
        // Map DB results to domain entities
        return $bookingsData->map(function ($bookingData) {
            return $this->mapToEntity($bookingData);
        })->all();
    }
    
    public function delete(string $id): bool
    {
        return DB::table('bookings')->where('id', $id)->delete() > 0;
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

**4. Service Provider Registration:**

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
    
    // Other service provider methods...
}
```

In this implementation, the repository interface is defined by the Use Cases layer, which better aligns with Clean Architecture principles. The Use Cases specify what they need from the outside world, and the IO layer provides the implementation. This maintains the dependency rule: dependencies point inward, with the Use Cases defining the interfaces and the outer layers implementing them.

## Model Factories

There are two approaches to organizing Model Factories:

### Approach 1: Laravel Conventional Structure

```
project-root/
├── database/
│   └── factories/
│       ├── Booking/                 # Booking domain factories
│       │   └── BookingFactory.php
│       ├── Payment/                 # Payment domain factories
│       │   └── TransactionFactory.php
│       └── User/                    # User domain factories
│           └── UserFactory.php
```

This approach maintains Laravel's conventional structure while still organizing factories by domain. It preserves the dependency rule - entities don't need to know about factories, but factories can know about entities.

Example implementation:

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

An alternative approach is to place model factories directly within the domain structure:

```
project-root/
├── app/
│   ├── Booking/
│   │   ├── Entities/
│   │   │   ├── Models/             # Domain models
│   │   │   │   ├── Booking.php
│   │   │   │   └── Factories/      # Model factories
│   │   │   │       └── BookingFactory.php
```

Example implementation:

```php
// app/Booking/Entities/Models/Factories/BookingFactory.php

namespace App\Booking\Entities\Models\Factories;

use App\Booking\Entities\Models\Booking;
use Illuminate\Database\Eloquent\Factories\Factory;

class BookingFactory extends Factory
{
    protected $model = Booking::class;
    
    public function definition()
    {
        return [
            'user_id' => \App\User\Entities\Models\Factories\UserFactory::new(),
            'start_date' => $this->faker->dateTimeBetween('+1 week', '+2 weeks'),
            'end_date' => $this->faker->dateTimeBetween('+3 weeks', '+4 weeks'),
            'status' => $this->faker->randomElement(['pending', 'confirmed', 'cancelled']),
        ];
    }
}
```

And in the model:

```php
// app/Booking/Entities/Models/Booking.php

namespace App\Booking\Entities\Models;

use App\Booking\Entities\Models\Factories\BookingFactory;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Booking extends Model
{
    use HasFactory;
    
    // Factory connection
    protected static function newFactory()
    {
        return BookingFactory::new();
    }
    
    // Domain methods...
}
```

This approach creates tighter integration between models and factories but introduces a bidirectional dependency.

## GraphQL Components

GraphQL components (using Lighthouse or other GraphQL implementations) are placed in the IO layer of each domain:

```
project-root/
├── app/
│   ├── Foundation/
│   │   ├── IO/
│   │   │   ├── GraphQL/          # Shared GraphQL components
│   │   │   │   ├── Directives/   # Custom directives
│   │   │   │   ├── Scalars/      # Custom scalar types
│   │   │   │   └── Interfaces/   # Shared interfaces
│   │
│   ├── Booking/
│   │   ├── IO/
│   │   │   ├── GraphQL/          # GraphQL components for Booking domain
│   │   │   │   ├── Queries/      # Query resolvers
│   │   │   │   ├── Mutations/    # Mutation resolvers
│   │   │   │   ├── Types/        # Object type definitions
│   │   │   │   └── Unions/       # Union type definitions
│   │
│   ├── Payment/
│   │   ├── IO/
│   │   │   ├── GraphQL/          # GraphQL components for Payment domain
│   │   │   │   ├── Queries/
│   │   │   │   ├── Mutations/
│   │   │   │   ├── Types/
│   │   │   │   └── Unions/
│   │
│   └── User/
│       ├── IO/
│       │   ├── GraphQL/          # GraphQL components for User domain
│       │   │   ├── Queries/
│       │   │   ├── Mutations/
│       │   │   ├── Types/
│       │   │   └── Unions/
```

### Example Implementation

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

This placement is appropriate because GraphQL components are part of your application's external interface layer, similar to REST controllers. They handle the translation between your domain's internal structures and the GraphQL schema that's exposed to clients.

In your Lighthouse schema, you would reference these classes:

```graphql
type Mutation {
    createBooking(input: CreateBookingInput!): Booking! @field(resolver: "App\\Booking\\IO\\GraphQL\\Mutations\\CreateBookingMutation")
}
```

The GraphQL namespace in the IO layer of each domain maintains clean architecture principles while properly categorizing GraphQL as an IO concern.

## Example Implementation

Here's an example of how models would fit in the Entities layer:

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

// app/Booking/UseCases/CancelBooking.php
namespace App\Booking\UseCases;

use App\Booking\Entities\Booking;

class CancelBooking
{
    public function execute(string $bookingId)
    {
        $booking = Booking::findOrFail($bookingId);
        
        if (!$booking->canBeCancelled()) {
            throw new BookingCancellationException('Booking cannot be cancelled');
        }
        
        $booking->cancel();
        $booking->save();
        
        return $booking;
    }
}
```

## Specifications

Each domain includes a `Specs` directory containing behavior specifications for that domain. Specs serve as both tests and living documentation, describing the expected behavior of components.

## Testing Support

Each domain includes a `Testing` directory that provides utilities to support testing:
- Mock implementations
- Testing helpers
- Fixtures and datasets

### Example Testing API Implementation

**1. Repository Mock (Basic Example)**

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

This example demonstrates how to test a CreateBooking use case by:
1. Creating a mock repository that captures the booking being saved
2. Setting up a request DTO with test data
3. Executing the use case
4. Verifying that the booking was created with the correct attributes
5. Confirming that the repository's save method was called with the right data

The test focuses on the behavior of the use case rather than the implementation details, ensuring that it correctly transforms the input data and interacts with the repository as expected.

## Benefits

1. **Clear Domain Focus**: The structure clearly communicates what the application does
2. **Maintainability**: Co-located specs make it easy to find and update tests
3. **Autonomy**: Each domain can evolve independently
4. **Scalability**: New domains can be added without affecting existing ones
5. **Clean Dependencies**: Dependencies flow inward, maintaining architectural integrity

This structure is designed to support large, complex applications while keeping the codebase maintainable, testable, and aligned with business domains.
