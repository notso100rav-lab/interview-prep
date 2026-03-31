# Chapter 3: Low-Level Design

> **Estimated study time:** 3 days | **Priority:** 🟡 Medium-High

---

## Key Concepts

- Object-Oriented Design (OOD) methodology
- UML class diagrams
- SOLID principles in practice
- Common OOP design problems: parking lot, elevator, library, vending machine
- Design pattern application

---

## OOD Methodology

For every LLD problem, follow this structure:

1. **Clarify requirements** — What are the core use cases? What is out of scope?
2. **Identify entities** — What are the main objects/nouns in the domain?
3. **Define relationships** — Inheritance, composition, association, aggregation
4. **Apply design patterns** — Which patterns solve the recurring sub-problems?
5. **Write interfaces and key classes** — Focus on public API, not implementation details
6. **Discuss extensibility** — How would you add feature X later?

---

## Questions & Answers

---

### Q1 — 🟢 Explain the SOLID principles with Java examples.

<details><summary>Click to reveal answer</summary>

**S — Single Responsibility Principle**
> A class should have only one reason to change.

```java
// WRONG: UserService does too many things
class UserService {
    void createUser(User user) { ... }
    void sendWelcomeEmail(User user) { ... }  // email concern
    void saveToDatabase(User user) { ... }    // persistence concern
}

// RIGHT: separate concerns
class UserService {
    void createUser(User user) {
        userRepository.save(user);
        emailService.sendWelcomeEmail(user);
    }
}
class EmailService { void sendWelcomeEmail(User user) { ... } }
class UserRepository { void save(User user) { ... } }
```

**O — Open/Closed Principle**
> Open for extension, closed for modification.

```java
// WRONG: must modify the class to add new discount types
class DiscountCalculator {
    double calculate(Order order) {
        if (order.getType() == PREMIUM) return order.getTotal() * 0.9;
        if (order.getType() == VIP) return order.getTotal() * 0.8;
        return order.getTotal();
    }
}

// RIGHT: extend via new implementations
interface DiscountStrategy {
    double apply(double total);
}
class PremiumDiscount implements DiscountStrategy {
    public double apply(double total) { return total * 0.9; }
}
class VIPDiscount implements DiscountStrategy {
    public double apply(double total) { return total * 0.8; }
}
class DiscountCalculator {
    double calculate(Order order, DiscountStrategy strategy) {
        return strategy.apply(order.getTotal());
    }
}
```

**L — Liskov Substitution Principle**
> Subclasses must be substitutable for their base class without breaking behavior.

```java
// WRONG: Square violates LSP when substituted for Rectangle
class Rectangle {
    protected int width, height;
    void setWidth(int w) { this.width = w; }
    void setHeight(int h) { this.height = h; }
    int area() { return width * height; }
}
class Square extends Rectangle {
    // Forced to override both setters to maintain invariant
    void setWidth(int w) { this.width = w; this.height = w; }  // unexpected!
}

// RIGHT: use composition, not inheritance
interface Shape { int area(); }
class Rectangle implements Shape { ... }
class Square implements Shape { ... }
```

**I — Interface Segregation Principle**
> Clients should not be forced to depend on interfaces they don't use.

```java
// WRONG: forces printers to implement fax
interface MultifunctionDevice {
    void print(Document d);
    void scan(Document d);
    void fax(Document d);  // not all devices fax
}

// RIGHT: split into focused interfaces
interface Printer { void print(Document d); }
interface Scanner { void scan(Document d); }
interface Fax { void fax(Document d); }
class SimplePrinter implements Printer { ... }
class AllInOnePrinter implements Printer, Scanner, Fax { ... }
```

**D — Dependency Inversion Principle**
> High-level modules should not depend on low-level modules. Both should depend on abstractions.

```java
// WRONG: UserService depends on concrete MySQLUserRepository
class UserService {
    private MySQLUserRepository repo = new MySQLUserRepository();
}

// RIGHT: depend on the abstraction
class UserService {
    private final UserRepository repo;  // interface
    UserService(UserRepository repo) { this.repo = repo; }
}
// Now easy to swap: MySQLUserRepository, MongoUserRepository, InMemoryUserRepository
```

</details>

---

