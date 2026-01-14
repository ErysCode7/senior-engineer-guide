# Payment Integration with Stripe

## Overview

Stripe is the leading payment processing platform for online businesses. This tutorial covers implementing one-time payments, subscriptions, webhooks, and security best practices for handling payments in your application.

## Practical Use Cases

- **E-commerce**: Product purchases and checkouts
- **SaaS subscriptions**: Monthly/annual billing
- **Marketplaces**: Platform fees and payouts
- **Donations**: One-time or recurring contributions
- **On-demand services**: Pay-per-use pricing

## Step-by-Step Implementation

### 1. Setting Up Stripe with NestJS

```bash
npm install stripe @nestjs/config
```

```typescript
// payments/payments.module.ts
import { Module } from "@nestjs/common";
import { ConfigModule } from "@nestjs/config";
import { PaymentsController } from "./payments.controller";
import { PaymentsService } from "./payments.service";
import { StripeService } from "./stripe.service";
import { WebhookController } from "./webhook.controller";

@Module({
  imports: [ConfigModule],
  controllers: [PaymentsController, WebhookController],
  providers: [PaymentsService, StripeService],
  exports: [PaymentsService, StripeService],
})
export class PaymentsModule {}

// payments/stripe.service.ts
import { Injectable } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import Stripe from "stripe";

@Injectable()
export class StripeService {
  public readonly stripe: Stripe;

  constructor(private configService: ConfigService) {
    this.stripe = new Stripe(this.configService.get("STRIPE_SECRET_KEY"), {
      apiVersion: "2023-10-16",
    });
  }

  // Customer Management
  async createCustomer(email: string, name?: string): Promise<Stripe.Customer> {
    return this.stripe.customers.create({
      email,
      name,
    });
  }

  async getCustomer(customerId: string): Promise<Stripe.Customer> {
    return this.stripe.customers.retrieve(
      customerId
    ) as Promise<Stripe.Customer>;
  }

  async updateCustomer(
    customerId: string,
    data: Stripe.CustomerUpdateParams
  ): Promise<Stripe.Customer> {
    return this.stripe.customers.update(customerId, data);
  }

  // Payment Intent (One-time payments)
  async createPaymentIntent(
    amount: number,
    currency: string = "usd",
    customerId?: string
  ): Promise<Stripe.PaymentIntent> {
    return this.stripe.paymentIntents.create({
      amount: Math.round(amount * 100), // Convert to cents
      currency,
      customer: customerId,
      automatic_payment_methods: {
        enabled: true,
      },
    });
  }

  async confirmPaymentIntent(
    paymentIntentId: string
  ): Promise<Stripe.PaymentIntent> {
    return this.stripe.paymentIntents.confirm(paymentIntentId);
  }

  // Setup Intent (Save card for future use)
  async createSetupIntent(customerId: string): Promise<Stripe.SetupIntent> {
    return this.stripe.setupIntents.create({
      customer: customerId,
      payment_method_types: ["card"],
    });
  }

  // Payment Methods
  async attachPaymentMethod(
    paymentMethodId: string,
    customerId: string
  ): Promise<Stripe.PaymentMethod> {
    return this.stripe.paymentMethods.attach(paymentMethodId, {
      customer: customerId,
    });
  }

  async setDefaultPaymentMethod(
    customerId: string,
    paymentMethodId: string
  ): Promise<Stripe.Customer> {
    return this.stripe.customers.update(customerId, {
      invoice_settings: {
        default_payment_method: paymentMethodId,
      },
    });
  }

  async listPaymentMethods(
    customerId: string
  ): Promise<Stripe.PaymentMethod[]> {
    const paymentMethods = await this.stripe.paymentMethods.list({
      customer: customerId,
      type: "card",
    });
    return paymentMethods.data;
  }

  // Subscriptions
  async createSubscription(
    customerId: string,
    priceId: string,
    trialDays?: number
  ): Promise<Stripe.Subscription> {
    const subscriptionData: Stripe.SubscriptionCreateParams = {
      customer: customerId,
      items: [{ price: priceId }],
      payment_behavior: "default_incomplete",
      expand: ["latest_invoice.payment_intent"],
    };

    if (trialDays) {
      subscriptionData.trial_period_days = trialDays;
    }

    return this.stripe.subscriptions.create(subscriptionData);
  }

  async cancelSubscription(
    subscriptionId: string,
    immediately: boolean = false
  ): Promise<Stripe.Subscription> {
    if (immediately) {
      return this.stripe.subscriptions.cancel(subscriptionId);
    }
    return this.stripe.subscriptions.update(subscriptionId, {
      cancel_at_period_end: true,
    });
  }

  async updateSubscription(
    subscriptionId: string,
    newPriceId: string
  ): Promise<Stripe.Subscription> {
    const subscription = await this.stripe.subscriptions.retrieve(
      subscriptionId
    );

    return this.stripe.subscriptions.update(subscriptionId, {
      items: [
        {
          id: subscription.items.data[0].id,
          price: newPriceId,
        },
      ],
      proration_behavior: "create_prorations",
    });
  }

  // Refunds
  async createRefund(
    paymentIntentId: string,
    amount?: number
  ): Promise<Stripe.Refund> {
    const refundData: Stripe.RefundCreateParams = {
      payment_intent: paymentIntentId,
    };

    if (amount) {
      refundData.amount = Math.round(amount * 100);
    }

    return this.stripe.refunds.create(refundData);
  }

  // Invoices
  async getInvoice(invoiceId: string): Promise<Stripe.Invoice> {
    return this.stripe.invoices.retrieve(invoiceId);
  }

  async listInvoices(customerId: string): Promise<Stripe.Invoice[]> {
    const invoices = await this.stripe.invoices.list({
      customer: customerId,
      limit: 100,
    });
    return invoices.data;
  }

  // Products & Prices
  async createProduct(
    name: string,
    description?: string
  ): Promise<Stripe.Product> {
    return this.stripe.products.create({
      name,
      description,
    });
  }

  async createPrice(
    productId: string,
    amount: number,
    currency: string = "usd",
    recurring?: { interval: "month" | "year" }
  ): Promise<Stripe.Price> {
    const priceData: Stripe.PriceCreateParams = {
      product: productId,
      unit_amount: Math.round(amount * 100),
      currency,
    };

    if (recurring) {
      priceData.recurring = recurring;
    }

    return this.stripe.prices.create(priceData);
  }
}
```

