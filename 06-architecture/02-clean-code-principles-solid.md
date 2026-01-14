# Clean Code Principles & SOLID

## Overview

Clean code principles and SOLID design principles form the foundation of maintainable, scalable software. Clean code is easy to read, understand, and modify. SOLID principles provide guidelines for object-oriented design that reduce complexity, improve testability, and facilitate change. Together, they enable teams to build systems that evolve gracefully over time.

**SOLID Principles:**

- **S**ingle Responsibility Principle
- **O**pen/Closed Principle
- **L**iskov Substitution Principle
- **I**nterface Segregation Principle
- **D**ependency Inversion Principle

## Practical Use Cases

### 1. **Maintainable Codebases**

Long-term project sustainability

- Large enterprise applications
- Legacy system refactoring
- Team collaboration
- Onboarding new developers

### 2. **Testable Code**

Easier unit testing

- Isolated components
- Mocked dependencies
- Clear interfaces
- Predictable behavior

### 3. **Scalable Architecture**

Growing systems

- Adding new features
- Changing requirements
- Technology migrations
- Performance optimization

### 4. **Code Reviews**

Quality assurance

- Consistent standards
- Shared understanding
- Knowledge transfer
- Best practice enforcement

### 5. **Refactoring**

Improving existing code

- Technical debt reduction
- Performance improvements
- Security enhancements
- Bug fixes

## Single Responsibility Principle (SRP)

A class should have only one reason to change.

### ❌ Violating SRP

```typescript
// Bad: Class has multiple responsibilities
class User {
  constructor(public id: string, public name: string, public email: string) {}

  // Responsibility 1: User data validation
  validate(): boolean {
    return this.email.includes("@") && this.name.length > 0;
  }

  // Responsibility 2: Database operations
  async save(): Promise<void> {
    await database.query("INSERT INTO users...");
  }

  // Responsibility 3: Email sending
  async sendWelcomeEmail(): Promise<void> {
    await emailService.send({
      to: this.email,
      subject: "Welcome!",
      body: `Hello ${this.name}`,
    });
  }

  // Responsibility 4: Authentication
  async authenticate(password: string): Promise<boolean> {
    const hash = await bcrypt.hash(password, 10);
    return hash === this.passwordHash;
  }
}
```

### ✅ Following SRP

```typescript
// Good: Each class has a single responsibility

// Responsibility 1: User entity (data)
class User {
  constructor(
    public readonly id: string,
    public readonly name: string,
    public readonly email: string,
    public readonly passwordHash: string
  ) {}
}

// Responsibility 2: User validation
class UserValidator {
  validate(user: User): ValidationResult {
    const errors: string[] = [];

    if (!user.email.includes("@")) {
      errors.push("Invalid email format");
    }

    if (user.name.length < 2) {
      errors.push("Name must be at least 2 characters");
    }

    return {
      isValid: errors.length === 0,
      errors,
    };
  }
}

// Responsibility 3: User repository (database)
class UserRepository {
  async save(user: User): Promise<void> {
    await this.db.query(
      "INSERT INTO users (id, name, email, password_hash) VALUES ($1, $2, $3, $4)",
      [user.id, user.name, user.email, user.passwordHash]
    );
  }

  async findById(id: string): Promise<User | null> {
    const result = await this.db.query("SELECT * FROM users WHERE id = $1", [
      id,
    ]);
    return result.rows[0] ? this.mapToUser(result.rows[0]) : null;
  }
}

// Responsibility 4: Email service
class EmailService {
  async sendWelcomeEmail(user: User): Promise<void> {
    await this.emailClient.send({
      to: user.email,
      subject: "Welcome!",
      template: "welcome",
      data: { name: user.name },
    });
  }
}

// Responsibility 5: Authentication service
class AuthenticationService {
  async authenticate(user: User, password: string): Promise<boolean> {
    return bcrypt.compare(password, user.passwordHash);
  }
}

// Usage: Compose services
class UserService {
  constructor(
    private userRepository: UserRepository,
    private userValidator: UserValidator,
    private emailService: EmailService,
    private authService: AuthenticationService
  ) {}

  async registerUser(data: CreateUserDto): Promise<User> {
    // Validate
    const user = new User(
      uuid(),
      data.name,
      data.email,
      await bcrypt.hash(data.password, 10)
    );

    const validation = this.userValidator.validate(user);
    if (!validation.isValid) {
      throw new ValidationError(validation.errors);
    }

    // Save
    await this.userRepository.save(user);

    // Send email
    await this.emailService.sendWelcomeEmail(user);

    return user;
  }
}
```

