## Deployments

This section defines controls related to metadata deployment practices, configuration change governance, and production environment integrity. These controls ensure that organizations establish clear provenance for production changes, restrict high-risk metadata modifications to controlled deployment processes, and maintain continuous monitoring to detect unauthorized configuration drift.

### SBS-DEP-001: Require a Designated Deployment Identity for Metadata Changes

**Control Statement:** Salesforce production orgs must designate a single deployment identity that is exclusively used for all metadata deployments and high-risk configuration changes performed through automated or scripted release processes.

**Description:**  
A dedicated deployment identity (integration user) must be created and used as the sole account for CI/CD, metadata deployments, and automated release tooling. No human user—regardless of administrative privilege—may deploy metadata or execute automated deployment operations using their personal account.

**Risk:** <Badge type="warning" text="High" />  
Without a designated deployment identity, organizations cannot reliably attribute production changes—any administrator can deploy metadata, making it impossible to distinguish authorized CI/CD deployments from unauthorized manual changes. This loss of provenance prevents security teams from detecting unauthorized modifications, investigating configuration drift, or determining whether a change was part of an approved release. Attackers or malicious insiders can make direct production changes that blend into legitimate administrative activity, and incident responders cannot reconstruct the timeline of configuration changes during a breach investigation.

**Audit Procedure:**  
1. Identify the user account designated as the deployment identity.  
2. Enumerate all recent metadata deployments using tooling such as Deployment Status, Metadata API logs, or audit logs.  
3. Verify that all deployments were executed by the designated deployment identity.  
4. Flag any metadata deployment performed by a human user or non-deployment identity.

**Remediation:**  
1. Create or identify a dedicated deployment identity.  
2. Reconfigure CI/CD pipelines, release management tooling, and automated deployment scripts to authenticate exclusively with the deployment identity.  
3. Revoke deployment permissions from all human users.  
4. Re-deploy any metadata last deployed by a human user to restore provenance.

**Default Value:**  
Salesforce does not create or enforce a dedicated deployment identity by default.

### SBS-DEP-002: Establish and Maintain a List of High-Risk Metadata Types Prohibited from Direct Production Editing

**Control Statement:** Salesforce production orgs must maintain an explicit list of high-risk metadata types that must never be edited directly in production by human users, defaulting at minimum to the SBS baseline list while allowing organizations to extend or refine it as needed.

**Description:**  
Organizations must adopt the SBS baseline list of prohibited direct-in-production changes—which includes Apex Classes, Apex Triggers, LWCs, Aura Components, Profiles, Permission Set definitions, Remote Site Settings, Named Credentials, and core authentication or session security settings—and maintain this list as an internal policy. Organizations may extend this list or define exceptions, but the minimum baseline must be included and documented.

**Risk:** <Badge type="warning" text="High" />  
Without an explicit list of high-risk metadata types, organizations cannot define or enforce deployment governance boundaries—leaving critical configuration categories (Apex code, authentication settings, outbound connectivity, permissions) open to uncontrolled direct production editing. Security teams cannot distinguish between metadata that requires strict deployment controls and metadata that can be safely edited manually, resulting in inconsistent governance and gaps in change attribution. The absence of a defined list also prevents effective monitoring (SBS-DEP-003), as there is no baseline to compare against when detecting unauthorized changes.

**Audit Procedure:**  
1. Obtain the organization's documented list of high-risk metadata types prohibited from direct production editing.  
2. Confirm that the list, at minimum, includes all SBS baseline categories.  
3. Review the list for any documented exceptions and verify they are formally approved.  
4. Verify that only the deployment identity has modify permissions for metadata types on the list.

**Remediation:**  
1. Adopt the SBS baseline list of prohibited direct-in-production metadata changes.  
2. Add any organization-specific items or exceptions as needed.  
3. Remove modify permissions for these metadata types from all human users.  
4. Ensure all future changes to listed metadata types are performed exclusively by the deployment identity.