### 2. Payment Service Layer

```typescript
// payments/payments.service.ts
import { Injectable, BadRequestException } from "@nestjs/common";
import { StripeService } from "./stripe.service";
import { UsersService } from "../users/users.service";
import Stripe from "stripe";

@Injectable()
export class PaymentsService {
  constructor(
    private stripeService: StripeService,
    private usersService: UsersService
  ) {}

  async createCheckoutSession(
    userId: string,
    priceId: string,
    successUrl: string,
    cancelUrl: string
  ) {
    const user = await this.usersService.findOne(userId);

    // Get or create Stripe customer
    let customerId = user.stripeCustomerId;
    if (!customerId) {
      const customer = await this.stripeService.createCustomer(
        user.email,
        `${user.firstName} ${user.lastName}`
      );
      customerId = customer.id;
      await this.usersService.update(userId, { stripeCustomerId: customerId });
    }

    // Create checkout session
    const session = await this.stripeService.stripe.checkout.sessions.create({
      customer: customerId,
      payment_method_types: ["card"],
      line_items: [
        {
          price: priceId,
          quantity: 1,
        },
      ],
      mode: "payment",
      success_url: successUrl,
      cancel_url: cancelUrl,
      metadata: {
        userId,
      },
    });

    return { sessionId: session.id, url: session.url };
  }

  async createSubscriptionCheckout(
    userId: string,
    priceId: string,
    trialDays?: number
  ) {
    const user = await this.usersService.findOne(userId);

    let customerId = user.stripeCustomerId;
    if (!customerId) {
      const customer = await this.stripeService.createCustomer(user.email);
      customerId = customer.id;
      await this.usersService.update(userId, { stripeCustomerId: customerId });
    }

    const sessionData: Stripe.Checkout.SessionCreateParams = {
      customer: customerId,
      payment_method_types: ["card"],
      line_items: [
        {
          price: priceId,
          quantity: 1,
        },
      ],
      mode: "subscription",
      success_url: `${process.env.FRONTEND_URL}/subscription/success?session_id={CHECKOUT_SESSION_ID}`,
      cancel_url: `${process.env.FRONTEND_URL}/subscription/cancel`,
      metadata: {
        userId,
      },
    };

    if (trialDays) {
      sessionData.subscription_data = {
        trial_period_days: trialDays,
      };
    }

    const session = await this.stripeService.stripe.checkout.sessions.create(
      sessionData
    );

    return { sessionId: session.id, url: session.url };
  }

  async createPaymentIntent(userId: string, amount: number) {
    const user = await this.usersService.findOne(userId);

    if (!user.stripeCustomerId) {
      throw new BadRequestException("No payment method on file");
    }

    const paymentIntent = await this.stripeService.createPaymentIntent(
      amount,
      "usd",
      user.stripeCustomerId
    );

    return {
      clientSecret: paymentIntent.client_secret,
    };
  }

  async cancelSubscription(userId: string, immediately: boolean = false) {
    const user = await this.usersService.findOne(userId);

    if (!user.subscriptionId) {
      throw new BadRequestException("No active subscription");
    }

    const subscription = await this.stripeService.cancelSubscription(
      user.subscriptionId,
      immediately
    );

    await this.usersService.update(userId, {
      subscriptionStatus: immediately ? "canceled" : "canceling",
    });

    return subscription;
  }

  async updateSubscription(userId: string, newPriceId: string) {
    const user = await this.usersService.findOne(userId);

    if (!user.subscriptionId) {
      throw new BadRequestException("No active subscription");
    }

    const subscription = await this.stripeService.updateSubscription(
      user.subscriptionId,
      newPriceId
    );

    return subscription;
  }

  async listPaymentMethods(userId: string) {
    const user = await this.usersService.findOne(userId);

    if (!user.stripeCustomerId) {
      return [];
    }

    return this.stripeService.listPaymentMethods(user.stripeCustomerId);
  }

  async getInvoices(userId: string) {
    const user = await this.usersService.findOne(userId);

    if (!user.stripeCustomerId) {
      return [];
    }

    return this.stripeService.listInvoices(user.stripeCustomerId);
  }

  async refundPayment(paymentIntentId: string, amount?: number) {
    return this.stripeService.createRefund(paymentIntentId, amount);
  }
}

// payments/payments.controller.ts
import {
  Controller,
  Post,
  Get,
  Body,
  UseGuards,
  Patch,
  Delete,
} from "@nestjs/common";
import { ApiTags, ApiBearerAuth, ApiOperation } from "@nestjs/swagger";
import { PaymentsService } from "./payments.service";
import { JwtAuthGuard } from "../auth/guards/jwt-auth.guard";
import { CurrentUser } from "../auth/decorators/current-user.decorator";

@ApiTags("payments")
@Controller("payments")
@UseGuards(JwtAuthGuard)
@ApiBearerAuth()
export class PaymentsController {
  constructor(private paymentsService: PaymentsService) {}

  @Post("checkout")
  @ApiOperation({ summary: "Create checkout session for one-time payment" })
  createCheckout(
    @CurrentUser("id") userId: string,
    @Body() body: { priceId: string; successUrl: string; cancelUrl: string }
  ) {
    return this.paymentsService.createCheckoutSession(
      userId,
      body.priceId,
      body.successUrl,
      body.cancelUrl
    );
  }

  @Post("subscription/checkout")
  @ApiOperation({ summary: "Create subscription checkout" })
  createSubscriptionCheckout(
    @CurrentUser("id") userId: string,
    @Body() body: { priceId: string; trialDays?: number }
  ) {
    return this.paymentsService.createSubscriptionCheckout(
      userId,
      body.priceId,
      body.trialDays
    );
  }

  @Post("payment-intent")
  @ApiOperation({ summary: "Create payment intent" })
  createPaymentIntent(
    @CurrentUser("id") userId: string,
    @Body() body: { amount: number }
  ) {
    return this.paymentsService.createPaymentIntent(userId, body.amount);
  }

  @Delete("subscription")
  @ApiOperation({ summary: "Cancel subscription" })
  cancelSubscription(
    @CurrentUser("id") userId: string,
    @Body() body: { immediately?: boolean }
  ) {
    return this.paymentsService.cancelSubscription(userId, body.immediately);
  }

  @Patch("subscription")
  @ApiOperation({ summary: "Update subscription plan" })
  updateSubscription(
    @CurrentUser("id") userId: string,
    @Body() body: { newPriceId: string }
  ) {
    return this.paymentsService.updateSubscription(userId, body.newPriceId);
  }

  @Get("payment-methods")
  @ApiOperation({ summary: "List saved payment methods" })
  listPaymentMethods(@CurrentUser("id") userId: string) {
    return this.paymentsService.listPaymentMethods(userId);
  }

  @Get("invoices")
  @ApiOperation({ summary: "Get invoice history" })
  getInvoices(@CurrentUser("id") userId: string) {
    return this.paymentsService.getInvoices(userId);
  }
}
```

