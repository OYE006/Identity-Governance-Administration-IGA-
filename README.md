# Identity-Governance-Administration-IGA
This project focuses on Identity Governance &amp; Administration (IGA) and lifecycle management (Joiner-Mover-Leaver) using an enterprise identity directory structure, completely independent of cloud storage buckets.


## Part 1: Designing the Automated JML Lifecycle Workflow

The recommended design uses Workday as the authoritative HR System of Record and Microsoft Entra ID as the central identity and access-control platform. Access should be granted primarily through role/attribute-based groups.


### The Joiner — Automated Onboarding
   
  1. Trigger Source:

     Workday HR record for a new employee or contractor, including employment status, department, job title, manager, location, and start date.
  
  2. Workflow:

     - HR creates the worker record in Workday.
     - Workday sends the worker's identity attributes to the IGA/IDP provisioning system.
     - The IGA platform creates the user's corporate identity in Entra ID.
     - The user's department, job function, employment type, and other attributes are evaluated against predefined access policies.
     - The user is automatically placed into the appropriate security groups,

       for example:
 
            Marketing-All
            Engineering-All
            Finance-All
            Sale-All
            Temporary-Contractors
       
     - Group memberships provision access to approved SaaS applications such as GitHub, Jira, Salesforce, and Workday.
     - The user receives an enrollment invitation for MFA.
     - Conditional Access/security policies require MFA before the account can access protected corporate resources.
     - The IGA platform records the provisioning event for audit purposes.
  
  3. System of Record:

     - Employee identity/employment data: Workday
     - Authentication and corporate identity: Entra ID/Okta
     - Authorization: IGA-managed security groups and application roles

  4. Security benefit:

     IT does not manually create accounts or assign individual permissions. Access is derived from authoritative HR attributes and approved roles.


### The Mover — Automated Department/Role Transfer

  1. Trigger Source:

     A change to department, job title, manager, employment type, or other authoritative attribute in Workday.

  2. Workflow:
     
     -	HR updates the employee's department in Workday.
     -	The change is synchronized to the IGA/IdP platform.
     -	The IGA engine detects the change from the employee's previous role to the new role.
     -	Existing role-based group memberships associated with the old department are automatically removed.
     -	New groups associated with the employee's new department are automatically assigned.
     -	Application entitlements inherited from the old groups are revoked.
     -	New application permissions are provisioned according to the employee's new role.
     -	The change is recorded in the access-governance audit trail.
     -	Where required, high-risk/new access can be routed to the employee's manager or application owner for approval.

            Example:
            Marketing → Engineering
            Before:
            Marketing-All → Marketing CRM + Marketing Database
            After the HR change:
            Remove Marketing-All → revoke Marketing access
            Add Engineering-All → provision Engineering tools
            The key principle is remove before/add, or otherwise ensure the resulting access is calculated from the employee's current role rather than simply accumulating permissions.

  3. System of Record:
     -	Workday: authoritative employment/department information
     -	IGA/Entra ID: identity and group lifecycle
     -	Application: final enforcement of application permissions

  4. Security benefit:

     This prevents permission creep, where employees retain privileges from previous positions.


### The Leaver — Automated Offboarding/Kill-Switch

  1. Trigger Source:

     Workday termination event or contractor contract-end date.

  2. Workflow:
     - HR enters the termination/end date into Workday.
     - The IGA system receives the termination event.
     - The user's account is immediately disabled/blocked.
     - Active authentication sessions and refresh tokens are revoked.
     - Group memberships and application assignments are removed or disabled.
     - Provisioning connectors send deprovisioning commands to connected SaaS applications.
     - Administrative roles are removed immediately.
     - API keys, credentials, and other identity-linked access mechanisms are disabled or rotated where supported.
     - The event is written to centralized audit logs.
     - Security/IT receives an alert confirming completion or identifying applications where automated deprovisioning failed.
  
  3. System of Record:
     -	Workday: termination status and effective date
     -	IGA/Entra ID: identity lifecycle state
     -	Connected SaaS applications: application-level enforcement
  
  4. Security benefit:
     A former employee cannot continue accessing corporate systems after termination simply because an administrator forgot to manually disable an account.
     Important implementation point: A true enterprise kill-switch should not rely solely on blocking the Entra ID sign-in. Connected SaaS applications should use automated provisioning/
     deprovisioning (for example, SCIM) and appropriate session/token revocation mechanisms.



## Part 2: Performing an Access Certification & Audit Campaign

### Detailed Audit Decisions

  **User Account: j.doe@omnicorp.com**
  
  Role / Department: Marketing Manager
  
  Last Login Activity: 2 days ago
  
  Active Entitlements / Groups: Marketing-All, Jira-Users, Prod-Database-Admin
  
  Compliance Violation / Risk Identified: Marketing Manager has Prod-Database-Admin, This creates a least-privilege and potentially risk.
  
  Action Required: Modify
  
  Justification: Remove Prod-Database-Admin and maintain only Marketing access.

====================================================

  **User Account: c.smith@contractor.io**
  
  Role / Department: External Developer
  
  Last Login Activity: 145 days ago
  
  Active Entitlements / Groups: GitHub-Core, AWS-Dev-Access
  
  Compliance Violation / Risk Identified: External developer has been inactive for 145 days while retaining GitHub and AWS development access. This violates dormant-account/lifecycle
  controls and creates unnecessary external access risk.
  
  Action Required: Revoke
  
  Justification: Disable the account and remove access.

