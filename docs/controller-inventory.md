# Controller Method Inventory

Complete reference of all controllers and their methods in TeamYellow.

---

## HomeController

**Authorization:** None (public)
**Dependencies:** `ILogger<HomeController>`

### Methods

#### 1. `Index()` — GET `/`

- **Auth:** Public (no attribute)
- **Returns:** View or redirect
- **Business Rules:**
  - If authenticated AND role = `Administrator` → redirect to `Admin/UserRoleIndex`
  - ⚠️ TODO: Commented-out redirects for `Manager` and `Paid_Counselor`/`Free_Counselor` roles
  - Otherwise → return Home Index view (landing page)

#### 2. `Privacy()` — GET `/Home/Privacy`

- **Returns:** Privacy policy view (no logic)

#### 3. `Error()` — GET `/Home/Error`

- **Cache:** `NoStore`
- **Returns:** Error view with `RequestId` from `Activity.Current` or `HttpContext.TraceIdentifier`

---

## AdminController

**Authorization:** `[Authorize(Roles = "Administrator")]` — all methods
**Dependencies:** `RoleRepository`, `UserRepository`, `UserRoleRepository`, `UserLogRepository`, `ILogger`

### Methods

#### 1. `UserRoleIndex()` — GET `/Admin/UserRoleIndex`

- **Calls:** `_userRepository.GetAllUsersAsync()`
- **Returns:** View with list of `UserVM` (email only)
- **Purpose:** Entry point for admin — lists all Identity users

#### 2. `UserRoleDetail(userName, message, isError)` — GET `/Admin/UserRoleDetail`

- **Params:** `string userName`, `string message = ""`, `bool isError = false`
- **Calls:** `_userRoleRepository.GetUserRolesAsync(userName)`
- **Returns:** View with list of `UserRoleVM`, passes message/isError via `ViewBag`
- **Purpose:** Shows all roles assigned to a specific user

#### 3. `UserRoleCreate(email?)` — GET `/Admin/UserRoleCreate`

- **Calls:** `_roleRepository.GetRoleSelectListAsync()`, `_userRepository.GetUserSelectListAsync(email)`
- **Returns:** Create form with role and user dropdowns
- **Purpose:** Display form to assign role to user

#### 4. `UserRoleCreate(UserRoleVM)` — POST `/Admin/UserRoleCreate`

- **Params:** `UserRoleVM` (Email, RoleName)
- **Validation:** `ModelState.IsValid`
- **Calls:** `_userRoleRepository.AddUserRoleAsync(email, roleName)`
- **Success:** Redirect to `UserRoleDetail` with success message
- **Failure:** Re-render form with error ("role might already exist for this user")

#### 5. `UserRoleDelete(email, roleName)` — GET `/Admin/UserRoleDelete`

- **Params:** `string email`, `string roleName`
- **Business Rule:** If email or roleName is empty → redirect to `UserRoleIndex` with error
- **Returns:** Confirmation view with `UserRoleVM`

#### 6. `UserRoleDelete(UserRoleVM)` — POST `/Admin/UserRoleDelete`

- **Params:** `UserRoleVM`
- **Validation:** `ModelState.IsValid` + Anti-forgery token
- **Critical Business Rule:** If `RoleName == "Administrator"` AND `Email == currentUser.Email` → BLOCK with error ("You cannot remove the Administrator role from your own account.")
- **Calls:** `_userRoleRepository.RemoveUserRoleAsync(email, roleName)`
- **Success:** Redirect to `UserRoleDetail` with success message
- **Failure:** Re-render with error

#### 7. `RoleIndex(message)` — GET `/Admin/RoleIndex`

- **Calls:** `_roleRepository.GetAllRolesVMAsync()`
- **Returns:** View with list of `RoleVM`, message in `ViewBag`

#### 8. `RoleCreate()` — GET `/Admin/RoleCreate`

- **Returns:** Create form with empty `RoleVM`

#### 9. `RoleCreate(RoleVM)` — POST `/Admin/RoleCreate`

- **Params:** `RoleVM` (RoleName)
- **Validation:** `ModelState.IsValid`
- **Calls:** `_roleRepository.CreateRoleAsync(roleName)`
- **Success:** Redirect to `RoleIndex` with success message
- **Failure:** Re-render with error ("may already exist")

#### 10. `RoleDelete(roleName)` — GET `/Admin/RoleDelete`

- **Business Rule:** If roleName empty → redirect to `RoleIndex` with error
- **Business Rule:** If role not found → redirect to `RoleIndex` with error
- **Returns:** Confirmation view with `RoleVM`

#### 11. `RoleDelete(RoleVM)` — POST `/Admin/RoleDelete`

- **Validation:** `ModelState.IsValid` + Anti-forgery token
- **Calls:** `_roleRepository.DeleteRoleAsync(roleName)`
- **Business Rule:** Delete fails if users are currently assigned to this role
- **Success:** Redirect to `RoleIndex`
- **Failure:** Re-render ("may have users attached")

#### 12. `UserLogAll()` — GET `/Admin/UserLogAll`

- **Calls:** `_userLogRepository.GetAllAsync()`
- **Returns:** View with list of `UserLogVM` (sorted by LogInTime DESC)

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