# SaaS Application Architecture

## Table of Contents

1. [System Overview](#system-overview)
2. [Multi-Tenancy Architecture](#multi-tenancy-architecture)
3. [Authentication & Authorization](#authentication--authorization)
4. [Billing System](#billing-system)
5. [Subscription Management](#subscription-management)
6. [Onboarding Flow](#onboarding-flow)
7. [Feature Flags](#feature-flags)
8. [Usage Tracking](#usage-tracking)
9. [API Rate Limiting](#api-rate-limiting)
10. [Data Isolation](#data-isolation)
11. [Scalability Patterns](#scalability-patterns)
12. [Key Takeaways](#key-takeaways)

## System Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Frontend (React)                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  Dashboard   │  │   Billing    │  │  Settings    │     │
│  │   Module     │  │   Module     │  │   Module     │     │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │
└─────────┼──────────────────┼──────────────────┼─────────────┘
          │                  │                  │
┌─────────▼──────────────────▼──────────────────▼─────────────┐
│                    API Gateway                                │
│         (Authentication, Rate Limiting, Routing)              │
└─────────┬────────────────────────────────────────────────────┘
          │
    ┌─────┴─────┬──────────────┬──────────────┬────────────┐
    │           │              │              │            │
┌───▼────┐ ┌───▼────┐  ┌──────▼──────┐ ┌────▼─────┐ ┌───▼───┐
│ Tenant │ │ User   │  │   Billing   │ │  Usage   │ │Feature│
│Service │ │Service │  │   Service   │ │ Service  │ │Service│
└───┬────┘ └───┬────┘  └──────┬──────┘ └────┬─────┘ └───┬───┘
    │          │               │             │           │
┌───▼──────────▼───────────────▼─────────────▼───────────▼───┐
│              Database Layer (Multi-Tenant)                   │
│  ┌───────────┐  ┌────────────┐  ┌─────────────┐           │
│  │ Tenant DB │  │  User DB   │  │ Billing DB  │           │
│  └───────────┘  └────────────┘  └─────────────┘           │
└──────────────────────────────────────────────────────────────┘
```

## Multi-Tenancy Architecture

### Tenant Isolation Strategies

```typescript
// Tenant types and configuration
interface Tenant {
  id: string;
  name: string;
  slug: string;
  domain?: string;
  plan: SubscriptionPlan;
  status: TenantStatus;
  settings: TenantSettings;
  limits: ResourceLimits;
  createdAt: Date;
  updatedAt: Date;
}

type TenantStatus = 'active' | 'suspended' | 'trial' | 'canceled';

interface TenantSettings {
  branding: BrandingConfig;
  features: FeatureFlags;
  locale: string;
  timezone: string;
  customDomain?: string;
}

interface BrandingConfig {
  logo?: string;
  primaryColor: string;
  secondaryColor: string;
  favicon?: string;
  customCSS?: string;
}

interface ResourceLimits {
  users: number;
  storage: number; // in GB
  apiCallsPerMonth: number;
  projectsPerUser: number;
}

// Tenant context provider
interface TenantContextValue {
  tenant: Tenant | null;
  isLoading: boolean;
  switchTenant: (tenantId: string) => Promise<void>;
  updateTenant: (updates: Partial<Tenant>) => Promise<void>;
}

const TenantContext = createContext<TenantContextValue | undefined>(undefined);

export const TenantProvider: React.FC<{ children: React.ReactNode }> = ({
  children,
}) => {
  const [tenant, setTenant] = useState<Tenant | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    loadTenantFromContext();
  }, []);

  const loadTenantFromContext = async () => {
    try {
      // Extract tenant from subdomain or path
      const tenantId = extractTenantId();
      if (tenantId) {
        const tenantData = await tenantApi.get(tenantId);
        setTenant(tenantData);
      }
    } catch (error) {
      console.error('Failed to load tenant:', error);
    } finally {
      setIsLoading(false);
    }
  };

  const switchTenant = async (tenantId: string) => {
    setIsLoading(true);
    try {
      const tenantData = await tenantApi.get(tenantId);
      setTenant(tenantData);
      // Update URL or redirect to new tenant context
      window.location.href = `https://${tenantData.slug}.myapp.com`;
    } catch (error) {
      console.error('Failed to switch tenant:', error);
    } finally {
      setIsLoading(false);
    }
  };

  const updateTenant = async (updates: Partial<Tenant>) => {
    if (!tenant) return;

    try {
      const updated = await tenantApi.update(tenant.id, updates);
      setTenant(updated);
    } catch (error) {
      console.error('Failed to update tenant:', error);
      throw error;
    }
  };

  return (
    <TenantContext.Provider
      value={{ tenant, isLoading, switchTenant, updateTenant }}
    >
      {children}
    </TenantContext.Provider>
  );
};

export const useTenant = () => {
  const context = useContext(TenantContext);
  if (!context) {
    throw new Error('useTenant must be used within TenantProvider');
  }
  return context;
};

// Extract tenant ID from URL
function extractTenantId(): string | null {
  // Strategy 1: Subdomain (tenant.myapp.com)
  const subdomain = window.location.hostname.split('.')[0];
  if (subdomain && subdomain !== 'www') {
    return subdomain;
  }

  // Strategy 2: Path (/tenant/tenant-id)
  const pathMatch = window.location.pathname.match(/^\/tenant\/([^\/]+)/);
  if (pathMatch) {
    return pathMatch[1];
  }

  // Strategy 3: Custom domain mapping
  const customDomain = window.location.hostname;
  return localStorage.getItem(`tenant_for_${customDomain}`) || null;
}
```

### Database Isolation Patterns

```typescript
// Pattern 1: Shared schema with tenant_id
interface SharedSchemaModel {
  id: string;
  tenantId: string;
  // other fields
}

class SharedSchemaRepository<T extends SharedSchemaModel> {
  constructor(
    private db: Database,
    private tableName: string,
    private tenantContext: () => string,
  ) {}

  async find(id: string): Promise<T | null> {
    const tenantId = this.tenantContext();
    return await this.db.query<T>(
      `SELECT * FROM ${this.tableName} WHERE id = ? AND tenant_id = ?`,
      [id, tenantId],
    );
  }

  async findAll(filters?: Record<string, any>): Promise<T[]> {
    const tenantId = this.tenantContext();
    let query = `SELECT * FROM ${this.tableName} WHERE tenant_id = ?`;
    const params: any[] = [tenantId];

    if (filters) {
      Object.entries(filters).forEach(([key, value]) => {
        query += ` AND ${key} = ?`;
        params.push(value);
      });
    }

    return await this.db.query<T[]>(query, params);
  }

  async create(data: Omit<T, "id" | "tenantId">): Promise<T> {
    const tenantId = this.tenantContext();
    return await this.db.insert<T>(this.tableName, {
      ...data,
      tenantId,
    });
  }

  async update(id: string, data: Partial<T>): Promise<T> {
    const tenantId = this.tenantContext();
    return await this.db.update<T>(this.tableName, { id, tenantId }, data);
  }

  async delete(id: string): Promise<void> {
    const tenantId = this.tenantContext();
    await this.db.delete(this.tableName, { id, tenantId });
  }
}

// Pattern 2: Separate databases per tenant
class IsolatedDatabaseManager {
  private connections: Map<string, Database> = new Map();

  async getConnection(tenantId: string): Promise<Database> {
    if (this.connections.has(tenantId)) {
      return this.connections.get(tenantId)!;
    }

    const dbConfig = await this.getDatabaseConfig(tenantId);
    const connection = await Database.connect(dbConfig);
    this.connections.set(tenantId, connection);

    return connection;
  }

  private async getDatabaseConfig(tenantId: string): Promise<DatabaseConfig> {
    // Retrieve tenant-specific database credentials
    return {
      host: process.env.DB_HOST!,
      database: `tenant_${tenantId}`,
      user: process.env.DB_USER!,
      password: process.env.DB_PASSWORD!,
    };
  }

  async closeConnection(tenantId: string): Promise<void> {
    const connection = this.connections.get(tenantId);
    if (connection) {
      await connection.close();
      this.connections.delete(tenantId);
    }
  }
}
```

## Authentication & Authorization

### Auth Types

```typescript
interface User {
  id: string;
  email: string;
  name: string;
  tenantId: string;
  role: UserRole;
  permissions: Permission[];
  status: UserStatus;
  lastLoginAt?: Date;
  createdAt: Date;
}

type UserRole = 'owner' | 'admin' | 'member' | 'viewer';
type UserStatus = 'active' | 'invited' | 'suspended';

interface Permission {
  resource: string;
  actions: Action[];
}

type Action = 'create' | 'read' | 'update' | 'delete' | 'manage';

interface AuthToken {
  accessToken: string;
  refreshToken: string;
  expiresIn: number;
  tokenType: 'Bearer';
}

// Role-based access control
class RBACService {
  private rolePermissions: Record<UserRole, Permission[]> = {
    owner: [
      { resource: '*', actions: ['create', 'read', 'update', 'delete', 'manage'] },
    ],
    admin: [
      { resource: 'users', actions: ['create', 'read', 'update', 'delete'] },
      { resource: 'projects', actions: ['create', 'read', 'update', 'delete'] },
      { resource: 'billing', actions: ['read', 'update'] },
    ],
    member: [
      { resource: 'projects', actions: ['create', 'read', 'update'] },
      { resource: 'users', actions: ['read'] },
    ],
    viewer: [
      { resource: 'projects', actions: ['read'] },
      { resource: 'users', actions: ['read'] },
    ],
  };

  can(user: User, resource: string, action: Action): boolean {
    const rolePermissions = this.rolePermissions[user.role];

    // Check wildcard permission
    const wildcardPermission = rolePermissions.find((p) => p.resource === '*');
    if (wildcardPermission?.actions.includes(action)) {
      return true;
    }

    // Check specific resource permission
    const resourcePermission = rolePermissions.find((p) => p.resource === resource);
    if (resourcePermission?.actions.includes(action)) {
      return true;
    }

    // Check user-specific permissions
    const userPermission = user.permissions.find((p) => p.resource === resource);
    return userPermission?.actions.includes(action) || false;
  }

  getPermittedActions(user: User, resource: string): Action[] {
    const actions: Set<Action> = new Set();

    const rolePermissions = this.rolePermissions[user.role];

    // Add role permissions
    rolePermissions.forEach((p) => {
      if (p.resource === '*' || p.resource === resource) {
        p.actions.forEach((a) => actions.add(a));
      }
    });

    // Add user-specific permissions
    const userPermission = user.permissions.find((p) => p.resource === resource);
    userPermission?.actions.forEach((a) => actions.add(a));

    return Array.from(actions);
  }
}

// Auth hook
export function useAuth() {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const rbac = new RBACService();

  useEffect(() => {
    loadUser();
  }, []);

  const loadUser = async () => {
    try {
      const token = localStorage.getItem('access_token');
      if (token) {
        const userData = await authApi.me();
        setUser(userData);
      }
    } catch (error) {
      console.error('Failed to load user:', error);
    } finally {
      setIsLoading(false);
    }
  };

  const login = async (email: string, password: string): Promise<void> => {
    const { accessToken, refreshToken } = await authApi.login(email, password);
    localStorage.setItem('access_token', accessToken);
    localStorage.setItem('refresh_token', refreshToken);
    await loadUser();
  };

  const logout = async (): Promise<void> => {
    await authApi.logout();
    localStorage.removeItem('access_token');
    localStorage.removeItem('refresh_token');
    setUser(null);
  };

  const can = (resource: string, action: Action): boolean => {
    return user ? rbac.can(user, resource, action) : false;
  };

  return { user, isLoading, login, logout, can };
}

// Protected route component
export const ProtectedRoute: React.FC<{
  children: React.ReactNode;
  requiredRole?: UserRole;
  requiredPermission?: { resource: string; action: Action };
}> = ({ children, requiredRole, requiredPermission }) => {
  const { user, isLoading, can } = useAuth();
  const navigate = useNavigate();

  useEffect(() => {
    if (!isLoading && !user) {
      navigate('/login');
    } else if (user && requiredRole && user.role !== requiredRole) {
      navigate('/unauthorized');
    } else if (
      user &&
      requiredPermission &&
      !can(requiredPermission.resource, requiredPermission.action)
    ) {
      navigate('/unauthorized');
    }
  }, [user, isLoading, requiredRole, requiredPermission]);

  if (isLoading) {
    return <LoadingScreen />;
  }

  return user ? <>{children}</> : null;
};
```

## Billing System

### Subscription Types

```typescript
interface SubscriptionPlan {
  id: string;
  name: string;
  description: string;
  price: PlanPrice;
  features: PlanFeature[];
  limits: ResourceLimits;
  billingInterval: BillingInterval;
  trialDays: number;
}

interface PlanPrice {
  monthly: number;
  yearly: number;
  currency: string;
}

type BillingInterval = "monthly" | "yearly";

interface PlanFeature {
  name: string;
  description: string;
  included: boolean | number;
  tooltip?: string;
}

interface Subscription {
  id: string;
  tenantId: string;
  planId: string;
  status: SubscriptionStatus;
  currentPeriodStart: Date;
  currentPeriodEnd: Date;
  cancelAtPeriodEnd: boolean;
  trialEnd?: Date;
  paymentMethodId?: string;
  billingInterval: BillingInterval;
  amount: number;
  currency: string;
  createdAt: Date;
}

type SubscriptionStatus =
  | "active"
  | "trialing"
  | "past_due"
  | "canceled"
  | "unpaid";

interface Invoice {
  id: string;
  subscriptionId: string;
  tenantId: string;
  amount: number;
  currency: string;
  status: InvoiceStatus;
  dueDate: Date;
  paidAt?: Date;
  items: InvoiceItem[];
  createdAt: Date;
}

type InvoiceStatus = "draft" | "open" | "paid" | "void" | "uncollectible";

interface InvoiceItem {
  description: string;
  quantity: number;
  unitAmount: number;
  total: number;
}
```

### Billing Service

```typescript
class BillingService {
  constructor(
    private stripeClient: Stripe,
    private db: Database,
  ) {}

  async createSubscription(
    tenantId: string,
    planId: string,
    paymentMethodId: string,
    billingInterval: BillingInterval,
  ): Promise<Subscription> {
    const plan = await this.getPlan(planId);
    const amount =
      billingInterval === "monthly" ? plan.price.monthly : plan.price.yearly;

    // Create Stripe subscription
    const stripeSubscription = await this.stripeClient.subscriptions.create({
      customer: await this.getOrCreateStripeCustomer(tenantId),
      items: [{ price: this.getStripePriceId(planId, billingInterval) }],
      default_payment_method: paymentMethodId,
      trial_period_days: plan.trialDays,
      metadata: {
        tenantId,
        planId,
      },
    });

    // Save subscription to database
    const subscription: Subscription = {
      id: stripeSubscription.id,
      tenantId,
      planId,
      status: stripeSubscription.status as SubscriptionStatus,
      currentPeriodStart: new Date(
        stripeSubscription.current_period_start * 1000,
      ),
      currentPeriodEnd: new Date(stripeSubscription.current_period_end * 1000),
      cancelAtPeriodEnd: stripeSubscription.cancel_at_period_end,
      trialEnd: stripeSubscription.trial_end
        ? new Date(stripeSubscription.trial_end * 1000)
        : undefined,
      paymentMethodId,
      billingInterval,
      amount,
      currency: plan.price.currency,
      createdAt: new Date(),
    };

    await this.db.insert("subscriptions", subscription);

    // Update tenant plan
    await this.db.update("tenants", { id: tenantId }, { plan });

    return subscription;
  }

  async changePlan(
    subscriptionId: string,
    newPlanId: string,
    billingInterval: BillingInterval,
  ): Promise<Subscription> {
    const subscription = await this.getSubscription(subscriptionId);
    const newPlan = await this.getPlan(newPlanId);

    // Update Stripe subscription
    const stripeSubscription = await this.stripeClient.subscriptions.update(
      subscriptionId,
      {
        items: [
          {
            id: subscription.items[0].id,
            price: this.getStripePriceId(newPlanId, billingInterval),
          },
        ],
        proration_behavior: "create_prorations",
        metadata: {
          planId: newPlanId,
        },
      },
    );

    // Update database
    const updatedSubscription = {
      ...subscription,
      planId: newPlanId,
      billingInterval,
      amount:
        billingInterval === "monthly"
          ? newPlan.price.monthly
          : newPlan.price.yearly,
    };

    await this.db.update(
      "subscriptions",
      { id: subscriptionId },
      updatedSubscription,
    );

    return updatedSubscription;
  }

  async cancelSubscription(
    subscriptionId: string,
    immediately: boolean = false,
  ): Promise<Subscription> {
    if (immediately) {
      await this.stripeClient.subscriptions.cancel(subscriptionId);
    } else {
      await this.stripeClient.subscriptions.update(subscriptionId, {
        cancel_at_period_end: true,
      });
    }

    const updates = immediately
      ? { status: "canceled" as SubscriptionStatus }
      : { cancelAtPeriodEnd: true };

    await this.db.update("subscriptions", { id: subscriptionId }, updates);

    return await this.getSubscription(subscriptionId);
  }

  async handleWebhook(event: Stripe.Event): Promise<void> {
    switch (event.type) {
      case "customer.subscription.updated":
        await this.handleSubscriptionUpdated(
          event.data.object as Stripe.Subscription,
        );
        break;

      case "customer.subscription.deleted":
        await this.handleSubscriptionDeleted(
          event.data.object as Stripe.Subscription,
        );
        break;

      case "invoice.paid":
        await this.handleInvoicePaid(event.data.object as Stripe.Invoice);
        break;

      case "invoice.payment_failed":
        await this.handlePaymentFailed(event.data.object as Stripe.Invoice);
        break;
    }
  }

  private async handleSubscriptionUpdated(
    stripeSubscription: Stripe.Subscription,
  ): Promise<void> {
    await this.db.update(
      "subscriptions",
      { id: stripeSubscription.id },
      {
        status: stripeSubscription.status as SubscriptionStatus,
        currentPeriodStart: new Date(
          stripeSubscription.current_period_start * 1000,
        ),
        currentPeriodEnd: new Date(
          stripeSubscription.current_period_end * 1000,
        ),
        cancelAtPeriodEnd: stripeSubscription.cancel_at_period_end,
      },
    );
  }

  private async handleSubscriptionDeleted(
    stripeSubscription: Stripe.Subscription,
  ): Promise<void> {
    await this.db.update(
      "subscriptions",
      { id: stripeSubscription.id },
      { status: "canceled" as SubscriptionStatus },
    );

    // Suspend tenant
    const tenantId = stripeSubscription.metadata.tenantId;
    await this.db.update(
      "tenants",
      { id: tenantId },
      { status: "suspended" as TenantStatus },
    );
  }

  private async handleInvoicePaid(invoice: Stripe.Invoice): Promise<void> {
    await this.db.insert("invoices", {
      id: invoice.id,
      subscriptionId: invoice.subscription as string,
      tenantId: invoice.metadata.tenantId,
      amount: invoice.amount_paid / 100,
      currency: invoice.currency,
      status: "paid" as InvoiceStatus,
      dueDate: new Date(invoice.due_date! * 1000),
      paidAt: new Date(invoice.status_transitions.paid_at! * 1000),
      items: invoice.lines.data.map((line) => ({
        description: line.description || "",
        quantity: line.quantity || 1,
        unitAmount: line.price!.unit_amount! / 100,
        total: line.amount / 100,
      })),
      createdAt: new Date(),
    });
  }

  private async handlePaymentFailed(invoice: Stripe.Invoice): Promise<void> {
    const subscription = await this.db.query(
      "SELECT * FROM subscriptions WHERE id = ?",
      [invoice.subscription],
    );

    if (subscription) {
      await this.db.update(
        "subscriptions",
        { id: subscription.id },
        { status: "past_due" as SubscriptionStatus },
      );

      // Send notification to tenant
      await this.notifyPaymentFailed(subscription.tenantId, invoice);
    }
  }

  private async getOrCreateStripeCustomer(tenantId: string): Promise<string> {
    const tenant = await this.db.query("SELECT * FROM tenants WHERE id = ?", [
      tenantId,
    ]);

    if (tenant.stripeCustomerId) {
      return tenant.stripeCustomerId;
    }

    const customer = await this.stripeClient.customers.create({
      email: tenant.email,
      name: tenant.name,
      metadata: { tenantId },
    });

    await this.db.update(
      "tenants",
      { id: tenantId },
      { stripeCustomerId: customer.id },
    );

    return customer.id;
  }

  private getStripePriceId(planId: string, interval: BillingInterval): string {
    // Map internal plan IDs to Stripe price IDs
    const priceMap: Record<string, Record<BillingInterval, string>> = {
      starter: {
        monthly: "price_starter_monthly",
        yearly: "price_starter_yearly",
      },
      professional: {
        monthly: "price_pro_monthly",
        yearly: "price_pro_yearly",
      },
      enterprise: {
        monthly: "price_enterprise_monthly",
        yearly: "price_enterprise_yearly",
      },
    };

    return priceMap[planId][interval];
  }

  private async getPlan(planId: string): Promise<SubscriptionPlan> {
    return await this.db.query("SELECT * FROM plans WHERE id = ?", [planId]);
  }

  private async getSubscription(subscriptionId: string): Promise<Subscription> {
    return await this.db.query("SELECT * FROM subscriptions WHERE id = ?", [
      subscriptionId,
    ]);
  }

  private async notifyPaymentFailed(
    tenantId: string,
    invoice: Stripe.Invoice,
  ): Promise<void> {
    // Send email notification
    console.log(`Payment failed for tenant ${tenantId}, invoice ${invoice.id}`);
  }
}
```

### Billing UI Components

```typescript
export const PricingTable: React.FC = () => {
  const [plans, setPlans] = useState<SubscriptionPlan[]>([]);
  const [billingInterval, setBillingInterval] = useState<BillingInterval>('monthly');
  const { tenant } = useTenant();

  useEffect(() => {
    loadPlans();
  }, []);

  const loadPlans = async () => {
    const data = await billingApi.getPlans();
    setPlans(data);
  };

  const handleSelectPlan = async (planId: string) => {
    // Navigate to checkout
    window.location.href = `/billing/checkout?plan=${planId}&interval=${billingInterval}`;
  };

  return (
    <div className="pricing-table">
      <div className="billing-toggle">
        <button
          className={billingInterval === 'monthly' ? 'active' : ''}
          onClick={() => setBillingInterval('monthly')}
        >
          Monthly
        </button>
        <button
          className={billingInterval === 'yearly' ? 'active' : ''}
          onClick={() => setBillingInterval('yearly')}
        >
          Yearly (Save 20%)
        </button>
      </div>

      <div className="plans-grid">
        {plans.map((plan) => {
          const price =
            billingInterval === 'monthly'
              ? plan.price.monthly
              : plan.price.yearly;
          const isCurrentPlan = tenant?.plan.id === plan.id;

          return (
            <div key={plan.id} className="plan-card">
              <h3>{plan.name}</h3>
              <p className="plan-description">{plan.description}</p>

              <div className="plan-price">
                <span className="currency">$</span>
                <span className="amount">{price}</span>
                <span className="interval">/{billingInterval === 'monthly' ? 'mo' : 'yr'}</span>
              </div>

              {plan.trialDays > 0 && (
                <div className="trial-notice">
                  {plan.trialDays} days free trial
                </div>
              )}

              <ul className="features-list">
                {plan.features.map((feature, idx) => (
                  <li key={idx}>
                    <CheckIcon />
                    <span>{feature.name}</span>
                    {feature.tooltip && <Tooltip text={feature.tooltip} />}
                  </li>
                ))}
              </ul>

              <button
                className={`select-plan-btn ${isCurrentPlan ? 'current' : ''}`}
                onClick={() => handleSelectPlan(plan.id)}
                disabled={isCurrentPlan}
              >
                {isCurrentPlan ? 'Current Plan' : 'Select Plan'}
              </button>
            </div>
          );
        })}
      </div>
    </div>
  );
};

export const BillingDashboard: React.FC = () => {
  const { tenant } = useTenant();
  const [subscription, setSubscription] = useState<Subscription | null>(null);
  const [invoices, setInvoices] = useState<Invoice[]>([]);

  useEffect(() => {
    loadBillingData();
  }, [tenant]);

  const loadBillingData = async () => {
    if (!tenant) return;

    const [subData, invoicesData] = await Promise.all([
      billingApi.getSubscription(tenant.id),
      billingApi.getInvoices(tenant.id),
    ]);

    setSubscription(subData);
    setInvoices(invoicesData);
  };

  const handleCancelSubscription = async () => {
    if (!subscription) return;

    const confirmed = window.confirm(
      'Are you sure you want to cancel your subscription?'
    );
    if (!confirmed) return;

    await billingApi.cancelSubscription(subscription.id);
    await loadBillingData();
  };

  if (!subscription) {
    return <LoadingSpinner />;
  }

  return (
    <div className="billing-dashboard">
      <section className="subscription-section">
        <h2>Current Subscription</h2>

        <div className="subscription-info">
          <div className="info-row">
            <span>Plan:</span>
            <strong>{tenant?.plan.name}</strong>
          </div>
          <div className="info-row">
            <span>Status:</span>
            <StatusBadge status={subscription.status} />
          </div>
          <div className="info-row">
            <span>Amount:</span>
            <strong>
              ${subscription.amount} / {subscription.billingInterval}
            </strong>
          </div>
          <div className="info-row">
            <span>Next billing date:</span>
            <strong>{formatDate(subscription.currentPeriodEnd)}</strong>
          </div>
        </div>

        <div className="subscription-actions">
          <button className="btn-primary">Change Plan</button>
          <button className="btn-secondary" onClick={handleCancelSubscription}>
            Cancel Subscription
          </button>
        </div>
      </section>

      <section className="invoices-section">
        <h2>Billing History</h2>

        <table className="invoices-table">
          <thead>
            <tr>
              <th>Date</th>
              <th>Amount</th>
              <th>Status</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {invoices.map((invoice) => (
              <tr key={invoice.id}>
                <td>{formatDate(invoice.createdAt)}</td>
                <td>
                  ${invoice.amount} {invoice.currency.toUpperCase()}
                </td>
                <td>
                  <StatusBadge status={invoice.status} />
                </td>
                <td>
                  <button onClick={() => downloadInvoice(invoice.id)}>
                    Download
                  </button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </section>
    </div>
  );
};
```

## Onboarding Flow

### Onboarding Types

```typescript
interface OnboardingStep {
  id: string;
  title: string;
  description: string;
  component: React.ComponentType<StepProps>;
  required: boolean;
  order: number;
}

interface StepProps {
  onNext: (data: any) => void;
  onBack: () => void;
  data: Record<string, any>;
}

interface OnboardingProgress {
  tenantId: string;
  currentStep: number;
  completedSteps: string[];
  stepData: Record<string, any>;
  completed: boolean;
  startedAt: Date;
  completedAt?: Date;
}
```

### Onboarding Service

```typescript
class OnboardingService {
  private steps: OnboardingStep[] = [
    {
      id: "tenant_setup",
      title: "Set Up Your Workspace",
      description: "Tell us about your organization",
      component: TenantSetupStep,
      required: true,
      order: 1,
    },
    {
      id: "invite_team",
      title: "Invite Your Team",
      description: "Collaborate with your team members",
      component: InviteTeamStep,
      required: false,
      order: 2,
    },
    {
      id: "configure_integrations",
      title: "Connect Integrations",
      description: "Connect your favorite tools",
      component: IntegrationsStep,
      required: false,
      order: 3,
    },
    {
      id: "create_first_project",
      title: "Create Your First Project",
      description: "Get started with your first project",
      component: FirstProjectStep,
      required: true,
      order: 4,
    },
  ];

  async getProgress(tenantId: string): Promise<OnboardingProgress> {
    const progress = await db.query<OnboardingProgress>(
      "SELECT * FROM onboarding_progress WHERE tenant_id = ?",
      [tenantId],
    );

    return progress || this.initializeProgress(tenantId);
  }

  async updateProgress(
    tenantId: string,
    stepId: string,
    data: any,
  ): Promise<OnboardingProgress> {
    const progress = await this.getProgress(tenantId);

    progress.completedSteps.push(stepId);
    progress.stepData[stepId] = data;
    progress.currentStep++;

    if (progress.currentStep >= this.steps.length) {
      progress.completed = true;
      progress.completedAt = new Date();
    }

    await db.update("onboarding_progress", { tenantId }, progress);

    return progress;
  }

  private initializeProgress(tenantId: string): OnboardingProgress {
    return {
      tenantId,
      currentStep: 0,
      completedSteps: [],
      stepData: {},
      completed: false,
      startedAt: new Date(),
    };
  }

  getSteps(): OnboardingStep[] {
    return this.steps.sort((a, b) => a.order - b.order);
  }

  getStep(id: string): OnboardingStep | undefined {
    return this.steps.find((s) => s.id === id);
  }
}
```

### Onboarding Component

```typescript
export const OnboardingFlow: React.FC = () => {
  const { tenant } = useTenant();
  const [progress, setProgress] = useState<OnboardingProgress | null>(null);
  const [currentStep, setCurrentStep] = useState(0);
  const onboardingService = new OnboardingService();
  const steps = onboardingService.getSteps();

  useEffect(() => {
    loadProgress();
  }, [tenant]);

  const loadProgress = async () => {
    if (!tenant) return;
    const data = await onboardingService.getProgress(tenant.id);
    setProgress(data);
    setCurrentStep(data.currentStep);
  };

  const handleNext = async (stepData: any) => {
    if (!tenant || !progress) return;

    const step = steps[currentStep];
    const updated = await onboardingService.updateProgress(
      tenant.id,
      step.id,
      stepData
    );

    setProgress(updated);

    if (updated.completed) {
      // Redirect to dashboard
      window.location.href = '/dashboard';
    } else {
      setCurrentStep(updated.currentStep);
    }
  };

  const handleBack = () => {
    setCurrentStep((prev) => Math.max(0, prev - 1));
  };

  const handleSkip = () => {
    setCurrentStep((prev) => Math.min(steps.length - 1, prev + 1));
  };

  if (!progress) {
    return <LoadingSpinner />;
  }

  const step = steps[currentStep];
  const StepComponent = step.component;
  const progressPercentage = ((currentStep + 1) / steps.length) * 100;

  return (
    <div className="onboarding-flow">
      <div className="onboarding-header">
        <h1>Welcome to MyApp!</h1>
        <div className="progress-bar">
          <div
            className="progress-fill"
            style={{ width: `${progressPercentage}%` }}
          />
        </div>
        <div className="step-indicator">
          Step {currentStep + 1} of {steps.length}
        </div>
      </div>

      <div className="onboarding-content">
        <h2>{step.title}</h2>
        <p>{step.description}</p>

        <StepComponent
          onNext={handleNext}
          onBack={handleBack}
          data={progress.stepData[step.id] || {}}
        />

        {!step.required && (
          <button className="skip-button" onClick={handleSkip}>
            Skip this step
          </button>
        )}
      </div>

      <div className="onboarding-footer">
        <div className="steps-navigation">
          {steps.map((s, idx) => (
            <div
              key={s.id}
              className={`step-dot ${idx === currentStep ? 'active' : ''} ${
                progress.completedSteps.includes(s.id) ? 'completed' : ''
              }`}
            />
          ))}
        </div>
      </div>
    </div>
  );
};

// Example step component
const TenantSetupStep: React.FC<StepProps> = ({ onNext, data }) => {
  const [formData, setFormData] = useState({
    companyName: data.companyName || '',
    industry: data.industry || '',
    teamSize: data.teamSize || '',
    timezone: data.timezone || Intl.DateTimeFormat().resolvedOptions().timeZone,
  });

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    onNext(formData);
  };

  return (
    <form onSubmit={handleSubmit} className="onboarding-form">
      <div className="form-group">
        <label>Company Name *</label>
        <input
          type="text"
          value={formData.companyName}
          onChange={(e) =>
            setFormData({ ...formData, companyName: e.target.value })
          }
          required
        />
      </div>

      <div className="form-group">
        <label>Industry *</label>
        <select
          value={formData.industry}
          onChange={(e) =>
            setFormData({ ...formData, industry: e.target.value })
          }
          required
        >
          <option value="">Select an industry</option>
          <option value="technology">Technology</option>
          <option value="healthcare">Healthcare</option>
          <option value="finance">Finance</option>
          <option value="education">Education</option>
          <option value="other">Other</option>
        </select>
      </div>

      <div className="form-group">
        <label>Team Size *</label>
        <select
          value={formData.teamSize}
          onChange={(e) =>
            setFormData({ ...formData, teamSize: e.target.value })
          }
          required
        >
          <option value="">Select team size</option>
          <option value="1-10">1-10</option>
          <option value="11-50">11-50</option>
          <option value="51-200">51-200</option>
          <option value="201+">201+</option>
        </select>
      </div>

      <div className="form-group">
        <label>Timezone</label>
        <select
          value={formData.timezone}
          onChange={(e) =>
            setFormData({ ...formData, timezone: e.target.value })
          }
        >
          {Intl.supportedValuesOf('timeZone').map((tz) => (
            <option key={tz} value={tz}>
              {tz}
            </option>
          ))}
        </select>
      </div>

      <button type="submit" className="btn-primary">
        Continue
      </button>
    </form>
  );
};
```

## Feature Flags

### Feature Flag System

```typescript
interface FeatureFlag {
  id: string;
  name: string;
  description: string;
  enabled: boolean;
  rolloutPercentage?: number;
  targetPlans?: string[];
  targetTenants?: string[];
  expiresAt?: Date;
}

class FeatureFlagService {
  private flags: Map<string, FeatureFlag> = new Map();

  async initialize(): Promise<void> {
    const flags = await this.loadFlags();
    flags.forEach((flag) => this.flags.set(flag.id, flag));
  }

  isEnabled(flagId: string, tenantId: string, planId: string): boolean {
    const flag = this.flags.get(flagId);
    if (!flag) return false;

    // Check expiration
    if (flag.expiresAt && new Date() > flag.expiresAt) {
      return false;
    }

    // Check if disabled globally
    if (!flag.enabled) return false;

    // Check plan targeting
    if (flag.targetPlans && !flag.targetPlans.includes(planId)) {
      return false;
    }

    // Check tenant targeting
    if (flag.targetTenants && !flag.targetTenants.includes(tenantId)) {
      return false;
    }

    // Check rollout percentage
    if (flag.rolloutPercentage !== undefined) {
      const hash = this.hashTenantId(tenantId);
      return hash < flag.rolloutPercentage;
    }

    return true;
  }

  private hashTenantId(tenantId: string): number {
    let hash = 0;
    for (let i = 0; i < tenantId.length; i++) {
      hash = (hash << 5) - hash + tenantId.charCodeAt(i);
      hash |= 0;
    }
    return Math.abs(hash % 100);
  }

  private async loadFlags(): Promise<FeatureFlag[]> {
    return await db.query<FeatureFlag[]>('SELECT * FROM feature_flags');
  }
}

// React hook
export function useFeatureFlag(flagId: string): boolean {
  const { tenant } = useTenant();
  const featureFlagService = new FeatureFlagService();

  return tenant
    ? featureFlagService.isEnabled(flagId, tenant.id, tenant.plan.id)
    : false;
}

// Feature gate component
export const FeatureGate: React.FC<{
  feature: string;
  fallback?: React.ReactNode;
  children: React.ReactNode;
}> = ({ feature, fallback = null, children }) => {
  const isEnabled = useFeatureFlag(feature);
  return isEnabled ? <>{children}</> : <>{fallback}</>;
};
```

## Usage Tracking

### Usage Tracking System

```typescript
interface UsageMetric {
  tenantId: string;
  metric: string;
  value: number;
  timestamp: Date;
  metadata?: Record<string, any>;
}

class UsageTrackingService {
  async trackUsage(
    tenantId: string,
    metric: string,
    value: number = 1,
    metadata?: Record<string, any>,
  ): Promise<void> {
    const usage: UsageMetric = {
      tenantId,
      metric,
      value,
      timestamp: new Date(),
      metadata,
    };

    await db.insert("usage_metrics", usage);

    // Check if tenant exceeds limits
    await this.checkLimits(tenantId, metric, value);
  }

  async getCurrentUsage(
    tenantId: string,
    metric: string,
    period: "day" | "month" = "month",
  ): Promise<number> {
    const startDate = this.getStartDate(period);

    const result = await db.query<{ total: number }>(
      `SELECT SUM(value) as total 
       FROM usage_metrics 
       WHERE tenant_id = ? AND metric = ? AND timestamp >= ?`,
      [tenantId, metric, startDate],
    );

    return result?.total || 0;
  }

  private async checkLimits(
    tenantId: string,
    metric: string,
    addedValue: number,
  ): Promise<void> {
    const tenant = await db.query<Tenant>(
      "SELECT * FROM tenants WHERE id = ?",
      [tenantId],
    );

    const limit = this.getLimit(tenant.plan, metric);
    if (!limit) return;

    const currentUsage = await this.getCurrentUsage(tenantId, metric);
    const newUsage = currentUsage + addedValue;

    if (newUsage > limit) {
      // Send notification or block action
      await this.handleLimitExceeded(tenant, metric, newUsage, limit);
    }
  }

  private getLimit(plan: SubscriptionPlan, metric: string): number | null {
    const limitMap: Record<string, keyof ResourceLimits> = {
      api_calls: "apiCallsPerMonth",
      storage: "storage",
      users: "users",
      projects: "projectsPerUser",
    };

    const limitKey = limitMap[metric];
    return limitKey ? plan.limits[limitKey] : null;
  }

  private getStartDate(period: "day" | "month"): Date {
    const now = new Date();
    if (period === "day") {
      return new Date(now.getFullYear(), now.getMonth(), now.getDate());
    } else {
      return new Date(now.getFullYear(), now.getMonth(), 1);
    }
  }

  private async handleLimitExceeded(
    tenant: Tenant,
    metric: string,
    currentUsage: number,
    limit: number,
  ): Promise<void> {
    console.log(
      `Tenant ${tenant.id} exceeded limit for ${metric}: ${currentUsage}/${limit}`,
    );
    // Send notification, block API calls, etc.
  }
}
```

## API Rate Limiting

### Rate Limiter

```typescript
interface RateLimitConfig {
  windowMs: number;
  maxRequests: number;
  message?: string;
}

class RateLimiter {
  private store: Map<string, number[]> = new Map();

  async checkLimit(
    key: string,
    config: RateLimitConfig,
  ): Promise<{ allowed: boolean; remaining: number; resetAt: Date }> {
    const now = Date.now();
    const windowStart = now - config.windowMs;

    // Get existing requests
    let requests = this.store.get(key) || [];

    // Remove expired requests
    requests = requests.filter((time) => time > windowStart);

    // Check limit
    const allowed = requests.length < config.maxRequests;

    if (allowed) {
      requests.push(now);
      this.store.set(key, requests);
    }

    return {
      allowed,
      remaining: Math.max(0, config.maxRequests - requests.length),
      resetAt: new Date(now + config.windowMs),
    };
  }

  reset(key: string): void {
    this.store.delete(key);
  }
}

// Express middleware
export function rateLimitMiddleware(config: RateLimitConfig) {
  const limiter = new RateLimiter();

  return async (req: Request, res: Response, next: NextFunction) => {
    const tenantId = req.headers["x-tenant-id"] as string;
    if (!tenantId) {
      return res.status(400).json({ error: "Missing tenant ID" });
    }

    const key = `rate_limit:${tenantId}:${req.path}`;
    const result = await limiter.checkLimit(key, config);

    res.setHeader("X-RateLimit-Limit", config.maxRequests);
    res.setHeader("X-RateLimit-Remaining", result.remaining);
    res.setHeader("X-RateLimit-Reset", result.resetAt.toISOString());

    if (!result.allowed) {
      return res.status(429).json({
        error: config.message || "Too many requests",
        retryAfter: result.resetAt,
      });
    }

    next();
  };
}
```

## Key Takeaways

### 1. **Multi-Tenancy Design**

Choose between shared schema with tenant_id or isolated databases based on compliance and scale requirements. Implement proper tenant context throughout the application.

### 2. **Secure Authentication**

Use JWT tokens with short expiration for access tokens and longer-lived refresh tokens. Implement role-based access control (RBAC) with granular permissions.

### 3. **Flexible Billing**

Integrate with Stripe or similar payment processors. Support multiple plans, trial periods, and prorated upgrades/downgrades. Handle webhooks for subscription events.

### 4. **Smooth Onboarding**

Guide users through setup with progressive disclosure. Make non-critical steps optional. Track progress and allow resuming later.

### 5. **Feature Management**

Use feature flags for gradual rollouts and A/B testing. Enable features by plan, tenant, or percentage. Support time-limited features.

### 6. **Usage Tracking**

Monitor resource usage against plan limits. Send proactive notifications before limits are reached. Provide usage dashboards to users.

### 7. **API Rate Limiting**

Implement rate limiting per tenant to ensure fair usage. Use sliding window algorithms for accurate limiting. Provide clear error messages with retry information.

### 8. **Data Isolation**

Ensure strict data isolation between tenants. Add tenant_id to all queries. Use database constraints to prevent cross-tenant data leaks.

### 9. **Scalable Architecture**

Design for horizontal scaling from the start. Use caching strategically. Implement database read replicas for reporting queries.

### 10. **Compliance & Security**

Support data export and deletion for GDPR compliance. Encrypt sensitive data at rest and in transit. Implement audit logging for sensitive operations.

---

**Further Reading:**

- Stripe Billing Documentation
- Multi-Tenancy Patterns by Microsoft
- SaaS Security Best Practices
- Feature Flag Systems (LaunchDarkly, Unleash)