### 3. Webhook Handler (Critical!)

```typescript
// payments/webhook.controller.ts
import {
  Controller,
  Post,
  Headers,
  RawBodyRequest,
  Req,
  BadRequestException,
} from "@nestjs/common";
import { Request } from "express";
import { StripeService } from "./stripe.service";
import { UsersService } from "../users/users.service";
import Stripe from "stripe";

@Controller("webhooks")
export class WebhookController {
  constructor(
    private stripeService: StripeService,
    private usersService: UsersService
  ) {}

  @Post("stripe")
  async handleWebhook(
    @Headers("stripe-signature") signature: string,
    @Req() request: RawBodyRequest<Request>
  ) {
    if (!signature) {
      throw new BadRequestException("Missing stripe-signature header");
    }

    let event: Stripe.Event;

    try {
      event = this.stripeService.stripe.webhooks.constructEvent(
        request.rawBody,
        signature,
        process.env.STRIPE_WEBHOOK_SECRET
      );
    } catch (err) {
      throw new BadRequestException(`Webhook Error: ${err.message}`);
    }

    // Handle the event
    switch (event.type) {
      case "checkout.session.completed":
        await this.handleCheckoutCompleted(event.data.object);
        break;

      case "customer.subscription.created":
      case "customer.subscription.updated":
        await this.handleSubscriptionUpdate(event.data.object);
        break;

      case "customer.subscription.deleted":
        await this.handleSubscriptionDeleted(event.data.object);
        break;

      case "invoice.payment_succeeded":
        await this.handleInvoicePaymentSucceeded(event.data.object);
        break;

      case "invoice.payment_failed":
        await this.handleInvoicePaymentFailed(event.data.object);
        break;

      case "payment_intent.succeeded":
        await this.handlePaymentIntentSucceeded(event.data.object);
        break;

      case "payment_intent.payment_failed":
        await this.handlePaymentIntentFailed(event.data.object);
        break;

      default:
        console.log(`Unhandled event type: ${event.type}`);
    }

    return { received: true };
  }

  private async handleCheckoutCompleted(session: Stripe.Checkout.Session) {
    const userId = session.metadata?.userId;
    if (!userId) return;

    // Update user with payment info
    await this.usersService.update(userId, {
      stripeCustomerId: session.customer as string,
      hasPaid: true,
    });

    // If it's a subscription
    if (session.subscription) {
      await this.usersService.update(userId, {
        subscriptionId: session.subscription as string,
        subscriptionStatus: "active",
      });
    }
  }

  private async handleSubscriptionUpdate(subscription: Stripe.Subscription) {
    const customerId = subscription.customer as string;
    const user = await this.usersService.findByStripeCustomerId(customerId);

    if (user) {
      await this.usersService.update(user.id, {
        subscriptionId: subscription.id,
        subscriptionStatus: subscription.status,
        currentPeriodEnd: new Date(subscription.current_period_end * 1000),
      });
    }
  }

  private async handleSubscriptionDeleted(subscription: Stripe.Subscription) {
    const customerId = subscription.customer as string;
    const user = await this.usersService.findByStripeCustomerId(customerId);

    if (user) {
      await this.usersService.update(user.id, {
        subscriptionId: null,
        subscriptionStatus: "canceled",
      });
    }
  }

  private async handleInvoicePaymentSucceeded(invoice: Stripe.Invoice) {
    const customerId = invoice.customer as string;
    const user = await this.usersService.findByStripeCustomerId(customerId);

    if (user) {
      // Send receipt email
      // Update payment history
      console.log(`Payment succeeded for user ${user.id}`);
    }
  }

  private async handleInvoicePaymentFailed(invoice: Stripe.Invoice) {
    const customerId = invoice.customer as string;
    const user = await this.usersService.findByStripeCustomerId(customerId);

    if (user) {
      // Send payment failed email
      // Alert user to update payment method
      console.log(`Payment failed for user ${user.id}`);
    }
  }

  private async handlePaymentIntentSucceeded(
    paymentIntent: Stripe.PaymentIntent
  ) {
    // Handle successful one-time payment
    console.log(`PaymentIntent ${paymentIntent.id} succeeded`);
  }

  private async handlePaymentIntentFailed(paymentIntent: Stripe.PaymentIntent) {
    // Handle failed payment
    console.log(`PaymentIntent ${paymentIntent.id} failed`);
  }
}

// main.ts - IMPORTANT: Add raw body parser for webhooks
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";
import { json } from "express";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Raw body for Stripe webhooks
  app.use(
    "/webhooks/stripe",
    json({ verify: (req, res, buf) => (req["rawBody"] = buf) })
  );

  await app.listen(3000);
}
bootstrap();
```

