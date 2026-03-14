# Subscription Flow Description

> Status: Verified with stakeholder — 2026-03-13
> Gap noted: Dashboard redirect after success not yet implemented in code.

---

## Overview

```
Anyone → Browse Plans → [Not logged in] → Sign Up/Login
                      → [Logged in]     → Checkout → Free path OR Paid path
                                                    → Success → Dashboard
```

---

## Phase 1 — Plan Discovery (Public)

- Route: `GET /Plan` (`PlanController.Index`)
- **Access:** Public (`[AllowAnonymous]`) — no login required
- Displays all active plans ordered by price: Free ($0) → Monthly ($49.99) → Yearly ($499.99)
- Each plan shows: name, description, price, features list
- If user is `Paid_Counselor`: their current plan card is highlighted (`ViewData["CurrentPlanId"]`)
- Non-authenticated users see all plans with no highlighting

---

## Phase 2 — Click Subscribe (Authentication Gate)

- Route: `GET /Plan/Checkout/{planId}` (`PlanController.Checkout`)
- **Access:** `[Authorize(Roles = "Registered_Visitor, Free_Counselor, Paid_Counselor")]`

```
User clicks "Subscribe" on a plan card
│
├── NOT logged in
│     └── ASP.NET Identity redirects → /Identity/Account/Login (or /Account/Register)
│           └── After login/register → returns to Checkout page
│
└── IS logged in → Validation checks:
      ├── Plan not found or IsActive = false            → 404 Not Found
      ├── Free_Counselor clicking Free plan             → redirect (already subscribed)
      ├── Paid_Counselor clicking Free plan             → redirect (cannot downgrade)
      └── Paid_Counselor clicking their current plan    → redirect (already subscribed)
      └── All checks pass → Show Checkout view (PlanVM)
```

---

## Phase 3A — Free Plan Flow

- Route: `POST /Subscription/Subscribe` (`SubscriptionController.Subscribe`)
- Triggered by: Submit on Checkout page for planId=1 (Price = $0.00)

```
SubscriptionController.Subscribe(planId=1)
│
├── 1. Get current IdentityUser
│
├── 2. Counsellor lookup / auto-create
│     ├── CounsellorRepository.GetByUserIdAsync(userId)
│     └── [If no Counsellor] → generate unique PractitionerLicenceId (format: A123456)
│           └── CounsellorRepository.CreateAsync(new Counsellor { ... })
│
├── 3. Duplicate subscription check
│     └── SubscriptionRepository.GetActiveSubscriptionByCounsellorId()
│           └── [If same planId] → redirect to Plan Index with error
│
├── 4. SubscriptionService.CreatePayPalOrder(planId=1, ...)
│     └── Plan.Price == 0 → return "" → skip PayPal entirely ✅
│
├── 5. SubscriptionService.SubscribeFree(counsellorId, userName, planId=1)
│     ├── Check for existing active subscription
│     │     └── [If exists on different plan] → UpdateSubscription(Status = Cancelled)
│     ├── SubscriptionRepository.CreateSubscription()
│     │     Status = Active, CycleStart = now, CycleEnd = now + 1 month
│     └── TransactionRepository.CreateTransaction()
│           Amount = 0.00, Provider = "PayPal", Status = Captured (audit record)
│
├── 6. AssignCounsellorRole(userId, "Free_Counselor")
│     ├── Remove: Registered_Visitor (if held)
│     ├── Remove: Paid_Counselor (if held — plan downgrade edge case)
│     ├── Add: Free_Counselor
│     └── SignInManager.RefreshSignInAsync() → auth cookie updated immediately
│
└── 7. → Show Success view (SubscriptionSuccessVM: message, plan name, cycle dates)
          ⚠️ GAP: No redirect to Dashboard yet — needs implementation
          Expected: redirect to Counsellor/Index (dashboard)
```

---

## Phase 3B — Paid Plan Flow (PayPal)

- Route: `POST /Subscription/Subscribe` (`SubscriptionController.Subscribe`)
- Triggered by: Submit on Checkout page for planId=2 (Monthly) or planId=3 (Yearly)

```
SubscriptionController.Subscribe(planId=2 or 3)
│
├── 1-3. Same as Free flow (user lookup, counsellor create, duplicate check)
│
├── 4. SubscriptionService.CreatePayPalOrder(planId, returnUrl, cancelUrl)
│     ├── PlanRepository.GetPlanById(planId) → Price = 49.99 or 499.99
│     ├── PayPalService.GetAccessToken() → cached OAuth2 token from PayPal API
│     ├── PayPalService.CreateOrder(amount, "CAD", returnUrl, cancelUrl, customId)
│     │     └── POST https://api-m.sandbox.paypal.com/v2/checkout/orders
│     └── Returns PayPal approval URL
│
└── 5. Redirect(approvalUrl) → User leaves app, goes to PayPal website
```

**On PayPal — two outcomes:**

### 3B-i: User APPROVES payment