### Q2 — 🟡 Design a Parking Lot system.

<details><summary>Click to reveal answer</summary>

**Requirements:**
- Multiple levels, each with multiple spots
- Spot types: compact, large, handicapped, electric (with charger)
- Track which spots are occupied
- Calculate parking fee based on duration and spot type
- Support entry/exit tickets

**Key entities:**

```java
// Spot types
enum SpotType { COMPACT, LARGE, HANDICAPPED, ELECTRIC }
enum VehicleType { MOTORCYCLE, CAR, TRUCK, ELECTRIC_CAR }

// Vehicle
abstract class Vehicle {
    protected String licensePlate;
    protected VehicleType type;
}
class Car extends Vehicle {
    Car(String plate) { this.licensePlate = plate; this.type = VehicleType.CAR; }
}
class Truck extends Vehicle {
    Truck(String plate) { this.licensePlate = plate; this.type = VehicleType.TRUCK; }
}

// Parking Spot
class ParkingSpot {
    private final int id;
    private final SpotType type;
    private final int level;
    private Vehicle parkedVehicle;

    boolean isAvailable() { return parkedVehicle == null; }
    boolean canFit(VehicleType vehicleType) {
        return switch (type) {
            case COMPACT -> vehicleType == VehicleType.MOTORCYCLE || vehicleType == VehicleType.CAR;
            case LARGE -> true;  // fits all
            case HANDICAPPED -> vehicleType == VehicleType.CAR;
            case ELECTRIC -> vehicleType == VehicleType.ELECTRIC_CAR;
        };
    }
    void park(Vehicle v) { this.parkedVehicle = v; }
    Vehicle unpark() {
        Vehicle v = this.parkedVehicle;
        this.parkedVehicle = null;
        return v;
    }
}

// Level
class Level {
    private final int number;
    private final List<ParkingSpot> spots;

    Optional<ParkingSpot> findAvailableSpot(VehicleType vehicleType) {
        return spots.stream()
            .filter(s -> s.isAvailable() && s.canFit(vehicleType))
            .findFirst();
    }
}

// Ticket
class Ticket {
    private final String ticketId;
    private final Vehicle vehicle;
    private final ParkingSpot spot;
    private final LocalDateTime entryTime;
    private LocalDateTime exitTime;

    Duration getDuration() {
        return Duration.between(entryTime,
            exitTime != null ? exitTime : LocalDateTime.now());
    }
}

// Fee strategy (Strategy pattern)
interface FeeStrategy {
    BigDecimal calculate(Ticket ticket);
}
class HourlyFeeStrategy implements FeeStrategy {
    private static final Map<SpotType, BigDecimal> RATES = Map.of(
        SpotType.COMPACT, new BigDecimal("2.00"),
        SpotType.LARGE, new BigDecimal("4.00"),
        SpotType.ELECTRIC, new BigDecimal("5.00")
    );
    public BigDecimal calculate(Ticket ticket) {
        long hours = ticket.getDuration().toHours() + 1;  // round up
        return RATES.get(ticket.getSpot().getType())
            .multiply(BigDecimal.valueOf(hours));
    }
}

// ParkingLot (Singleton — one lot, one system)
class ParkingLot {
    private static ParkingLot instance;
    private final List<Level> levels;
    private final Map<String, Ticket> activeTickets = new ConcurrentHashMap<>();
    private final FeeStrategy feeStrategy;

    private ParkingLot(List<Level> levels, FeeStrategy feeStrategy) {
        this.levels = levels;
        this.feeStrategy = feeStrategy;
    }

    static synchronized ParkingLot getInstance(List<Level> levels, FeeStrategy strategy) {
        if (instance == null) instance = new ParkingLot(levels, strategy);
        return instance;
    }

    Ticket enter(Vehicle vehicle) {
        for (Level level : levels) {
            Optional<ParkingSpot> spot = level.findAvailableSpot(vehicle.getType());
            if (spot.isPresent()) {
                spot.get().park(vehicle);
                Ticket ticket = new Ticket(UUID.randomUUID().toString(),
                    vehicle, spot.get(), LocalDateTime.now());
                activeTickets.put(ticket.getTicketId(), ticket);
                return ticket;
            }
        }
        throw new ParkingLotFullException("No available spot for " + vehicle.getType());
    }

    BigDecimal exit(String ticketId) {
        Ticket ticket = activeTickets.remove(ticketId);
        if (ticket == null) throw new InvalidTicketException(ticketId);
        ticket.setExitTime(LocalDateTime.now());
        ticket.getSpot().unpark();
        return feeStrategy.calculate(ticket);
    }
}
```

