# Razorpay Payment Integration Guide

This guide walks you through wiring Razorpay checkout for selling Punchly plans.

---

## Step 1: Razorpay Account Setup

1. Sign up at https://razorpay.com
2. Complete KYC (business verification)
3. Go to Settings → API Keys
4. Copy:
   - **Key ID** (starts with `rzp_live_` or `rzp_test_`)
   - **Key Secret** (keep this secret!)
5. Save test keys locally; production keys go to Vercel only

---

## Step 2: Install Razorpay SDK

```bash
cd smart-timekeeper-main
bun add razorpay  # server-side SDK
```

Note: For client-side, we'll load from CDN instead of npm.

---

## Step 3: Add Environment Variables

**Locally (`.env`):**
```
RAZORPAY_KEY_ID="rzp_test_xxxxx"
RAZORPAY_KEY_SECRET="test_secret_xxxxx"
```

**Vercel (Settings → Environment Variables):**
```
RAZORPAY_KEY_ID=rzp_live_xxxxx
RAZORPAY_KEY_SECRET=live_secret_xxxxx
```

---

## Step 4: Create Payment Server Functions

Create file: `src/lib/payment.functions.ts`

```typescript
import { createServerFn } from "@tanstack/react-start";
import { requireSupabaseAuth } from "@/integrations/supabase/auth-middleware";
import Razorpay from "razorpay";

const razorpay = new Razorpay({
  key_id: process.env.RAZORPAY_KEY_ID!,
  key_secret: process.env.RAZORPAY_KEY_SECRET!,
});

/**
 * Create a Razorpay order for the given plan
 */
export const createRazorpayOrder = createServerFn({ method: "POST" })
  .middleware([requireSupabaseAuth])
  .inputValidator((data: { plan_id: string; tenant_id: string }) => data)
  .handler(async ({ data, context }) => {
    const { supabase, userId } = context;
    const { plan_id, tenant_id } = data;

    // 1. Verify user is admin of this tenant
    const { data: role } = await supabase.rpc("has_role", {
      _user_id: userId,
      _role: "client_admin",
      _tenant_id: tenant_id,
    });
    if (!role) throw new Error("Forbidden: not admin of this tenant");

    // 2. Get plan details
    const { data: plan } = await supabase
      .from("plans")
      .select("*")
      .eq("id", plan_id)
      .single();
    if (!plan) throw new Error("Plan not found");

    // 3. Calculate amount in paise (₹1 = 100 paise)
    const amountInPaise = Math.round(plan.price * 100);

    // 4. Create Razorpay order
    const order = await razorpay.orders.create({
      amount: amountInPaise,
      currency: "INR",
      receipt: `order_${tenant_id}_${Date.now()}`,
      notes: {
        tenant_id,
        plan_id,
      },
    });

    // 5. Store order in DB (for tracking)
    const { error: storeErr } = await supabase.from("payment_orders").insert({
      tenant_id,
      razorpay_order_id: order.id,
      plan_id,
      amount_paise: amountInPaise,
      status: "pending",
    });
    if (storeErr) throw new Error(storeErr.message);

    return {
      order_id: order.id,
      amount: plan.price,
      currency: "INR",
      key_id: process.env.RAZORPAY_KEY_ID!,
    };
  });

/**
 * Verify payment after Razorpay callback
 * (Called from webhook)
 */
export const verifyRazorpayPayment = createServerFn({ method: "POST" })
  .inputValidator(
    (data: {
      razorpay_order_id: string;
      razorpay_payment_id: string;
      razorpay_signature: string;
    }) => data
  )
  .handler(async ({ data }) => {
    const { supabaseAdmin } = await import(
      "@/integrations/supabase/client.server"
    );

    const { razorpay_order_id, razorpay_payment_id, razorpay_signature } = data;

    // 1. Verify signature (important: prevents fake payments)
    const body = razorpay_order_id + "|" + razorpay_payment_id;
    const expectedSignature = require("crypto")
      .createHmac("sha256", process.env.RAZORPAY_KEY_SECRET!)
      .update(body)
      .digest("hex");

    if (expectedSignature !== razorpay_signature) {
      throw new Error("Invalid payment signature");
    }

    // 2. Get order from DB
    const { data: order } = await supabaseAdmin
      .from("payment_orders")
      .select("*, plans(*)")
      .eq("razorpay_order_id", razorpay_order_id)
      .single();

    if (!order) throw new Error("Order not found");
    if (!order.plans) throw new Error("Plan not found");

    // 3. Create subscription
    const { plans: plan } = order;
    const expiresAt =
      plan.billing === "lifetime"
        ? null
        : plan.billing === "monthly"
          ? new Date(Date.now() + 30 * 86400000).toISOString()
          : new Date(Date.now() + 365 * 86400000).toISOString();

    const { error: subErr } = await supabaseAdmin.from("subscriptions").insert({
      tenant_id: order.tenant_id,
      plan_id: plan.id,
      razorpay_payment_id,
      status: "active",
      expires_at: expiresAt,
    });
    if (subErr) throw new Error(subErr.message);

    // 4. Update order status
    await supabaseAdmin
      .from("payment_orders")
      .update({ status: "completed" })
      .eq("id", order.id);

    // 5. Send confirmation email (optional)
    // await sendConfirmationEmail(order.tenant_id, plan.name);

    return { success: true, subscription: { plan: plan.name, expires_at: expiresAt } };
  });
```