```
PayPal redirects → GET /Subscription/Success?token={orderId}
SubscriptionController.Success(orderId)
│
├── 1. Validate orderId not empty → else redirect Home with error
├── 2. Get current IdentityUser
├── 3. CounsellorRepository.GetByUserIdAsync() → must exist (created in step 2 of Subscribe)
│
├── 4. SubscriptionService.CompletePayPalSubscription(orderId, counsellorId, userName)
│     ├── PayPalService.CaptureOrder(orderId)
│     │     └── POST /v2/checkout/orders/{orderId}/capture → PayPal
│     │           Returns: (captureId, customId=planId)
│     │
│     ├── TransactionRepository.ExistsByProviderOrderId(captureId)
│     │     └── [If duplicate] → return AlreadySubscribed (prevent double-charge)
│     │
│     ├── PlanRepository.GetPlanById(planId from customId)
│     │
│     ├── SubscriptionRepository.GetActiveSubscriptionByCounsellorId()
│     │     └── [If exists on different plan] → UpdateSubscription(Status = Cancelled)
│     │
│     ├── SubscriptionRepository.CreateSubscription()
│     │     Status = Active
│     │     CycleStart = now
│     │     CycleEnd = now + 1 year  (BillingType = "Yearly")
│     │              = now + 1 month (BillingType = "Monthly")
│     │
│     └── TransactionRepository.CreateTransaction()
│           PayerName, Amount, Currency = "CAD"
│           Provider = "PayPal", ProviderOrderId = captureId
│           Status = Captured, PaidAt = now
│
├── 5. AssignCounsellorRole(userId, "Paid_Counselor")
│     ├── Remove: Registered_Visitor, Free_Counselor (if held)
│     ├── Add: Paid_Counselor
│     └── SignInManager.RefreshSignInAsync() → auth cookie updated immediately
│
└── 6. → Show Success view (SubscriptionSuccessVM)
          ⚠️ GAP: No redirect to Dashboard yet — needs implementation
          Expected: redirect to Counsellor/Index (dashboard)
```

### 3B-ii: User CANCELS on PayPal

```
PayPal redirects → GET /Subscription/Cancel
SubscriptionController.Cancel()
│
└── Show Cancel view
      No charge made, no database records created
```

---

## Phase 4 — Upgrade Path (Free → Paid)

When a `Free_Counselor` subscribes to Monthly or Yearly:

```
Free_Counselor → Checkout(planId=2) → Subscribe(planId=2)
│
├── Existing Free subscription found (Status = Active)
├── Free subscription → Status = Cancelled (UpdatedAt = now)
├── New Monthly subscription created (Status = Active)
├── PaymentTransaction recorded for new subscription
└── Role: Free_Counselor removed → Paid_Counselor assigned
```

---

## Database Records Created Per Flow

| Flow | Subscription Table | PaymentTransaction Table |
|------|--------------------|--------------------------|
| Free plan (new) | 1 row created (Active) | 1 row (Amount = 0.00) |
| Free plan (upgrade from paid) | Old row updated (Cancelled) + 1 new (Active) | 1 new row (Amount = 0.00) |
| Paid plan (new) | 1 row created (Active) | 1 row (Amount = 49.99 or 499.99) |
| Paid plan (upgrade) | Old row updated (Cancelled) + 1 new (Active) | 1 new row |
| Cancel on PayPal | No change | No change |

---

## Code Gap: Dashboard Redirect

**Current behavior:** After successful subscription (free or paid), the app shows a `Success` view with subscription details and stops there.

**Expected behavior:** Redirect to Counsellor dashboard (`Counsellor/Index`).

**What needs to change:**

```csharp
// SubscriptionController.Subscribe() — free plan path
// CURRENT:
return View("Success", vm);

// SHOULD BE (after dashboard is implemented):
return RedirectToAction("Index", "Counsellor");
// or pass success message via TempData:
TempData["SuccessMessage"] = result == SubscriptionResult.PlanChanged
    ? "Your plan has been updated successfully."
    : "You have successfully subscribed to the Free plan.";
return RedirectToAction("Index", "Counsellor");
```

Same change needed in `SubscriptionController.Success()` (paid plan path).

Also requires `HomeController.Index()` to uncomment and implement the Counsellor redirect:
```csharp
// Currently commented out — needs to be activated
if (User.IsInRole("Paid_Counselor") || User.IsInRole("Free_Counselor"))
    return RedirectToAction("Index", "Counsellor");
```

---

## Role Transition Summary

```
[New User Registration]
        ↓
  Registered_Visitor
        ↓
  ┌─────────────────┐
  │  Chooses Plan   │
  └────────┬────────┘
           │
    ┌──────┴──────┐
    ▼             ▼
Free Plan      Paid Plan (Monthly/Yearly)
    ↓               ↓
Free_Counselor  Paid_Counselor
    │
    └─ Can upgrade → Paid_Counselor
                      (Free subscription cancelled)
```
