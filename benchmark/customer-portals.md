## Customer Portals

This section defines controls related to secure configuration and development practices for Salesforce customer portals, including Experience Cloud sites, Communities, and other external-facing Salesforce platforms. These controls ensure that organizations implement proper access controls, data isolation, and secure coding practices when exposing Salesforce functionality to external users.

### SBS-CPORTAL-001: Prevent Insecure Direct Object Reference (IDOR) in Portal Apex

**Control Statement:**  
All Apex methods exposed to Experience Cloud or customer portal users must enforce server-side authorization for every record accessed or modified. User-supplied parameters (including record IDs, filters, field names, or relationship references) must not be trusted as the basis for access control and must be validated against the running user's sharing, CRUD, and FLS permissions before use.

**Description:**  
Portal-exposed Apex methods are callable by external users and therefore must not rely on client-provided identifiers or query inputs to determine record access. Accepting record IDs, filter criteria, field lists, or relationship paths without validating the running user's access creates Insecure Direct Object Reference (IDOR) vulnerabilities.

Methods must:

* Run `with sharing` unless explicitly justified.
* Enforce object- and field-level permissions.
* Prevent user-controlled SOQL structure (e.g., dynamic WHERE clauses or field lists).
* Restrict data scope to records the authenticated user is authorized to access according to business rules.

Parameters may be accepted when required for legitimate functionality, but must be validated server-side before querying or performing DML.

**Risk:** <Badge type="danger" text="Critical" />  
If portal-exposed Apex trusts user-controlled parameters to determine record access, external users can manipulate inputs to retrieve or modify unauthorized records. This may enable record enumeration, data exfiltration, or data corruption. IDOR vulnerabilities represent a critical authorization boundary failure.

**Audit Procedure:**  
1. Identify all Apex classes exposing `@AuraEnabled`, `@InvocableMethod`, or `@RestResource` methods accessible to portal users.
2. Review method parameters of type `Id`, `String`, collections, or maps that could influence record access.
3. Verify that:

   * Classes run `with sharing` or implement equivalent authorization checks.
   * Record access is validated before query or DML.
   * CRUD and FLS are enforced.
   * Dynamic SOQL does not incorporate unsanitized user input.
4. Attempt to access unauthorized records by manipulating record IDs or filter inputs from a portal user session.
5. Flag any method that relies solely on user-supplied parameters to control record access as noncompliant.

**Remediation:**  
* Enforce `with sharing` on portal-facing classes by default.
* Scope queries using the running user's context where possible.
* If record IDs are accepted as parameters, verify access using sharing or `UserRecordAccess` before returning or modifying data.
* Remove user-controlled query structure and whitelist allowable filter inputs.
* Enforce CRUD and FLS on all returned or modified records.

**Default Value:**  
Salesforce does not validate user-supplied parameters in custom Apex. Developers are responsible for implementing server-side authorization controls in all portal-exposed methods.

### SBS-CPORTAL-002: Restrict Guest User Record Access

**Control Statement:** Unauthenticated guest users in customer portals must be restricted to authentication and registration flows only, with no direct access to business objects or custom Apex methods that query organizational data.

**Description:**  
Organizations must configure customer portal guest user profiles to prohibit access to all business-related standard and custom objects, limiting guest user capabilities exclusively to authentication flows (login, registration, password reset, self-service account creation). Guest users must not be granted object-level permissions, field-level access, or the ability to invoke custom Apex methods that return organizational data.

When business requirements necessitate limited guest user access to specific public data (such as knowledge articles, public case submission forms, or product catalogs), organizations must:
- Implement a dedicated service layer architecture where controllers invoke secure service classes that perform explicit access validation
- Use allowlist-based data access (explicitly define queryable records, never accept parameters)
- Apply additional validation using `UserInfo.getUserType() == 'Guest'` to enforce restricted logic paths
- Consider rate limiting and CAPTCHA protection to prevent enumeration attacks

**Risk:** <Badge type="danger" text="Critical" />  
Guest users represent the highest-risk trust boundary in Salesforce portals—they are unauthenticated, have zero accountability, generate minimal audit trail, and operate with potential adversarial intent. When guest users are granted object permissions or can invoke custom Apex methods, attackers can systematically enumerate organizational data without even creating an account. Historical Salesforce security updates have repeatedly addressed guest user permission defaults because vendors consistently misconfigure this boundary. A single guest-accessible method that queries user records, cases, accounts, or custom objects creates a public API for data exfiltration accessible to anyone on the internet. This constitutes a Critical boundary violation: unauthenticated attackers access organizational data with no authentication required.

