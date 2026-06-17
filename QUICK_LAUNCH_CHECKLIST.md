# 🚨 IMMEDIATE ACTION PLAN — BEFORE SELLING

## This Week (8-10 hours of work)

### TODAY (Do First)
```
BLOCKING: Cannot sell until these are fixed
Time: 45 minutes
```

**1. Fix Exposed Supabase Keys (15 min)**
- [ ] Go to https://app.supabase.com → Your project → Settings → API
- [ ] Click "Regenerate" on all keys
- [ ] Add new keys to Vercel: Settings → Environment Variables
  - SUPABASE_URL
  - SUPABASE_PUBLISHABLE_KEY
  - SUPABASE_PROJECT_ID
- [ ] Update `.env` locally with new keys
- [ ] Redeploy to Vercel

**2. Remove Secrets from Git (10 min)**
```bash
# From project root
echo ".env" >> .gitignore
echo ".env.local" >> .gitignore
echo ".env.*.local" >> .gitignore
git rm --cached .env
git add .gitignore
git commit -m "Remove .env from git history - add to gitignore"
git push origin main
```

**3. Add Privacy Policy (1 hour)**
- [ ] Create `src/routes/privacy.tsx`
- [ ] Include: data types, retention, DPDP Act compliance
- [ ] Add link in footer

**4. Add Terms of Service (1 hour)**
- [ ] Create `src/routes/terms.tsx`
- [ ] Include: refund policy, account suspension, acceptable use

---

### TUESDAY (Next Priority)
```
REVENUE BLOCKING: No payments can be processed
Time: 2.5 hours
```

**5. Wire Razorpay Checkout**
- [ ] Install SDK: `bun add razorpay`
- [ ] Create `src/lib/payment.functions.ts` with `createRazorpayOrder()`
- [ ] Add webhook endpoint: `src/routes/webhook/razorpay.ts`
- [ ] Connect "Pay Now" button on pricing page to checkout modal
- [ ] Test with Razorpay test keys first
- [ ] Switch to production keys before going live

**Files to Create:**
- `src/lib/payment.functions.ts` — Order creation
- `src/lib/payment.helpers.ts` — Webhook validation
- `src/routes/webhook/razorpay.ts` — Payment callback
- `src/components/RazorpayCheckoutModal.tsx` — UI

**Test:**
```
1. Go to pricing page
2. Click "Pay Now" for monthly plan
3. Razorpay modal appears
4. Complete test payment (use test card: 4111 1111 1111 1111, any future date, any CVV)
5. Verify subscription created in Supabase
6. Check order stored in payments table
```

---

### WEDNESDAY (Testing & Monitoring)
```
PRODUCT RISK: Must test on real phones
Time: 3-4 hours
```

**6. Test Check-In on Real Phones**
- [ ] Install app on Android phone (smartpunch.vercel.app)
- [ ] Install app on iPhone (smartpunch.vercel.app)
- [ ] Test camera permission flow
- [ ] Test GPS permission flow
- [ ] Take selfie, verify upload works
- [ ] Test with slow internet (throttle to 3G)
- [ ] Test offline: check-in without network, then sync when online
- [ ] Check battery drain (5 min of GPS + camera)

**Success Criteria:**
- No crashes or freezes
- Permission errors show friendly messages
- Photos upload within 5 seconds (on good network)
- Offline queue works and syncs on reconnect

**7. Add Error Monitoring**
- [ ] Sign up at sentry.io (free tier)
- [ ] Create project, get DSN
- [ ] Add Sentry to package.json: `bun add @sentry/react`
- [ ] Initialize in `src/start.ts`
- [ ] Add to Vercel env vars: `VITE_SENTRY_DSN="..."`
- [ ] Wrap all server functions in error handler
- [ ] Test: cause an error in dev, see it in Sentry dashboard

---

### THURSDAY (Verify Everything)
```
FINAL CHECK: All systems working
Time: 2 hours
```

**8. Test Complete Flow**
- [ ] Sign up as new user
- [ ] Create company + branch
- [ ] Add 2 staff members
- [ ] Check in as staff (with photo + GPS)
- [ ] Check admin dashboard shows attendance
- [ ] Request leave as staff
- [ ] Approve leave as admin
- [ ] Generate payslip
- [ ] Test billing: buy a plan
- [ ] Verify all errors are captured in Sentry

**9. Deploy to Vercel**
- [ ] Run build locally: `bun run build`
- [ ] Fix any TypeScript errors
- [ ] Push to main branch
- [ ] Vercel auto-deploys
- [ ] Verify live at https://smartpunch.vercel.app
- [ ] Test one more end-to-end flow on live

---

## HOLD UNTIL v1.1 (Post-Launch)

These don't block selling, do them after first 3 customers:

- [ ] Google OAuth integration (2-3 hours)
- [ ] WhatsApp notifications (3-4 hours)
- [ ] SMS OTP backup (1-2 hours)
- [ ] Email digest notifications (2 hours)
- [ ] Audit log viewer UI (1-2 hours)

---

## Rollout Plan

**Week 1:** Fix all IMMEDIATE issues above  
**Week 2:** Sell to 1-2 beta customers (preferably friends/test companies)  
**Week 3:** Gather feedback, iterate  
**Week 4:** Open to public  

---

## Key Contacts When Selling

Before your first customer signs up, prepare:

**1. Legal/Compliance Contact**
- Indian data protection lawyer (₹5k-15k for initial review)
- For DPDP Act compliance + privacy policy review

**2. Razorpay Account**
- Sign up at razorpay.com
- Create API keys (test + production)
- Set up webhook for payment confirmations

**3. Sentry Account**
- Sign up at sentry.io
- Create project for error tracking

**4. Customer Support Plan**
- Email address (support@punchly.app or similar)
- How you'll respond to issues
- Who handles onboarding

---

## Support/Questions?

If stuck on:
- Razorpay: Read https://razorpay.com/docs/api/orders/
- Sentry: Read https://docs.sentry.io/platforms/javascript/
- Supabase: Read https://supabase.com/docs
- TanStack: Read https://tanstack.com/start/latest

**Critical:** Post questions in code, don't try to hack through unknown issues.
