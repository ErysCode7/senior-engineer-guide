# Payment Integration with Stripe

## Overview

Stripe is the leading payment processor for online businesses. This guide covers integrating Stripe with Express.js to handle payments, subscriptions, webhooks, and secure payment flows with TypeScript.

## Setup

```bash
npm install stripe
npm install -D @types/stripe
```

```typescript
// src/config/stripe.ts
import Stripe from 'stripe';

if (!process.env.STRIPE_SECRET_KEY) {
  throw new Error('STRIPE_SECRET_KEY is required');
}

export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY, {
  apiVersion: '2024-11-20.acacia',
  typescript: true,
});

export default stripe;
```

```env
# .env
STRIPE_SECRET_KEY=sk_test_...
STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
```

## Payment Intents (One-time Payments)

### 1. Payment Service

```typescript
// src/services/payment.service.ts
import stripe from '../config/stripe';
import { AppError } from '../utils/AppError';
import { prisma } from '../config/database';

export class PaymentService {
  async createPaymentIntent(
    amount: number,
    currency: string = 'usd',
    userId: string,
    metadata?: Record<string, string>
  ) {
    const paymentIntent = await stripe.paymentIntents.create({
      amount: Math.round(amount * 100), // Convert to cents
      currency,
      metadata: {
        userId,
        ...metadata,
      },
    });

    await prisma.payment.create({
      data: {
        userId,
        stripePaymentIntentId: paymentIntent.id,
        amount,
        currency,
        status: paymentIntent.status,
      },
    });

    return {
      clientSecret: paymentIntent.client_secret,
      paymentIntentId: paymentIntent.id,
    };
  }

  async confirmPayment(paymentIntentId: string) {
    const paymentIntent = await stripe.paymentIntents.retrieve(paymentIntentId);

    if (paymentIntent.status === 'succeeded') {
      await prisma.payment.update({
        where: { stripePaymentIntentId: paymentIntentId },
        data: { status: 'succeeded', paidAt: new Date() },
      });
    }

    return paymentIntent;
  }

  async getPaymentHistory(userId: string) {
    return prisma.payment.findMany({
      where: { userId },
      orderBy: { createdAt: 'desc' },
    });
  }

  async refundPayment(paymentIntentId: string, amount?: number) {
    const payment = await prisma.payment.findUnique({
      where: { stripePaymentIntentId: paymentIntentId },
    });

    if (!payment) {
      throw new AppError('Payment not found', 404);
    }

    const refund = await stripe.refunds.create({
      payment_intent: paymentIntentId,
      amount: amount ? Math.round(amount * 100) : undefined,
    });

    await prisma.payment.update({
      where: { stripePaymentIntentId: paymentIntentId },
      data: { status: 'refunded', refundedAt: new Date() },
    });

    return refund;
  }
}

export default new PaymentService();
```

### 2. Payment Controller

```typescript
// src/controllers/payment.controller.ts
import { Request, Response } from 'express';
import { asyncHandler } from '../utils/asyncHandler';
import paymentService from '../services/payment.service';

export class PaymentController {
  createPaymentIntent = asyncHandler(async (req: Request, res: Response) => {
    const { amount, currency, metadata } = req.body;
    const userId = req.user!.id;

    const result = await paymentService.createPaymentIntent(
      amount,
      currency,
      userId,
      metadata
    );

    res.json({
      success: true,
      data: result,
    });
  });

  confirmPayment = asyncHandler(async (req: Request, res: Response) => {
    const { paymentIntentId } = req.params;

    const paymentIntent = await paymentService.confirmPayment(paymentIntentId);

    res.json({
      success: true,
      data: paymentIntent,
    });
  });

  getPaymentHistory = asyncHandler(async (req: Request, res: Response) => {
    const userId = req.user!.id;

    const payments = await paymentService.getPaymentHistory(userId);

    res.json({
      success: true,
      data: payments,
    });
  });

  refundPayment = asyncHandler(async (req: Request, res: Response) => {
    const { paymentIntentId } = req.params;
    const { amount } = req.body;

    const refund = await paymentService.refundPayment(paymentIntentId, amount);

    res.json({
      success: true,
      data: refund,
    });
  });
}

export default new PaymentController();
```

## Subscriptions

### 1. Subscription Service

