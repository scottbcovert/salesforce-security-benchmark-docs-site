# For Security Vendors

This page provides guidance for security tooling vendors who want to integrate SBS compliance capabilities into their products.

## Machine-Readable Format

SBS controls are available in a structured XML format designed for programmatic consumption. Each release includes a complete XML representation of all controls, enabling vendors to build automated scanners, compliance dashboards, and reporting tools without hardcoding control text.

## Accessing the XML

The XML file is published with each SBS release on GitHub:

```
https://github.com/Salesforce-Security-Benchmark/releases/download/v1.0.0/sbs-controls.xml
```

Replace `v1.0.0` with the desired version. Always use a specific version to ensure consistency in your tooling.

## XML Structure

The XML file follows a predictable structure:

```xml
<sbs_benchmark version="1.0.0" xmlns="https://securitybenchmark.dev/sbs/v1">
  <metadata>
    <title>Security Benchmark for Salesforce</title>
    <version>1.0.0</version>
    <total_controls>24</total_controls>
  </metadata>
  <controls>
    <category>
      <name>OAuth Security</name>
      <description>This section defines controls related to OAuth-enabled Connected Apps...</description>
      <control id="SBS-OAUTH-001">
        <title>Require Formal Installation and Access Governance for Connected Apps</title>
        <statement>Organizations must formally install all connected apps...</statement>
        <description>The organization must ensure that any connected app...</description>
        <rationale>Implicitly created OAuth connections inherit configuration...</rationale>
        <audit_procedure>1. Enumerate all user-authorized OAuth connected apps...
2. Identify all connected apps that are not formally installed...</audit_procedure>
        <remediation>1. Formally install any connected app...
2. Configure the installed connected app's policies...</remediation>
        <default_value>When a user first authenticates to a connected app...</default_value>
      </control>
    </category>
  </controls>
</sbs_benchmark>
```

## Remediation Scope and Task Generation

Starting with SBS version 0.2.0, controls may include additional metadata to support **deterministic task generation** in compliance tools. This metadata ensures that when a control fails, your tool can generate clear, actionable tasks without inventing language or guessing intent.

### Remediation Scope

Each control maps to one of four remediation scopes that reflect fundamentally different types of work:

- **org** — One org-level configuration change (e.g., "Enable SSO enforcement for production users")
- **entity** — One task per noncompliant entity (e.g., "Formally install Connected App: Slack")
- **mechanism** — Implement tooling or automated process (e.g., "Implement regulated data detection for Long Text Area fields")
- **inventory** — Establish and maintain system of record (e.g., "Establish and maintain SSO bypass user inventory")

Controls that include remediation metadata will have a `<remediation_scope>` block in the XML:

```xml
<control id="SBS-OAUTH-003">
  <title>Inventory and Criticality Classification of OAuth-Enabled Connected Apps</title>
  <statement>...</statement>
  <description>...</description>
  <rationale>...</rationale>
  <audit_procedure>...</audit_procedure>
  <remediation>...</remediation>
  
  <remediation_scope>
    <scope>inventory</scope>
  </remediation_scope>
  
  <task>
    <title_template>Establish and maintain Connected App criticality inventory</title_template>
  </task>
  
  <default_value>...</default_value>
</control>
```

### Understanding Scope Types

**org** — Org-level configuration changes
- One task per Salesforce org
- Typically involves enabling/disabling settings or configuring org-wide policies
- Example: Enable SSO enforcement setting + assign permissions
- Implementation: Configuration changes, usually no custom code

**entity** — Per-entity remediation
- One task per noncompliant entity
- Vendors scan org, identify violations, generate tasks for each
- Example: 5 Connected Apps not installed → 5 tasks
- Implementation: Bulk actions, list views, per-item remediation

**mechanism** — Build and deploy capability
- One task to implement tooling or automation
- Requires development, deployment, and ongoing operation
- Example: Build custom scanner for regulated data detection
- Implementation: Custom code, monitoring, alerting infrastructure

**inventory** — System of record maintenance
- One task to establish and maintain documentation
- Focuses on governance and external tracking systems
- Example: Create inventory of SSO bypass users with justifications
- Implementation: Integration with CMDB/GRC tools, periodic reconciliation

### Task Title Templates

The `<task>` block contains a `<title_template>` that vendors should use to generate tasks. Templates use mustache-style syntax (e.g., double curly braces around `entity.name`) to insert dynamic values for entity-scoped controls.

**Why this matters:**

Instead of your tool inventing task names like "Fix Connected App Issues" or "Resolve OAUTH-003," you generate the exact task title specified by SBS.

**For entity-scoped controls:**
```
Formally install Connected App: Slack
Formally install Connected App: Jira
Formally install Connected App: DocuSign
```

**For other scopes:**
```
Implement SSO enforcement for production users (org)
Establish and maintain SSO bypass user inventory (inventory)
Implement regulated data detection for Long Text Area fields (mechanism)
```

This ensures consistency across all tools that implement SBS, creates better user experiences, and eliminates ambiguity about what remediation action is required.

### Implementation Example

```python
import xml.etree.ElementTree as ET

# Parse control metadata
control = tree.find(".//control[@id='SBS-OAUTH-001']")
scope_elem = control.find('remediation_scope/scope')
task_template = control.find('task/title_template').text

# Scan org and determine compliance
scan_result = scan_control(org, 'SBS-OAUTH-001')

# Generate tasks based on scope
scope = scope_elem.text if scope_elem is not None else 'org'

if scope == 'entity':
    # Generate one task per noncompliant entity
    for entity in scan_result.noncompliant_entities:
        task_title = task_template.replace('{{entity.name}}', entity.name)
        create_task(task_title, control_id='SBS-OAUTH-001', entity_id=entity.id)

elif scope == 'org':
    # Generate one task for the org if noncompliant
    if not scan_result.compliant:
        create_task(task_template, control_id='SBS-OAUTH-001')

elif scope == 'inventory':
    # Generate one task if inventory gaps exist
    if scan_result.missing_inventory_count > 0:
        create_task(task_template, control_id='SBS-OAUTH-001', 
                   context={'missing': scan_result.missing_inventory_count})

elif scope == 'mechanism':
    # Generate one task if mechanism not implemented
    if not scan_result.mechanism_exists:
        create_task(task_template, control_id='SBS-OAUTH-001')
```

