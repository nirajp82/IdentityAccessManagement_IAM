# 📘 Section: Access Control Models (RBAC, ABAC, PBAC)

### Context: The "MoneyGuard" Banking App

In a small .NET app, you might just use `[Authorize(Roles="Admin")]`. But in an enterprise like MoneyGuard, hardcoding roles eventually breaks because the rules get too complex. You need a structured model to manage *who* gets to do *what*.

We will explore the evolution of these models: from **Static (RBAC)** to **Dynamic (ABAC)** to **Logic-Driven (PBAC)**.

---

### 1. RBAC (Role-Based Access Control)

**"The Standard Approach"**

RBAC is the most common model. Instead of assigning permissions directly to users (which is a nightmare to manage), you assign permissions to a **Role**, and then assign users to that Role.

* **The Concept:**
* **User:** Alice
* **Role:** "Teller"
* **Permission:** "View Account Balance"
* **Logic:** Alice is a Teller -> Tellers can View Balance -> Therefore, Alice can View Balance.


* **MoneyGuard Example:**
* **Scenario:** You have 1,000 employees. 500 are Tellers, 50 are Managers.
* **Implementation:** You create a "Teller" role in Azure AD/Okta. You give that role read-access to the `AccountService`. When you hire Alice, you just add her to the "Teller" group.
* **The Limitation (Role Explosion):** Suddenly, the NY branch needs Tellers who *can't* see VIP accounts. You create a "NY_Teller" role. Then the London branch needs Tellers who can approve small loans. You create "London_Loan_Teller". Soon, you have 5,000 roles for 1,000 people.



### 2. ABAC (Attribute-Based Access Control)

**"The Context-Aware Approach"**

ABAC solves "Role Explosion" by making decisions at runtime based on **attributes** (data tags).

* **The Concept:** Access is calculated using a formula:



* **Key Attributes:**
* **Subject (User):** Department, Clearance Level, Location.
* **Resource (Data):** Classification (Confidential/Public), Owner, Value.
* **Environment:** Time of day, IP address, Device health.


* **MoneyGuard Example:**
* **Scenario:** Alice (Teller) wants to view a VIP Account from home.
* **The Policy:** "Allow access IF User.Role = 'Teller' AND Resource.Type = 'Account' AND User.Location = 'OfficeNetwork'."
* **Result:** Alice is a Teller (Pass), accessing an Account (Pass), but she is at Home (Fail). **Access Denied.**
* **Why it's better:** You didn't need a "WorkFromHome_Teller" role. You just added a rule about location.



### 3. PBAC (Policy-Based Access Control)

**"The Logic-Driven Approach"**

PBAC is the modern evolution of ABAC. While ABAC focuses on the *attributes*, PBAC focuses on the **Policy Logic** itself. It often uses a centralized "Policy Engine" (like OPA - Open Policy Agent) to make decisions.

* **The Concept:** Decouple the code from the rules. Your .NET code asks the Policy Engine: "Can Alice do X?" The Engine runs a script and says "Yes/No".
* **MoneyGuard Example:**
* **The Complexity:** A transaction over $10,000 requires 2 managers to approve it, but only if the risk score is low.
* **The Implementation:** You don't write this `if/else` logic in C#. You write a Policy (in a language like Rego or XACML) and store it centrally.
* **The Flow:**
1. MoneyGuard API sends a JSON payload to the Policy Engine.
2. Policy Engine evaluates the complex rule.
3. Returns `{"allow": true}`.





---

### ⚔️ Comparison: When to use what?

| Feature | RBAC (Roles) | ABAC (Attributes) | PBAC (Policies) |
| --- | --- | --- | --- |
| **Best For** | Coarse-grained access (Menu visibility, basic admin vs user). | Fine-grained access (Row-level security, specific data fields). | Complex business logic & regulatory compliance. |
| **Management** | Easy to understand. Hard to scale. | Hard to set up initially. Scales infinitely. | Requires specialized skills (Policy as Code). |
| **.NET Mapping** | `User.IsInRole("Admin")` | `Requirement: User.Dept == Resouce.Dept` | External call to OPA or `PolicyServer`. |
| **Example** | "Managers can approve loans." | "Managers can approve loans *only* for their own branch." | "Managers can approve loans *if* the risk score < 50 AND it's a weekday." |

---

### 🚀 Real-World Enterprise Use Case

**Scenario:** The "Regional Data Restriction" at MoneyGuard.

**The Requirement:** MoneyGuard operates in the US and Europe (EU). Due to GDPR, US employees **must not** see the personal data of EU customers.

**Step-by-Step Implementation:**

1. **RBAC Layer (The Baseline):**
* We assign Alice (US Employee) the **"CustomerSupport"** Role.
* This gives her access to the *Support Application*. She can log in and see the menu.


2. **ABAC Layer (The Filter):**
* Alice searches for "Hans" (a German customer).
* **The Check:** The database query or API middleware intercepts the request.
* **The Attributes:**
* `User.Region = "US"`
* `Customer.Region = "EU"`


* **The Rule:** `Deny IF User.Region != Customer.Region`.
* **The Outcome:** The system returns "Record Restricted" or hides the row entirely.



**If we used only RBAC:** We would need "US_CustomerSupport" and "EU_CustomerSupport" roles and separate databases or complex code to keep them apart. ABAC handles it with one rule.

---

### ❓ FAQ

**Q1: Can I use RBAC and ABAC together?**
- **A:** Yes! This is the industry standard. Use **RBAC** for high-level access (can I login? can I see the 'Admin' tab?) and **ABAC** for low-level data access (can I see *this specific* record?). This is often called "RBAC with Attribute constraints."

**Q2: Isn't ABAC slow if it has to check attributes every time?**
- **A:** It can be if designed poorly. In a .NET Enterprise app, you typically cache the user's attributes in their Claims Principal (ID Token) when they log in. You only fetch the *Resource* attributes from the DB in real-time.

**Q3: Is PBAC just ABAC with a marketing name?**
- **A:** Mostly, yes. But PBAC implies you are managing policies as *code* (version controlled, tested) rather than just clicking checkboxes in a UI.
