# 🔒 Security Documentation - 403 Classroom Seat Selection System

## Table of Contents
1. [Security Overview](#security-overview)
2. [Authentication & Access Control](#authentication--access-control)
3. [Data Protection](#data-protection)
4. [Firebase Security Rules](#firebase-security-rules)
5. [Deployment Security Checklist](#deployment-security-checklist)
6. [Incident Response](#incident-response)
7. [Best Practices](#best-practices)

---

## Security Overview

This document outlines all security measures implemented in the 403 Classroom Seat Selection System and provides guidance for secure deployment and operation.

### Current Security Level: **Enhanced** ⭐⭐⭐⭐

| Component | Status | Notes |
|-----------|--------|-------|
| XSS Protection | ✅ Hardened | HTML escaping on all user inputs |
| CSRF Protection | ✅ Implemented | Token-based CSRF prevention |
| Authentication | ✅ Enhanced | Rate limiting, password strength validation |
| Data Encryption | ⚠️ Partial | Client-side session encryption only |
| Backend Validation | ❌ Missing | **Recommended for production** |
| HTTPS | ✅ Required | Firebase uses HTTPS by default |
| API Security | ⚠️ Partial | Firebase Rules configuration needed |

---

## Authentication & Access Control

### Password Requirements

**Minimum complexity:**
- ✅ Minimum 8 characters
- ✅ At least 1 uppercase letter (A-Z)
- ✅ At least 1 lowercase letter (a-z)
- ✅ At least 1 number (0-9)

**Example strong passwords:**
- ✅ Admin2024Seat
- ✅ Classroom123Safe
- ❌ admin (too weak)
- ❌ 12345678 (no letters)

### Login Security

**Features Implemented:**
- **Rate Limiting**: Maximum 5 login attempts per 15 minutes
- **Session Timeout**: Sessions cleared on logout
- **Account Whitelist**: Only whitelisted admin accounts can access the system
- **Failed Login Tracking**: Excessive attempts display warning

**Code Reference:**
```javascript
const loginLimiter = new LoginRateLimiter(5, 15 * 60 * 1000);
// Max 5 attempts within 15 minutes
```

### Admin Account Whitelist

**How to set up:**
1. Go to Admin Backend → "雲端安全設定"
2. Enter admin usernames separated by commas: `admin01, admin02, admin03`
3. Click "儲存名單"

**Benefits:**
- Restricts admin access to authorized personnel only
- Prevents unauthorized admin account creation
- Centralized access control

---

## Data Protection

### XSS (Cross-Site Scripting) Prevention

**All user inputs are escaped using:**
```javascript
function escapeHTML(str) {
    if (!str || typeof str !== 'string') return "";
    const div = document.createElement('div');
    div.textContent = str;
    return div.innerHTML;
}
```

**Protected areas:**
- ✅ Student names
- ✅ Class codes
- ✅ URL parameters
- ✅ Admin input fields
- ✅ Display output

### CSRF (Cross-Site Request Forgery) Protection

**Token-based prevention:**
```javascript
let csrfToken = generateCSRFToken();
// Tokens invalidated on logout
```

### Data Storage

| Data | Storage Method | Encryption | Accessibility |
|------|---|---|---|
| Admin Auth Token | sessionStorage | Base64 encoded | Current session only |
| Student Names | Firebase Realtime DB | Server-side (recommend) | Public (name masked for others) |
| Password | Firebase Realtime DB | Server-side (must be) | Server-side only |
| Class Roster | localStorage | None (consider hashing) | Local device only |

### Name Masking

Student names are automatically masked to protect privacy:
- Your own seat: **Full name displayed**
- Other students' seats: **Name[0] + Ｏ + Name[3:]** (e.g., "李Ｏ明")
- Admin backend: **Full names visible** (for verification)

---

## Firebase Security Rules

### ⚠️ CRITICAL: Current Firebase Security Status

**Current State: VULNERABLE** 
The default Firebase configuration allows public read/write access. This must be secured immediately.

### Recommended Firebase Security Rules

Copy and paste these rules into your Firebase Console:

**Path:** Firebase Console → Realtime Database → Rules

```json
{
  "rules": {
    "room403": {
      "$classId": {
        "title": {
          ".read": true,
          ".write": false
        },
        "roster": {
          ".read": true,
          ".write": false
        },
        "seats": {
          ".read": true,
          ".write": "root.child('settings').child('authorized_ips').val().contains(root.getRoot().now())",
          "$seatId": {
            ".validate": "newData.isString() && newData.val().length > 0 && newData.val().length <= 20"
          }
        }
      }
    },
    "settings": {
      "pwd_403": {
        ".read": false,
        ".write": false
      },
      "accounts_403": {
        ".read": false,
        ".write": false
      }
    }
  }
}
```

### How to Deploy Firebase Rules

1. **Go to Firebase Console**
   - Navigate to: `https://console.firebase.google.com/`
   - Select your project
   - Go to Realtime Database → Rules

2. **Replace default rules** with the recommended rules above

3. **Click "Publish"**

4. **Test the rules** by verifying:
   - Students can read class info ✅
   - Students can write to seats ✅
   - Students cannot read/modify passwords ✅
   - Unauthorized users cannot access admin settings ✅

### Alternative: Backend Authentication

**For enhanced security, implement server-side authentication:**

1. Create a backend endpoint for seat updates
2. Verify admin token before allowing database writes
3. Log all admin actions
4. Implement JWT tokens
5. Add request signing with HMAC

**Recommended backend stack:**
- Node.js + Express
- Firebase Admin SDK
- JWT tokens
- Request logging

---

## Deployment Security Checklist

### Pre-Deployment

- [ ] Change default admin password (must meet strength requirements)
- [ ] Set admin account whitelist with authorized personnel only
- [ ] Review and deploy Firebase Security Rules
- [ ] Enable HTTPS (use custom domain with SSL certificate)
- [ ] Set strong Content Security Policy headers
- [ ] Remove/disable debug logging in production
- [ ] Test XSS protection with sample malicious input: `<script>alert('XSS')</script>`
- [ ] Verify rate limiting works: attempt login 6 times rapidly
- [ ] Review all admin accounts have strong passwords
- [ ] Backup current database configuration

### Deployment Environment

```bash
# Server Security Headers (example for nginx)
add_header X-Frame-Options "DENY";
add_header X-Content-Type-Options "nosniff";
add_header X-XSS-Protection "1; mode=block";
add_header Referrer-Policy "strict-origin-when-cross-origin";
add_header Content-Security-Policy "default-src 'self'; script-src 'self' https://cdnjs.cloudflare.com; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' https:; connect-src 'self' https://*.firebasedatabase.app; frame-ancestors 'none';";
```

### Post-Deployment

- [ ] Monitor Firebase database for unusual activity
- [ ] Check admin login logs regularly
- [ ] Review password change history
- [ ] Test disaster recovery procedures
- [ ] Monitor for XSS/injection attempts in logs
- [ ] Set up alerts for failed login attempts
- [ ] Perform monthly security reviews

---

## Incident Response

### If You Suspect a Security Breach:

#### 1. **Immediate Actions (Minutes 0-5)**
```javascript
// Logout all sessions
sessionStorage.removeItem("isAdminAuthed_403");
sessionStorage.removeItem("adminAuthToken_403");

// Change admin password immediately
// Go to Admin Panel → "雲端安全設定" → "變更密碼"
```

#### 2. **Investigation (Minutes 5-30)**
- [ ] Check browser console for errors: `F12 → Console`
- [ ] Review network requests: `F12 → Network → XHR`
- [ ] Check Firebase console for unexpected data modifications
- [ ] Review admin login attempts in browser history
- [ ] Look for suspicious class codes or rosters

#### 3. **Recovery (Minutes 30+)**
- [ ] Delete compromised class periods: "徹底清除當前班期"
- [ ] Update all admin passwords
- [ ] Update admin account whitelist
- [ ] Review and restore from backup if needed
- [ ] Clear all browser caches and cookies
- [ ] Notify all administrators of the incident

### Potential Attack Vectors & Prevention

| Attack Vector | Risk | Prevention |
|---|---|---|
| **XSS via Student Name** | HIGH | Input escaping + validation |
| **Weak Password** | HIGH | Enforce complexity requirements |
| **Brute Force Login** | MEDIUM | Rate limiting (5 attempts/15 min) |
| **Session Hijacking** | MEDIUM | Use sessionStorage + HTTPS |
| **CSRF Attack** | LOW | CSRF token validation |
| **Firebase Unauthorized Access** | CRITICAL | Implement Security Rules |
| **Man-in-the-Middle (MITM)** | MEDIUM | Use HTTPS only |
| **Malicious QR Code** | LOW | QR codes generated dynamically |

---

## Best Practices

### For Administrators

✅ **DO:**
- Use a strong password (8+ chars, uppercase, lowercase, numbers)
- Change password every 30 days
- Log out when leaving the computer
- Only use on secure networks (avoid public WiFi)
- Verify student names carefully during check-in
- Keep device software updated
- Use browser-based password manager (e.g., Bitwarden, LastPass)
- Document admin access in case of audit

❌ **DON'T:**
- Share admin password via email or chat
- Use the same password for multiple systems
- Leave admin session open unattended
- Access from public computers
- Disable browser security features
- Click suspicious links
- Use outdated browsers
- Store passwords in plain text files

### For System Administrators

**Setup Recommendations:**
1. Implement centralized authentication (OAuth2/SAML)
2. Set up monitoring alerts for failed logins
3. Enable audit logging for all admin actions
4. Use VPN for admin access
5. Implement IP whitelisting for admin accounts
6. Regular security audits (quarterly)
7. Penetration testing annually

**Backup Strategy:**
```javascript
// Export seats regularly
// Location: Admin Panel → "🖨️ 匯出當前座位表（存檔）"
// Recommended: Daily backups during active class periods
```

### For Database Administrators

**Firebase Configuration:**
```json
{
  "database": {
    "rules": "see-above",
    "backup": "enabled",
    "monitoring": "enabled",
    "logging": "enabled"
  }
}
```

**Monitoring:**
- Enable Firebase alerts
- Set up usage alerts
- Monitor for unusual data patterns
- Review access logs weekly

---

## Technical Implementation Details

### Security Functions Reference

#### 1. **escapeHTML(str)**
- **Purpose:** Prevent XSS attacks
- **Usage:** `escapeHTML(userInput)`
- **Output:** HTML-safe string

#### 2. **validatePasswordStrength(pwd)**
- **Purpose:** Enforce password complexity
- **Returns:** `true` if password meets requirements
- **Requirements:** 8+ chars, uppercase, lowercase, digits

#### 3. **generateCSRFToken()**
- **Purpose:** Generate unique CSRF tokens
- **Returns:** UUID-formatted token
- **Usage:** Automatically called on page load

#### 4. **LoginRateLimiter**
- **Purpose:** Prevent brute force attacks
- **Max Attempts:** 5 per 15 minutes
- **Warning:** Displayed after 5 failed attempts

#### 5. **secureStore() / secureRetrieve()**
- **Purpose:** Encrypt sensitive session data
- **Method:** Base64 encoding (upgrade to AES for production)
- **Storage:** sessionStorage (cleared on browser close)

---

## Compliance & Certifications

### GDPR Compliance

The system handles student personal data (names). Ensure:
- ✅ Students consent to name storage
- ✅ Data deletion on class completion
- ✅ Privacy policy clearly displayed
- ✅ Right to access student data implemented
- ✅ Data breach notification procedures established

### Recommended Actions:
1. Add privacy notice to the system
2. Implement data retention policy (auto-delete after 90 days)
3. Create data export functionality for compliance
4. Document data processing activities

---

## 403 Classroom Specific Configuration

### Seat Layout
- **Total Seats:** 36 (9 rows × 4 columns)
- **Special Seats:** 9-1 = Report/Volunteer desk
- **Door Position:** Left side (前門/後門)

### Firebase Paths
- **Data Path:** `/room403/{classId}/seats/`
- **Metadata Path:** `/room403/{classId}/title`
- **Roster Path:** `/room403/{classId}/roster`
- **Settings Path:** `/settings/pwd_403` and `/settings/accounts_403`

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-06-19 | Initial security hardening for 403 |
| 1.1 | TBD | Add backend authentication |
| 1.2 | TBD | Implement GDPR compliance |
| 2.0 | TBD | Full OAuth2 integration |

---

## Support & Questions

For security-related questions or to report vulnerabilities:

1. **Report Security Issues**: Create a private security advisory
2. **Ask Questions**: Open an issue with `[SECURITY]` prefix
3. **Suggest Improvements**: Submit a pull request with security enhancements

---

## Security Disclaimer

⚠️ **Important:**
- This application is provided "as-is"
- Security depends on proper configuration and deployment
- Client-side security has limitations; backend authentication is strongly recommended for production
- Regularly update all dependencies and security patches
- No warranty for data breaches or unauthorized access

For production deployment, consider hiring a security consultant for a full penetration test.

---

**Last Updated:** 2026-06-19  
**Maintained By:** Security Team  
**Review Schedule:** Quarterly
