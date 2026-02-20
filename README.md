# Alta Entrega - Providers (Capabilities Adapters)

**Provider-Agnostic Interfaces | Vendor Abstraction Layer**

---

## 🎯 Visão Geral

Repositório de **Capabilities Adapters** que abstraem vendors externos, permitindo:

- ✅ UI nunca menciona vendors (Stripe, Supabase, etc.)
- ✅ Troca de providers sem alterar código da aplicação
- ✅ Múltiplos providers para a mesma capability
- ✅ A/B testing de providers por tenant
- ✅ Fallback automático em caso de falhas

**Filosofia:** Vendors são implementação, não contrato.

---

## 🏗️ Estrutura

```
alta-entrega-providers/
├── src/
│   ├── auth/
│   │   ├── interface.ts              # AuthProvider interface
│   │   ├── supabase.adapter.ts       # Supabase implementation
│   │   ├── keycloak.adapter.ts       # Keycloak implementation
│   │   └── index.ts
│   ├── payments/
│   │   ├── interface.ts              # PaymentProvider interface
│   │   ├── stripe.adapter.ts         # Stripe implementation
│   │   ├── pagseguro.adapter.ts      # PagSeguro implementation
│   │   └── index.ts
│   ├── notifications/
│   │   ├── interface.ts              # NotificationProvider interface
│   │   ├── sendgrid.adapter.ts       # SendGrid (email)
│   │   ├── twilio.adapter.ts         # Twilio (SMS)
│   │   └── index.ts
│   ├── storage/
│   │   ├── interface.ts              # StorageProvider interface
│   │   ├── minio.adapter.ts          # MinIO (S3-compatible)
│   │   ├── s3.adapter.ts             # AWS S3
│   │   └── index.ts
│   └── search/
│       ├── interface.ts              # SearchProvider interface
│       ├── meilisearch.adapter.ts    # Meilisearch
│       ├── elasticsearch.adapter.ts  # Elasticsearch
│       └── index.ts
├── tests/
│   ├── auth/
│   ├── payments/
│   └── notifications/
└── docs/
    ├── auth-providers.md
    ├── payment-providers.md
    └── adding-new-provider.md
```

---

## 📦 Capabilities

### 1. Auth Provider

**Interface:**
```typescript
interface AuthProvider {
  signUp(email: string, password: string): Promise<User>;
  signIn(email: string, password: string): Promise<Session>;
  signOut(token: string): Promise<void>;
  verify(token: string): Promise<User>;
  refreshToken(refreshToken: string): Promise<Session>;
  resetPassword(email: string): Promise<void>;
}
```

**Implementações:**
- ✅ Supabase (MVP)
- ⏳ Keycloak (Enterprise, SSO, SAML)

---

### 2. Payment Provider

**Interface:**
```typescript
interface PaymentProvider {
  authorize(params: AuthorizeParams): Promise<PaymentResult>;
  capture(transactionId: string): Promise<PaymentResult>;
  refund(transactionId: string, amount?: number): Promise<RefundResult>;
  getStatus(transactionId: string): Promise<PaymentStatus>;
  createCustomer(params: CustomerParams): Promise<Customer>;
  getCustomer(customerId: string): Promise<Customer>;
}
```

**Implementações:**
- ✅ Stripe (Internacional)
- ⏳ PagSeguro (Brasil)
- ⏳ Mercado Pago (LATAM)

---

### 3. Notification Provider

**Interface:**
```typescript
interface NotificationProvider {
  sendEmail(params: EmailParams): Promise<NotificationResult>;
  sendSMS(params: SMSParams): Promise<NotificationResult>;
  sendPush(params: PushParams): Promise<NotificationResult>;
  getStatus(notificationId: string): Promise<DeliveryStatus>;
}
```

**Implementações:**
- ✅ SendGrid (Email)
- ✅ Twilio (SMS)
- ⏳ OneSignal (Push)

---

### 4. Storage Provider

**Interface:**
```typescript
interface StorageProvider {
  upload(file: File, path: string): Promise<UploadResult>;
  download(path: string): Promise<File>;
  delete(path: string): Promise<void>;
  getUrl(path: string, expiresIn?: number): Promise<string>;
  list(prefix: string): Promise<StorageObject[]>;
}
```

**Implementações:**
- ✅ MinIO (Dev/Staging)
- ⏳ AWS S3 (Production)
- ⏳ Cloudflare R2 (CDN)

---

### 5. Search Provider

**Interface:**
```typescript
interface SearchProvider {
  index(documents: Document[]): Promise<IndexResult>;
  search(query: string, filters?: Filters): Promise<SearchResult[]>;
  update(documentId: string, document: Document): Promise<void>;
  delete(documentId: string): Promise<void>;
}
```

**Implementações:**
- ✅ Meilisearch (Dev/Prod)
- ⏳ Elasticsearch (Analytics)

---

## 🚀 Usage

### Exemplo: Payment Provider

```typescript
import { PaymentProvider } from '@alta-entrega/providers';

// ❌ WRONG (menciona vendor)
import Stripe from 'stripe';
const stripe = new Stripe(process.env.STRIPE_KEY);
const payment = await stripe.paymentIntents.create({ ... });

// ✅ CORRECT (usa capability)
const result = await PaymentProvider.authorize({
  amount: 10000, // cents
  currency: 'BRL',
  customerId: 'cus_123',
  metadata: { order_id: 'ord_456' },
});
```