### Controls Without Metadata

Not all controls have remediation metadata yet. For controls without `<remediation_scope>` or `<task>` blocks, treat them as `org`-scoped and generate one task per org using your own naming conventions.

As SBS evolves, more controls will include metadata to improve task generation consistency.

## Understanding What SBS Provides

**SBS defines *what* to check, not *how* to check it.**

The XML file provides structured control requirements—statements, descriptions, audit procedures, and remediation guidance—but it does not include the technical implementation for scanning Salesforce orgs. Vendors must develop their own algorithms, API queries, and evaluation logic to determine compliance.

### Why Vendors Must Implement Scanning Logic

SBS controls do not map directly to simple binary Salesforce metadata attributes. For example:

- **SBS-ACS-002** requires that all API-enabled authorizations have documented justifications in a system of record. This requires querying profiles/permission sets via Metadata API, identifying which grant "API Enabled," and then checking an external system of record for justifications.

- **SBS-CHG-003** requires monitoring for unauthorized metadata changes. This might involve polling the Setup Audit Trail API, comparing changes against a deployment identity, and alerting on violations.

- **SBS-OAUTH-002** requires maintaining an inventory with criticality ratings. The vendor must query Connected Apps, cross-reference an external inventory, and validate completeness.

Each control requires vendor-specific logic combining Salesforce API calls, data correlation, business rule evaluation, and integration with external systems.

**SBS provides the compliance standard. Vendors provide the automation.**

## Example Integration

Here's a simplified example of how a vendor might use the XML:

```python
import xml.etree.ElementTree as ET
import requests

# Load SBS controls
response = requests.get('https://github.com/.../sbs-controls.xml')
tree = ET.fromstring(response.content)

# Map control IDs to your scanning functions
SCANNERS = {
    'SBS-AUTH-001': check_sso_enforcement,
    'SBS-ACS-002': check_api_enabled_justifications,
    'SBS-OAUTH-001': check_connected_app_installation,
    # ... vendor implements these functions
}

# Iterate through controls and run scans
for control in tree.findall('.//control'):
    control_id = control.get('id')
    scanner = SCANNERS.get(control_id)
    
    if scanner:
        result = scanner(salesforce_org)
        print(f"{control_id}: {'PASS' if result.compliant else 'FAIL'}")
    else:
        print(f"{control_id}: Not implemented")
```

The vendor must implement each `check_*` function using Salesforce APIs, metadata queries, and business logic appropriate to their architecture.

## Sample XML Excerpt

Here's a complete control as it appears in the XML:

```xml
<control id="SBS-ACS-002">
  <title>Documented Justification for All API-Enabled Authorizations</title>
  <statement>Every authorization granting the "API Enabled" permission must have documented business or technical justification recorded in a system of record.</statement>
  <description>All profiles, permission sets, and permission set groups that grant the "API Enabled" permission must be recorded in a designated system of record with a documented business or technical justification for requiring API access.</description>
  <rationale>The "API Enabled" permission allows users to authenticate to and interact with Salesforce via APIs, which enables large-scale data extraction, modification, or destructive operations.</rationale>
  <audit_procedure>1. Enumerate all profiles, permission sets, and permission set groups that include the "API Enabled" permission
2. Compare the enumerated list against the organization's designated system of record
3. Verify that every profile, permission set, and permission set group has a corresponding entry
4. Confirm that each entry includes clear justification
5. Flag as noncompliant any authorizations lacking documentation</audit_procedure>
  <remediation>1. Remove the "API Enabled" permission from any profile, permission set, or permission set group that lacks documented justification
2. For any authorization that legitimately requires API access, add or update the rationale
3. Reconcile and update the system of record</remediation>
  <default_value>Salesforce does not require or maintain a system of record for API-enabled authorizations. The "API Enabled" permission is disabled by default for standard profiles.</default_value>
</control>
```

## Implementation Recommendations

### Version Pinning

Always reference a specific SBS version in your product to ensure consistency for your customers and prevent unexpected changes when SBS is updated.

### Caching

Cache the XML file locally to avoid repeated network requests during scans.

### Incremental Implementation

You don't need to support all 24 controls immediately. Start with high-value controls that are easily automated (e.g., SBS-AUTH-001 for SSO enforcement), then expand coverage over time.

### Leverage Audit Procedures

Each control includes detailed audit procedures. Use these as a blueprint for your scanning logic—they describe exactly what to query, compare, and validate.

## Marketing Your SBS Support

When you've integrated SBS into your product, you can market it as:

- "**SBS-Compliant Scanning**" — Full coverage of all 24 controls
- "**Automated SBS Reporting**" — Generate compliance reports in seconds
- "**SBS Gap Analysis**" — Identify which controls are failing and why
- "**Built for SBS v1.0**" — Version-specific implementation

## Questions or Feedback

If you're building SBS integration and have questions about the XML format or specific controls, please open an issue on the GitHub repository or contact the maintainers.

SBS is vendor-neutral and designed to support the entire Salesforce security tooling ecosystem. We welcome implementations from all vendors and are happy to provide guidance.

