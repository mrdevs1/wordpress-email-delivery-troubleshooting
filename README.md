# WordPress Email Delivery Troubleshooting

A practical guide for diagnosing and fixing WordPress email delivery problems, including SMTP configuration, `wp_mail()`, PHP mail issues, DNS authentication, spam delivery, WooCommerce notifications, plugin conflicts, and server-level email problems.

---

## Overview

WordPress websites depend on email for many important functions:

* Password reset emails
* User registration notifications
* Contact form messages
* Administrator notifications
* WooCommerce order emails
* Customer account emails
* Payment notifications
* Security alerts
* Plugin-generated notifications

An email problem can happen at several different stages:

```text
WordPress
   ↓
wp_mail()
   ↓
PHP / SMTP
   ↓
Mail Server / SMTP Provider
   ↓
DNS Authentication
   ↓
Recipient Mail Server
   ↓
Inbox / Spam
```

The fact that WordPress reports an email as successfully processed does not necessarily mean that the recipient received it. `wp_mail()` returning `true` only indicates that the mail-sending process accepted the request.

---

## Common Email Problems

Typical symptoms include:

| Problem                                   | Possible Cause                            |
| ----------------------------------------- | ----------------------------------------- |
| WordPress sends no emails                 | `wp_mail()` or server mail problem        |
| Password reset email missing              | Mail delivery or spam filtering           |
| Contact form email missing                | SMTP/plugin configuration                 |
| WooCommerce order email missing           | Order status, email settings, or delivery |
| Emails arrive in spam                     | Authentication/reputation/content         |
| Gmail receives but Outlook does not       | Recipient filtering/reputation            |
| SMTP authentication fails                 | Incorrect credentials/settings            |
| Connection timeout                        | Firewall/network/SMTP port                |
| Sender address rejected                   | Invalid or unauthorized sender            |
| Emails delayed                            | Queue, provider, or recipient filtering   |
| Emails sent from wrong address            | Plugin/theme/custom code                  |
| Test email works but WooCommerce does not | WooCommerce configuration or conflict     |

---

## 1. Identify the Exact Failure

Before changing anything, determine what is actually happening.

Ask:

1. Is WordPress generating the email?
2. Is `wp_mail()` being called?
3. Is SMTP configured?
4. Does the SMTP connection succeed?
5. Does the SMTP provider accept the message?
6. Is the recipient server accepting it?
7. Did the message reach spam/junk?
8. Is only one type of email affected?
9. Are all recipients affected?
10. Did the problem begin after a plugin, theme, DNS, or hosting change?

### Basic diagnostic classification

```text
No email generated
        ↓
WordPress / Plugin / Theme problem

Email generated but sending fails
        ↓
SMTP / PHP / Server problem

SMTP accepts email
        ↓
Check provider logs and recipient delivery

Email reaches recipient but goes to spam
        ↓
Authentication / reputation / content problem
```

---

## 2. Understand `wp_mail()`

WordPress uses the `wp_mail()` function to send email.

Basic example:

```php
wp_mail(
    'customer@example.com',
    'Test Email',
    'This is a test message.'
);
```

A successful return value does not prove that the recipient received the message.

It generally means the mail-sending process accepted the request.

---

## 3. Test WordPress Email

Create a controlled test instead of immediately troubleshooting WooCommerce or a contact-form plugin.

For example, temporarily test:

```php
wp_mail(
    'your-test@example.com',
    'WordPress Email Test',
    'This is a WordPress email delivery test.'
);
```

### Important

Do not leave temporary test code inside production files.

After testing:

1. Remove the test code.
2. Clear relevant caches.
3. Test again using the normal application workflow.

---

## 4. Check SMTP Configuration

SMTP is generally more reliable for transactional WordPress email than relying on the hosting server's default mail function.

Check:

```text
SMTP Host
SMTP Port
Encryption
Authentication
Username
Password/API credential
From Address
Reply-To Address
```

Common encryption and port combinations include:

```text
STARTTLS → 587
SSL/TLS  → 465
```

The exact settings depend on the SMTP provider.

### Common SMTP problems

* Wrong SMTP hostname
* Wrong port
* Incorrect username
* Incorrect password
* Expired credentials
* Incorrect encryption
* SMTP authentication disabled
* Firewall blocking outbound connection
* Provider rejecting the sender
* DNS problems
* Provider account restrictions

---

## 5. SMTP Authentication Errors

Common errors may look like:

```text
Authentication failed
535 Authentication failed
Username or password rejected
Authentication unsuccessful
```

Check:

### Username