### Configuração (tenant-based)

```typescript
// Tenant pode ter provider específico
const tenantConfig = await db.tenant.findUnique({
  where: { id: tenantId },
  select: { payment_provider: true },
});

// Resolve provider dinamicamente
const provider = PaymentProvider.resolve(tenantConfig.payment_provider);
const result = await provider.authorize(params);
```

---

## 🔧 Adicionando Novo Provider

### 1. Implemente a interface

```typescript
// src/payments/mercadopago.adapter.ts
import { PaymentProvider, AuthorizeParams, PaymentResult } from './interface';
import MercadoPago from 'mercadopago';

export class MercadoPagoAdapter implements PaymentProvider {
  private client: MercadoPago;

  constructor(config: MercadoPagoConfig) {
    this.client = new MercadoPago(config.accessToken);
  }

  async authorize(params: AuthorizeParams): Promise<PaymentResult> {
    const payment = await this.client.payment.create({
      transaction_amount: params.amount / 100,
      description: params.description,
      payment_method_id: params.paymentMethodId,
      payer: { email: params.customerEmail },
    });

    return {
      transactionId: payment.id.toString(),
      status: this.mapStatus(payment.status),
      amount: payment.transaction_amount * 100,
      providerData: payment,
    };
  }

  private mapStatus(status: string): PaymentStatus {
    // Map MercadoPago status to internal status
    const statusMap = {
      approved: 'captured',
      pending: 'authorized',
      rejected: 'failed',
    };
    return statusMap[status] || 'pending';
  }

  // Implement other methods...
}
```

### 2. Registre o adapter

```typescript
// src/payments/index.ts
import { StripeAdapter } from './stripe.adapter';
import { MercadoPagoAdapter } from './mercadopago.adapter';

const providers = {
  stripe: StripeAdapter,
  mercadopago: MercadoPagoAdapter,
};

export class PaymentProvider {
  static resolve(providerName: string) {
    const Provider = providers[providerName];
    if (!Provider) {
      throw new Error(`Unknown payment provider: ${providerName}`);
    }
    return new Provider(config);
  }

  // Convenience methods
  static async authorize(params: AuthorizeParams) {
    const provider = this.resolve(getDefaultProvider());
    return provider.authorize(params);
  }
}
```

### 3. Adicione testes

```typescript
// tests/payments/mercadopago.test.ts
import { MercadoPagoAdapter } from '../../src/payments/mercadopago.adapter';

describe('MercadoPagoAdapter', () => {
  it('should authorize payment', async () => {
    const adapter = new MercadoPagoAdapter(testConfig);
    const result = await adapter.authorize({
      amount: 10000,
      currency: 'BRL',
      customerId: 'test',
    });

    expect(result.status).toBe('authorized');
    expect(result.transactionId).toBeDefined();
  });
});
```

---

## 🧪 Testing

### Contract Tests (garantem interface)

```typescript
import { testPaymentProvider } from './contract-tests';
import { StripeAdapter } from '../src/payments/stripe.adapter';
import { MercadoPagoAdapter } from '../src/payments/mercadopago.adapter';

// Todos os adapters devem passar nos mesmos testes
describe('Payment Providers', () => {
  testPaymentProvider('Stripe', new StripeAdapter(config));
  testPaymentProvider('MercadoPago', new MercadoPagoAdapter(config));
});
```

### Mock Provider (testes)

```typescript
export class MockPaymentProvider implements PaymentProvider {
  async authorize(params: AuthorizeParams) {
    return {
      transactionId: 'mock_tx_123',
      status: 'authorized',
      amount: params.amount,
    };
  }
  // ...
}
```

---

## 📊 Feature Flags (Provider A/B Testing)

```typescript
import { unleash } from '@alta-entrega/feature-flags';

const provider = unleash.isEnabled('use-mercadopago', { tenantId })
  ? 'mercadopago'
  : 'stripe';

const result = await PaymentProvider.resolve(provider).authorize(params);
```

---

## 🔄 Fallback Strategy

```typescript
export class ResilientPaymentProvider {
  async authorize(params: AuthorizeParams): Promise<PaymentResult> {
    const primaryProvider = PaymentProvider.resolve('stripe');
    const fallbackProvider = PaymentProvider.resolve('mercadopago');

    try {
      return await primaryProvider.authorize(params);
    } catch (error) {
      logger.warn('Primary payment provider failed, using fallback', { error });
      return await fallbackProvider.authorize(params);
    }
  }
}
```

---

## 📖 Documentação Adicional

- [Adding New Provider Guide](./docs/adding-new-provider.md)
- [Auth Providers Comparison](./docs/auth-providers.md)
- [Payment Providers Comparison](./docs/payment-providers.md)
- [Provider Configuration](./docs/configuration.md)

---

## 🤝 Contributing

Repositório proprietário. Contribuições internas apenas.

---

## 📄 License

Proprietary - All rights reserved © Alta Entrega 2025