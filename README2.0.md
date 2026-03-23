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
Address objects represent IP subnets. Currently, the playbook creates one source and one destination object.

### For Single IP (Current Setup)
- Edit `source_ip` and `destination_ip` as strings (e.g., `"10.10.199.0/24"`).
- Object names are fixed: `WEB_TIER_SUBNET` and `DATABASE_TIER_SUBNET`.

### For Multiple IPs
1. Change `source_ip` and `destination_ip` to lists:
   ```yaml
   source_ip: ["10.10.199.0/24", "10.10.198.0/24"]
   destination_ip: ["10.10.200.0/24", "10.10.201.0/24"]
   ```
2. The playbook will create multiple objects with names like `WEB_TIER_SUBNET_0`, `WEB_TIER_SUBNET_1`, etc.
3. **Note**: If adding multiple IPs, the address groups will automatically include all created objects.

### Custom Object Names
To use custom names, modify `source_object_name` and `destination_object_name`. For multiple, the playbook appends an index.

## Adding Address Groups
Address groups collect address objects. Currently, two groups are created.

- **Source Group**: `PRESET_WEB_TO_DATABASE_SOURCE` includes the source object(s).
- **Destination Group**: `PRESET_WEB_TO_DATABASE_DESTINATION` includes the destination object(s).

### Modifying Groups
- Change `source_group_name` and `destination_group_name` for custom names.
- The groups automatically reference the created objects. No manual addition needed.

### Adding More Groups
If you need additional groups (e.g., for different tiers):
1. Add new variables in `panorama_vars.yml` (e.g., `extra_group_name: "EXTRA_GROUP"`).
2. Add new tasks in `panorama_policy_complete.yml` to create them (copy the existing group creation logic).
3. Reference them in rules as needed.

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
Run the playbook. It creates 2 source objects and 1 destination object, with groups including all.

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
- **Module Errors**: Ensure collections are installed and Panorama is reachable.
- **Variable Issues**: Check YAML syntax in `panorama_vars.yml`.
- **Existing Items**: The playbook skips creation if items exist (idempotent).
- **Multiple IPs**: If using lists, ensure the playbook logic handles loops (it does for rules; for objects, it auto-indexes).
- **Logs**: Review Ansible output for "changed" status.

For advanced customizations (e.g., dynamic IPs), consult the Ansible documentation or modify the playbook tasks.

---

**Version**: 2.0  
**Last Updated**: March 23, 2026  
**Contact**: Operational Team</content>
<parameter name="filePath">c:\Users\111986\Downloads\ansible-panos-policy-automation-demo-main\README2.0.md