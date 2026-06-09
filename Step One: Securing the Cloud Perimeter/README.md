# Securing the Cloud Perimeter
## Summary
This project documents the foundational architecture, initial configuration, and essential defense-in-depth hygiene required when initializing a greenfield AWS organization. The first hours of a cloud environment's lifecycle are the most critical; leaving a master account unprotected or unmonitored invites immediate compromise. This implementation covers the deployment of multi-factor authentication (MFA) on the root identity, the engineering of proactive billing guardrails, and the initialization of a centralized identity infrastructure using AWS IAM Identity Center.

By establishing distinct administrative groups, deploying vetted permission sets, and enforcing the Principle of Least Privilege (PoLP), this architecture systematically reduces the cloud tenant's attack surface. Instead of relying on vulnerable, long-lived access keys or allowing engineers to operate with unrestricted root credentials, this setup transitions the environment to a secure, audit-ready workforce directory. This baseline configuration serves as the hardened, production-grade foundation for all future security operations, logging deployments, and infrastructure labs within this ecosystem.

This documentation doubles as a hands-on project write-up and a practical guide for anyone looking to build or audit their own cloud security perimeter. If you are following along with these steps to harden your own sandbox and run into any technical hurdles or have questions about the architectural choices, please feel free to reach out to me directly on [LinkedIn](https://linkedin.com/in/haydn-kuti).

---

## IAM Identity Center: Initialization and Root Hardening
Operating directly out of an AWS root account for daily administration is one of the highest risks in cloud security. The root user is the ultimate identity within the account boundary; it possesses unrestricted power, cannot be limited by explicit IAM policies, and its compromise represents a total tenant takeover. To mitigate this risk, the primary objective of this phase is to securely lock away the root credentials and shift all operational tasks to an enterprise identity provider.

### Enabling IAM Identity Center
Upon initializing the master billing account and landing on the AWS Management Console home, the first step is to establish a centralized directory service.
1. Navigate to the top global search bar and look for **IAM Identity Center** (formerly AWS Single Sign-On).
2. Selecting this service routes you to the main initialization dashboard.
3. If this is a fresh account, you'll be prompted with a clear initialization banner. Click **Enable** to activate the service within your current region.

Enabling IAM Identity Center creates a dedicated, isolated identity provider instance tied directly to your organization. This acts as a centralized authentication hub, allowing you to manage workforce identities, groups, and cross-account access configurations from a single control plane without needing to build local IAM users in every separate AWS account.

### Root Identity Hardening with Multi-Factor Authentication
With the identity infrastructure initialized, the absolute highest priority is to implement strong authentication controls on the root account before configuring any downstream resources. Multi-factor authentication adds a critical layer of defense, ensuring that even if the root password is leaked, harvested through phishing, or exposed via credential stuffing, an attacker still can't gain access without physical possession of the second factor.

AWS provides three primary authentication factors for hardening an account:
* **Passkeys or Security Keys:** Hardware devices or platform authenticators that utilize FIDO/WebAuthn standards (such as a YubiKey or biometric laptop sensors). These offer the strongest resistance against modern, proxy-based phishing attacks.
* **Authenticator Apps:** Soft-token applications (like Google Authenticator or Microsoft Authenticator) that generate a time-based one-time password (TOTP). This is a highly accessible and standard operational choice for lab and mid-tier enterprise deployments.
* **Hardware TOTP Tokens:** Physical key fobs that rotate alphanumeric codes on a time delay, completely isolated from network-connected operating systems.

To complete the hardening sequence, select your preferred authentication factor from the menu, follow the on-screen prompt to scan the generated QR code or link the hardware key, and input the sequential verification codes to bind the device to the root account. Once confirmed, the root credentials should be treated as a break-glass asset, used only for critical subscription or organization-level modifications, with daily work transitioning entirely to the identity portal.

---

## AWS Budgets: Proactive Financial Defenses
With the foundational identity security and root hardening protocols established, the next immediate phase is implementing fiscal security controls. In a cloud environment, financial visibility is an extension of security operations. Because AWS infrastructure scales dynamically on demand, an unmonitored credential compromise or an engineering misconfiguration can trigger a massive cascade of automated resource provisioning. Without explicit boundaries, a tenant can accumulate thousands of dollars in unauthorized charges within hours.

By engineering a proactive billing guardrail, you establish an early-warning system. If an attacker breaches a downstream environment and immediately begins spinning up high-performance compute instances for cryptocurrency mining or distributed attacks, the budget alert acts as a canary in the coal mine, notifying security personnel long before the standard monthly billing cycle closes.

### Configuring the Monthly Cost Guardrail
To deploy a financial boundary, return to the global search bar at the top of the management console and search for **Budgets** to access the AWS Budgets dashboard.
The console provides several customizable budget templates engineered for different structural requirements:
* **Zero Spend Budget:** An ultra-tight perimeter that triggers an automated alert the moment your account registers any accrued cost above $0.01. This is highly effective for free-tier tracking or completely dormant accounts.
* **Monthly Cost Budget:** A baseline financial tracker that monitors your aggregate monthly expenditures and evaluates them against a static threshold. This template tracks both actual accrued costs and forecasted spending trends.
* **Daily Savings Plans Coverage Budget:** An operational metric tracker that alerts engineering teams if their actual usage drops below a designated Savings Plan coverage target, preventing under-utilization of pre-purchased capacity.
* **Daily Reservation Utilization Budget:** A monitoring tool that triggers an alert when the utilization of Reserved Instances (such as EC2 or RDS commitments) drops below a specific percentage threshold, ensuring cost efficiency.

For an active lab or testing environment designed to simulate a development sandbox, selecting the **Monthly Cost Budget** template offers the best balance of tracking flexibility and rigid constraint.

To complete the setup process:
1. Select the **Monthly Cost Budget** template from the configuration wizard.
2. Define a meaningful, recognizable title for the budget naming convention to ensure clear context in alerting logs.
3. Input your maximum monthly target threshold. For a standard security lab environment, setting a hard cap of **$10.00** ensures that you're notified well before any minor experimentation incurs significant personal or corporate costs.
4. Input your primary operational email address within the notification recipients block. This ensures that the system can route automated alerts directly to your inbox the moment the AWS billing engine forecasts that your active resource deployment will breach your $10.00 threshold.
5. Confirm the structural details and finalize the creation wizard.

Once saved, the AWS billing engine continuously parses your active resource consumption against this metric, giving you an automated financial tripwire that protects the perimeter from run-away infrastructure costs.

---

## IAM Identity Center: Structural Role Separation
With root identity protections and systemic financial monitoring active, the next phase focuses on architectural engineering inside IAM Identity Center. Managing access controls for individual identities in an enterprise rapidly becomes an operational and auditing nightmare. To achieve scalable administrative control, security operations relies on **Role-Based Access Control (RBAC)**.

Instead of attaching granular security policies directly to individual user accounts, permissions are bound directly to specialized operational groups. When an employee onboard or transitions roles, they're simply moved into or out of the corresponding group. This drastically reduces administrative overhead and minimizes the risk of privilege creep, where a user accumulates excessive, unneeded access privileges over time.

### Engineering Isolated Operational Groups
To build out a resilient structural framework that simulates a cross-functional corporate technical team, navigate to the left-hand navigation pane within the IAM Identity Center console and select **Groups**.
1. Click the **Create group** button to open the configuration dashboard.
2. Define explicit group names and detailed operational descriptions that cleanly separate duties across three distinct organizational lanes:
* **`NetOps_Admin`:** Dedicated to network infrastructure engineers. This team manages core connectivity architecture, virtual private clouds (VPCs), transit gateways, routing tables, and perimeter firewall structures.
  * Edit: Just a quick heads up for future lab steps. If you're using the NetOps_Admin role like I am, you'll actually need to go into the root account under IAM Identity Center and add the `AmazonEC2FullAccess` policy directly to that permission set. By default, it only has network rights, which will cause a validation error when you try to launch the instance later on. Adding that policy fixes it right up.
* **`SysOps_Admin`:** Dedicated to systems engineers responsible for core workload operations, handling compute orchestration, provisioning, systems lifecycle management, and daily configuration tasks.
* **`SecOps_Audit`:** Dedicated to the security operations and compliance team. This identity boundary is designed strictly for continuous observation, compliance checking, security log monitoring, and threat visibility without infrastructure modification privileges.

Finalize the creation wizard for each separate group to lock this directory structure into your identity tenant.

### Selecting and Deploying Predefined Permission Sets
With the directory groups established, you must build the underlying permission frameworks that define exactly what actions these groups can execute inside the infrastructure. Within IAM Identity Center, these templates are called **Permission Sets**. They function as centralized IAM role definitions that AWS automates and scales across your organizational accounts.

Navigate to **Permission sets** in the left-hand navigation pane and click **Create permission set**. The console offers a choice between building highly specialized custom granular policies or deploying **Predefined permission sets**. Predefined sets are built and fully maintained by AWS, mapped directly to standard corporate job functions to ensure stability, accuracy, and broad compatibility with core cloud services.

The AWS managed predefined library provides an extensive portfolio of functional templates, including:
* **`AdministratorAccess`:** Full, unrestricted privileges to every service and resource within the tenant boundary.
* **`AWSManagementConsoleAdministratorAccess`:** Complete administrative control over console customizations, resource discovery, and user-facing notifications.
* **`Billing`:** Isolated access to financial statements, payment settings, invoicing, and budget tracking tools.
* **`DatabaseAdministrator`:** Granular control over structured data stores, caching layers, and database clusters.
* **`DataScientist`:** Specialized permissions for machine learning modeling, big-data processing, and analytical tooling.
* **`NetworkAdministrator`:** Full operational access to configure, scale, and secure networking infrastructure and connectivity maps.
* **`PowerUserAccess`:** Full administrative control over all application services and development tools, with an explicit denial to modify identity policies or user directories.
* **`ReadOnlyAccess`:** High-level operational view across resources, allowing engineers to audit configurations without seeing sensitive data or modifying assets.
* **`SecurityAudit`:** Vetted read-only access explicitly mapped to security tool configurations, access evaluation, and audit log inspection.
* **`SupportUser`:** Permissions built specifically for interacting with AWS Support representatives and checking basic diagnostic statistics.
* **`SystemAdministrator`:** Broad administrative control over compute systems, virtualization layers, storage buckets, and general backend configurations.
* **`ViewOnlyAccess`:** The tightest observational baseline, allowing visibility into resource presence without deep metadata mapping or configuration details.

To provision the required permission boundaries for your multi-role directory, step through the creation wizard three separate times to generate your templates using these exact mappings:
1. Select the predefined **`NetworkAdministrator`** template, configure a standard **1-hour or 12-hour session duration** based on your operational window, and name the set to align cleanly with your network engineering lane.
2. Repeat the process using the **`SystemAdministrator`** template to establish the foundational ruleset for the systems operations team.
3. Establish the security audit line by selecting the **`SecurityAudit`** template, ensuring the configuration matches your compliance requirements.

### Binding Groups to the AWS Account Perimeter
Having the directory groups and the permission sets sitting separate in the console does not actually grant access. The final critical link is explicitly binding the groups to your target accounts through the permission sets. This establishes the structural routing map that determines who can access what assets.
1. Navigate to **AWS accounts** in the left multi-account permissions tree.
2. Select the checkbox next to your primary root organizational management node, listed as your lab title (**`HK_Lab`**).
3. Click **Assign users or groups** to initialize the deployment wizard.
4. Flip to the **Groups** tab at the top of the interface and select your target group (for example, `NetOps_Admin`). Click **Next**.
5. Select the corresponding permission set template (`NetworkAdministrator`) that matches the group's job function. Click **Next**.
6. Review the structural assignment parameters and submit.

AWS IAM Identity Center instantly takes over, processing an automated background routine that securely provisions a corresponding, managed local IAM role directly into the target AWS account. Repeat this sequence for both the systems and security audit groups.

Once complete, your organizational architecture is perfectly aligned: your directory groups are systematically mapped directly to their corresponding AWS managed privilege baselines, setting up an environment that is completely isolated and ready for human operator onboarding.

---

## User Directory Population and Role Isolation
With the structural group framework and permission set mappings securely bound to the AWS account perimeter, the final phase of building out this centralized identity ecosystem is seeding the directory with operational users.

Populating the environment with distinct human identities allows you to transition your primary infrastructure duties away from the hazardous root account and into a controlled human operator workflow. Furthermore, by deliberately provisioning separate placeholder accounts for the other technical departments, you establish a real-world testing baseline. This gives you a clear way to validate that your security walls are holding and that role separation is active across all organizational units.

### Creating the Primary Human Identity
To begin onboarding operators into the enterprise tenant, select **Users** from the left-hand navigation pane within the IAM Identity Center console and click **Add user**.

The user provisioning wizard breaks down the account configuration into distinct identity details:
1. **Username:** Establish a clear, distinct naming convention that separates a person's core human identity from standard service accounts. To distinguish this primary daily account from the ultimate root user, assign the username **`<name>-operator`**.
2. **Identity Attributes:** Input the relevant administrative contact information, including the primary engineer's first name, last name, and display name.
3. **Authentication Delivery:** Define how the initial password onboarding sequence will be handled. The console provides two primary mechanisms for this:
* **Email Invitation:** AWS automatically routes a secure, time-sensitive registration link directly to the user's email inbox. The engineer clicks the link to accept the organization's invitation and set their permanent credential. This mirrors true corporate onboarding hygiene and is the method used for the primary `<name>-operator` identity.
* **One-Time Password Generation:** The console instantly generates a temporary, single-use credential that an administrator can copy and securely share with the end-user. This is highly efficient for rapid provisioning or sandboxed laboratory additions where configuring separate email delivery loops is unnecessary.

### Group Assignment and Subaddressing Deployment
Once the baseline identity attributes are configured, click **Next** to advance to the group membership assignment plane. To bind `<name>-operator` to the designated network engineering lane, select the checkbox for the **`NetOps_Admin`** group. Finalize the wizard to initiate the background provisioning routine.

To complete the corporate simulation and build out a robust environment for cross-functional auditing, step through the wizard two more times to provision your secondary placeholder identities:
* **The Security Auditor:** Create username **`alice-audit`** and assign her membership strictly to the **`SecOps_Audit`** group.
* **The Systems Engineer:** Create username **`sam-sysops`** and assign him membership strictly to the **`SysOps_Admin`** group.

> ### Directory Scaling Trick: Email Plus-Address Subaddressing
> 
> AWS IAM Identity Center requires every single user in the directory to possess a completely unique email address. To avoid the operational friction of registering and managing multiple auxiliary email inboxes just to run a lab environment, utilize email plus-address subaddressing.
> Most modern email providers allow you to append a `+` symbol and any string of characters immediately before the `@` domain identifier (for example, `yourprimaryemail+alice@gmail.com`). The AWS identity provider reads this as a completely separate, unique account string, while your underlying email provider safely routes all automated system messages right back into your single primary inbox.

For these secondary accounts, configure their initial credentials using the **One-Time Password Generation** option to bypass the email invitation loop, instantly creating their entries directly inside the directory control plane.

---

## Technical Validation and Deployment Closure
With all three human operators safely seated in the directory list, Phase 1 of the enterprise identity project is officially complete. You can validate the successful deployment of your architecture by verifying the active user session dynamics directly within the management console.

### The Secure Access Portal Portal and First Human Login
To transition out of the root user environment entirely, navigate back to the main IAM Identity Center **Dashboard** and locate the **Settings summary** block on the right side of the screen. Inside this metadata block sits your unique, organization-specific **AWS access portal URL**. This link serves as the single, hardened front door for your entire workforce.

To verify the permission isolation under the hood:
1. Copy the custom access portal URL and launch it inside a fresh **Incognito or Private Browsing window** to ensure your active browser session does not conflict with your existing root cookies.
2. Accept the initial email invitation for `<name>-operator`, establish a complex permanent password, and log into the portal.
3. Upon entering the minimalist workforce dashboard, expand the active account dropdown folder labeled **`HK_Lab`**.

Because you engineered strict role isolation, the `<name>-operator` identity is completely barred from seeing the global systems environment or security configuration panes. The portal dynamically evaluates your group memberships and displays exactly **one** operational avenue: the **`NetworkAdministrator`** console path.

### Enforcing the Perimeter Boundary
Clicking the **Management Console** link next to the `NetworkAdministrator` assignment launches you directly into the active AWS production console as a human operator. Looking up at the identity badge in the top right corner of the user interface confirms your active session status, displaying a secure role-assumed string indicating you are successfully acting through your restricted identity.

The ultimate proof of your architecture is clearly visible right on the console home page. Because a network administrator has no intrinsic privileges to monitor financial overhead or audit system compute states, multiple default console widgets will immediately display hard **Access Denied** warnings.

Your perimeter walls are up, your root account is securely locked down behind physical multi-factor authentication, your proactive financial guardrails are actively scanning for anomalies, and your human operators are strictly confined to their authorized operational lanes. You have successfully engineered a scalable, zero-trust foundation for all future cloud deployments.

---

## Continuous Telemetry Auditing: CloudWatch Telemetry Ingestion
A hardened cloud perimeter is only as effective as your visibility into it. In security operations, defense in depth demands robust, continuous log ingestion so that you have eyes on every API call, resource modification, and authentication event across the entire infrastructure.

AWS has shifted modern logging standards toward a highly streamlined approach using **CloudWatch Telemetry Ingestion**. Previously, tracking infrastructure modifications meant building a local trail in AWS CloudTrail and manually routing those logs into a storage bucket or stream. Now, AWS leverages service-linked channels to ingest CloudTrail event telemetry directly into Amazon CloudWatch Logs via global telemetry enablement rules. This approach completely bypasses the legacy per-trail configurations, allowing security teams to automatically audit and ingest activity across all regions and accounts from a single control plane.

### Configuring Direct Telemetry Enablement Rules
To establish an automated logging pipeline, use the global search bar at the top of the management console to navigate to the **CloudWatch** dashboard.
1. From the left-hand navigation pane, look under the log management sub-menus and select **Telemetry config**, then flip over to the **Enablement rules** tab.
2. Click the configuration action button to open the telemetry enablement wizard.
3. Locate **AWS CloudTrail** in the listed telemetry data sources and initiate the rule configuration.
4. **Specify Scope:** Assign a highly recognizable, descriptive title to your new enablement rule. Under the scope options, choose to target **All regions**. This is an essential security baseline; tracking activity in unused regions ensures that if an adversary attempts to evade detection by spinning up illicit workloads in a distant geographic zone, their actions are immediately captured.
5. **Select Data Options:** Choose **Management events** to track control plane operations, such as user logins, policy modifications, and service creations.

### Architectural Logging Storage: Managed vs. Unmanaged Groups
When defining where your telemetry data lands, the system requires you to choose a destination structure. This is where you determine how tightly controlled your audit trail is:
* **Managed Log Groups (Recommended):** This selection provisions a strictly isolated, AWS-hardened logging repository. Managed log groups automatically enforce built-in deletion protection, strict cryptographic source validation, and automated sub-grouping or naming conventions based on the event type (such as `aws/cloudtrail/[event-type]`). Most importantly, they enforce ingestion restrictions ensuring that only the authorized AWS service channel can write data to them, preventing attackers from tampering with the logs.
* **Unmanaged Log Groups:** This legacy style gives the administrator full control over group naming patterns but lacks built-in source validation or automated deletion safety nets. There are no native ingestion restrictions, meaning any service or identity with broad log permissions can write to or alter the stream.

For a secure, audit-ready architecture, select **Managed Log Groups**.

### Retention Lifecycle Engineering
Activity logs are invaluable for digital forensics and incident response (DFIR), but storing endless volumes of high-frequency telemetry indefinitely will quickly balloon storage costs. To practice proper data lifestyle management without sacrificing immediate investigative capabilities, adjust the retention setting.

While production environments may require months or years of archival storage for compliance, setting a hard retention threshold of **1 month** for a standard security lab strikes the perfect balance. This window gives you more than enough time to analyze historical logs, build monitoring metrics, and run queries without letting data bloat accrue unnecessary costs on your monthly balance.

Confirm your ruleset configuration and submit. Once active, CloudWatch immediately deploys your service-linked roles and begins capturing a definitive, immutable record of every administrative action taken across your cloud perimeter.

---

## User Directory Population & Multi-Role Validation
With the foundational tenant defenses, continuous logging ingestion, and group policies securely provisioned, the final phase of building out this centralized cloud identity architecture is populating the directory with operational users and conducting live access audits.

Managing permissions at scale relies on the strict enforcement of the Principle of Least Privilege (PoLP). To prove that our structural walls work in practice, we need to transition away from the unrestricted root user account entirely and onboard distinct human operator identities. By leveraging separate profiles for different corporate roles, we can systematically verify that users are successfully confined to their authorized operational lanes.

### Seeding the Workforce Directory
To begin onboarding operators into the enterprise tenant, select **Users** from the left-hand navigation pane within the IAM Identity Center console and click **Add user**.

The user provisioning wizard guides you through setting up distinct identity attributes. For the primary administrator account, establish the username **`<name>-operator`**. To configure authentication delivery, choose the **Email Invitation** option. This mirrors true corporate onboarding hygiene, prompting AWS to route a secure, time-sensitive registration link directly to the operator's primary inbox to set a permanent, complex credential.

To complete the corporate simulation and build out a robust environment for cross-department auditing, step through the wizard to provision secondary placeholder identities:
* **The Security Auditor:** Create username **`alice-audit`** and assign her membership strictly to the **`SecOps_Audit`** group.
* **The Systems Engineer:** Create username **`sam-sysops`** and assign him membership strictly to the **`SysOps_Admin`** group.

> ### Directory Scaling Trick: Email Plus-Address Subaddressing
> 
> AWS IAM Identity Center requires every user account in the directory to possess a completely unique email address. To avoid the operational friction of registering and managing multiple auxiliary email inboxes just to run a lab environment, utilize email plus-address subaddressing.
> Most modern email providers allow you to append a `+` symbol and any string of characters immediately before the `@` domain identifier (for example, `yourprimaryemail+alice@gmail.com`). The AWS identity provider reads this as a completely separate, unique account string, while your underlying email provider safely routes all automated system messages right back into your single primary inbox.

For these secondary accounts, configure their initial credentials using the **One-Time Password Generation** option to bypass the email invitation loop, instantly creating their entries directly inside the directory control plane and assigning them to their respective groups.

### Initializing the Human Operator Session
To validate the permission isolation under the hood, locate the organization-specific **AWS access portal URL** within the Identity Center dashboard metadata block. This link serves as the single, hardened front door for your entire workforce.
1. Copy the custom access portal URL and launch it inside a fresh **Incognito or Private Browsing window** to ensure your active browser session does not conflict with your existing root cookies.
2. Log into the portal using the newly configured `<name>-operator` credentials.
3. Upon entering the workforce dashboard, expand the active account dropdown folder labeled **`HK_Lab`**.

Because you engineered strict role isolation, the identity provider dynamically evaluates your group memberships and displays exactly **one** operational avenue: the **`NetworkAdministrator`** console path. You have zero visibility into security logs or general compute instances.

### Enforcing the Perimeter Boundary
Clicking the **Management Console** link next to the `NetworkAdministrator` assignment launches you directly into the active AWS production console as a restricted human operator. Looking up at the identity badge in the top right corner of the user interface confirms your active session status, displaying a secure role-assumed string indicating you are successfully acting through your restricted identity.

The ultimate proof of your architecture is clearly visible right on the console home page. Because a network administrator has no intrinsic privileges to monitor financial overhead or audit system compute states, multiple default console widgets will immediately display hard **Access Denied** warnings. Your perimeter walls are up, your root account is securely locked down behind physical multi-factor authentication, and your human operators are strictly confined to their authorized operational lanes.

---

## Final Notes
This project attempts to demonstrate the transition of a vulnerable, greenfield AWS tenant into a hardened, enterprise-grade cloud environment. By executing this deployment, we've achieved several critical security milestones:
* **Eliminated Root Exposure:** The unrestricted master root account has been reinforced with multi-factor authentication and completely isolated from daily administrative tasks.
* **Established Role-Based Access Control:** Individual user identities are managed centrally through group bindings, eliminating the risks associated with long-lived access keys or individual privilege creep.
* **Deployed Continuous Audit Visibility:** With CloudWatch Telemetry Ingestion active across all regions, the tenant maintains an immutable record of control plane operations, laying the groundwork for future digital forensics and incident response (DFIR) labs.
* **Enforced Proactive Cost Boundaries:** The implementation of a monthly budget guardrail ensures that any unauthorized resource spikes or configuration drifts are caught and countered before causing fiscal damage.

This baseline architecture establishes a secure, zero-trust foundation. With the infrastructure perimeter thoroughly stabilized, future labs within this repository will focus on deploying active workloads, configuring advanced logging metrics, and simulating purple-team incident response exercises within our isolated operational lanes.