```typescript
// src/services/subscription.service.ts
import stripe from '../config/stripe';
import { prisma } from '../config/database';
import { AppError } from '../utils/AppError';

export class SubscriptionService {
  async createCustomer(userId: string, email: string, name?: string) {
    const existingCustomer = await prisma.user.findUnique({
      where: { id: userId },
      select: { stripeCustomerId: true },
    });

    if (existingCustomer?.stripeCustomerId) {
      return existingCustomer.stripeCustomerId;
    }

    const customer = await stripe.customers.create({
      email,
      name,
      metadata: { userId },
    });

    await prisma.user.update({
      where: { id: userId },
      data: { stripeCustomerId: customer.id },
    });

    return customer.id;
  }

  async createSubscription(
    userId: string,
    priceId: string,
    paymentMethodId?: string
  ) {
    const user = await prisma.user.findUnique({ where: { id: userId } });

    if (!user?.stripeCustomerId) {
      throw new AppError('Customer not found', 404);
    }

    if (paymentMethodId) {
      await stripe.paymentMethods.attach(paymentMethodId, {
        customer: user.stripeCustomerId,
      });

      await stripe.customers.update(user.stripeCustomerId, {
        invoice_settings: {
          default_payment_method: paymentMethodId,
        },
      });
    }

    const subscription = await stripe.subscriptions.create({
      customer: user.stripeCustomerId,
      items: [{ price: priceId }],
      payment_behavior: 'default_incomplete',
      payment_settings: { save_default_payment_method: 'on_subscription' },
      expand: ['latest_invoice.payment_intent'],
    });

    await prisma.subscription.create({
      data: {
        userId,
        stripeSubscriptionId: subscription.id,
        stripePriceId: priceId,
        status: subscription.status,
        currentPeriodStart: new Date(subscription.current_period_start * 1000),
        currentPeriodEnd: new Date(subscription.current_period_end * 1000),
      },
    });

    return subscription;
  }

  async cancelSubscription(subscriptionId: string, immediately: boolean = false) {
    const subscription = await stripe.subscriptions.update(subscriptionId, {
      cancel_at_period_end: !immediately,
    });

    if (immediately) {
      await stripe.subscriptions.cancel(subscriptionId);
    }

    await prisma.subscription.update({
      where: { stripeSubscriptionId: subscriptionId },
      data: {
        status: immediately ? 'canceled' : 'active',
        cancelAt: immediately ? new Date() : new Date(subscription.cancel_at! * 1000),
      },
    });

    return subscription;
  }

  async updateSubscription(subscriptionId: string, newPriceId: string) {
    const subscription = await stripe.subscriptions.retrieve(subscriptionId);

    const updated = await stripe.subscriptions.update(subscriptionId, {
      items: [
        {
          id: subscription.items.data[0].id,
          price: newPriceId,
        },
      ],
      proration_behavior: 'always_invoice',
    });

    await prisma.subscription.update({
      where: { stripeSubscriptionId: subscriptionId },
      data: { stripePriceId: newPriceId },
    });

    return updated;
  }

  async getSubscription(userId: string) {
    return prisma.subscription.findFirst({
      where: { userId, status: { in: ['active', 'trialing'] } },
    });
  }
}

export default new SubscriptionService();
```

### 2. Subscription Controller

```typescript
// src/controllers/subscription.controller.ts
import { Request, Response } from 'express';
import { asyncHandler } from '../utils/asyncHandler';
import subscriptionService from '../services/subscription.service';

export class SubscriptionController {
  createSubscription = asyncHandler(async (req: Request, res: Response) => {
    const { priceId, paymentMethodId } = req.body;
    const userId = req.user!.id;

    const subscription = await subscriptionService.createSubscription(
      userId,
      priceId,
      paymentMethodId
    );

    res.json({
      success: true,
      data: subscription,
    });
  });

  cancelSubscription = asyncHandler(async (req: Request, res: Response) => {
    const { subscriptionId } = req.params;
    const { immediately } = req.body;

    const subscription = await subscriptionService.cancelSubscription(
      subscriptionId,
      immediately
    );

    res.json({
      success: true,
      data: subscription,
    });
  });

  updateSubscription = asyncHandler(async (req: Request, res: Response) => {
    const { subscriptionId } = req.params;
    const { priceId } = req.body;

    const subscription = await subscriptionService.updateSubscription(
      subscriptionId,
      priceId
    );

    res.json({
      success: true,
      data: subscription,
    });
  });

  getSubscription = asyncHandler(async (req: Request, res: Response) => {
    const userId = req.user!.id;

    const subscription = await subscriptionService.getSubscription(userId);

    res.json({
      success: true,
      data: subscription,
    });
  });
}

export default new SubscriptionController();
```

## Webhooks

