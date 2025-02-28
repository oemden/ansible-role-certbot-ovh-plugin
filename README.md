# Ansible Role: Certbot

An Ansible role to install and configure Certbot with OVH DNS plugin integration for Let's Encrypt certificate issuance and renewal.

## Overview

This role installs Certbot and configures it to use the OVH DNS plugin for DNS-01 challenge validation. This approach allows for obtaining wildcard certificates and doesn't require exposing your server to the internet for validation.

## Requirements

- Ansible 2.9 or higher
- Python 3.x on the target system
- OVH API credentials (for the OVH DNS plugin)

## Role Variables

<details>
<summary><b>Click to view Installation Settings</b></summary>

```yaml
# Installation method - pip (recommended) or package manager
certbot_use_pip: true

# Required packages for Certbot installation
certbot_required_packages:
  - python3
  - python3-venv
  - ca-certificates
  - openssl

# Virtual environment settings
certbot_venv_dir: "/opt/certbot-venv"
```
</details>

<details>
<summary><b>Click to view DNS Provider Configuration</b></summary>

```yaml
# DNS provider for DNS-01 challenge (currently supports OVH)
certbot_dns_provider: "ovh"

# Packages to install for DNS plugin
certbot_dns_plugin_packages:
  - certbot-dns-ovh
```
</details>

<details>
<summary><b>Click to view OVH Specific Settings</b></summary>

```yaml
# Directory to store OVH credentials
certbot_ovh_credentials_dir: "/etc/letsencrypt/ovh"

# OVH API endpoint
certbot_ovh_endpoint: "ovh-eu"  # Options: ovh-eu, ovh-ca, ovh-us

# OVH API credentials
certbot_ovh_application_key: ""
certbot_ovh_application_secret: ""
certbot_ovh_consumer_key: ""

# DNS propagation delay in seconds
certbot_ovh_propagation_seconds: 60
```
</details>

<details>
<summary><b>Click to view Certbot Configuration</b></summary>

```yaml
# Certbot directory settings
certbot_config_dir: "/etc/letsencrypt"
certbot_work_dir: "/var/lib/letsencrypt"
certbot_logs_dir: "/var/log/letsencrypt"

# Certificate settings
certbot_admin_email: "admin@example.com"
certbot_staging: false  # Set to true for testing
certbot_force_renew: false
certbot_additional_args: ""
```
</details>

<details>
<summary><b>Click to view Certificate Renewal Settings</b></summary>

```yaml
# Use cron instead of systemd timer
certbot_use_cron: false

# Cron job settings (if certbot_use_cron is true)
certbot_cron_minute: "30"
certbot_cron_hour: "3"
certbot_cron_weekday: "1"  # Monday
```
</details>

<details>
<summary><b>Click to view Certificate Configuration</b></summary>

```yaml
# Certificates to obtain
certbot_certs:
  - domains:
      - example.com
      - www.example.com
      - "*.example.com"  # Wildcard certificate requires DNS validation
    email: admin@example.com  # Optional, defaults to certbot_admin_email
    cert_name: example-com    # Optional
```
</details>

<details>
<summary><b>Click to view Post-Renewal Hooks</b></summary>

```yaml
# Post-renewal hooks to execute after certificate renewal
certbot_post_hooks:
  - name: reload-nginx.sh
    services:
      - nginx
```
</details>

## Example Playbook

<details>
<summary><b>Click to view Example Playbook</b></summary>

```yaml
- hosts: web_servers
  vars:
    certbot_admin_email: "webmaster@example.com"
    certbot_ovh_endpoint: "ovh-eu"
    certbot_ovh_application_key: "your_app_key"
    certbot_ovh_application_secret: "your_app_secret"
    certbot_ovh_consumer_key: "your_consumer_key"
    
    # Enable DNS management (optional)
    certbot_manage_dns: true
    
    certbot_certs:
      - domains:
          - example.com
          - www.example.com
          - "*.example.com"
        cert_name: example-com
        dns_managed: true  # Enable DNS management for this cert
        create_www: true   # Create a www CNAME record
      
      - domains:
          - api.example.com
        cert_name: api-example-com
        dns_managed: true  # Manage DNS for this subdomain
        # No create_www, so defaults to false
    
    certbot_post_hooks:
      - name: reload-services.sh
        services:
          - nginx
          - haproxy
  roles:
    - certbot
```
</details>

## DNS Record Management

<details>
<summary><b>Click to view DNS Management Configuration</b></summary>

This role can automatically manage DNS records for your certificates when used with OVH DNS. This makes it easy to set up new subdomains without manual DNS configuration.

### Prerequisites

1. Install the required Ansible collection:
   ```bash
   ansible-galaxy collection install ansible714.ovh
   ```

2. Enable DNS management in your playbook:
   ```yaml
   certbot_manage_dns: true
   ```

### Configuration Options

Add these variables to your defaults/main.yml file:

```yaml
# DNS Management Settings
certbot_manage_dns: false  # Set to true to enable DNS management
certbot_create_www_alias: false  # Set to false by default to avoid unexpected CNAME records
```

### Using DNS Management

In your playbook, you can control DNS record creation per certificate:

```yaml
certbot_certs:
  - domains:
      - example.com
      - www.example.com
    cert_name: example-com
    dns_managed: true  # Enable DNS management for this certificate
    create_www: true   # Create a www CNAME record
  
  - domains:
      - api.example.com
    cert_name: api-example-com
    dns_managed: true  # Manage DNS for this subdomain
    # No create_www, so defaults to false
```

