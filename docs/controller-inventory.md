# Controller Method Inventory

Complete reference of all controllers and their methods in TeamYellow.

---
## PlanController

**Authorization:** `[Authorize]` (class-level), overridden per method
**Dependencies:** `IPlanService`, `ICounsellorRepository`, `ISubscriptionRepository`, `UserManager<IdentityUser>`

### Methods

#### 1. `Index()` — GET `/Plan`

- **Auth:** `[AllowAnonymous]` — public
- **Calls:**
  - `_planService.GetActivePlans()` (always)
  - If authenticated AND `Paid_Counselor`:
    - `_userManager.GetUserAsync(User)`
    - `_counsellorRepository.GetByUserIdAsync(user.Id)`
    - `_subscriptionRepository.GetActiveSubscriptionByCounsellorId(counsellorId)`
- **Returns:** View with `List<PlanVM>`
- **Business Rule:** If user is `Paid_Counselor`, injects `ViewData["CurrentPlanId"]` so the view can highlight their active plan
- **Notes:** Free and non-authenticated users see all plans with no highlighting

#### 2. `Checkout(id)` — GET `/Plan/Checkout/{id}`

- **Auth:** `[Authorize(Roles = "Registered_Visitor,Paid_Counselor,Free_Counselor")]`
- **Params:** `int id` (PlanId)
- **Calls:**
  - `_planService.GetPlanById(id)`
  - If `Paid_Counselor`: `_counsellorRepository.GetByUserIdAsync`, `_subscriptionRepository.GetActiveSubscriptionByCounsellorId`
- **Business Rules:**
  - If plan not found or `!IsActive` → `NotFound()`
  - If `Free_Counselor` AND `plan.Price == 0` → redirect to Plan Index with error ("already subscribed to this plan")
  - If `Paid_Counselor` AND `plan.Price == 0` → redirect to Plan Index with error ("cannot downgrade to Free plan from here")
  - If `Paid_Counselor` AND `subscription.PlanId == id` → redirect with error ("already subscribed to this plan")
- **Returns:** Checkout view with `PlanVM`

### Private Helpers

```csharp
private static PlanVM MapToPlanVM(Plan plan)
// Maps Plan → PlanVM with features collection
```

---

## SubscriptionController

**Authorization:** `[Authorize(Roles = "Registered_Visitor,Paid_Counselor,Free_Counselor")]`
**Dependencies:** `ISubscriptionService`, `UserManager`, `ICounsellorRepository`, `ISubscriptionRepository`, `SignInManager`, `ILogger`

### Methods

#### 1. `Subscribe(planId)` — POST `/Subscription/Subscribe`

- **Auth:** `[ValidateAntiForgeryToken]`
- **Params:** `int planId`
- **Full Flow:**
  1. Get current `IdentityUser` → if null, return `Unauthorized()`
  2. Generate `returnUrl` and `cancelUrl` for PayPal callbacks
  3. Find `Counsellor` by `user.Id`
     - **If no counsellor exists:** Auto-generate unique `PractitionerLicenceId` (format: `[A-Z][0-9]{6}`) and call `_counsellorRepository.CreateAsync()`
  4. Check `_subscriptionRepository.GetActiveSubscriptionByCounsellorId()` → if already subscribed to `planId`, redirect with error
  5. Call `_subscriptionService.CreatePayPalOrder(planId, returnUrl, cancelUrl)`
  - **Free plan path** (returns empty string):
    - `_subscriptionService.SubscribeFree(counsellorId, userName, planId)`
    - `AssignCounsellorRole(userId, "Free_Counselor")`
    - Return `Success` view with `SubscriptionSuccessVM`
  - **Paid plan path** (returns PayPal URL):
    - `return Redirect(approvalUrl)` → user goes to PayPal
- **Error Handling:** `KeyNotFoundException` → 404; all others → log error + redirect to Plan Index

#### 2. `Success(orderId)` — GET `/Subscription/Success?token={orderId}`

- **Params:** `[FromQuery(Name = "token")] string orderId` (PayPal returns this)
- **Full Flow:**
  1. If `orderId` is empty → redirect to Home Index with error
  2. Get current user → if null, `Unauthorized()`
  3. Find counsellor by userId → if null, redirect Home with error
  4. `_subscriptionService.CompletePayPalSubscription(orderId, counsellorId, userName)`
     - Returns `SubscriptionResult.AlreadySubscribed` → redirect to Plan Index
  5. `AssignCounsellorRole(userId, "Paid_Counselor")`
  6. Fetch updated subscription and build `SubscriptionSuccessVM`
  7. Return `Success` view
- **Error Handling:** Any exception → redirect to Home Index ("Payment failed. Please try again.")

#### 3. `Cancel()` — GET `/Subscription/Cancel`

- **Returns:** Cancel view (no logic — user aborted PayPal payment)

### Private Helpers

```csharp
private async Task AssignCounsellorRole(string userId, string targetRole)
```
- **Logic:**
  1. Find user by ID
  2. Remove all of: `Registered_Visitor`, `Free_Counselor`, `Paid_Counselor`
  3. Add `targetRole` if not already assigned
  4. `_signInManager.RefreshSignInAsync(user)` — refreshes auth cookie so new role takes effect immediately

---

## Business Rules Summary (Cross-Controller)

| Rule | Where Enforced |
|------|---------------|
| Admin cannot remove own Administrator role | `AdminController.UserRoleDelete` (POST) |
| Role deletion blocked if users assigned | `RoleRepository.DeleteRoleAsync` |
| Free_Counselor cannot re-subscribe to Free plan | `PlanController.Checkout` |
| Paid_Counselor cannot downgrade to Free via Checkout | `PlanController.Checkout` |
| Paid_Counselor cannot subscribe to current plan again | `PlanController.Checkout` + `SubscriptionController.Subscribe` |
| Auto-create Counsellor profile on first subscribe | `SubscriptionController.Subscribe` |
| PractitionerLicenceId must be unique (retry loop) | `SubscriptionController.Subscribe` |
| Duplicate PayPal payment prevention | `TransactionRepository.ExistsByProviderOrderId` |
| Subscription cycle: 1 year for Yearly, 1 month otherwise | `SubscriptionRepository.CreateSubscription` |
| Role refresh after subscription (cookie refresh) | `SubscriptionController.AssignCounsellorRole` |