**Design patterns used:**
- **Singleton**: `ParkingLot` — only one instance
- **Strategy**: `FeeStrategy` — swap hourly, flat-rate, or subscription pricing
- **Factory**: `VehicleFactory.create(type, plate)` to instantiate vehicles

</details>

---

### Q3 — 🟡 Design an Elevator Control System.

<details><summary>Click to reveal answer</summary>

**Requirements:**
- N elevators, M floors
- Passengers can request elevator from any floor (external request) specifying direction (Up/Down)
- Passengers inside elevator can request a destination floor (internal request)
- Optimize for minimum wait time / minimize direction changes
- Handle edge cases: elevator at capacity, emergency stop

**Key entities:**

```java
enum Direction { UP, DOWN, IDLE }
enum ElevatorState { MOVING, STOPPED, MAINTENANCE }

// External request (from floor panel)
record ExternalRequest(int floor, Direction direction) {}

// Internal request (from inside elevator)
record InternalRequest(int destinationFloor) {}

// Elevator
class Elevator {
    private final int id;
    private int currentFloor;
    private Direction direction;
    private ElevatorState state;
    private final TreeSet<Integer> upQueue;    // floors to visit going up
    private final TreeSet<Integer> downQueue;  // floors to visit going down (descending)
    private int currentLoad;
    private final int capacity;

    // SCAN algorithm: service requests in one direction, then reverse
    void addRequest(int floor) {
        if (floor > currentFloor || direction == Direction.UP) {
            upQueue.add(floor);
        } else {
            downQueue.add(floor);
        }
    }

    int nextFloor() {
        if (direction == Direction.UP && !upQueue.isEmpty()) {
            return upQueue.first();  // lowest floor going up
        }
        if (direction == Direction.DOWN && !downQueue.isEmpty()) {
            return downQueue.last(); // highest floor going down
        }
        // Switch direction
        if (direction == Direction.UP && !downQueue.isEmpty()) {
            direction = Direction.DOWN;
            return downQueue.last();
        }
        if (direction == Direction.DOWN && !upQueue.isEmpty()) {
            direction = Direction.UP;
            return upQueue.first();
        }
        direction = Direction.IDLE;
        return currentFloor;
    }

    void moveTo(int floor) {
        // Simulate movement, open/close doors, update currentFloor
        currentFloor = floor;
        if (direction == Direction.UP) upQueue.remove(floor);
        else downQueue.remove(floor);
    }

    // Cost function for dispatcher: estimate time to reach floor
    int costToServe(int floor, Direction requestDirection) {
        int distance = Math.abs(currentFloor - floor);
        // If elevator moving toward the floor in the same direction, low cost
        boolean samePath = (direction == requestDirection) &&
            ((direction == Direction.UP && floor >= currentFloor) ||
             (direction == Direction.DOWN && floor <= currentFloor));
        return samePath ? distance : distance + 10;  // penalty for direction change
    }
}

// Dispatcher (Observer pattern: external requests → best elevator)
class ElevatorDispatcher {
    private final List<Elevator> elevators;

    // Assign request to elevator with lowest cost
    void dispatch(ExternalRequest request) {
        Elevator best = elevators.stream()
            .filter(e -> e.getState() != ElevatorState.MAINTENANCE)
            .filter(e -> e.getCurrentLoad() < e.getCapacity())
            .min(Comparator.comparingInt(
                e -> e.costToServe(request.floor(), request.direction())))
            .orElseThrow(() -> new NoElevatorAvailableException());

        best.addRequest(request.floor());
    }

    // Internal request: passenger already in elevator, just add floor
    void addInternalRequest(int elevatorId, InternalRequest request) {
        elevators.stream()
            .filter(e -> e.getId() == elevatorId)
            .findFirst()
            .ifPresent(e -> e.addRequest(request.destinationFloor()));
    }
}
```