---

## Step 5: Create Webhook Endpoint

Create file: `src/routes/webhook/razorpay.ts`

```typescript
import { json } from "@tanstack/react-start";
import { verifyRazorpayPayment } from "@/lib/payment.functions";

/**
 * Webhook for Razorpay payment events
 * Called by Razorpay after payment is complete
 */
export async function POST(req: Request) {
  try {
    const body = await req.json();

    // Razorpay sends events, we care about payment.authorized
    if (body.event !== "payment.authorized") {
      return json({ ok: true }); // Ignore other events
    }

    const { razorpay_order_id, razorpay_payment_id, razorpay_signature } =
      body.payload.payment.entity;

    // Verify and process payment
    await verifyRazorpayPayment({
      razorpay_order_id,
      razorpay_payment_id,
      razorpay_signature,
    });

    return json({ success: true });
  } catch (err) {
    console.error("[Webhook] Payment error:", err);
    // Return 200 to acknowledge receipt (prevents Razorpay retry)
    return json({ error: (err as Error).message }, { status: 200 });
  }
}
```

---

## Step 6: Create Database Table

Run in Supabase SQL editor:

```sql
-- Payment orders table
CREATE TABLE IF NOT EXISTS payment_orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  plan_id UUID NOT NULL REFERENCES plans(id),
  razorpay_order_id TEXT NOT NULL UNIQUE,
  amount_paise INTEGER NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending', -- pending, completed, failed
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- RLS: tenants can see their own orders
ALTER TABLE payment_orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY "tenant_sees_own_orders" ON payment_orders
  FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM user_roles
      WHERE user_roles.tenant_id = payment_orders.tenant_id
        AND user_roles.user_id = auth.uid()
    )
  );

-- Admin: create orders
CREATE POLICY "admin_create_orders" ON payment_orders
  FOR INSERT
  WITH CHECK (
    EXISTS (
      SELECT 1 FROM user_roles
      WHERE user_roles.tenant_id = payment_orders.tenant_id
        AND user_roles.user_id = auth.uid()
        AND user_roles.role = 'client_admin'
    )
  );
```

---

## Step 7: Create Checkout Modal Component

Create file: `src/components/RazorpayCheckoutModal.tsx`

```typescript
import { useState } from "react";
import { Button } from "@/components/ui/button";
import { Dialog, DialogContent, DialogHeader, DialogTitle } from "@/components/ui/dialog";
import { createRazorpayOrder } from "@/lib/payment.functions";
import { useQuery } from "@tanstack/react-query";

interface RazorpayCheckoutModalProps {
  isOpen: boolean;
  onClose: () => void;
  planId: string;
  planName: string;
  amount: number;
  tenantId: string;
  onSuccess?: () => void;
}

export function RazorpayCheckoutModal({
  isOpen,
  onClose,
  planId,
  planName,
  amount,
  tenantId,
  onSuccess,
}: RazorpayCheckoutModalProps) {
  const [isProcessing, setIsProcessing] = useState(false);

  const handlePayment = async () => {
    setIsProcessing(true);
    try {
      // 1. Create order on backend
      const orderData = await createRazorpayOrder({
        plan_id: planId,
        tenant_id: tenantId,
      });

      // 2. Load Razorpay script if not already loaded
      if (!(window as any).Razorpay) {
        const script = document.createElement("script");
        script.src = "https://checkout.razorpay.com/v1/checkout.js";
        script.async = true;
        document.head.appendChild(script);
        await new Promise((r) => setTimeout(r, 500)); // Wait for script
      }

      // 3. Open Razorpay checkout
      const razorpay = new (window as any).Razorpay({
        key: orderData.key_id,
        order_id: orderData.order_id,
        amount: orderData.amount * 100, // Convert to paise
        currency: "INR",
        name: "Punchly",
        description: `Subscribe to ${planName}`,
        handler(response: any) {
          // 4. Verify payment on backend (via webhook)
          console.log("Payment successful:", response);
          onSuccess?.();
          onClose();
        },
        modal: {
          ondismiss() {
            console.log("Checkout closed");
          },
        },
      });

      razorpay.open();
    } catch (err) {
      console.error("Payment error:", err);
      alert("Payment failed. Please try again.");
    } finally {
      setIsProcessing(false);
    }
  };

  return (
    <Dialog open={isOpen} onOpenChange={onClose}>
      <DialogContent>
        <DialogHeader>
          <DialogTitle>Subscribe to {planName}</DialogTitle>
        </DialogHeader>
        <div className="space-y-4">
          <div className="text-center text-3xl font-bold">₹{amount}</div>
          <Button
            onClick={handlePayment}
            disabled={isProcessing}
            className="w-full"
            size="lg"
          >
            {isProcessing ? "Processing..." : "Pay with Razorpay"}
          </Button>
        </div>
      </DialogContent>
    </Dialog>
  );
}
```