Make sure the complete email address is being used if the provider requires it.

Example:

```text
support@example.com
```

### Password/API Key

Verify:

* Credential is correct
* Credential has not expired
* Credential has required permissions
* Old credentials were not recently revoked

### Encryption

Make sure the SMTP encryption matches the provider's requirements.

---

## 6. Check DNS

Email delivery depends heavily on DNS.

Important records include:

```text
MX
TXT
SPF
DKIM
DMARC
```

### MX

MX records tell other mail servers where email for a domain should be delivered.

Check:

```bash
dig MX example.com
```

or:

```bash
nslookup -type=MX example.com
```

---

## 7. SPF Troubleshooting

SPF identifies which servers and services are authorized to send email for a domain.

Example structure:

```text
example.com TXT
v=spf1 include:provider.example -all
```

Do not blindly copy this example into production.

Your SPF record must reflect the actual services authorized to send mail.

### Common SPF problems

* Multiple SPF records
* Missing sending provider
* Incorrect IP address
* Invalid `include`
* Too many DNS lookups
* Old mail service still listed
* SPF record syntax errors

### Check SPF

```bash
dig TXT example.com
```

Look for:

```text
v=spf1
```

---

## 8. DKIM Troubleshooting

DKIM adds a cryptographic signature to outgoing email.

A typical setup involves:

```text
selector._domainkey.example.com
```

The exact selector depends on the mail provider.

### Check DKIM

```bash
dig TXT selector._domainkey.example.com
```

Verify that:

* DKIM is enabled
* DNS record exists
* Correct selector is being used
* Public key matches the provider configuration
* DNS changes have propagated

---

## 9. DMARC Troubleshooting

DMARC helps receiving mail servers evaluate whether messages align with the domain's authentication policies.

A basic example:

```text
_dmarc.example.com TXT
v=DMARC1; p=none
```

A monitoring policy can be useful when initially deploying DMARC.

Do not change DMARC to a strict enforcement policy without understanding your legitimate sending sources.

### Check DMARC

```bash
dig TXT _dmarc.example.com
```

Look for:

```text
v=DMARC1
```

---

## 10. Avoid Multiple SPF Records

A common DNS mistake is creating multiple TXT records containing SPF.

Bad configuration:

```text
v=spf1 include:provider-a.example ~all
v=spf1 include:provider-b.example ~all
```

A domain should have a single SPF policy.

If multiple services send email, combine their authorized mechanisms into one valid SPF record.

---

## 11. Check the From Address

Use a sender address associated with your own domain.

Example:

```text
From: WordPress <wordpress@example.com>
```

Avoid using unrelated public addresses as the sender when your website is actually sending through another mail system.

For WooCommerce, the sender information can be configured under:

```text
WooCommerce
→ Settings
→ Emails
```

---

## 12. Check Reply-To

The `From` address and `Reply-To` address serve different purposes.

Example:

```text
From: store@example.com
Reply-To: customer@example.com
```

This can allow staff to reply to a customer while keeping the authorized sender address consistent.

Be careful when custom code or plugins modify these headers.

---

## 13. WooCommerce Email Troubleshooting

WooCommerce transactional emails can fail for different reasons.

Check:

```text
WooCommerce
→ Settings
→ Emails
```

Verify that the relevant notification is enabled.

WooCommerce provides separate settings for individual email notifications, recipients, sender identity, templates, and other options.

---

### 13.1 Check Order Status

An order may not trigger an expected email if the required order state was never reached.

For example:

```text
Pending Payment
        ↓
Payment Completed
        ↓
Processing
        ↓
Completed
```

If a payment gateway fails to update the order correctly, the expected email may not be generated.

---

## 14. Check WooCommerce Email Logs

For supported current WooCommerce versions, inspect:

```text
WooCommerce
→ Status
→ Logs
```

Look for the transactional email log.

Possible outcomes include:

```text
Sent
Failed
Disabled
Skipped
```

These results help determine whether the problem occurred before or during mail delivery.

---

## 15. WooCommerce Plugin/Theme Conflicts

A plugin or theme can interfere with:

* Order status updates
* Email triggers
* Email headers
* Recipients
* Templates
* Checkout processing
* Payment callbacks

### Safe conflict-testing workflow

Create a backup first.

Then, if possible:

```text
Keep:
WooCommerce

Temporarily disable:
Other plugins

Use:
Default theme
```

Test again.

If the problem disappears:

```text
Enable plugins one by one
        ↓
Test after each change
        ↓
Identify the conflicting component
```

---

## 16. WordPress Password Reset Emails

If password reset messages are missing, test:

```text
Lost Password
        ↓
WordPress generates email
        ↓
wp_mail()
        ↓
SMTP / Mail Server
        ↓
Recipient
```

Check:

* SMTP configuration
* WordPress site URL
* Admin email
* Sender address
* SMTP logs
* Spam folder
* Mail provider logs

---

## 17. Contact Form Email Problems

If a contact form submits successfully but no email arrives:

Check separately:

### Form submission

```text
Form submitted?
```

### WordPress email

```text
wp_mail() called?
```

### SMTP

```text
SMTP accepted message?
```

### Delivery

```text
Recipient received message?
```

This prevents confusing a form-processing problem with an email-delivery problem.

---

## 18. Gmail / Outlook Delivery Problems

If email reaches some providers but not others:

Compare:

```text
Gmail
Outlook
Yahoo
Business email
```

Check:

* SPF
* DKIM
* DMARC
* Sender domain
* SMTP provider
* Bounce messages
* Provider reputation
* Spam/junk folders
* Message headers

Do not assume that successful delivery to one provider proves universal deliverability.

---

## 19. Check Spam and Junk Folders

Always check:

```text
Inbox
Spam
Junk
Promotions
Other
Quarantine
```

For business mail systems, administrators may also have quarantine or filtering systems.

If a message is in spam, inspect the complete message headers when possible.

Look for authentication results such as:

```text
SPF: PASS
DKIM: PASS
DMARC: PASS
```

---

## 20. Inspect Email Headers

Email headers can reveal where a message traveled and how receiving servers evaluated it.

Look for:

```text
Received:
Return-Path:
From:
Reply-To:
Authentication-Results:
DKIM-Signature:
Message-ID:
```

Example:

```text
Authentication-Results:
spf=pass
dkim=pass
dmarc=pass
```

Do not publish private email headers containing sensitive information in public GitHub issues.

Redact:

* Email addresses
* IP addresses where appropriate
* Message IDs
* Internal hostnames
* Authentication tokens
* Private server information

---

## 21. PHP and Server-Level Checks

On self-managed hosting, check the server configuration.

Possible areas:

```text
PHP configuration
MTA configuration
Firewall
Outbound SMTP restrictions
DNS resolver
Server hostname
Reverse DNS
Mail queue
System logs
```

Depending on the server environment, common MTAs include:

```text
Exim
Postfix
```

Do not change production mail configuration without understanding how existing websites depend on it.

---

## 22. SMTP Port Connectivity

If SMTP connections fail, test network connectivity where appropriate.

Example:

```bash
nc -vz smtp.example.com 587
```

or:

```bash
nc -vz smtp.example.com 465
```

A failed connection may indicate:

* Firewall restriction
* Incorrect hostname
* Incorrect port
* Provider restriction
* Network routing problem

A successful TCP connection does not guarantee successful SMTP authentication.

---

## 23. Check Mail Logs

For self-managed servers, mail logs can be extremely useful.

Depending on your mail server, logs may be located somewhere such as:

```text
/var/log/maillog
```

or:

```text
/var/log/mail.log
```

Check the actual logging configuration of the server before assuming a specific path.

Search for:

```text
SMTP
authentication
rejected
deferred
bounced
timeout
connection
recipient
```

---

## 24. Check the Mail Queue

On servers that manage their own mail queue, a growing queue may indicate delivery problems.

Check:

```text
Queued
Deferred
Rejected
Bounced
```

A queue problem can be caused by:

* Recipient server errors
* DNS problems
* Authentication issues
* Network failures
* Provider throttling
* Incorrect routing

Do not blindly delete queued mail without understanding what the messages are and why they are queued.

---

## 25. Check Security Software

Server security tools can interfere with mail connections.

Check:

```text
Firewall
CSF
cPHulk
Imunify
ModSecurity
Hosting security policies
SMTP restrictions
```

Verify whether outbound connections are being blocked.

---

## 26. Test From the WordPress Environment

A useful diagnostic sequence is:

```text
WordPress test
      ↓
SMTP test
      ↓
Provider log
      ↓
Recipient mailbox
      ↓
Message headers
```

Do not change multiple variables at the same time.

Change one thing, test, and record the result.

---

## 27. Plugin Conflict Testing

If email stopped working after installing or updating a plugin:

1. Record current configuration.
2. Create a backup.
3. Test the email.
4. Temporarily disable suspected plugins.
5. Test again.
6. Switch to a default theme if necessary.
7. Compare results.
8. Re-enable components one at a time.

Avoid making unnecessary changes on production sites.

---

## 28. WordPress Debug Logging