### 1. Webhook Handler

```typescript
// src/controllers/webhook.controller.ts
import { Request, Response } from 'express';
import stripe from '../config/stripe';
import { prisma } from '../config/database';

export class WebhookController {
  handleStripeWebhook = async (req: Request, res: Response) => {
    const sig = req.headers['stripe-signature'];

    if (!sig) {
      return res.status(400).send('No signature');
    }

    let event;

    try {
      event = stripe.webhooks.constructEvent(
        req.body,
        sig,
        process.env.STRIPE_WEBHOOK_SECRET!
      );
    } catch (err: any) {
      return res.status(400).send(`Webhook Error: ${err.message}`);
    }

    try {
      switch (event.type) {
        case 'payment_intent.succeeded':
          await this.handlePaymentIntentSucceeded(event.data.object);
          break;

        case 'payment_intent.payment_failed':
          await this.handlePaymentIntentFailed(event.data.object);
          break;

        case 'customer.subscription.created':
        case 'customer.subscription.updated':
          await this.handleSubscriptionUpdate(event.data.object);
          break;

        case 'customer.subscription.deleted':
          await this.handleSubscriptionDeleted(event.data.object);
          break;

        case 'invoice.payment_succeeded':
          await this.handleInvoicePaymentSucceeded(event.data.object);
          break;

        case 'invoice.payment_failed':
          await this.handleInvoicePaymentFailed(event.data.object);
          break;

        default:
          console.log(`Unhandled event type: ${event.type}`);
      }

      res.json({ received: true });
    } catch (error) {
      console.error('Webhook handler error:', error);
      res.status(500).send('Webhook handler failed');
    }
  };

  private async handlePaymentIntentSucceeded(paymentIntent: any) {
    await prisma.payment.update({
      where: { stripePaymentIntentId: paymentIntent.id },
      data: {
        status: 'succeeded',
        paidAt: new Date(),
      },
    });
  }

  private async handlePaymentIntentFailed(paymentIntent: any) {
    await prisma.payment.update({
      where: { stripePaymentIntentId: paymentIntent.id },
      data: { status: 'failed' },
    });
  }

  private async handleSubscriptionUpdate(subscription: any) {
    await prisma.subscription.upsert({
      where: { stripeSubscriptionId: subscription.id },
      update: {
        status: subscription.status,
        currentPeriodStart: new Date(subscription.current_period_start * 1000),
        currentPeriodEnd: new Date(subscription.current_period_end * 1000),
      },
      create: {
        userId: subscription.metadata.userId,
        stripeSubscriptionId: subscription.id,
        stripePriceId: subscription.items.data[0].price.id,
        status: subscription.status,
        currentPeriodStart: new Date(subscription.current_period_start * 1000),
        currentPeriodEnd: new Date(subscription.current_period_end * 1000),
      },
    });
  }

  private async handleSubscriptionDeleted(subscription: any) {
    await prisma.subscription.update({
      where: { stripeSubscriptionId: subscription.id },
      data: {
        status: 'canceled',
        cancelAt: new Date(),
      },
    });
  }

  private async handleInvoicePaymentSucceeded(invoice: any) {
    // Update subscription or send receipt email
    console.log('Invoice paid:', invoice.id);
  }

  private async handleInvoicePaymentFailed(invoice: any) {
    // Notify user of payment failure
    console.log('Invoice payment failed:', invoice.id);
  }
}

export default new WebhookController();
```

### 2. Webhook Route (Raw Body Parser)

```typescript
// src/routes/webhook.routes.ts
import { Router } from 'express';
import express from 'express';
import webhookController from '../controllers/webhook.controller';

const router = Router();

// Stripe requires raw body for webhook signature verification
router.post(
  '/stripe',
  express.raw({ type: 'application/json' }),
  webhookController.handleStripeWebhook
);

export default router;
```

```typescript
// src/server.ts - Important: raw body parser before JSON parser
import express from 'express';
import webhookRoutes from './routes/webhook.routes';

const app = express();

// Webhook routes MUST come before express.json()
app.use('/api/v1/webhooks', webhookRoutes);

// Then add JSON parser for other routes
app.use(express.json());

// Other routes...
```

## Payment Routes