## Open/Closed Principle (OCP)

Software entities should be open for extension but closed for modification.

### ❌ Violating OCP

```typescript
// Bad: Must modify class to add new payment methods
class PaymentProcessor {
  processPayment(amount: number, method: string): void {
    if (method === "credit_card") {
      this.processCreditCard(amount);
    } else if (method === "paypal") {
      this.processPayPal(amount);
    } else if (method === "crypto") {
      this.processCrypto(amount);
    }
    // Adding new method requires modifying this class
  }

  private processCreditCard(amount: number): void {
    console.log(`Processing credit card payment: $${amount}`);
  }

  private processPayPal(amount: number): void {
    console.log(`Processing PayPal payment: $${amount}`);
  }

  private processCrypto(amount: number): void {
    console.log(`Processing crypto payment: $${amount}`);
  }
}
```

### ✅ Following OCP

```typescript
// Good: Use abstraction to allow extension

// Abstract payment method
interface PaymentMethod {
  process(amount: number): Promise<PaymentResult>;
  getName(): string;
}

// Concrete implementations
class CreditCardPayment implements PaymentMethod {
  constructor(private stripeClient: StripeClient) {}

  async process(amount: number): Promise<PaymentResult> {
    const result = await this.stripeClient.charge({
      amount,
      currency: "usd",
    });

    return {
      success: result.status === "succeeded",
      transactionId: result.id,
    };
  }

  getName(): string {
    return "Credit Card";
  }
}

class PayPalPayment implements PaymentMethod {
  constructor(private paypalClient: PayPalClient) {}

  async process(amount: number): Promise<PaymentResult> {
    const order = await this.paypalClient.createOrder({
      amount,
      currency: "USD",
    });

    return {
      success: order.status === "APPROVED",
      transactionId: order.id,
    };
  }

  getName(): string {
    return "PayPal";
  }
}

class CryptoPayment implements PaymentMethod {
  constructor(private web3Client: Web3Client) {}

  async process(amount: number): Promise<PaymentResult> {
    const transaction = await this.web3Client.sendTransaction({
      value: amount,
    });

    return {
      success: transaction.status === "confirmed",
      transactionId: transaction.hash,
    };
  }

  getName(): string {
    return "Cryptocurrency";
  }
}

// Payment processor (closed for modification, open for extension)
class PaymentProcessor {
  private methods: Map<string, PaymentMethod> = new Map();

  registerMethod(key: string, method: PaymentMethod): void {
    this.methods.set(key, method);
  }

  async processPayment(
    amount: number,
    methodKey: string
  ): Promise<PaymentResult> {
    const method = this.methods.get(methodKey);

    if (!method) {
      throw new Error(`Payment method ${methodKey} not found`);
    }

    console.log(`Processing payment via ${method.getName()}`);
    return method.process(amount);
  }
}

// Usage: Register payment methods
const processor = new PaymentProcessor();
processor.registerMethod("credit_card", new CreditCardPayment(stripeClient));
processor.registerMethod("paypal", new PayPalPayment(paypalClient));
processor.registerMethod("crypto", new CryptoPayment(web3Client));

// Adding new payment method doesn't require modifying PaymentProcessor
processor.registerMethod("bank_transfer", new BankTransferPayment(bankClient));
```

## Liskov Substitution Principle (LSP)

Objects of a superclass should be replaceable with objects of its subclasses without breaking the application.

### ❌ Violating LSP