When appropriate, enable WordPress debugging temporarily.

Example:

```php
define( 'WP_DEBUG', true );
define( 'WP_DEBUG_LOG', true );
define( 'WP_DEBUG_DISPLAY', false );
```

The log is normally written to:

```text
wp-content/debug.log
```

After troubleshooting, review whether debugging should remain enabled.

Never expose PHP errors directly to visitors on a production website.

---

## 29. Check Application Logs

Look for:

```text
PHP fatal errors
PHP warnings
Plugin errors
Theme errors
SMTP errors
API errors
Payment gateway errors
WooCommerce errors
```

An email problem may actually be caused by an application error occurring before the email is generated.

---

## 30. Check Recent Changes

Ask:

```text
What changed immediately before the problem?
```

Possible changes:

* WordPress update
* WooCommerce update
* Plugin update
* Theme update
* PHP version change
* DNS change
* Hosting migration
* SMTP provider change
* Password/API key change
* Firewall change
* Cloudflare/DNS change
* Email provider change

Recent changes are often the fastest path to the root cause.

---

## 31. Delivery vs. Sending

These are different problems.

### Sending problem

```text
WordPress
   ↓
SMTP
   X
```

The message never successfully leaves the website or mail system.

### Delivery problem

```text
WordPress
   ↓
SMTP
   ↓
Recipient Server
   ↓
Spam / Reject / Delay
```

The sending process works, but the recipient does not place the message in the expected inbox.

Always identify which stage is failing.

---

## 32. Common Mistakes

Avoid these troubleshooting mistakes:

### Changing DNS randomly

DNS changes should be based on a specific diagnosis.

### Adding multiple SPF records

Use one SPF policy per domain.

### Changing DMARC aggressively

Start with a controlled policy and understand legitimate senders first.

### Changing SMTP ports randomly

Use the provider's documented configuration.

### Assuming `wp_mail()` success means delivery

It does not.

### Disabling every plugin immediately

Use controlled conflict testing.

### Deleting mail queues without investigation

Queued messages may contain useful diagnostic information.

### Changing production mail configuration without backup

Record the current configuration first.

---

## 33. Safe Troubleshooting Workflow

Use this sequence:

```text
1. Reproduce the problem
        ↓
2. Identify which email is affected
        ↓
3. Check WordPress/WooCommerce settings
        ↓
4. Test wp_mail()
        ↓
5. Check SMTP configuration
        ↓
6. Check SMTP/provider logs
        ↓
7. Check DNS
        ↓
8. Check SPF/DKIM/DMARC
        ↓
9. Check recipient spam/junk
        ↓
10. Inspect message headers
        ↓
11. Check server logs
        ↓
12. Test plugin/theme conflicts
        ↓
13. Apply the smallest safe fix
        ↓
14. Retest
        ↓
15. Document the result
```

---

## 34. Troubleshooting Checklist

### WordPress

* [ ] WordPress is functioning normally
* [ ] `wp_mail()` has been tested
* [ ] No fatal PHP errors
* [ ] SMTP configuration checked
* [ ] Sender address checked
* [ ] Reply-To checked
* [ ] Recent plugin changes reviewed

### WooCommerce

* [ ] Correct order status
* [ ] Email notification enabled
* [ ] Recipient configured
* [ ] WooCommerce logs checked
* [ ] Payment gateway status checked
* [ ] Plugin/theme conflict tested
* [ ] Test email sent

### SMTP

* [ ] SMTP hostname correct
* [ ] Port correct
* [ ] Encryption correct
* [ ] Username correct
* [ ] Credential valid
* [ ] Provider accepts connection
* [ ] Provider accepts sender

### DNS

* [ ] MX checked
* [ ] SPF checked
* [ ] DKIM checked
* [ ] DMARC checked
* [ ] No duplicate SPF
* [ ] Sending services authorized

### Server

* [ ] Firewall checked
* [ ] SMTP connectivity tested
* [ ] Mail logs checked
* [ ] Mail queue checked
* [ ] PHP errors checked
* [ ] DNS resolution checked

### Recipient

* [ ] Inbox checked
* [ ] Spam checked
* [ ] Junk checked
* [ ] Quarantine checked
* [ ] Headers inspected
* [ ] Bounce message reviewed

---

## 35. Quick Diagnostic Table