```typescript
// src/routes/payment.routes.ts
import { Router } from 'express';
import paymentController from '../controllers/payment.controller';
import subscriptionController from '../controllers/subscription.controller';
import { authenticate } from '../middleware/auth.middleware';

const router = Router();

router.use(authenticate);

// One-time payments
router.post('/payment-intent', paymentController.createPaymentIntent);
router.get('/payment-intent/:paymentIntentId', paymentController.confirmPayment);
router.get('/history', paymentController.getPaymentHistory);
router.post('/refund/:paymentIntentId', paymentController.refundPayment);

// Subscriptions
router.post('/subscription', subscriptionController.createSubscription);
router.get('/subscription', subscriptionController.getSubscription);
router.patch('/subscription/:subscriptionId', subscriptionController.updateSubscription);
router.delete('/subscription/:subscriptionId', subscriptionController.cancelSubscription);

export default router;
```

## Customer Portal

```typescript
// src/services/billing-portal.service.ts
import stripe from '../config/stripe';
import { prisma } from '../config/database';
import { AppError } from '../utils/AppError';

export class BillingPortalService {
  async createPortalSession(userId: string, returnUrl: string) {
    const user = await prisma.user.findUnique({ where: { id: userId } });

    if (!user?.stripeCustomerId) {
      throw new AppError('No customer found', 404);
    }

    const session = await stripe.billingPortal.sessions.create({
      customer: user.stripeCustomerId,
      return_url: returnUrl,
    });

    return session.url;
  }
}

export default new BillingPortalService();
```

## Testing

```typescript
// src/__tests__/payment.test.ts
import request from 'supertest';
import app from '../server';
import stripe from '../config/stripe';

describe('Payment Integration', () => {
  let authToken: string;

  beforeAll(async () => {
    const res = await request(app)
      .post('/api/v1/auth/login')
      .send({ email: 'test@example.com', password: 'password123' });
    authToken = res.body.data.accessToken;
  });

  describe('POST /api/v1/payments/payment-intent', () => {
    it('should create a payment intent', async () => {
      const res = await request(app)
        .post('/api/v1/payments/payment-intent')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          amount: 99.99,
          currency: 'usd',
        });

      expect(res.status).toBe(200);
      expect(res.body.data).toHaveProperty('clientSecret');
      expect(res.body.data).toHaveProperty('paymentIntentId');
    });
  });

  describe('POST /api/v1/payments/subscription', () => {
    it('should create a subscription', async () => {
      const testPriceId = 'price_test123';

      const res = await request(app)
        .post('/api/v1/payments/subscription')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          priceId: testPriceId,
          paymentMethodId: 'pm_card_visa',
        });

      expect(res.status).toBe(200);
      expect(res.body.data).toHaveProperty('id');
      expect(res.body.data.status).toBeDefined();
    });
  });

  describe('Webhook', () => {
    it('should handle payment_intent.succeeded event', async () => {
      const event = stripe.webhooks.generateTestHeaderString({
        payload: JSON.stringify({
          type: 'payment_intent.succeeded',
          data: { object: { id: 'pi_test123' } },
        }),
        secret: process.env.STRIPE_WEBHOOK_SECRET!,
      });

      const res = await request(app)
        .post('/api/v1/webhooks/stripe')
        .set('stripe-signature', event)
        .send({ type: 'payment_intent.succeeded' });

      expect(res.status).toBe(200);
    });
  });
});
```

## Best Practices

1. **Never store card details** - let Stripe handle it
2. **Use payment intents** for better fraud prevention
3. **Validate webhook signatures** to prevent spoofing
4. **Handle idempotency** to prevent duplicate charges
5. **Use metadata** to link payments to your data
6. **Test with Stripe CLI** for webhook development
7. **Implement retry logic** for failed payments
8. **Store minimal payment data** - link via Stripe IDs
9. **Use customer portal** for self-service billing
10. **Monitor webhook delivery** in Stripe Dashboard

## Security Checklist

- ✅ Use HTTPS in production
- ✅ Verify webhook signatures
- ✅ Never log sensitive data
- ✅ Keep API keys in environment variables
- ✅ Use different keys for test/production
- ✅ Implement rate limiting on payment endpoints
- ✅ Validate amounts server-side
- ✅ Use SCA-compliant payment methods
- ✅ Handle 3D Secure authentication
- ✅ Log all payment activities

## Key Takeaways

1. **Payment Intents** are the modern way to accept payments
2. **Subscriptions** automate recurring billing
3. **Webhooks** keep your database in sync with Stripe
4. **Raw body parser** is required for webhook verification
5. **Stripe CLI** simplifies webhook testing
6. **Customer Portal** reduces support burden
7. **Metadata** links Stripe objects to your data
8. **Test mode** allows safe development

Stripe provides excellent documentation and test cards for development.