### How It Works

When enabled, the role will:
1. Install the Python OVH module in the Certbot virtualenv
2. Use the OVH API to create A records for your domains pointing to your server's IP
3. Optionally create CNAME records for www subdomains
4. Refresh the DNS zone to apply changes
5. Then proceed with certificate issuance as normal

This approach allows for fully automated domain configuration and certificate issuance in a single playbook run.

</details>

## OVH API Setup

<details>
<summary><b>Click to view OVH API Credentials (Manual Method)</b></summary>

To use the OVH DNS plugin, you need to create API credentials through the OVH API console:

1. Go to the [OVH API create token page](https://api.ovh.com/createToken/)
2. Enter a name and description for your application
3. Set the validity period (unlimited is recommended for automation)
4. Set the necessary access rights:
   - GET /domain/zone/*
   - PUT /domain/zone/*
   - POST /domain/zone/*
   - DELETE /domain/zone/*
5. Create the token and note the values for:
   - Application Key (certbot_ovh_application_key)
   - Application Secret (certbot_ovh_application_secret)
   - Consumer Key (certbot_ovh_consumer_key)
</details>

<details>
<summary><b>Click to view API Application Setup (Automated Method)</b></summary>

### Generate an API Application:

1. Go to https://api.ovh.com/createApp/ to create an API application, which gives you an `application_key` and `application_secret`. 
    
    This is essentially creating your API identity:
    
    ```
    Application name	my-certbot
    Application description	My Certbot API TOKEN
    Application key	fakeapplicationkeyff0152bb0
    Application secret	fakeApplicationsecret12345abcdef67890ghijklkmnopoqr
    ```
    
2. To generate your consumer key (token) that has the specific permissions you need, run a curl command like this:
    
    ```bash
    curl -XPOST -H "X-Ovh-Application: fakeapplicationkeyff0152bb0" \
    -H "Content-type: application/json" \
    https://eu.api.ovh.com/1.0/auth/credential \
    -d '{
      "accessRules": [
        {"method": "GET", "path": "/domain/zone/example.com/*"},
        {"method": "POST", "path": "/domain/zone/example.com/*"},
        {"method": "PUT", "path": "/domain/zone/example.com/*"},
        {"method": "DELETE", "path": "/domain/zone/example.com/record/*"}
      ],
      "redirection": "https://ovh.com"
    }'
    ```
    
    > **Note:** `"redirection": "https://example-redirect-uri.com"` can be anything, it is just needed by OVH tool and must be a valid URI format.
    >
    > You can:
    > 1. Run the curl command once on your admin computer
    > 2. Visit the validation URL **once** to authorize the token
    > 3. Store the resulting application key, application secret, and consumer key securely
    > 4. Use these same credentials on any VM, server, or in any automation tool like Ansible
    >
    > This approach is great for automation - you go through the manual authorization step just once, and then you can use the same credentials in your Certbot OVH DNS plugin configuration files on any VM or server where you need to issue or renew certificates.
    >
    > The credentials don't expire unless you specifically set an expiration date when creating them or manually revoke them from your OVH account.
    
3. **Example Output:**
    
    ```json
    {
      "consumerKey": "someSupersecretkey1234",
      "validationUrl": "https://www.ovh.com/auth/sso/api?credentialToken=superfakecredentialtoken63ed3ba1f12cb668b52a4292b1f2fdf4de025",
      "state": "pendingValidation"
    }
    ```
    
    Visit the validationUrl in your browser to authorize the token. After validation, the app is active and the token can be used by certbot with the OVH plugin and the ini file.
</details>

<details>
<summary><b>Click to view Certbot OVH INI File Configuration</b></summary>

Example `ovh.ini` file that will be placed at `{{ certbot_ovh_credentials_dir }}/ovh.ini`:

```ini
# OVH API credentials used by Certbot
dns_ovh_endpoint = ovh-eu
dns_ovh_application_key = fakeapplicationkeyff0152bb0
dns_ovh_application_secret = fakeApplicationsecret12345abcdef67890ghijklkmnopoqr
dns_ovh_consumer_key = someSupersecretkey1234
```

This file will be used by the Certbot DNS plugin to make the necessary API calls to OVH for DNS verification.
</details>

## Implementation Notes

This role installs Certbot and the OVH DNS plugin in a Python virtual environment to avoid conflicts with system Python packages. This approach works well with modern Debian-based systems that implement PEP 668 (externally-managed-environment).

The virtual environment is created at the location specified by `certbot_venv_dir` (default: `/opt/certbot-venv`), and all Certbot commands use this environment.

### Python Dependencies

The role manages the following Python dependencies in the virtualenv:
- certbot (core functionality)
- certbot-dns-ovh (OVH DNS plugin)
- ovh (Python OVH API client for DNS management)

### Working with Modern Debian Systems

Modern Debian-based systems (Ubuntu 22.04+, Debian 12+) use externally managed environments as per PEP 668. This role handles these restrictions by:

1. Installing all Python dependencies in a dedicated virtualenv
2. Using the virtualenv for all Certbot operations
3. Using a custom Python script for DNS management that runs within the virtualenv

This approach avoids conflicts with system Python packages while still providing all required functionality.

## License

MIT

## Author Information

olive oem