| Symptom                               | First Things to Check                      |
| ------------------------------------- | ------------------------------------------ |
| No WordPress emails                   | `wp_mail()`, SMTP                          |
| No password reset                     | SMTP, spam, PHP errors                     |
| No contact-form emails                | Form plugin + SMTP                         |
| No WooCommerce emails                 | Order status + WooCommerce email settings  |
| SMTP authentication error             | Credentials + encryption                   |
| SMTP timeout                          | Host + port + firewall                     |
| Email in spam                         | SPF + DKIM + DMARC + reputation            |
| Gmail works, Outlook fails            | Headers + recipient filtering              |
| All emails suddenly stopped           | Recent changes + SMTP/provider             |
| Emails delayed                        | Provider logs + queue                      |
| Wrong sender                          | WooCommerce/plugin/WordPress configuration |
| Only one plugin affected              | Plugin configuration/conflict              |
| WooCommerce email skipped             | Order/email conditions                     |
| WooCommerce email failed              | Logs + mail system                         |
| SMTP works but recipient gets nothing | Provider logs + recipient filtering        |

---

## 36. Document the Investigation

For professional troubleshooting, record:

```text
Website:
Date:
WordPress Version:
WooCommerce Version:
PHP Version:
Hosting Environment:
SMTP Provider:
Affected Email:
Sender Address:
Recipient Provider:
Observed Error:
Relevant Log:
Changes Made:
Test Result:
Final Resolution:
```

This makes future troubleshooting much faster.

---

## 37. Security and Privacy

Email troubleshooting can expose sensitive information.

Never publish:

```text
SMTP passwords
API keys
Authentication tokens
Private email contents
Customer personal information
Payment information
Full private mail headers
```

Before sharing logs publicly, redact sensitive information.

Example:

```text
username@example.com
```

can be replaced with:

```text
user@example.com
```

---

## 38. Prevention Best Practices

For reliable WordPress email delivery:

* Use a properly configured SMTP service.
* Use a domain-based sender address.
* Configure SPF correctly.
* Configure DKIM.
* Publish an appropriate DMARC policy.
* Monitor bounce messages.
* Monitor SMTP provider logs.
* Keep WordPress and plugins updated.
* Test WooCommerce transactional emails after major changes.
* Keep backups before server configuration changes.
* Document mail infrastructure.
* Avoid unnecessary DNS modifications.
* Monitor domain and sending reputation.

---

## 39. Official References

### WordPress

* `wp_mail()` Developer Reference:
  [https://developer.wordpress.org/reference/functions/wp_mail/](https://developer.wordpress.org/reference/functions/wp_mail/)

### WooCommerce

* Email Troubleshooting:
  [https://woocommerce.com/document/email-faq/](https://woocommerce.com/document/email-faq/)

* Email Settings:
  [https://woocommerce.com/document/configuring-woocommerce-settings/emails/](https://woocommerce.com/document/configuring-woocommerce-settings/emails/)

* WooCommerce Troubleshooting:
  [https://woocommerce.com/documentation/woocommerce/get-help/troubleshooting-get-help/](https://woocommerce.com/documentation/woocommerce/get-help/troubleshooting-get-help/)

---

## 40. Final Diagnostic Flow

Use this flow when investigating a real production issue:

```text
                    EMAIL PROBLEM
                          │
                          ▼
              Is the email generated?
                    │           │
                   NO          YES
                    │           │
                    ▼           ▼
          Check WordPress   Check SMTP
          plugin/theme      / mail system
                    │           │
                    └─────┬─────┘
                          ▼
                  Was it accepted?
                    │           │
                   NO          YES
                    │           │
                    ▼           ▼
              Fix SMTP /     Check provider
              server issue   delivery logs
                                │
                                ▼
                       Recipient received?
                          │           │
                         NO          YES
                          │           │
                          ▼           ▼
                    Check spam,    Problem
                    headers, DNS   resolved
                    and filtering
```

---

## Important Notes

This repository is intended as a troubleshooting reference for WordPress administrators, developers, hosting providers, and support teams.

Email delivery depends on multiple systems. A WordPress website can successfully generate a message while the recipient server later rejects, delays, filters, or places that message in spam.

Always identify the failing stage before changing configuration.

For production websites:

1. Take a backup before major changes.
2. Record the existing configuration.
3. Make one controlled change at a time.
4. Test after each change.
5. Keep a record of the final configuration.

---

## Contributing

Contributions are welcome.

If you find an additional troubleshooting scenario:

1. Open an issue.
2. Explain the symptoms.
3. Include the relevant error message.
4. Describe the environment.
5. Remove sensitive information.
6. Explain the confirmed solution when available.

---

## License

This documentation is provided for educational and troubleshooting purposes.

Use the procedures carefully and adapt them to your own hosting environment, mail provider, and WordPress configuration.
