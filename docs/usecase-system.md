# Use Case Diagram: TeamYellow System

## Diagram 1 — Actor Use Case Map

```mermaid
flowchart LR
    Guest(["👤 Guest"])
    RV(["👤 Registered Visitor"])
    FC(["👤 Free Counsellor"])
    PC(["👤 Paid Counsellor"])
    Admin(["👤 Administrator"])

    subgraph PUB["Public Access"]
        UC1("Browse Plans")
        UC2("Register Account")
        UC3("Login / Logout")
    end

    subgraph SUB["Subscription Management"]
        UC4("View Plan Checkout")
        UC5("Subscribe: Free Plan")
        UC6("Subscribe: Paid Plan")
        UC7("Upgrade to Paid Plan")
        UC8("Switch Plan")
    end

    subgraph ADM["Administration"]
        UC12("View All Users")
        UC13("Assign Role to User")
        UC14("Remove Role from User")
        UC15("Create Role")
        UC16("Delete Role")
        UC17("View Login Logs")
    end

    Guest  --> UC1 & UC2 & UC3
    RV     --> UC1 & UC4 & UC5 & UC6
    FC     --> UC1 & UC7
    PC     --> UC1 & UC8
    Admin  --> UC12 & UC13 & UC14 & UC15 & UC16 & UC17
```

## Diagram 2 — Include Relationships

```mermaid
flowchart TD
    UC5("Subscribe: Free Plan")
    UC6("Subscribe: Paid Plan")
    UC7("Upgrade to Paid Plan")
    UC8("Switch Plan")
    UC9("Process PayPal Payment")
    UC10("Auto-Create Counsellor Profile")
    UC11("Cancel Existing Subscription")

    UC5 -. "include" .-> UC10
    UC6 -. "include" .-> UC10
    UC6 -. "include" .-> UC9
    UC7 -. "include" .-> UC9
    UC7 -. "include" .-> UC11
    UC8 -. "include" .-> UC9
    UC8 -. "include" .-> UC11
```

## Diagram 3 — Actor Generalization

```mermaid
flowchart TD
    Guest["👤 Guest"]
    RV["👤 Registered Visitor"]
    FC["👤 Free Counsellor"]
    PC["👤 Paid Counsellor"]
    Admin["👤 Administrator"]
    PayPal["⬛ PayPal (external system)"]

    Guest -->|generalizes| RV
    RV -->|generalizes| FC
    RV -->|generalizes| PC
```

---

## Use Case Descriptions

### Public Access

| Use Case | Actor(s) | Description |
| -------- | -------- | ----------- |
| Browse Plans | Guest, all authenticated | View all active plans, prices, and features. Paid_Counselor sees current plan highlighted. |
| Register Account | Guest | Create a new account. User receives `Registered_Visitor` role. |
| Login / Logout | Guest (login), all authenticated (logout) | Authenticate using ASP.NET Identity. Session tracked in `UserLog` table. |

### Subscription Management

| Use Case | Actor(s) | Description |
| -------- | -------- | ----------- |
| View Plan Checkout | Registered_Visitor, Free_Counselor, Paid_Counselor | See plan details before confirming subscription. Validates eligibility (no duplicate/downgrade). |
| Subscribe to Free Plan | Registered_Visitor | Select Free plan. No payment required. include: Auto-Create Counsellor Profile. |
| Subscribe to Paid Plan | Registered_Visitor | Select Monthly or Yearly plan. include: Auto-Create Counsellor Profile + Process PayPal Payment. |
| Upgrade to Paid Plan | Free_Counselor | Upgrade from Free to Monthly/Yearly. include: Process PayPal Payment + Cancel Existing Subscription. |
| Switch Plan | Paid_Counselor | Switch between Monthly and Yearly. include: Process PayPal Payment + Cancel Existing Subscription. |
| Process PayPal Payment | System (triggered by above) | Create PayPal order, redirect to PayPal, capture payment, record transaction. |
| Auto-Create Counsellor Profile | System (triggered by subscribe) | If no Counsellor record exists for the user, generate a unique PractitionerLicenceId and create one. |
| Cancel Existing Subscription | System (triggered by upgrade/switch) | Mark the old active subscription as Cancelled before creating the new one. |

### Administration

| Use Case | Actor(s) | Description |
| -------- | -------- | ----------- |
| View All Users | Administrator | List all Identity users by email. |
| Assign Role to User | Administrator | Add a role to a selected user. |
| Remove Role from User | Administrator | Remove a role from a user. Cannot remove own Administrator role (business rule). |
| Create Role | Administrator | Add a new Identity role to the system. |
| Delete Role | Administrator | Delete a role. Blocked if users are currently assigned to it. |
| View Login Logs | Administrator | Audit log of all user login/logout sessions, including abandoned sessions. |