```typescript
// Bad: Square violates LSP by changing Rectangle behavior
class Rectangle {
  constructor(protected width: number, protected height: number) {}

  setWidth(width: number): void {
    this.width = width;
  }

  setHeight(height: number): void {
    this.height = height;
  }

  getArea(): number {
    return this.width * this.height;
  }
}

class Square extends Rectangle {
  constructor(size: number) {
    super(size, size);
  }

  // Violates LSP: Changes parent behavior
  setWidth(width: number): void {
    this.width = width;
    this.height = width; // Side effect!
  }

  setHeight(height: number): void {
    this.width = height; // Side effect!
    this.height = height;
  }
}

// This breaks when using Square
function testRectangle(rect: Rectangle): void {
  rect.setWidth(5);
  rect.setHeight(4);

  // Expects 20, but Square gives 16!
  console.assert(rect.getArea() === 20, "Area should be 20");
}

testRectangle(new Rectangle(0, 0)); // ✓ Works
testRectangle(new Square(0)); // ✗ Fails!
```

### ✅ Following LSP

```typescript
// Good: Use composition instead of inheritance

interface Shape {
  getArea(): number;
  getPerimeter(): number;
}

class Rectangle implements Shape {
  constructor(
    private readonly width: number,
    private readonly height: number
  ) {}

  getArea(): number {
    return this.width * this.height;
  }

  getPerimeter(): number {
    return 2 * (this.width + this.height);
  }

  getWidth(): number {
    return this.width;
  }

  getHeight(): number {
    return this.height;
  }
}

class Square implements Shape {
  constructor(private readonly size: number) {}

  getArea(): number {
    return this.size * this.size;
  }

  getPerimeter(): number {
    return 4 * this.size;
  }

  getSize(): number {
    return this.size;
  }
}

// Works with any Shape
function calculateArea(shape: Shape): number {
  return shape.getArea();
}

calculateArea(new Rectangle(5, 4)); // 20
calculateArea(new Square(5)); // 25
```

## Interface Segregation Principle (ISP)

Clients should not be forced to depend on interfaces they don't use.

### ❌ Violating ISP

```typescript
// Bad: Fat interface forces implementations to have unused methods
interface Worker {
  work(): void;
  eat(): void;
  sleep(): void;
  getMaintenance(): void; // Only for robots!
}

class Human implements Worker {
  work(): void {
    console.log("Human working");
  }

  eat(): void {
    console.log("Human eating");
  }

  sleep(): void {
    console.log("Human sleeping");
  }

  // Forced to implement even though humans don't need maintenance
  getMaintenance(): void {
    throw new Error("Humans do not get maintenance");
  }
}

class Robot implements Worker {
  work(): void {
    console.log("Robot working");
  }

  // Forced to implement even though robots don't eat/sleep
  eat(): void {
    throw new Error("Robots do not eat");
  }

  sleep(): void {
    throw new Error("Robots do not sleep");
  }

  getMaintenance(): void {
    console.log("Robot getting maintenance");
  }
}
```

### ✅ Following ISP

```typescript
// Good: Segregated interfaces

interface Workable {
  work(): void;
}

interface Eatable {
  eat(): void;
}

interface Sleepable {
  sleep(): void;
}

interface Maintainable {
  getMaintenance(): void;
}

// Human implements only relevant interfaces
class Human implements Workable, Eatable, Sleepable {
  work(): void {
    console.log("Human working");
  }

  eat(): void {
    console.log("Human eating");
  }

  sleep(): void {
    console.log("Human sleeping");
  }
}

// Robot implements only relevant interfaces
class Robot implements Workable, Maintainable {
  work(): void {
    console.log("Robot working");
  }

  getMaintenance(): void {
    console.log("Robot getting maintenance");
  }
}

// Functions depend only on what they need
function makeWork(worker: Workable): void {
  worker.work();
}

function provideMeal(eater: Eatable): void {
  eater.eat();
}

makeWork(new Human()); // ✓
makeWork(new Robot()); // ✓
provideMeal(new Human()); // ✓
// provideMeal(new Robot()); // ✗ Compile error (good!)
```

## Dependency Inversion Principle (DIP)

High-level modules should not depend on low-level modules. Both should depend on abstractions.

### ❌ Violating DIP