### 4. Frontend Integration (React Example)

```typescript
// React component for payment
import { loadStripe } from "@stripe/stripe-js";
import {
  Elements,
  PaymentElement,
  useStripe,
  useElements,
} from "@stripe/react-stripe-js";

const stripePromise = loadStripe("pk_test_...");

function CheckoutForm() {
  const stripe = useStripe();
  const elements = useElements();
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!stripe || !elements) return;

    setLoading(true);

    const { error } = await stripe.confirmPayment({
      elements,
      confirmParams: {
        return_url: `${window.location.origin}/payment/success`,
      },
    });

    if (error) {
      console.error(error);
    }

    setLoading(false);
  };

  return (
    <form onSubmit={handleSubmit}>
      <PaymentElement />
      <button disabled={!stripe || loading}>
        {loading ? "Processing..." : "Pay Now"}
      </button>
    </form>
  );
}

// Main component
function CheckoutPage() {
  const [clientSecret, setClientSecret] = useState("");

  useEffect(() => {
    // Get client secret from backend
    fetch("/api/payments/payment-intent", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ amount: 50.0 }),
    })
      .then((res) => res.json())
      .then((data) => setClientSecret(data.clientSecret));
  }, []);

  return (
    <Elements stripe={stripePromise} options={{ clientSecret }}>
      <CheckoutForm />
    </Elements>
  );
}
```