---

## Step 8: Update Pricing Page

In `src/routes/index.tsx`, wire up the checkout modal:

```typescript
import { useState } from "react";
import { RazorpayCheckoutModal } from "@/components/RazorpayCheckoutModal";

// In the pricing section:
export function PricingSection() {
  const [selectedPlan, setSelectedPlan] = useState<{
    id: string;
    name: string;
    price: number;
  } | null>(null);

  return (
    <>
      {plans.map((plan) => (
        <div key={plan.id}>
          {/* ... plan card UI ... */}
          <Button onClick={() => setSelectedPlan(plan)}>
            Pay Now
          </Button>
        </div>
      ))}

      <RazorpayCheckoutModal
        isOpen={!!selectedPlan}
        onClose={() => setSelectedPlan(null)}
        planId={selectedPlan?.id ?? ""}
        planName={selectedPlan?.name ?? ""}
        amount={selectedPlan?.price ?? 0}
        tenantId={currentTenant?.id ?? ""}
        onSuccess={() => {
          // Refresh subscriptions, show success message
          window.location.reload();
        }}
      />
    </>
  );
}
```

---

## Step 9: Configure Razorpay Webhook

1. Go to Razorpay Dashboard → Settings → Webhooks
2. Add webhook URL:
   ```
   https://smartpunch.vercel.app/webhook/razorpay
   ```
3. Select events:
   - `payment.authorized`
4. Copy webhook secret
5. Add to Vercel env vars:
   ```
   RAZORPAY_WEBHOOK_SECRET=whsec_xxxxx
   ```

---

## Step 10: Test Payment Flow

### Test Environment:

1. Create order:
   ```bash
   curl -X POST http://localhost:5173/api/payment/create-order \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer YOUR_TOKEN" \
     -d '{"plan_id": "plan_123", "tenant_id": "tenant_123"}'
   ```

2. Use test card in Razorpay modal:
   - Card: `4111 1111 1111 1111`
   - Expiry: Any future date (e.g., 12/25)
   - CVV: Any 3 digits

3. Check Supabase:
   - `payment_orders` table should have new "completed" order
   - `subscriptions` table should have new active subscription

### Production Checklist:

- [ ] Switch Razorpay keys to **live** (not test)
- [ ] Test with real card (small amount, ₹1)
- [ ] Verify email confirmation sent
- [ ] Confirm webhook webhook responses in Razorpay logs
- [ ] Monitor Sentry for errors

---

## Troubleshooting

**"Order not found" error:**
- Check Supabase `payment_orders` table has the order
- Verify RLS policies allow insert

**Payment doesn't show in dashboard:**
- Check webhook is being called (Razorpay → Webhooks → Logs)
- Check `subscriptions` table in Supabase
- Look at Sentry for errors

**Razorpay modal doesn't open:**
- Check browser console for errors
- Verify Razorpay script loaded (check Network tab)
- Check `key_id` is correct

---

## Security Notes

1. **Never expose `RAZORPAY_KEY_SECRET`** in frontend code
2. **Always verify signatures** in webhook (done above)
3. **Use HTTPS only** for webhook URL (Vercel provides this)
4. **Rotate webhook secret** periodically
5. **Log all payment events** for audit trail

---

## Next: Email Confirmations

After payment succeeds, send email to customer:

```typescript
// In verifyRazorpayPayment()
await sendConfirmationEmail({
  email: tenant.admin_email,
  company: tenant.name,
  plan: plan.name,
  expires_at: expiresAt,
});
```

For now, use free tier SendGrid or Mailgun.

---

## References

- Razorpay Orders API: https://razorpay.com/docs/api/orders/
- Razorpay Webhooks: https://razorpay.com/docs/webhooks/paymentauthorized/
- Test Cards: https://razorpay.com/docs/payments/payments-gateway/test-card-numbers/