```typescript
// Bad: High-level class depends on low-level implementation

class MySQLDatabase {
  save(data: any): void {
    console.log("Saving to MySQL:", data);
  }

  find(id: string): any {
    console.log("Finding in MySQL:", id);
    return { id, name: "John" };
  }
}

// UserService depends directly on MySQLDatabase
class UserService {
  private database = new MySQLDatabase(); // Tight coupling!

  createUser(userData: any): void {
    this.database.save(userData);
  }

  getUser(id: string): any {
    return this.database.find(id);
  }
}

// Can't switch to PostgreSQL without modifying UserService
```

### ✅ Following DIP

```typescript
// Good: Depend on abstraction

// Abstraction
interface Database {
  save(data: any): Promise<void>;
  find(id: string): Promise<any>;
}

// Low-level implementations
class MySQLDatabase implements Database {
  async save(data: any): Promise<void> {
    console.log("Saving to MySQL:", data);
    // MySQL-specific implementation
  }

  async find(id: string): Promise<any> {
    console.log("Finding in MySQL:", id);
    return { id, name: "John" };
  }
}

class PostgreSQLDatabase implements Database {
  async save(data: any): Promise<void> {
    console.log("Saving to PostgreSQL:", data);
    // PostgreSQL-specific implementation
  }

  async find(id: string): Promise<any> {
    console.log("Finding in PostgreSQL:", id);
    return { id, name: "John" };
  }
}

class MongoDatabase implements Database {
  async save(data: any): Promise<void> {
    console.log("Saving to MongoDB:", data);
    // MongoDB-specific implementation
  }

  async find(id: string): Promise<any> {
    console.log("Finding in MongoDB:", id);
    return { id, name: "John" };
  }
}

// High-level module depends on abstraction
class UserService {
  constructor(private database: Database) {} // Dependency injection!

  async createUser(userData: any): Promise<void> {
    await this.database.save(userData);
  }

  async getUser(id: string): Promise<any> {
    return this.database.find(id);
  }
}

// Easy to switch implementations
const mysqlService = new UserService(new MySQLDatabase());
const postgresService = new UserService(new PostgreSQLDatabase());
const mongoService = new UserService(new MongoDatabase());
```

## Clean Code Principles

### 1. Meaningful Names

```typescript
// ❌ Bad: Unclear names
const d = new Date();
const list = getData();
function proc(x: number): number {
  return x * 2;
}

// ✅ Good: Descriptive names
const currentDate = new Date();
const activeUsers = getActiveUsers();
function doubleValue(value: number): number {
  return value * 2;
}
```

### 2. Functions Should Do One Thing

```typescript
// ❌ Bad: Function does too much
function processUserData(user: User): void {
  // Validate
  if (!user.email.includes("@")) throw new Error("Invalid email");

  // Transform
  user.name = user.name.trim().toLowerCase();

  // Save
  database.save(user);

  // Send email
  emailService.send(user.email, "Welcome");

  // Log
  logger.info(`User ${user.id} processed`);
}

// ✅ Good: Each function does one thing
function validateUser(user: User): void {
  if (!user.email.includes("@")) {
    throw new ValidationError("Invalid email");
  }
}

function normalizeUserData(user: User): User {
  return {
    ...user,
    name: user.name.trim().toLowerCase(),
  };
}

function saveUser(user: User): Promise<void> {
  return database.save(user);
}

function sendWelcomeEmail(user: User): Promise<void> {
  return emailService.send(user.email, "Welcome");
}

async function processUserData(user: User): Promise<void> {
  validateUser(user);
  const normalizedUser = normalizeUserData(user);
  await saveUser(normalizedUser);
  await sendWelcomeEmail(normalizedUser);
  logger.info(`User ${user.id} processed`);
}
```

### 3. Avoid Magic Numbers

```typescript
// ❌ Bad: Magic numbers
function calculateDiscount(price: number): number {
  if (price > 1000) {
    return price * 0.2;
  }
  return price * 0.1;
}

// ✅ Good: Named constants
const PREMIUM_THRESHOLD = 1000;
const PREMIUM_DISCOUNT_RATE = 0.2;
const STANDARD_DISCOUNT_RATE = 0.1;

function calculateDiscount(price: number): number {
  if (price > PREMIUM_THRESHOLD) {
    return price * PREMIUM_DISCOUNT_RATE;
  }
  return price * STANDARD_DISCOUNT_RATE;
}
```