====================================================

  **User Account: a.jones@omnicorp.com**
  
  Role / Department: Former SysAdmin (Terminated)
  
  Last Login Activity: 180 days ago
  
  Active Entitlements / Groups: Global-Tenant-Admin, Domain-Controllers
  
  Compliance Violation / Risk Identified: Former SysAdmin was terminated 180 days ago but still has Global-Tenant-Admin and Domain-Controllers. This is a critical privileged-account
  violation.
  
  Action Required: Revoke
  
  Justification: Disable the identity, revoke sessions/tokens, remove all privileged roles, and investigate authentication/audit logs for activity since termination.

====================================================

  **User Account: m.chen@omnicorp.com**
  
  Role / Department: Transfer from Sales > Finance
  
  Last Login Activity: Today
  
  Active Entitlements / Groups: Sales-CRM-Full, Finance-Billing-System
  
  Compliance Violation / Risk Identified: Employee transferred from Sales to Finance but retained Sales-CRM-Full while receiving Finance-Billing-System. This is classic permission
  accumulation.
  
  Action Required: Modify 
  
  Justification: Remove Sales CRM privilege and retain Finance access appropriate to the new role.


## Part 3: Using a cloud identity provider environment such as Microsoft Entra ID to build an automated group-based lifecycle structure, provision a test contractor, assign group-based permissions, and execute an instant offboarding kill-switch.

### Step 1: Creating a Lifecycle Security Group

> Instead of managing permissions user-by-user (which leads to orphaned accounts and permission creep), enterprise governance requires group-based access control.

- Navigating to the Microsoft Azure Portal (portal.azure.com)
- In the search bar at the top, type and select Microsoft Entra ID to open central identity administration panel.
- Click on Groups > All groups
- Click the New group button at the top.
- Configuring the group settings:
  
      Group type: Select Security.
      Group name: Enter SG-Temporary-Contractors.
      Group description: Enter Lifecycle managed group for external short-term contractors.
      Membership type: Select Assigned (manual or rule-based assignment).
  
<img width="1352" height="566" alt="image" src="https://github.com/user-attachments/assets/38e1c768-1d74-4508-a3df-c6c478d6a088" />

- Click Create; Refresh the group list to verify SG-Temporary-Contractors appears.

<img width="1350" height="493" alt="image" src="https://github.com/user-attachments/assets/1da780b4-4770-40d0-95e9-91c646f79716" />


### Step 2: Provisioning a Test Lifecycle User (The Joiner)

- Simulating a new contractor joining OmniCorp under a governance-controlled identity profile.
- In the left-hand navigation menu (default directory) , click on Users > All users.
- Click + New user > Create new user.
- Fill out the core basic identity attributes + All properties as required:
  
      User principal name: Enter contractor.test (resulting in contractor.test@yourdomain.onmicrosoft.com).
      Display name: Enter Test Contractor.
      Password: Choose "Auto-generate password" or write down a secure temporary password.

  <img width="1351" height="579" alt="image" src="https://github.com/user-attachments/assets/a913ea9a-fb91-4b2a-94a3-729323b99b5e" />

- Click on next:Assignments To the section of Groups and roles on the user creation form:
  
      Click Add groups, search for SG-Temporary-Contractors, select it, and click Select.

  <img width="1338" height="575" alt="image" src="https://github.com/user-attachments/assets/bfb0c376-7977-48ab-9aad-73285630903f" />

- Click Review + create, then click Create. 

<img width="1341" height="585" alt="image" src="https://github.com/user-attachments/assets/e46afc89-7ac4-4421-b306-19c361fbd7c2" />

> The user is now officially provisioned and automatically linked to the contractor security group.


### Step 3: Mapping Group-Based Resource Access (Simulating Entitlements)

> Demonstrating how permissions scale securely via groups rather than individual assignments.

- In the Entra ID menu, navigate to Enterprise applications (or App registrations if testing application permissions).

  <img width="1362" height="492" alt="image" src="https://github.com/user-attachments/assets/6ef551f1-512f-4ef9-9492-d86aede3cc52" />

- Select a test application or simulation resource connected to your tenant (e.g., a mock SaaS dashboard or internal web app).
- Click on Users and groups under that application's configuration.
- Click + Add user/group:

      Under Users, select the group SG-Temporary-Contractors that you created in Step 2.
      Assign a role (e.g., Standard-Contractor-Access).

- Click Assign.

> Governance Takeaway: Any user added to SG-Temporary-Contractors instantly inherits access to this application without needing direct, manual permissions.


### Step 4: Execute the Instant Offboarding (The Leaver / Kill-Switch)

> Simulating the end of the contractor's lifecycle when their contract terminates or audit flags require an immediate lockdown.

- Go back to Users -> All users and click on your test user (contractor.test@yourdomain.onmicrosoft.com).
- At the top banner of the user's profile page, locate and click the Block sign-in icon / button (it may say Block account depending on your dashboard view).
- A confirmation prompt will appear asking: "Are you sure you want to block sign-in for Test Contractor?" Select Yes / Save.
- Verify the Governance & Security Impact:

  > Instant Session Revocation: Entra ID immediately invalidates all active refresh tokens and session assertions for this user. Even if the contractor currently has an open browser tab or       an active API token, their next request will fail authentication instantly.
  
  > Group Persistence for Audit: The user remains in the audit log history and group structure for compliance tracking, but their ability to authenticate is completely severed.
