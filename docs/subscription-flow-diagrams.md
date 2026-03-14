---
sidebar_position: 3
---

# Subscription Flow Diagrams

Complete visual documentation of the subscription flows in TeamYellow.

---

## Diagram 1: Free Subscription Flow (Detailed)

This diagram shows the complete flow when a user subscribes to the Free plan, including the automatic Counsellor profile creation logic.

```mermaid
sequenceDiagram
    actor User as User<br/>(Registered_Visitor)
    participant PC as PlanController
    participant SC as SubscriptionController
    participant SS as SubscriptionService
    participant CR as CounsellorRepository
    participant SR as SubscriptionRepository
    participant TR as TransactionRepository
    participant UM as UserManager
    participant SM as SignInManager
    participant DB as Database

    Note over User,DB: Phase 1: Browse Plans
    User->>PC: GET /Plan
    PC->>SS: GetActivePlans()
    SS->>DB: SELECT * FROM Plans WHERE IsActive = true
    DB-->>SS: [Free, Monthly, Yearly]
    SS-->>PC: List<Plan>
    PC-->>User: Display plans (sorted by price)

    Note over User,DB: Phase 2: Checkout
    User->>PC: Click "Get Free" button
    PC->>PC: Checkout(planId=1)
    PC->>SS: GetPlanById(1)
    SS->>DB: SELECT * FROM Plans WHERE PlanId = 1
    DB-->>SS: Free Plan ($0.00)
    SS-->>PC: Plan
    PC->>PC: Validate not already Free_Counselor
    PC-->>User: Show Checkout form (PlanVM)

    Note over User,DB: Phase 3: Subscribe - Counsellor Lookup/Creation
    User->>SC: POST /Subscription/Subscribe?planId=1
    SC->>UM: GetUserAsync(User)
    UM-->>SC: IdentityUser (userId, email)
    
    SC->>CR: GetByUserIdAsync(userId)
    CR->>DB: SELECT * FROM Counsellors WHERE UserId = ?
    DB-->>CR: null (no counsellor exists)
    CR-->>SC: null

    Note over SC,DB: Auto-Create Counsellor with Unique Licence ID
    SC->>SC: Generate licence ID loop
    loop Until unique licence ID found
        SC->>SC: Generate: [A-Z] + Random(100000-999999)
        SC->>CR: LicenceIdExistsAsync(licenceId)
        CR->>DB: SELECT COUNT(*) WHERE PractitionerLicenceId = ?
        DB-->>CR: 0 or 1
        CR-->>SC: exists (bool)
    end
    
    SC->>CR: CreateAsync(new Counsellor)
    Note right of SC: DisplayName = email<br/>PractitionerLicenceId = "A123456"<br/>IsActive = true
    CR->>DB: INSERT INTO Counsellors
    DB-->>CR: Counsellor (counsellorId)
    CR-->>SC: Counsellor

    Note over User,DB: Phase 4: Duplicate Subscription Check
    SC->>SR: GetActiveSubscriptionByCounsellorId(counsellorId)
    SR->>DB: SELECT * WHERE CounsellorId = ? AND Status = 'Active'
    DB-->>SR: null or Subscription
    SR-->>SC: existing subscription
    alt Already subscribed to same plan
        SC-->>User: Redirect with error "Already subscribed"
    end

    Note over User,DB: Phase 5: Create PayPal Order (Free Path)
    SC->>SS: CreatePayPalOrder(planId=1, returnUrl, cancelUrl)
    SS->>SS: Check Plan.Price
    Note right of SS: Price = $0.00 → Free plan
    SS-->>SC: "" (empty string = skip PayPal)

    Note over User,DB: Phase 6: Subscribe Free
    SC->>SS: SubscribeFree(counsellorId, userName, planId=1)
    
    alt Has active subscription on different plan
        SS->>SR: UpdateSubscription(oldSubscriptionId, Status = Cancelled)
        SR->>DB: UPDATE Subscriptions SET Status = 'Cancelled'
        DB-->>SR: Success
    end
    
    SS->>SR: CreateSubscription()
    Note right of SS: Status = Active<br/>CycleStart = now<br/>CycleEnd = now + 1 month<br/>PlanId = 1
    SR->>DB: INSERT INTO Subscriptions
    DB-->>SR: Subscription (subscriptionId)
    SR-->>SS: Subscription
    
    SS->>TR: CreateTransaction()
    Note right of SS: Amount = 0.00<br/>Provider = "PayPal"<br/>Status = Captured<br/>(audit record)
    TR->>DB: INSERT INTO PaymentTransactions
    DB-->>TR: Transaction
    TR-->>SS: Transaction
    SS-->>SC: SubscriptionResult.Created

    Note over User,DB: Phase 7: Role Assignment
    SC->>SC: AssignCounsellorRole(userId, "Free_Counselor")
    SC->>UM: GetRolesAsync(user)
    UM-->>SC: [Registered_Visitor]
    SC->>UM: RemoveFromRolesAsync(user, [Registered_Visitor])
    UM->>DB: DELETE FROM AspNetUserRoles
    DB-->>UM: Success
    SC->>UM: AddToRoleAsync(user, "Free_Counselor")
    UM->>DB: INSERT INTO AspNetUserRoles
    DB-->>UM: Success
    SC->>SM: RefreshSignInAsync(user)
    Note right of SC: Updates auth cookie immediately

    Note over User,DB: Phase 8: Success
    SC->>SR: GetActiveSubscriptionWithPlanByCounsellorId()
    SR->>DB: SELECT Subscription, Plan
    DB-->>SR: Subscription + Plan details
    SR-->>SC: Subscription with Plan
    SC-->>User: Show Success view<br/>(plan name, cycle dates)
    Note over User: ⚠️ TODO: Should redirect to<br/>Counsellor/Index (dashboard)
```

