## File Security

This section defines controls related to file and content security in Salesforce environments. These controls ensure that organizations maintain appropriate protections, governance, and lifecycle management over files, documents, and content shared within or outside the organization—reducing the risk of unauthorized access, data leakage, and exposure of sensitive information.

### SBS-FILE-001: Require Expiry Dates on Public Content Links

**Control Statement:** Organizations must ensure that Public Content links have an appropriate expiry date.

**Description:**  
The organization must ensure that any content shared via Public Content links has an appropriate expiry date set dependent upon the classification of the content. The expiry date could be never for non-sensitive content—a PDF of the organization's Privacy Policy as an example—or it could be less than a week for sensitive information.

**Risk:** <Badge type="tip" text="Moderate" />  
Without an expiry date, Public Content links remain permanently accessible, extending the window of potential exposure indefinitely. While the link itself must be obtained by an unauthorized party for access to occur, perpetually valid links increase the cumulative risk of data exposure through link leakage, sharing, or discovery. Time-bounded links reduce the blast radius of any single link compromise and support data lifecycle governance.

**Audit Procedure:**  
1. Enumerate all `ContentDistribution` object records via the SOAP/REST API or Apex.
2. Identify all records where `PreferencesExpires = false`.
3. Flag any Public Content links without expiry dates for review.

**Remediation:**  
1. For each flagged content distribution record, determine the sensitivity classification of the associated content.
2. Set an appropriate expiry date on the `ContentDistribution` object based on content classification.
3. Establish organizational policy defining maximum link lifetimes by data classification.

**Default Value:**  
When a user manually creates a Public Content link on a piece of content, Salesforce suggests an expiry date. This can be overridden by the user. In the past, the default was no expiry date.

### SBS-FILE-002: Require Passwords on Public Content Links for Sensitive Content

**Control Statement:** Organizations must ensure that Public Content links to sensitive content have a password.

**Description:**  
The organization must ensure that any sensitive content shared via Public Content links has a password set to protect the content if the link is intercepted or inadvertently shared.

**Risk:** <Badge type="warning" text="High" />  
Without a password, anyone who obtains an unexpired Public Content link—through interception, accidental sharing, or link harvesting—can immediately access the associated data. For sensitive content, this creates a direct path to data exposure that requires only link acquisition. Password protection adds an authentication layer that prevents opportunistic access and limits the impact of link compromise, supporting breach containment and regulatory compliance for sensitive data handling.

**Audit Procedure:**  
1. Enumerate all `ContentDistribution` object records via the SOAP/REST API or Apex.
2. Identify all records where `Password` is null.
3. Cross-reference with content classification to identify sensitive content lacking password protection.
4. Flag any Public Content links to sensitive content without passwords for review.

**Remediation:**  
1. For each flagged content distribution record, determine the sensitivity classification of the associated content.
2. For sensitive content, set a password on the ContentDistribution record via the Salesforce UI.
3. Communicate the password to intended recipients through a separate, secure channel.
4. Establish organizational policy requiring password protection for all Public Content links to sensitive data.

**Default Value:**  
When a user manually creates a Public Content link on a piece of content, the default is to not have a password.

### SBS-FILE-003: Periodic Review and Cleanup of Public Content Links

**Control Statement:** Organizations must implement a recurring process to review all active Public Content links and remove or remediate links that are no longer required, lack appropriate controls, or were created outside of current policy.

**Description:**  
The organization must establish a defined cadence (e.g., quarterly) to scan all `ContentDistribution` records and review active Public Content links. This review should identify links that are forgotten, no longer needed, were created before current security controls were implemented, resulted from accidental sharing, or otherwise do not comply with organizational policy. Identified links must be remediated by applying appropriate controls (expiry dates, passwords) or deleted if no longer required.

**Risk:** <Badge type="tip" text="Moderate" />  
Without periodic review, Public Content links accumulate over time—including legacy links created before security policies were established, links that have outlived their business purpose, and links created through accidental or unauthorized sharing. These forgotten links represent persistent exposure that may go undetected indefinitely. While this control does not prevent initial link creation issues, it provides a governance mechanism to identify and remediate accumulated risk, supporting defense-in-depth and reducing the organization's overall exposure footprint.

**Audit Procedure:**  
1. Verify the organization has a documented process for periodic Public Content link review.
2. Confirm the review cadence is defined (e.g., quarterly, monthly) and appropriate for the organization's risk profile.
3. Obtain evidence of recent review execution (e.g., scan results, remediation records, review meeting notes).
4. Verify that reviews include all active `ContentDistribution` records.
5. Confirm that identified issues are tracked through remediation or deletion.
6. Flag organizations without a documented review process or evidence of recent execution.

**Remediation:**  
1. Establish a documented process for periodic review of all `ContentDistribution` records.
2. Define a review cadence appropriate to organizational risk tolerance (quarterly recommended as a baseline).
3. Create a scanning mechanism (script, report, or tool) to enumerate all active Public Content links.
4. Define review criteria to identify links requiring remediation: missing expiry dates, missing passwords on sensitive content, links older than a defined threshold, or links to content no longer requiring external sharing.
5. Assign ownership for the review process and remediation actions.
6. Maintain records of each review cycle for audit purposes.

**Default Value:**  
Salesforce does not provide a built-in mechanism for periodic review of Public Content links; organizations must implement this process manually or through custom tooling.
