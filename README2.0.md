# Panorama Policy Automation Playbook - Operational Guide (v2.0)

This guide provides step-by-step instructions for the operational team on how to configure and extend the Panorama policy automation using Ansible. The playbook (`panorama_policy_complete.yml`) automates the creation of address objects, groups, application groups, and security rules in Palo Alto Networks Panorama.

## Table of Contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [File Structure](#file-structure)
- [Configuration Variables](#configuration-variables)
- [Adding Address Objects](#adding-address-objects)
- [Adding Address Groups](#adding-address-groups)
- [Adding Application Groups](#adding-application-groups)
- [Adding Security Rules](#adding-security-rules)
- [Running the Playbook](#running-the-playbook)
- [Examples](#examples)
- [Troubleshooting](#troubleshooting)

## Overview
The playbook creates policy components in Panorama:
- **Address Objects**: IP subnets (e.g., 10.10.199.0/24).
- **Address Groups**: Collections of address objects.
- **Application Groups**: Collections of applications (e.g., web-browsing).
- **Security Rules**: Firewall rules using the above groups.

All configurations are defined in `panorama_vars.yml`. Edit this file to add or modify items—do not edit the main playbook file.

## Prerequisites
- Ansible installed on the control machine.
- Palo Alto Networks Ansible collections: `ansible-galaxy collection install paloaltonetworks.panos`.
- Environment variables set: `ANSIBLE_NET_USERNAME` and `ANSIBLE_NET_PASSWORD` for Panorama authentication.
- Access to Panorama API (ensure the `ip_address` in `provider` is reachable).
- Basic knowledge of YAML and Panorama concepts.

## File Structure
```
ansible-panos-policy-automation-demo-main/
├── panorama_policy_complete.yml  # Main playbook (do not edit)
├── panorama_vars.yml             # Configuration variables (edit this)
├── README2.0.md                  # This guide
└── collections/requirements.yml  # Ansible collections
```

## Configuration Variables
All settings are in `panorama_vars.yml`. Key sections:

- **provider**: Panorama connection details.
- **panorama_device_group**: Target device group (e.g., "Lab").
- **panorama_rulebase**: Rulebase type (e.g., "pre-rulebase").
- **source_ip / destination_ip**: IP subnets for objects (can be single string or list for multiple).
- **Object/Group Names**: Custom names for objects and groups.
- **rules**: List of security rules.

## Adding Address Objects
Address objects represent IP subnets. The playbook supports both single and multiple IPs.

### For Single IP
- Define `source_ip` and `destination_ip` as strings:
   ```yaml
   source_ip: "10.10.199.0/24"
   destination_ip: "10.10.200.0/24"
   ```
- Creates objects: `WEB_TIER_SUBNET_0` and `DATABASE_TIER_SUBNET_0`.

### For Multiple IPs (Recommended)
1. Define `source_ip` and `destination_ip` as lists:
   ```yaml
   source_ip: ["10.10.199.0/24", "10.10.198.0/24"]
   destination_ip: ["10.10.200.0/24", "10.10.201.0/24"]
   ```
2. The playbook automatically creates indexed objects:
   - Source: `WEB_TIER_SUBNET_0`, `WEB_TIER_SUBNET_1`, ...
   - Destination: `DATABASE_TIER_SUBNET_0`, `DATABASE_TIER_SUBNET_1`, ...
3. Each object corresponds to one IP in the list (by index).

### Custom Object Names
Modify `source_object_name` and `destination_object_name` in `panorama_vars.yml`:
   ```yaml
   source_object_name: "APP_SUBNET"
   destination_object_name: "DB_SUBNET"
   ```
With multiple IPs, objects will be named: `APP_SUBNET_0`, `APP_SUBNET_1`, etc.

## Adding Address Groups
Address groups collect address objects. Two groups are created automatically.

- **Source Group**: `PRESET_WEB_TO_DATABASE_SOURCE` includes ALL source objects.
- **Destination Group**: `PRESET_WEB_TO_DATABASE_DESTINATION` includes ALL destination objects.

### How Groups Work
- The playbook automatically detects the number of source/destination IPs and includes all corresponding objects.
- Example: With 2 source IPs, the group includes `[WEB_TIER_SUBNET_0, WEB_TIER_SUBNET_1]`.
- **No manual configuration needed**—groups are built dynamically.

### Modifying Group Names
- Change `source_group_name` and `destination_group_name` in `panorama_vars.yml` for custom names:
   ```yaml
   source_group_name: "MY_SOURCE_GROUP"
   destination_group_name: "MY_DEST_GROUP"
   ```

### Adding More Groups
If you need additional groups (e.g., for different tiers):
1. Add new variables in `panorama_vars.yml` (e.g., `extra_group_name: "EXTRA_GROUP"`).
2. Create the corresponding address objects first (or reference existing ones).
3. Add new tasks in `panorama_policy_complete.yml` to create the group (copy the existing group creation logic).
4. Reference the group in security rules.

## Adding Application Groups
Application groups collect applications. Currently, one group is created: `PRESET_WEB_TO_DATABASE` with "web-browsing".

### Modifying the Application Group
- Change `application_group_name` for a custom name.
- To add more applications, edit the `value` list in the playbook (under Step 5).

### Adding More Application Groups
1. Add new variables (e.g., `extra_app_group_name: "EXTRA_APP_GROUP"`).
2. Add tasks in the playbook to create them.
3. Reference in rules.

## Adding Security Rules
Rules are defined as a list in `rules`. Each rule is a dictionary with parameters like `rule_name`, `source_zone`, etc.

### Adding a New Rule
1. Append a new dictionary to the `rules` list in `panorama_vars.yml`:
   ```yaml
   rules:
     - rule_name: "Existing Rule"
       # ... existing config ...
     - rule_name: "New Rule Name"
       source_zone: "trust"
       destination_zone: "untrust"
       source_ip: "any"  # Or reference a group, e.g., "{{ source_group_name }}"
       destination_ip: "any"
       application: "ssl"
       service: "any"
       action: "allow"
       description: "Description of the new rule"
       tag_name: "Automated"
   ```
2. The playbook will create it automatically.

### Rule Parameters
- `rule_name`: Unique name.
- `source_zone` / `destination_zone`: Zones (e.g., "any", "trust").
- `source_ip` / `destination_ip`: IPs or groups (use variable references like `"{{ source_group_name }}"`).
- `application`: Apps or groups (e.g., `"{{ application_group_name }}"`).
- `service`: Services (e.g., "any").
- `action`: "allow", "deny", etc.
- `description`: Human-readable description.
- `tag_name`: Tag for organization.

## Running the Playbook
1. Set environment variables:
   ```bash
   export ANSIBLE_NET_USERNAME=your_username
   export ANSIBLE_NET_PASSWORD=your_password
   ```
2. Run the playbook:
   ```bash
   ansible-playbook panorama_policy_complete.yml
   ```
3. Monitor output for creation status. Changes are committed to Panorama and pushed to the device group.

## Examples

### Example 1: Add Multiple IPs
Update `panorama_vars.yml`:
```yaml
source_ip: ["10.10.199.0/24", "10.10.198.0/24"]
destination_ip: ["10.10.200.0/24"]
```
Run the playbook:
```bash
ansible-playbook panorama_policy_complete.yml
```
**Result**:
- Source objects created: `WEB_TIER_SUBNET_0` (10.10.199.0/24), `WEB_TIER_SUBNET_1` (10.10.198.0/24)
- Destination object: `DATABASE_TIER_SUBNET_0` (10.10.200.0/24)
- Source group includes: `[WEB_TIER_SUBNET_0, WEB_TIER_SUBNET_1]`
- Destination group includes: `[DATABASE_TIER_SUBNET_0]`

### Example 2: Add a New Security Rule
Update `panorama_vars.yml`:
```yaml
rules:
  - rule_name: "Match Web Server to Database Tier"
    # ... existing ...
  - rule_name: "Allow SSL from Trust to Untrust"
    source_zone: "trust"
    destination_zone: "untrust"
    source_ip: "any"
    destination_ip: "any"
    application: "ssl"
    service: "any"
    action: "allow"
    description: "Allow SSL traffic outbound"
    tag_name: "Automated"
```
Run the playbook to create the new rule.

## Troubleshooting

### Common Issues

**Multiple IPs not creating objects**
- Verify `source_ip` and `destination_ip` are properly formatted as lists in YAML:
  ```yaml
  source_ip: ["10.10.199.0/24", "10.10.198.0/24"]  # Correct
  source_ip: "10.10.199.0/24, 10.10.198.0/24"     # Incorrect
  ```
- Check Ansible output for object creation status (look for "changed": true).

**Groups not including all objects**
- Ensure objects are created first. Check Panorama UI to verify object existence.
- Groups are built dynamically based on the number of IPs—verify the list length matches.

**Module errors**
- Ensure Palo Alto Networks Ansible collections are installed:
  ```bash
  ansible-galaxy collection install paloaltonetworks.panos
  ```
- Verify Panorama is reachable at the configured IP address.
- Check credentials: `ANSIBLE_NET_USERNAME` and `ANSIBLE_NET_PASSWORD` environment variables.

**Existing items skipped**
- The playbook is idempotent—if objects/groups already exist, they are skipped ("changed": false).
- To force recreation, manually delete items in Panorama and re-run the playbook.

**YAML syntax errors**
- Validate `panorama_vars.yml` syntax:
  ```bash
  python -m yaml < panorama_vars.yml
  ```
- Ensure proper indentation (2 spaces, not tabs).

**For Advanced Help**
- Review Ansible debug output: `ansible-playbook -vvv panorama_policy_complete.yml`
- Consult Palo Alto Networks Ansible collection documentation.

---

**Version**: 2.1  
**Last Updated**: March 23, 2026  
**Contact**: Operational Team</content>
<parameter name="filePath">c:\Users\111986\Downloads\ansible-panos-policy-automation-demo-main\README2.0.md