**Key Points:**
- Counsellor profile auto-created with unique licence ID (format: 1 letter + 6 digits)
- Zero-dollar transaction recorded for audit purposes
- Role changes from `Registered_Visitor` → `Free_Counselor`
- Auth cookie refreshed immediately (no re-login needed)

---

## Diagram 2: Paid Subscription Flow with PayPal (Success & Cancel Paths)

This diagram shows the complete PayPal integration including both success and cancel scenarios.

```mermaid
sequenceDiagram
    actor User as User<br/>(Registered_Visitor)
    participant SC as SubscriptionController
    participant SS as SubscriptionService
    participant PS as PayPalService
    participant PayPal as PayPal API
    participant CR as CounsellorRepository
    participant SR as SubscriptionRepository
    participant TR as TransactionRepository
    participant UM as UserManager
    participant SM as SignInManager
    participant DB as Database

    Note over User,DB: Phase 1-3: Same as Free Flow
    Note right of User: User browses plans,<br/>clicks Checkout,<br/>counsellor created if needed

    Note over User,DB: Phase 4: Create PayPal Order
    User->>SC: POST /Subscription/Subscribe?planId=2
    Note right of User: Monthly plan ($49.99)
    
    SC->>SC: [Counsellor lookup/creation - same as Free flow]
    SC->>SC: [Duplicate check - same as Free flow]
    
    SC->>SS: CreatePayPalOrder(planId=2, returnUrl, cancelUrl)
    SS->>DB: SELECT * FROM Plans WHERE PlanId = 2
    DB-->>SS: Plan (Price = 49.99)
    
    SS->>PS: CreateOrder(49.99, "CAD", returnUrl, cancelUrl, customId=2)
    
    Note over PS,PayPal: OAuth2 Token Request
    PS->>PayPal: POST /v1/oauth2/token
    Note right of PS: ClientId + ClientSecret<br/>(Basic Auth)
    PayPal-->>PS: access_token (cached)
    
    Note over PS,PayPal: Create Order
    PS->>PayPal: POST /v2/checkout/orders
    Note right of PS: Body:<br/>{amount: 49.99, currency: CAD,<br/>return_url, cancel_url,<br/>custom_id: "2"}
    PayPal-->>PS: {id: order_123, status: CREATED,<br/>links: [{rel: approve, href: ...}]}
    PS-->>SS: approvalUrl
    SS-->>SC: approvalUrl
    
    SC-->>User: Redirect(approvalUrl)
    User->>PayPal: [User navigates to PayPal]

    Note over User,PayPal: User on PayPal Website
    PayPal->>User: Show payment details<br/>(Amount: $49.99 CAD)
    
    alt User Approves Payment
        User->>PayPal: Click "Pay Now"
        PayPal->>PayPal: Authorize payment (hold funds)
        PayPal-->>User: Redirect to returnUrl
        
        Note over User,DB: SUCCESS PATH
        User->>SC: GET /Subscription/Success?token=order_123
        
        SC->>UM: GetUserAsync(User)
        UM-->>SC: IdentityUser
        
        SC->>CR: GetByUserIdAsync(userId)
        CR->>DB: SELECT * FROM Counsellors
        DB-->>CR: Counsellor
        CR-->>SC: Counsellor
        
        SC->>SS: CompletePayPalSubscription(order_123, counsellorId, userName)
        
        Note over SS,PayPal: Capture Payment
        SS->>PS: CaptureOrder(order_123)
        PS->>PayPal: POST /v2/checkout/orders/order_123/capture
        PayPal->>PayPal: Transfer funds from payer to merchant
        PayPal-->>PS: {capture_id: capture_456,<br/>custom_id: "2",<br/>status: COMPLETED}
        PS-->>SS: (captureId, customId)
        
        Note over SS,DB: Duplicate Payment Check
        SS->>TR: ExistsByProviderOrderId(capture_456)
        TR->>DB: SELECT COUNT(*) WHERE ProviderOrderId = ?
        DB-->>TR: 0 (not duplicate)
        TR-->>SS: false
        
        alt Duplicate payment detected
            SS-->>SC: SubscriptionResult.AlreadySubscribed
            SC-->>User: Redirect with error
        end
        
        SS->>DB: SELECT * FROM Plans WHERE PlanId = 2
        DB-->>SS: Plan (Monthly)
        
        alt Has active subscription on different plan
            SS->>SR: UpdateSubscription(Status = Cancelled)
            SR->>DB: UPDATE Subscriptions
            DB-->>SR: Success
        end
        
        SS->>SR: CreateSubscription()
        Note right of SS: Status = Active<br/>CycleStart = now<br/>CycleEnd = now + 1 month<br/>PlanId = 2
        SR->>DB: INSERT INTO Subscriptions
        DB-->>SR: Subscription (subscriptionId)
        
        SS->>TR: CreateTransaction()
        Note right of SS: PayerName, Amount = 49.99<br/>Currency = CAD<br/>Provider = PayPal<br/>ProviderOrderId = capture_456<br/>Status = Captured
        TR->>DB: INSERT INTO PaymentTransactions
        DB-->>TR: Transaction
        
        SS-->>SC: SubscriptionResult.Created
        
        Note over SC,DB: Role Assignment
        SC->>SC: AssignCounsellorRole(userId, "Paid_Counselor")
        SC->>UM: RemoveFromRolesAsync([Registered_Visitor, Free_Counselor])
        UM->>DB: DELETE FROM AspNetUserRoles
        SC->>UM: AddToRoleAsync("Paid_Counselor")
        UM->>DB: INSERT INTO AspNetUserRoles
        SC->>SM: RefreshSignInAsync(user)
        
        SC->>SR: GetActiveSubscriptionWithPlanByCounsellorId()
        SR->>DB: SELECT Subscription JOIN Plan
        DB-->>SR: Subscription + Plan
        SR-->>SC: Subscription
        
        SC-->>User: Show Success view
        Note over User: ⚠️ TODO: Redirect to dashboard
        
    else User Cancels Payment
        User->>PayPal: Click "Cancel and Return"
        PayPal->>PayPal: No payment made (order abandoned)
        PayPal-->>User: Redirect to cancelUrl
        
        Note over User,SC: CANCEL PATH
        User->>SC: GET /Subscription/Cancel
        SC-->>User: Show Cancel view
        Note right of User: "Payment cancelled.<br/>No charges made."
        Note over DB: No database records created
    end
```