### 4. Error Handling

```typescript
// ❌ Bad: Silent failures
function parseJSON(data: string): any {
  try {
    return JSON.parse(data);
  } catch (error) {
    return null; // Lost error information!
  }
}

// ✅ Good: Proper error handling
function parseJSON(data: string): any {
  try {
    return JSON.parse(data);
  } catch (error) {
    throw new ParseError(`Failed to parse JSON: ${error.message}`, { data });
  }
}

// Even better: Type-safe error handling
type Result<T, E = Error> =
  | { success: true; value: T }
  | { success: false; error: E };

function parseJSON(data: string): Result<any> {
  try {
    return { success: true, value: JSON.parse(data) };
  } catch (error) {
    return { success: false, error: error as Error };
  }
}

// Usage
const result = parseJSON(jsonString);
if (result.success) {
  console.log(result.value);
} else {
  console.error(result.error);
}
```

### 5. DRY (Don't Repeat Yourself)

```typescript
// ❌ Bad: Repeated code
function validateEmail(email: string): boolean {
  return email.includes("@") && email.includes(".");
}

function validateUsername(username: string): boolean {
  return username.length >= 3 && username.length <= 20;
}

function validatePassword(password: string): boolean {
  return password.length >= 8 && /[A-Z]/.test(password);
}

// ✅ Good: Extract common pattern
interface ValidationRule<T> {
  validate(value: T): boolean;
  message: string;
}

class LengthRule implements ValidationRule<string> {
  constructor(
    private min: number,
    private max: number,
    public message: string = `Length must be between ${min} and ${max}`
  ) {}

  validate(value: string): boolean {
    return value.length >= this.min && value.length <= this.max;
  }
}

class RegexRule implements ValidationRule<string> {
  constructor(private pattern: RegExp, public message: string) {}

  validate(value: string): boolean {
    return this.pattern.test(value);
  }
}

class Validator<T> {
  constructor(private rules: ValidationRule<T>[]) {}

  validate(value: T): { isValid: boolean; errors: string[] } {
    const errors: string[] = [];

    for (const rule of this.rules) {
      if (!rule.validate(value)) {
        errors.push(rule.message);
      }
    }

    return {
      isValid: errors.length === 0,
      errors,
    };
  }
}

// Usage
const emailValidator = new Validator([
  new RegexRule(/@/, "Must contain @"),
  new RegexRule(/\./, "Must contain domain"),
]);

const passwordValidator = new Validator([
  new LengthRule(8, 100, "Must be at least 8 characters"),
  new RegexRule(/[A-Z]/, "Must contain uppercase letter"),
]);
```

## Best Practices

### 1. **Code Organization**

- Group related functionality
- Clear folder structure
- Consistent naming conventions
- Separate concerns

### 2. **Comments**

- Explain why, not what
- Update comments with code
- Use self-documenting code
- Document complex algorithms

### 3. **Testing**

- Write testable code
- Use dependency injection
- Test one thing at a time
- Test edge cases

### 4. **Refactoring**

- Small, incremental changes
- Always have tests first
- One refactoring at a time
- Review and validate

### 5. **Code Reviews**

- Check SOLID principles
- Look for code smells
- Verify tests exist
- Ensure documentation

### 6. **Continuous Improvement**

- Regular code review
- Technical debt tracking
- Refactoring sprints
- Team learning

## Key Takeaways

✅ **Single Responsibility: One class, one reason to change**  
✅ **Open/Closed: Extend behavior without modifying code**  
✅ **Liskov Substitution: Subtypes must be substitutable**  
✅ **Interface Segregation: Small, focused interfaces**  
✅ **Dependency Inversion: Depend on abstractions**  
✅ **Meaningful names improve code readability**  
✅ **Functions should do one thing well**  
✅ **DRY principle prevents code duplication**  
✅ **Proper error handling is essential**  
✅ **Clean code is easier to maintain and test**

Clean code and SOLID principles are not just academic concepts—they're practical tools for building maintainable, scalable software systems.