**Default Value:**  
Salesforce does not provide native restrictions or guidance preventing direct production edits to high-risk metadata.


### SBS-DEP-003: Monitor and Alert on Unauthorized Modifications to High-Risk Metadata

**Control Statement:** Salesforce production orgs must implement a monitoring capability that detects and reports any modification to high-risk metadata performed by a user other than the designated deployment identity.

**Description:**  
Organizations must maintain a monitoring process—manual or automated—that reviews administrative and metadata changes and identifies when high-risk metadata (as defined in SBS-DEP-002 or extended by the organization) is modified by a human user instead of the designated deployment identity. The monitoring method may use Salesforce APIs, audit logs, export files, CLI tooling, vendor tools, or any combination, provided it reliably detects unauthorized changes within the organization’s defined review interval.

**Risk:** <Badge type="warning" text="High" />  
Without monitoring for unauthorized metadata changes, organizations cannot detect when high-risk configuration is modified outside the approved deployment process—allowing malicious changes, accidental drift, or insider threats to persist undetected. Security teams lose the ability to identify unauthorized modifications to authentication settings, permission structures, Apex code, or outbound connectivity until a breach or incident reveals the gap. This impairs detection, investigation, and response capabilities for configuration-related security events, extending attacker dwell time and preventing timely remediation of unauthorized changes.

**Audit Procedure:**  
1. Interview system owners to identify the monitoring method(s) used for detecting changes to high-risk metadata.  
2. Review documentation describing how the monitoring process works—whether manual log review, automated scripts, API queries, CLI workflows, scheduled exports, or vendor tools.  
3. Verify that the monitoring process includes:  
   - Coverage of all high-risk metadata types defined by the organization and required by SBS-DEP-002.  
   - A review interval appropriate to the organization's change-management expectations (e.g., daily, weekly, or aligned with release cycles).  
   - A method for identifying the user who performed each change.  
4. Examine historical monitoring records or logs to confirm the process has been performed consistently.  
5. Flag noncompliance if no monitoring system exists or if the system cannot detect unauthorized human modifications to high-risk metadata.

**Remediation:**  
1. Implement a monitoring mechanism capable of identifying modifications to high-risk metadata and attributing them to the responsible user. Acceptable approaches include:  
   - Manual periodic review of the Salesforce Setup Audit Trail,  
   - Exporting audit logs for review,  
   - Scheduled API or CLI queries comparing metadata changes,  
   - Custom scripts,  
   - Vendor-based monitoring tools.  
2. Ensure the monitoring method covers all high-risk metadata types listed in the organization’s defined prohibited-direct-edit list.  
3. Define a repeatable review interval and assign responsibility for conducting the review.  
4. Document the monitoring approach and maintain records of reviews and findings.

**Default Value:**  
Salesforce does not provide built-in monitoring or alerting for unauthorized direct-in-production metadata changes; organizations must implement their own processes.

### SBS-DEP-004 — Establish Source-Driven Development Process

**Control Statement**  
Meaningful Salesforce metadata changes must be deployed through a source-driven, automated, and deterministic deployment process, except where the platform does not provide programmatic deployment support.

**Description:**  
Organizations must track all meaningful metadata changes in a centralized version control system and deploy them using an automated, repeatable, and deterministic process; manual changes in production are permitted only for metadata types that Salesforce does not expose for programmatic deployment.

**Risk:** <Badge type="warning" text="High" />  
Without a source-driven deployment process, organizations lose the verifiable audit trail that connects production configuration to approved changes—making it impossible to determine what changed, when, by whom, and whether it was authorized. Security teams cannot investigate configuration-related incidents, restore known-good state during outages, or attribute changes during forensic analysis. Manual production changes bypass code review, testing, and approval workflows, enabling unauthorized, accidental, or malicious modifications to security-sensitive settings without accountability or detection.