## Best Practices

### 1. Security

```typescript
// Always verify webhooks
const event = stripe.webhooks.constructEvent(rawBody, signature, webhookSecret);

// Never trust client-side amount
// Always calculate on server:
const amount = calculateOrderAmount(items);

// Use idempotency keys for safety
await stripe.paymentIntents.create(
  {
    amount: 1000,
    currency: "usd",
  },
  {
    idempotencyKey: orderId,
  }
);
```

### 2. Error Handling

```typescript
try {
  const paymentIntent = await stripe.paymentIntents.create({
    amount: 1000,
    currency: "usd",
  });
} catch (error) {
  if (error.type === "StripeCardError") {
    // Card was declined
  } else if (error.type === "StripeInvalidRequestError") {
    // Invalid parameters
  } else {
    // Other error
  }
}
```

### 3. Testing

```typescript
// Use test mode cards
const TEST_CARDS = {
  SUCCESS: "4242424242424242",
  DECLINED: "4000000000000002",
  REQUIRES_AUTH: "4000002500003155",
};

// Test webhooks with Stripe CLI
// stripe listen --forward-to localhost:3000/webhooks/stripe
```

## Key Takeaways

1. **Use webhooks** - Critical for subscription updates
2. **Validate everything** - Never trust client-side data
3. **Handle failures** - Payment failures are common
4. **Test thoroughly** - Use test mode extensively
5. **Secure webhooks** - Verify webhook signatures
6. **Idempotency** - Prevent duplicate charges
7. **PCI compliance** - Never store card details
8. **Monitor dashboard** - Watch for issues in Stripe dashboard

Stripe is powerful but requires careful implementation. Always handle webhooks properly and test all payment flows thoroughly!