**Design patterns used:**
- **Strategy**: different dispatch algorithms (SCAN, LOOK, FCFS)
- **Observer**: floor panels notify dispatcher of external requests
- **State**: `ElevatorState` (MOVING, STOPPED, MAINTENANCE)

</details>

---

### Q4 — 🟡 Design a Library Management System.

<details><summary>Click to reveal answer</summary>

**Requirements:**
- Members can search for books (by title, author, ISBN)
- Members can borrow and return books
- Fine calculation for late returns
- Staff can add/remove books, manage members
- Reserve books that are currently checked out

**Key entities:**

```java
// Book and BookItem (one Book can have many physical copies)
class Book {
    private String isbn;
    private String title;
    private String author;
    private String genre;
}

enum BookStatus { AVAILABLE, CHECKED_OUT, RESERVED, LOST }

class BookItem {
    private String barcode;      // unique physical copy ID
    private Book book;
    private BookStatus status;
    private LocalDate dueDate;

    boolean isAvailable() { return status == BookStatus.AVAILABLE; }
}

// Member
class Member {
    private String memberId;
    private String name;
    private int booksBorrowedCount;
    private static final int MAX_BOOKS = 5;

    boolean canBorrow() { return booksBorrowedCount < MAX_BOOKS; }
}

// Loan
class Loan {
    private String loanId;
    private Member member;
    private BookItem bookItem;
    private LocalDate borrowDate;
    private LocalDate dueDate;
    private LocalDate returnDate;

    BigDecimal calculateFine() {
        if (returnDate == null || !returnDate.isAfter(dueDate)) {
            return BigDecimal.ZERO;
        }
        long overdueDays = ChronoUnit.DAYS.between(dueDate, returnDate);
        return BigDecimal.valueOf(overdueDays).multiply(new BigDecimal("0.50"));
    }
}

// Reservation
class Reservation {
    private String reservationId;
    private Member member;
    private Book book;
    private LocalDateTime reservedAt;
    private ReservationStatus status;  // PENDING, FULFILLED, CANCELLED
}

// Search (Strategy pattern for different search criteria)
interface BookSearchStrategy {
    List<Book> search(String query, BookCatalog catalog);
}
class TitleSearch implements BookSearchStrategy { ... }
class AuthorSearch implements BookSearchStrategy { ... }
class ISBNSearch implements BookSearchStrategy { ... }

// Library (Facade)
class Library {
    private final BookCatalog catalog;
    private final LoanRepository loanRepo;
    private final ReservationRepository reservationRepo;
    private final NotificationService notificationService;

    Loan checkOut(Member member, String barcode) {
        if (!member.canBorrow()) throw new BorrowLimitExceededException();
        BookItem item = catalog.findByBarcode(barcode);
        if (!item.isAvailable()) throw new BookNotAvailableException(barcode);

        item.setStatus(BookStatus.CHECKED_OUT);
        LocalDate due = LocalDate.now().plusWeeks(2);
        item.setDueDate(due);
        member.setBorrowCount(member.getBooksBorrowedCount() + 1);

        Loan loan = new Loan(member, item, LocalDate.now(), due);
        loanRepo.save(loan);
        return loan;
    }

    BigDecimal returnBook(Member member, String barcode) {
        Loan loan = loanRepo.findActiveLoan(member, barcode);
        loan.setReturnDate(LocalDate.now());
        BigDecimal fine = loan.calculateFine();

        BookItem item = loan.getBookItem();
        // Check if anyone has reserved this book
        reservationRepo.findNextPendingReservation(item.getBook())
            .ifPresentOrElse(
                res -> {
                    item.setStatus(BookStatus.RESERVED);
                    notificationService.notifyMember(res.getMember(),
                        "Your reserved book is now available!");
                },
                () -> item.setStatus(BookStatus.AVAILABLE));

        loanRepo.save(loan);
        return fine;
    }
}
```

</details>

---

### Q5 — 🔴 Design a Vending Machine.

<details><summary>Click to reveal answer</summary>

**Requirements:**
- Accepts coins and notes (various denominations)
- Dispenses product when correct amount inserted
- Returns change
- Handle out-of-stock, insufficient funds
- Admin mode: restock products, collect cash