**Audit Procedure:**  
1. Identify the organization’s standard deployment process and designated deployment identity as defined in SBS-CHG-001.  
2. Review recent production metadata changes and their associated deployment records.  
3. Verify that changes deployable through Salesforce’s programmatic deployment mechanisms originated from centralized version control.  
4. Confirm that any manual production changes are limited to metadata types that Salesforce does not support for programmatic deployment.  
5. Flag any manually applied changes that could have been deployed through the source-driven process.

**Remediation:**  
1. Establish and maintain a centralized version control repository for Salesforce metadata.  
2. Implement or enforce an automated deployment pipeline that deploys changes exclusively from version control.  
3. Restrict direct production changes for metadata types that support programmatic deployment.  
4. Document and periodically review any required manual production changes for metadata types lacking deployment support.

**Default Value:**  
Salesforce allows direct manual changes to most metadata in production and does not require source control or automated deployments by default.

### SBS-DEP-005: Implement Secret Scanning for Salesforce Source Repositories

**Control Statement:** Organizations using source-driven development for Salesforce must implement automated secret scanning on all repositories containing Salesforce metadata, configuration, or deployment scripts to detect and prevent the exposure of credentials, access tokens, and other sensitive authentication material.

**Description:**  
CI/CD pipelines for Salesforce deployments typically require long-lived access tokens, refresh tokens, or other credentials to authenticate as the designated deployment identity. If these credentials are hardcoded in scripts, configuration files, or committed to version control, they become accessible to anyone with repository access—including contractors, consultants, former employees, or attackers who compromise the source control system. Organizations must implement automated secret scanning that runs on every commit and pull request to detect Salesforce-specific secrets (such as access tokens, refresh tokens, consumer secrets, and session IDs) as well as general credential patterns.

**Risk:** <Badge type="danger" text="Critical" />  
Exposed Salesforce credentials in source repositories represent a direct path to unauthorized production access—a supply chain attack vector that bypasses all other access controls. Contractors, consultants, or any party with repository access can extract hardcoded tokens and authenticate directly to production orgs with the full permissions of the deployment identity. Attackers who compromise source control systems or CI/CD infrastructure gain immediate access to production Salesforce environments. Unlike other credential exposures, Salesforce access tokens often have broad administrative permissions and long validity periods, making them high-value targets. Organizations cannot detect this exposure through Salesforce audit logs alone—the attacker authenticates with valid credentials, and their activity appears legitimate.

**Audit Procedure:**  
1. Identify all repositories containing Salesforce metadata, SFDX projects, deployment scripts, or CI/CD pipeline configurations.  
2. Verify that automated secret scanning is enabled on each repository—either through the source control platform's native capabilities (e.g., GitHub Secret Scanning, GitLab Secret Detection) or through third-party tooling.  
3. Confirm that the scanning configuration includes patterns for Salesforce-specific secrets (access tokens, refresh tokens, consumer keys/secrets, session IDs).  
4. Review scanning logs or dashboards to verify the tool is actively running and producing results.  
5. Verify that detected secrets trigger alerts and block merges or deployments until remediated.  
6. Flag noncompliance if any Salesforce-related repository lacks active secret scanning coverage.

**Remediation:**  
1. Enable secret scanning on all repositories containing Salesforce code, metadata, or deployment configurations using platform-native tools or third-party secret scanning solutions.  
2. Configure scanning rules to detect Salesforce-specific credential patterns in addition to general secrets.  
3. Implement pre-commit hooks or CI checks that block commits containing detected secrets.  
4. Immediately rotate any Salesforce access tokens, refresh tokens, or credentials that have been committed to version control—even if subsequently removed, as they persist in git history.  
5. Migrate credential storage to secure secrets management solutions (e.g., CI/CD platform secrets, vault systems) and remove all hardcoded credentials from repositories.  
6. Establish a periodic rotation schedule for Salesforce deployment credentials to limit the window of exposure if a secret is leaked.

