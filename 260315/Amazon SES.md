# Amazon SES(Simple Email Service)
## 1. User verification through E-mail
### 1) Architecture
```
┌──────────────────────────────────────────────────────────────┐
│             Email Verification Architecture                  │
│                                                              │
│  Client (browser / mobile)                                   │
│    │  POST /signup {email, name, password}                   │
│    ▼                                                         │
│  API Gateway                                                 │
│    │                                                         │
│    ▼                                                         │
│  SignUp Lambda                                               │
│    ├── Create user in DynamoDB (status: UNVERIFIED)          │
│    ├── Generate token → store in DynamoDB (TTL: 24h)         │
│    └── SES sends verification email                          │
│                │                                             │
│                ▼                                             │
│          User's Inbox                                        │
│                │  clicks verification link                   │
│                ▼                                             │
│  GET /verify?token=abc123                                    │
│    ▼                                                         │
│  API Gateway                                                 │
│    ▼                                                         │
│  Verify Lambda                                               │
│    ├── Look up token in DynamoDB                             │
│    ├── Update user status → VERIFIED                         │
│    ├── Delete used token                                     │
│    └── SES sends welcome email                               │
│                                                              │
│  DynamoDB Tables:                                            │
│    Users              → status: UNVERIFIED / VERIFIED        │
│    VerificationTokens → TTL auto-cleans expired tokens       │
│                                                              │
│  SES:                                                        │
│    Verified domain (yourapp.com) via Route 53                │
│    DKIM signed → lands in inbox not spam                     │
│    Bounce handling → protects sender reputation              │
└──────────────────────────────────────────────────────────────┘
```
### 2) Using your own Domain in SES
#### Why you see amazon.com in your eamil
- When you set up SES without custom domain configuration, your email looks like this
What user sees in Gmail:
```
  From: noreply@yourapp.com via amazonses.com
                              ↑
                         This appears because SES is
                         sending on behalf of your domain
                         but bounces go to amazonses.com

What the raw email headers show:
  From:          noreply@yourapp.com        ← your domain
  Return-Path:   0000014b8c87a59d-...       ← amazonses.com
                 @us-east-1.amazonses.com
  MAIL FROM:     bounce@amazonses.com       ← amazonses.com
```
#### Three things you need
```
1. From address        noreply@yourapp.com       ← already yours
2. Return-Path         mail.yourapp.com          ← needs custom MAIL FROM
3. DKIM signature      selector._domainkey.yourapp.com ← needs domain verification
```
## 2. Email Security in SES
![](ses_complete_setup_flow.svg)
### DKIM in SES
#### Reference
- DKIM in SES: https://docs.aws.amazon.com/ses/latest/dg/send-email-authentication-dkim.html
- Easy DKIM in Amazon SES: https://docs.aws.amazon.com/ses/latest/dg/send-email-authentication-dkim-easy.html
#### What it does?
- DKIM cryptographically signs every email you send — proving the email genuinely came from your domain and wasn't modified in transit.
```
SES sends email
      │
      ├── Signs email body + headers with PRIVATE key
      │   DKIM-Signature: v=1; a=rsa-sha256; d=yourapp.com;
      │                   b=abc123xyz...
      ▼
Gmail receives email
      │
      ├── Fetches PUBLIC key from Route 53 DNS
      │   selector1._domainkey.yourapp.com → TXT record
      │
      ├── Verifies signature matches
      │
      ├── Match    → email is genuine ✅ → inbox
      └── No match → email was tampered → spam/reject ❌
```
- Two options in SES
    + Easy DKIM
        * AWS generates and manages keys automatically
        * You just add the DNS records SES gives you
        * AWS rotates keys regularly for security
    + BYOD DKIM
        * You generate your own RSA key pair
  More control — enterprise requirement
        * You manage key rotation yourself
### SPF in SES
- SPF declares which mail servers are authorized to send email on behalf of your domain. It's a DNS record that receiving servers check.
```
Receiving server gets email from SES:
  "This email claims to be from yourapp.com"
  "Let me check who's allowed to send for yourapp.com"
        │
        ▼
DNS lookup: yourapp.com TXT record
  → "v=spf1 include:amazonses.com ~all"
  → Translation: "Only Amazon SES is authorized"
        │
        ├── SES IP? → Authorized ✅ → pass
        └── Unknown IP? → Not authorized ❌ → fail (~all = softfail)
```
### DMARC in SES
#### What It Does
- DMARC is the policy layer on top of DKIM and SPF.It tells receiving servers what to do when emails fail DKIM or SPF checks — and sends you reports about what's happening with your domain's email.
```
Receiving server runs DKIM + SPF checks
        │
        ├── Both pass  → deliver ✅
        │
        └── Either fails → check DMARC policy
                │
                ├── p=none       → deliver anyway, just report
                ├── p=quarantine → send to spam folder
                └── p=reject     → block completely
```
### Summary
|Feature|Answers|DNS Record|
|-------|-------|----------|
|DKIM|"Is this email genuine?"|`_domainkey` TXT|
|SPF|"Is this server authorized?"|Root Domain TXT|
|DMARC|"What to do when checks fail?"|`_dmarc`TXT|
|BIMI|"Show my logo in Inbox"|`_bimi`TXT|
|TLS|"Is transit encrypted?"|No DNS Record needed|