**State Machine design (State pattern):**

```java
// States
interface VendingMachineState {
    void insertCoin(VendingMachine machine, Coin coin);
    void selectProduct(VendingMachine machine, String productCode);
    void dispense(VendingMachine machine);
    void cancel(VendingMachine machine);
}

class IdleState implements VendingMachineState {
    public void insertCoin(VendingMachine m, Coin coin) {
        m.addToBalance(coin.getValue());
        m.setState(m.getHasMoneyState());
        System.out.println("Inserted: " + coin + ", Balance: " + m.getBalance());
    }
    public void selectProduct(VendingMachine m, String code) {
        System.out.println("Please insert coins first.");
    }
    public void dispense(VendingMachine m) { System.out.println("No product selected."); }
    public void cancel(VendingMachine m) { System.out.println("Nothing to cancel."); }
}

class HasMoneyState implements VendingMachineState {
    public void insertCoin(VendingMachine m, Coin coin) {
        m.addToBalance(coin.getValue());
        System.out.println("Total balance: " + m.getBalance());
    }
    public void selectProduct(VendingMachine m, String code) {
        Product product = m.getInventory().get(code);
        if (product == null || product.getQuantity() == 0) {
            System.out.println("Product unavailable.");
            m.setState(m.getOutOfStockState());
            return;
        }
        if (m.getBalance() < product.getPrice()) {
            System.out.println("Insufficient funds. Need: " + product.getPrice());
            return;
        }
        m.setSelectedProduct(product);
        m.setState(m.getProductSelectedState());
    }
    public void cancel(VendingMachine m) {
        System.out.println("Returning: " + m.getBalance());
        m.returnChange(m.getBalance());
        m.setState(m.getIdleState());
    }
    public void dispense(VendingMachine m) { System.out.println("Please select a product."); }
}

class ProductSelectedState implements VendingMachineState {
    public void insertCoin(VendingMachine m, Coin coin) {
        System.out.println("Product already selected. Dispensing...");
    }
    public void selectProduct(VendingMachine m, String code) {
        System.out.println("Already selected. Cancel first to change.");
    }
    public void dispense(VendingMachine m) {
        Product product = m.getSelectedProduct();
        product.decrementQuantity();
        double change = m.getBalance() - product.getPrice();
        System.out.println("Dispensing: " + product.getName());
        if (change > 0) {
            System.out.println("Change: " + change);
            m.returnChange(change);
        }
        m.resetBalance();
        m.setState(product.getQuantity() == 0 ?
            m.getOutOfStockState() : m.getIdleState());
    }
    public void cancel(VendingMachine m) {
        m.returnChange(m.getBalance());
        m.setState(m.getIdleState());
    }
}

// VendingMachine
class VendingMachine {
    private VendingMachineState currentState;
    private final VendingMachineState idleState = new IdleState();
    private final VendingMachineState hasMoneyState = new HasMoneyState();
    private final VendingMachineState productSelectedState = new ProductSelectedState();
    private final VendingMachineState outOfStockState = new OutOfStockState();

    private double balance;
    private Product selectedProduct;
    private final Map<String, Product> inventory;
    private final CoinDispenserStrategy changeDispenser;

    VendingMachine(Map<String, Product> inventory, CoinDispenserStrategy changeDispenser) {
        this.inventory = inventory;
        this.changeDispenser = changeDispenser;
        this.currentState = idleState;
    }

    // Delegate to state
    void insertCoin(Coin coin) { currentState.insertCoin(this, coin); }
    void selectProduct(String code) { currentState.selectProduct(this, code); }
    void dispense() { currentState.dispense(this); }
    void cancel() { currentState.cancel(this); }

    void returnChange(double amount) {
        changeDispenser.dispense(amount);  // Strategy: greedy coin selection
    }
}
```

**Design patterns used:**
- **State**: `VendingMachineState` — clean transitions without if/else chains
- **Strategy**: `CoinDispenserStrategy` — greedy vs exact change algorithms
- **Command**: each button press is a command (useful for undo/redo or audit)
- **Singleton**: `VendingMachine` instance (in an embedded system context)

</details>