**Audit Procedure:**  
1. Identify all guest user profiles used by customer portal sites (typically named "Site Guest User" or similar).  
2. Review object-level permissions for guest user profiles and verify that all business-related standard and custom objects have Read, Create, Edit, Delete permissions set to disabled.  
3. Enumerate all custom Apex classes containing `@AuraEnabled` methods and verify that none are accessible to guest users (either by checking profile permissions or testing invocation from guest context).  
4. For any guest-accessible functionality beyond authentication flows, verify implementation of service layer architecture with explicit access controls.  
5. Test by accessing the portal without authentication and attempting to invoke Apex methods or query objects via built-in Lightning controllers.  
6. Flag any guest user object permissions or method access as noncompliant.

**Remediation:**  
1. Remove all object-level permissions from guest user profiles except those explicitly required for authentication flows.  
2. Audit and remove guest user access to any custom Apex methods that query or return organizational data.  
3. For public data requirements (knowledge articles, case submission), implement service layer pattern:
   ```apex
   @AuraEnabled
   public static List<Knowledge__kav> getPublicArticles() {
       if (UserInfo.getUserType() == 'Guest') {
           // Allowlist-based, no parameters accepted
           return [SELECT Id, Title, Summary FROM Knowledge__kav 
                   WHERE PublicationStatus = 'Online' 
                   AND IsVisibleInPkb = true 
                   LIMIT 10];
       }
       throw new AuraHandledException('Access denied');
   }
   ```
4. Implement network-level rate limiting and CAPTCHA for guest-accessible endpoints.  
5. Review Salesforce security updates and apply guest user permission restrictions from recent releases.

**Default Value:**  
Salesforce has progressively restricted guest user default permissions in recent releases, but older orgs may retain permissive configurations. Guest user profiles do not prevent object access or Apex invocation by default—administrators must explicitly configure restrictions.


### SBS-CPORTAL-003: Inventory Portal-Exposed Apex Classes and Flows

**Control Statement:** Organizations must maintain an authoritative inventory of all Apex classes and Autolaunched Flows exposed to Experience Cloud sites, documenting which components are accessible to external and guest users.

**Description:**  
Organizations must document all Apex classes with `@AuraEnabled` methods and all Autolaunched Flows that can be invoked from Experience Cloud sites. The inventory must include which portal user profiles and permission sets can access each component.

**Risk:** <Badge type="warning" text="High" />  
Without a complete inventory of portal-exposed components, organizations cannot assess their external attack surface or enforce security reviews for externally accessible code. Security teams lose visibility into which business logic external users can invoke, preventing effective security testing, incident response, and access governance. This impairs the ability to detect unauthorized exposure of sensitive functionality or identify components requiring security hardening.

**Audit Procedure:**  
1. Request the organization's inventory of portal-exposed Apex classes and Flows from the designated system of record.
2. Query all Apex classes with `@AuraEnabled` methods accessible to portal user profiles.
3. Query all Autolaunched Flows invoked from Experience Cloud pages or components.
4. Verify each component appears in the inventory with documentation of which portal profiles can access it.
5. Flag any portal-exposed component missing from the inventory as noncompliant.

**Remediation:**  
1. Enumerate all Apex classes containing `@AuraEnabled` methods.
2. Enumerate all Autolaunched Flows embedded in Experience Cloud sites.
3. For each component, document which portal user profiles and permission sets have access.
4. Store the inventory in the designated system of record.
5. Establish a process to update the inventory when new components are exposed to portals.

**Default Value:**  
Salesforce does not require or maintain an inventory of portal-exposed components.

### SBS-CPORTAL-004: Prevent Parameter-Based Record Access in Portal-Exposed Flows

**Control Statement:** Autolaunched Flows exposed to customer portal users must not accept user-supplied input variables that directly determine which records are accessed.

**Description:**  
Flows invoked from Experience Cloud must not accept input variables for record IDs, object names, or filter criteria. All record access must be derived from the authenticated user's context using `$User.ContactId` or similar user context resources.

**Risk:** <Badge type="danger" text="Critical" />  
Flows accepting user-controlled input variables for record access create IDOR vulnerabilities allowing external users to access any record in the org. Because Autolaunched Flows run in system context without sharing by default, a single flow accepting a record ID input parameter bypasses all permissions and sharing rules. This constitutes a Critical boundary violation: unauthorized users access data they should never see, with no compensating controls required to fail.

**Audit Procedure:**  
1. Using the inventory from SBS-CPORTAL-003, identify all portal-exposed Autolaunched Flows.
2. For each flow, examine input variables for types that could contain record IDs (Text, Record, Text Collection).
3. Review flow logic to determine if input variables influence Get Records, Update Records, or Delete Records elements.
4. Flag any flow accepting user-supplied input variables that control record access as noncompliant.

**Remediation:**  
1. Refactor flows to eliminate input variables controlling record access.
2. Derive accessible records from authenticated user context (e.g., `$User.Id`, `$User.ContactId`, `$User.AccountId`).
3. Configure flows to run in user context ("With Sharing") where available.

**Default Value:**  
Salesforce does not prevent flows from accepting user-supplied input variables. Autolaunched Flows run in system context without sharing by default.