**Key Points:**
- **PayPal OAuth:** Access token cached for performance
- **PayPal Order Flow:** Create → Approve (user on PayPal) → Capture (funds transferred)
- **Duplicate Prevention:** `ProviderOrderId` (capture ID) checked before creating transaction
- **Cancel Path:** Clean exit, no database changes
- **Role Upgrade:** Can upgrade from `Free_Counselor` to `Paid_Counselor` (old subscription cancelled)

---

## Diagram 3: Subscription Decision Flow (Activity Diagram)

This diagram shows all the validation and decision points in the subscription process.

```mermaid
flowchart TD
    Start([User clicks Subscribe on Plan]) --> CheckAuth{Is user<br/>logged in?}
    
    CheckAuth -->|No| LoginRedirect[Redirect to /Identity/Account/Login]
    LoginRedirect --> ReturnToCheckout[After login: return to<br/>/Plan/Checkout]
    ReturnToCheckout --> ShowCheckout[Show Checkout form]
    
    CheckAuth -->|Yes| CheckPlan{Which Plan?}
    
    CheckPlan -->|Free Plan<br/>planId=1| CheckFreeCounsellor{Is user already<br/>Free_Counselor?}
    CheckFreeCounsellor -->|Yes| ErrorAlreadySubscribed[Error: Already subscribed<br/>to Free plan]
    ErrorAlreadySubscribed --> EndError([Redirect to /Plan])
    
    CheckFreeCounsellor -->|No| CheckPaidDowngrade{Is user<br/>Paid_Counselor?}
    CheckPaidDowngrade -->|Yes| ErrorCannotDowngrade[Error: Cannot downgrade<br/>from Paid to Free]
    ErrorCannotDowngrade --> EndError
    
    CheckPaidDowngrade -->|No| CounsellorLookup[Look up Counsellor<br/>by UserId]
    
    CheckPlan -->|Paid Plan<br/>planId=2 or 3| CounsellorLookup
    
    CounsellorLookup --> CounsellorExists{Counsellor<br/>exists?}
    
    CounsellorExists -->|No| AutoCreate[Auto-create Counsellor]
    AutoCreate --> GenerateLicence[Generate unique<br/>PractitionerLicenceId]
    GenerateLicence --> CheckLicenceUnique{Licence ID<br/>unique?}
    CheckLicenceUnique -->|No| GenerateLicence
    CheckLicenceUnique -->|Yes| CreateCounsellor[INSERT Counsellor record]
    CreateCounsellor --> DuplicateCheck
    
    CounsellorExists -->|Yes| DuplicateCheck[Get active subscription<br/>for this counsellor]
    
    DuplicateCheck --> HasActiveSubscription{Has active<br/>subscription?}
    
    HasActiveSubscription -->|Yes| SamePlan{Same<br/>PlanId?}
    SamePlan -->|Yes| ErrorDuplicate[Error: Already subscribed<br/>to this plan]
    ErrorDuplicate --> EndError
    
    SamePlan -->|No| MarkOldCancelled[Mark old subscription<br/>Status = Cancelled]
    MarkOldCancelled --> PlanTypeCheck
    
    HasActiveSubscription -->|No| PlanTypeCheck{Plan Price?}
    
    PlanTypeCheck -->|$0.00<br/>Free Plan| SkipPayPal[CreatePayPalOrder<br/>returns empty string]
    SkipPayPal --> CreateFreeSubscription[SubscribeFree:<br/>Create Subscription<br/>+ Transaction Amount=0]
    CreateFreeSubscription --> AssignFreeRole[Assign role:<br/>Free_Counselor]
    AssignFreeRole --> RefreshCookie[RefreshSignInAsync]
    RefreshCookie --> ShowSuccess[Show Success view]
    ShowSuccess --> TodoDashboard[⚠️ TODO: Redirect to<br/>Counsellor/Index]
    TodoDashboard --> EndSuccess([End])
    
    PlanTypeCheck -->|> $0.00<br/>Paid Plan| CreatePayPalOrder[CreatePayPalOrder:<br/>POST to PayPal API]
    CreatePayPalOrder --> RedirectPayPal[Redirect user to<br/>PayPal approval URL]
    RedirectPayPal --> UserOnPayPal{User action<br/>on PayPal?}
    
    UserOnPayPal -->|Approve| CapturePayment[CaptureOrder:<br/>POST to PayPal<br/>capture endpoint]
    CapturePayment --> DuplicatePaymentCheck{Payment already<br/>captured?}
    DuplicatePaymentCheck -->|Yes| ErrorDuplicatePayment[Error: Payment already<br/>processed]
    ErrorDuplicatePaymentCheck --> EndError
    
    DuplicatePaymentCheck -->|No| CreatePaidSubscription[Create Subscription<br/>+ Transaction with<br/>capture details]
    CreatePaidSubscription --> AssignPaidRole[Assign role:<br/>Paid_Counselor]
    AssignPaidRole --> RefreshCookie
    
    UserOnPayPal -->|Cancel| ShowCancel[Show Cancel view:<br/>No charges made]
    ShowCancel --> EndCancel([End - No DB changes])
    
    style ErrorAlreadySubscribed fill:#ffcccc
    style ErrorCannotDowngrade fill:#ffcccc
    style ErrorDuplicate fill:#ffcccc
    style ErrorDuplicatePayment fill:#ffcccc
    style CreateFreeSubscription fill:#ccffcc
    style CreatePaidSubscription fill:#ccffcc
    style ShowSuccess fill:#ccffcc
    style ShowCancel fill:#ffffcc
    style TodoDashboard fill:#ffeb99
```

**Decision Points:**
1. **Authentication Gate:** Login required to proceed
2. **Role Validation:** 
   - Free_Counselor cannot re-subscribe to Free
   - Paid_Counselor cannot downgrade via checkout
3. **Counsellor Auto-Creation:** Generated if doesn't exist
4. **Duplicate Prevention:** 
   - Subscription: Cannot subscribe to same plan twice
   - Payment: Capture ID checked (prevents double-charge)
5. **Plan Type Routing:** Price determines PayPal vs direct activation
6. **PayPal User Decision:** Approve or Cancel

---

## Summary

These three diagrams provide complete visibility into:
- **Free flow:** Direct activation with role assignment
- **Paid flow:** PayPal integration with payment capture
- **Decision logic:** All validation and branching paths

All business rules from the code are visualized, making it easy to:
- Onboard new team members
- Identify edge cases
- Plan refactoring work
- Document for stakeholders

**Next:** We can create similar diagrams for Admin role management and Plan browsing flows.