**Default Value:**  
Salesforce does not provide secret scanning capabilities; organizations must implement scanning through their source control platform or third-party tooling. Credentials can be freely committed to repositories without any native detection or prevention.

### SBS-DEP-006: Configure Salesforce CLI Connected App with Token Expiration Policies

**Control Statement:** Organizations must configure the Connected App used for Salesforce CLI authentication with refresh token expiration of 90 days or less and access token timeout of 15 minutes or less.

**Description:**  
Salesforce CLI stores OAuth tokens locally on developer workstations to enable command-line access to orgs. The default "Salesforce CLI" Connected App ships with refresh tokens and access tokens set to never expire, meaning stolen or leaked token files provide indefinite access to authorized orgs. Organizations must either create a dedicated Connected App for CLI usage or install and configure the default Connected App with appropriate token expiration policies—refresh tokens must expire within 90 days and access tokens must timeout within 15 minutes. When using a dedicated Connected App, organizations should use the `--client-id` flag with CLI authentication commands.

**Risk:** <Badge type="warning" text="High" />  
Salesforce CLI token files stored on local workstations represent a persistent credential exposure risk. If a laptop is stolen, reassigned without proper cleanup, or compromised by malware, attackers can extract token files that provide direct access to Salesforce orgs—including production environments. With the default Connected App configuration, these tokens never expire, giving attackers indefinite access that persists even after the original user's password is changed or their account is deactivated. The attack surface expands with each org a developer authenticates to, as token files accumulate credentials to sandboxes, Dev Hubs, and production orgs. Organizations cannot detect this credential theft through Salesforce audit logs because the attacker authenticates with valid tokens.

**Audit Procedure:**  
1. From Setup, navigate to Connected Apps OAuth Usage (or Apps → Connected Apps → Connected Apps OAuth Usage).  
2. Identify the Connected App(s) used for Salesforce CLI authentication—either the default "Salesforce CLI" app or a custom Connected App.  
3. Review the OAuth Policies for each CLI-related Connected App:  
   - Verify that Refresh Token Policy is set to "Expire refresh token after" with a value of 90 days or less.  
   - Verify that Session Policies Timeout Value is set to 15 minutes or less.  
4. If a custom Connected App is used, verify that developers are instructed to use the `--client-id` flag when authenticating.  
5. Flag noncompliance if any CLI-related Connected App has tokens set to never expire or exceeds the maximum allowed durations.

**Remediation:**  
1. Determine whether to use the default "Salesforce CLI" Connected App or create a dedicated Connected App for CLI authentication.  
2. If using the default app:  
   - From Setup, navigate to Connected Apps OAuth Usage.  
   - Locate "Salesforce CLI" and click Install (if not already installed), then Edit Policies.  
   - Set Refresh Token Policy to "Expire refresh token after: 90 Days" (or less).  
   - Set Session Policies Timeout Value to "15 minutes" (or less).  
3. If creating a dedicated Connected App:  
   - Create a new Connected App with OAuth enabled and appropriate callback URL.  
   - Configure refresh token expiry to 90 days or less and access token timeout to 15 minutes or less.  
   - Distribute the Consumer Key to developers and require use of `--client-id` flag.  
4. Communicate to developers that they will need to re-authenticate periodically when refresh tokens expire.  
5. Consider implementing compensating controls to protect locally stored token files, such as:  
   - Requiring full disk encryption (FileVault, BitLocker) on developer workstations.  
   - Enabling remote wipe capability for managed devices.  
   - Including Salesforce CLI token file cleanup in device offboarding procedures.  
   - Training developers to run `sf org logout --all` before returning or transferring devices.

**Default Value:**  
The default "Salesforce CLI" Connected App is configured with refresh tokens and access tokens that never expire. Organizations must explicitly install and configure the app or create a dedicated Connected App to enforce token expiration policies.