+++
# `title` is how your post will be listed and what will appear at the top of the post
title= "Integrating Valkey with LDAP and Active Directory Authentication"
# `date` is when your post will be published.
# For the most part, you can leave this as the day you _started_ the post.
# The maintainers will update this value before publishing
# The time is generally irrelevant in how Valkey published, so '01:01:01' is a good placeholder
date= 2025-06-17 01:01:01
# 'description' is what is shown as a snippet/summary in various contexts.
# You can make this the first few lines of the post or (better) a hook for readers.
# Aim for 2 short sentences.
description= "Valkey now supports seamless integration with enterprise identity management systems through the official valkey-ldap module."
# 'authors' are the folks who wrote or contributed to the post.
# Each author corresponds to a biography file (more info later in this document)
authors= [ "rdias"]
[extra]
featured = false
featured_image = "/assets/media/featured/random-03.webp"
+++


Valkey is a high-performance, open-source in-memory data store, and many organizations rely on it for caching, session management, and real-time analytics. As Valkey adoption grows in enterprise environments, integrating with centralized authentication systems like LDAP and Active Directory becomes essential for security and compliance.

In this post, we’ll take a deep dive into how Valkey’s [valkey-ldap module](https://github.com/valkey-io/valkey-ldap) enables LDAP authentication, the different authentication modes it supports, how to configure secure connections, and best practices for managing users and monitoring your setup.

---

## Why Integrate Valkey with LDAP?

LDAP (Lightweight Directory Access Protocol) is the backbone of many enterprise identity management systems, including Microsoft Active Directory and OpenLDAP. Integrating Valkey with LDAP allows you to:

- **Centralize authentication:** Manage user credentials and policies in one place.
- **Enforce enterprise security standards:** Use existing password policies, multi-factor authentication, and account lifecycle management.
- **Simplify user management:** Onboard and offboard users across all systems from a single directory.

---

## Overview of the valkey-ldap Module

The valkey-ldap module is compatible with Valkey 7.2.x and above. Once loaded, it enables Valkey to authenticate users against external LDAP directories, including Active Directory. This means you can leverage your organization’s existing authentication infrastructure without managing separate Valkey passwords for each user.

---

## LDAP Authentication Modes in Valkey

Valkey’s LDAP module supports two authentication modes, each designed for different directory structures and requirements:

### 1. Bind Mode

**Bind mode** is the simplest approach. When a user attempts to authenticate, the module constructs the user’s distinguished name (DN) by combining a configurable prefix and suffix with the username provided in the `AUTH` command. It then attempts to bind to the LDAP server using this DN and the password.

**When to use:**  
- Your LDAP directory has a straightforward structure.
- Usernames map directly to DN components.

**Example:**

Suppose your LDAP entry looks like this:

```
dn: cn=alice,ou=engineering,dc=valkey,dc=io
objectClass: person
objectClass: inetOrgPerson
cn: alice
sn: Alice
uid: alice
```

If the username is `alice`, you can set the prefix to `cn=` and the suffix to `,ou=engineering,dc=valkey,dc=io`. The module will attempt to bind as `cn=alice,ou=engineering,dc=valkey,dc=io`.

**Configuration:**

- `ldap.bind_dn_prefix`: Prefix to prepend to the username (e.g., `cn=`)
- `ldap.bind_dn_suffix`: Suffix to append to the username (e.g., `,ou=engineering,dc=valkey,dc=io`)

---

### 2. Search+Bind Mode

**Search+bind mode** is more flexible and suitable for complex directory structures. Instead of constructing the DN, the module first binds to the LDAP server (optionally as a search user), searches for the user entry using a configurable attribute (like `uid`), retrieves the DN from a specified attribute, and then attempts to bind as that user.

**When to use:**  
- Usernames do not map directly to DN components.
- Your directory structure is complex or nested.
- You need to search for users based on attributes other than the DN.

**Example:**

```
dn: cn=Ben Alex,ou=engineering,dc=valkey,dc=io
objectClass: person
objectClass: inetOrgPerson
entryDN: cn=Ben Alex,ou=engineering,dc=valkey,dc=io
cn: Ben Alex
sn: Alex
uid: ben
```

Here, the username is stored in the `uid` attribute, and the DN is in the `entryDN` attribute. The module will search for `uid=ben`, extract the DN from `entryDN`, and attempt to bind as that user.

**Configuration:**

- `ldap.search_bind_dn` / `ldap.search_bind_passwd`: Credentials for the search account (optional; anonymous bind if omitted)
- `ldap.search_base`: The base DN for the search
- `ldap.search_filter`: LDAP filter (default: `objectClass=*`)
- `ldap.search_attribute`: Attribute to match the username (e.g., `uid`)
- `ldap.search_dn_attribute`: Attribute containing the DN (e.g., `entryDN`)
- `ldap.search_scope`: Search scope (`base`, `one`, or `sub`)

---

## LDAP Authentication Flow (Sequence Diagram)

Below is a sequence diagram illustrating how authentication requests flow between the client, Valkey server, valkey-ldap module, and the LDAP server in both Bind and Search+Bind modes:

![LDAP authentication](/assets/media/pictures/ldap_authentication.png)

---

## Choosing Between Bind and Search+Bind

- **Bind mode** is faster and simpler, but only works if usernames map directly to DN components.
- **Search+bind mode** is more flexible and supports complex directory structures, but requires additional LDAP requests and configuration.

---

## Setting Up Valkey Users for LDAP Authentication

**Important:**  
LDAP is used only for authentication. Authorization (permissions) must be managed in Valkey itself using ACLs. Each LDAP user must also exist as a Valkey user.

**Example:**  
To allow the LDAP user `bob` to authenticate, create a Valkey user named `bob`:

```
ACL SETUSER bob on +@hash
```

If you want to ensure `bob` cannot log in with a password (only via LDAP), simply omit the password when creating the user.

---

## Configuring the LDAP Module

All configuration options are prefixed with `ldap.` and can be set at runtime using the `CONFIG SET` command. To view current LDAP-related settings:

```
CONFIG GET ldap.*
```

**Key options:**

- `ldap.servers`: Space-separated list of LDAP URLs (e.g., `ldap://ldap.example.com:389`)
- `ldap.auth_mode`: Set to `bind` or `search+bind`
- `ldap.bind_dn_prefix` / `ldap.bind_dn_suffix`: For bind mode
- `ldap.search_*`: For search+bind mode

---

## Securing LDAP Connections: LDAPS and STARTTLS

Security is critical when transmitting credentials. The LDAP module supports two secure connection methods:

### LDAPS (LDAP over SSL)

- Establishes an SSL/TLS session immediately after the TCP connection (typically on port 636).
- All communication is encrypted from the start.
- Use the `ldaps://` URL scheme.

### STARTTLS

- Starts with a standard, unencrypted LDAP connection (usually on port 389).
- The client issues a STARTTLS command to upgrade the connection to TLS.
- Enable by setting `ldap.use_starttls` to `yes`.

**Certificate Verification:**  
Set `ldap.tls_ca_cert_path` to the path of a trusted CA certificate to verify the LDAP server’s certificate.  
For client certificate authentication, use `ldap.tls_cert_path` and `ldap.tls_key_path`.

---

## Monitoring LDAP Integration

The module extends the output of the `INFO` command with a new `ldap_status` section, providing real-time status for each configured LDAP server:

- `host`: The server hostname
- `status`: `healthy` (reachable) or `unhealthy` (not responding)
- `ping_time_ms`: Round-trip time for a simple LDAP operation (shown for healthy servers)
- `error`: Error description (shown for unhealthy servers)

**Example:**

```
# ldap_status
ldap_server_0:host=ldap1.example.com,status=unhealthy,error=Timeout
ldap_server_1:host=ldap2.example.com,status=healthy,ping_time_ms=1.645
```

---

## Advanced Configuration

### Connection Pooling

LDAP servers are often shared resources. The `ldap.connection_pool_size` option controls how many connections the module will open to each LDAP server. Set this based on your expected authentication load and the server’s capacity.

### Failure Detection and Failover

The module periodically checks the health of each LDAP server (interval set by `ldap.failure_detector_interval`). If a server becomes unreachable, it is marked as "unhealthy" and authentication requests are automatically routed to another available server.

### Timeouts

- `ldap.timeout_connection`: How long to wait when connecting to an LDAP server before timing out (seconds)
- `ldap.timeout_ldap_operation`: How long to wait for an LDAP operation before timing out (seconds)

---


## Conclusion

Integrating Valkey with LDAP or Active Directory brings your Valkey deployment in line with enterprise security and identity management practices. With flexible authentication modes, robust monitoring, and secure connection options, the valkey-ldap module makes it easy to manage access at scale.

For more details and advanced usage, check out the [valkey-ldap module documentation](https://github.com/valkey-io/valkey-ldap).

---