# Cloudflare Security Patterns

Security integration patterns for building secure applications with Cloudflare Workers and security products.

## Table of Contents

1. [WAF + Workers Integration](#waf--workers-integration)
2. [Turnstile Integration](#turnstile-integration)
3. [Rate Limiting Patterns](#rate-limiting-patterns)
4. [Secrets Management](#secrets-management)
5. [API Security](#api-security)
6. [Input Validation](#input-validation)
7. [Security Headers](#security-headers)

---

## WAF + Workers Integration

### Overview

The Web Application Firewall (WAF) and Workers work together to provide layered security:
- **WAF**: Runs before Workers, blocks malicious requests at the edge
- **Workers**: Implement business logic with custom security rules

### Architecture Pattern

```
User Request
    ↓
WAF (Cloudflare Rules Engine)
    ├─→ Block (malicious)
    └─→ Allow
        ↓
    Workers (Custom Logic)
        ├─→ Additional validation
        ├─→ Rate limiting
        └─→ Business logic
            ↓
        Origin/Response
```

### When to Use WAF vs Workers

**Use WAF for:**
- Known attack patterns (SQL injection, XSS)
- IP-based blocking
- Geo-blocking
- Basic rate limiting
- Pre-configured rule sets

**Use Workers for:**
- Custom business logic validation
- Dynamic rate limiting based on user behavior
- Complex authentication flows
- API-specific protection
- Content transformation

### Common Integration Pattern

**WAF Configuration (via Dashboard or API)**:
```
# Example: Block common attacks, allow legitimate traffic
Rule 1: Block SQL injection patterns
Rule 2: Block XSS patterns
Rule 3: Rate limit /api/* to 100 req/min per IP
Rule 4: Allow all other traffic → Pass to Workers
```

**Workers Code**:
```typescript
// Worker handles allowed traffic with additional validation
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // WAF has already filtered malicious requests

    // Add custom validation
    if (!isValidRequest(request)) {
      return new Response('Invalid request', { status: 400 });
    }

    // Business logic
    return handleRequest(request, env);
  }
}
```

### Best Practices

1. **Layer security**: WAF for broad protection, Workers for specific rules
2. **Don't duplicate**: If WAF handles it well, don't re-implement in Workers
3. **Log and monitor**: Use Workers to log security events for analysis
4. **Fail securely**: Default to deny if validation fails

---

## Turnstile Integration

### Overview

Turnstile is Cloudflare's CAPTCHA alternative that provides bot protection without user friction.

**Use for:**
- Form submissions
- Login pages
- Account creation
- API endpoints that need bot protection

### Complete Integration Example

#### Frontend (HTML + JavaScript)

```html
<!DOCTYPE html>
<html>
<head>
  <title>Contact Form</title>
  <!-- Load Turnstile script -->
  <script src="https://challenges.cloudflare.com/turnstile/v0/api.js" async defer></script>
</head>
<body>
  <form id="contact-form">
    <input type="text" name="name" required />
    <input type="email" name="email" required />
    <textarea name="message" required></textarea>

    <!-- Turnstile widget -->
    <div class="cf-turnstile"
         data-sitekey="YOUR_SITE_KEY"
         data-callback="onTurnstileSuccess"></div>

    <button type="submit">Submit</button>
  </form>

  <script>
    let turnstileToken = null;

    function onTurnstileSuccess(token) {
      turnstileToken = token;
    }

    document.getElementById('contact-form').addEventListener('submit', async (e) => {
      e.preventDefault();

      if (!turnstileToken) {
        alert('Please complete the verification');
        return;
      }

      const formData = new FormData(e.target);
      formData.append('cf-turnstile-response', turnstileToken);

      const response = await fetch('/api/contact', {
        method: 'POST',
        body: formData
      });

      if (response.ok) {
        alert('Form submitted successfully!');
      } else {
        alert('Submission failed. Please try again.');
      }
    });
  </script>
</body>
</html>
```

#### Backend (Workers)

```typescript
// src/index.ts
interface Env {
  TURNSTILE_SECRET_KEY: string;
  CONTACT_FORM_SUBMISSIONS: KVNamespace;
}

async function verifyTurnstileToken(token: string, secret: string, ip: string): Promise<boolean> {
  const formData = new URLSearchParams();
  formData.append('secret', secret);
  formData.append('response', token);
  formData.append('remoteip', ip);

  const response = await fetch('https://challenges.cloudflare.com/turnstile/v0/siteverify', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded',
    },
    body: formData,
  });

  const data = await response.json<{ success: boolean }>();
  return data.success;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    if (request.method !== 'POST') {
      return new Response('Method not allowed', { status: 405 });
    }

    const formData = await request.formData();
    const turnstileToken = formData.get('cf-turnstile-response') as string;

    if (!turnstileToken) {
      return new Response('Missing Turnstile token', { status: 400 });
    }

    // Verify Turnstile token
    const clientIP = request.headers.get('CF-Connecting-IP') || '';
    const isValid = await verifyTurnstileToken(
      turnstileToken,
      env.TURNSTILE_SECRET_KEY,
      clientIP
    );

    if (!isValid) {
      return new Response('Turnstile verification failed', { status: 403 });
    }

    // Process form submission
    const name = formData.get('name') as string;
    const email = formData.get('email') as string;
    const message = formData.get('message') as string;

    // Store submission
    await env.CONTACT_FORM_SUBMISSIONS.put(
      `submission:${Date.now()}`,
      JSON.stringify({ name, email, message, timestamp: Date.now() })
    );

    return new Response('Form submitted successfully', { status: 200 });
  }
};
```

#### Configuration

```jsonc
// wrangler.jsonc
{
  "name": "contact-form-worker",
  "main": "src/index.ts",
  "compatibility_date": "2025-03-07",
  "compatibility_flags": ["nodejs_compat"],
  "kv_namespaces": [
    {
      "binding": "CONTACT_FORM_SUBMISSIONS",
      "id": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
    }
  ],
  "vars": {
    "TURNSTILE_SITE_KEY": "1x00000000000000000000AA"
  },
  "observability": {
    "enabled": true,
    "head_sampling_rate": 1
  }
}
```

**Set secret**:
```bash
echo "YOUR_SECRET_KEY" | npx wrangler secret put TURNSTILE_SECRET_KEY
```

### Turnstile Modes

**Managed Mode** (Recommended):
- Automatic challenge based on visitor behavior
- Invisible for low-risk visitors
- Shows challenge for suspicious behavior

**Non-Interactive Mode**:
- Always invisible
- No user interaction required
- Best for high-traffic APIs

**Invisible Mode**:
- Hidden widget
- Runs in background
- Good for seamless UX

### Best Practices

1. **Always verify server-side**: Never trust client-side validation alone
2. **Use secrets for keys**: Store secret key in Wrangler secrets
3. **Include IP address**: Pass client IP for better fraud detection
4. **Handle errors gracefully**: Provide clear feedback to users
5. **Rate limit endpoints**: Combine with rate limiting for defense in depth

---

## Rate Limiting Patterns

### Pattern 1: Simple Rate Limiting with KV

**Use for:** Low-traffic endpoints, simple rate limits

```typescript
interface Env {
  RATE_LIMIT: KVNamespace;
}

async function isRateLimited(
  key: string,
  limit: number,
  window: number,
  kv: KVNamespace
): Promise<boolean> {
  const count = await kv.get(key);
  const current = count ? parseInt(count) : 0;

  if (current >= limit) {
    return true; // Rate limited
  }

  // Increment counter
  await kv.put(key, (current + 1).toString(), { expirationTtl: window });
  return false;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const clientIP = request.headers.get('CF-Connecting-IP') || 'unknown';
    const key = `ratelimit:${clientIP}`;

    // 100 requests per 60 seconds
    if (await isRateLimited(key, 100, 60, env.RATE_LIMIT)) {
      return new Response('Rate limit exceeded', {
        status: 429,
        headers: {
          'Retry-After': '60'
        }
      });
    }

    // Process request
    return new Response('OK');
  }
};
```

### Pattern 2: Advanced Rate Limiting with Durable Objects

**Use for:** High-traffic endpoints, complex rate limiting logic

```typescript
import { DurableObject } from 'cloudflare:workers';

interface Env {
  RATE_LIMITER: DurableObjectNamespace;
}

export class RateLimiter extends DurableObject {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);
    const action = url.searchParams.get('action');

    if (action === 'check') {
      const allowed = await this.checkRateLimit();
      return Response.json({ allowed });
    }

    return new Response('Invalid action', { status: 400 });
  }

  async checkRateLimit(): Promise<boolean> {
    const now = Date.now();
    const window = 60000; // 60 seconds
    const limit = 100;

    // Get request timestamps from storage
    const timestamps = (await this.ctx.storage.get<number[]>('timestamps')) || [];

    // Remove expired timestamps
    const validTimestamps = timestamps.filter(ts => now - ts < window);

    if (validTimestamps.length >= limit) {
      return false; // Rate limited
    }

    // Add current timestamp
    validTimestamps.push(now);
    await this.ctx.storage.put('timestamps', validTimestamps);

    return true; // Allowed
  }
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const clientIP = request.headers.get('CF-Connecting-IP') || 'unknown';
    const id = env.RATE_LIMITER.idFromName(clientIP);
    const limiter = env.RATE_LIMITER.get(id);

    const response = await limiter.fetch(
      `https://fake-host/?action=check`
    );
    const { allowed } = await response.json<{ allowed: boolean }>();

    if (!allowed) {
      return new Response('Rate limit exceeded', { status: 429 });
    }

    // Process request
    return new Response('OK');
  }
};
```

### Pattern 3: Token Bucket Rate Limiting

**Use for:** Burst traffic handling with sustained rate limits

```typescript
export class TokenBucketRateLimiter extends DurableObject {
  private capacity = 100; // Max tokens
  private refillRate = 10; // Tokens per second

  async checkRateLimit(tokens: number = 1): Promise<boolean> {
    const now = Date.now();
    const state = await this.ctx.storage.get<{
      tokens: number;
      lastRefill: number;
    }>('bucket') || { tokens: this.capacity, lastRefill: now };

    // Refill tokens based on time elapsed
    const elapsed = (now - state.lastRefill) / 1000;
    const tokensToAdd = Math.floor(elapsed * this.refillRate);
    const currentTokens = Math.min(this.capacity, state.tokens + tokensToAdd);

    if (currentTokens >= tokens) {
      // Allow request, consume tokens
      await this.ctx.storage.put('bucket', {
        tokens: currentTokens - tokens,
        lastRefill: now
      });
      return true;
    }

    // Not enough tokens
    return false;
  }
}
```

---

## Secrets Management

### Using Secrets Store (NEW - Recommended)

**Best for:** Production applications with centralized secret management

```typescript
interface Env {
  SECRETS: Fetcher; // Secrets Store binding
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Access secret from Secrets Store
    const apiKeyResponse = await env.SECRETS.fetch(
      'https://secrets/API_KEY'
    );
    const apiKey = await apiKeyResponse.text();

    // Use secret
    const response = await fetch('https://api.example.com/data', {
      headers: {
        'Authorization': `Bearer ${apiKey}`
      }
    });

    return response;
  }
};
```

**Configuration**:
```jsonc
{
  "secrets_store": {
    "binding": "SECRETS"
  }
}
```

### Using Wrangler Secrets (Traditional)

**Best for:** Simple applications, individual secrets

```bash
# Set secret
echo "your-api-key" | npx wrangler secret put API_KEY

# Use in Worker
```

```typescript
interface Env {
  API_KEY: string; // Automatically available
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const response = await fetch('https://api.example.com/data', {
      headers: {
        'Authorization': `Bearer ${env.API_KEY}`
      }
    });

    return response;
  }
};
```

### Best Practices

1. **Never hardcode secrets** in source code
2. **Use environment-specific secrets** for staging vs production
3. **Rotate secrets regularly** using automated workflows
4. **Audit secret access** with logging
5. **Use Secrets Store** for centralized management in production

---

## API Security

### JWT Validation

```typescript
import jwt from '@tsndr/cloudflare-worker-jwt';

interface Env {
  JWT_SECRET: string;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const authHeader = request.headers.get('Authorization');

    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      return new Response('Unauthorized', { status: 401 });
    }

    const token = authHeader.substring(7);

    try {
      const isValid = await jwt.verify(token, env.JWT_SECRET);

      if (!isValid) {
        return new Response('Invalid token', { status: 401 });
      }

      const decoded = jwt.decode(token);

      // Process authenticated request
      return Response.json({ user: decoded.payload });

    } catch (error) {
      return new Response('Invalid token', { status: 401 });
    }
  }
};
```

### API Key Validation

```typescript
interface Env {
  API_KEYS: KVNamespace;
}

async function validateAPIKey(key: string, kv: KVNamespace): Promise<boolean> {
  const keyData = await kv.get(key);
  return keyData !== null;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const apiKey = request.headers.get('X-API-Key');

    if (!apiKey) {
      return new Response('Missing API key', { status: 401 });
    }

    if (!await validateAPIKey(apiKey, env.API_KEYS)) {
      return new Response('Invalid API key', { status: 403 });
    }

    // Process request
    return new Response('OK');
  }
};
```

### Request Signing

```typescript
async function verifySignature(
  request: Request,
  secret: string
): Promise<boolean> {
  const signature = request.headers.get('X-Signature');
  const timestamp = request.headers.get('X-Timestamp');

  if (!signature || !timestamp) {
    return false;
  }

  // Prevent replay attacks (5 minute window)
  const now = Date.now();
  const requestTime = parseInt(timestamp);
  if (Math.abs(now - requestTime) > 300000) {
    return false;
  }

  // Verify signature
  const body = await request.clone().text();
  const message = `${timestamp}.${body}`;

  const encoder = new TextEncoder();
  const key = await crypto.subtle.importKey(
    'raw',
    encoder.encode(secret),
    { name: 'HMAC', hash: 'SHA-256' },
    false,
    ['verify']
  );

  const signatureBytes = hexToBytes(signature);
  const messageBytes = encoder.encode(message);

  return await crypto.subtle.verify(
    'HMAC',
    key,
    signatureBytes,
    messageBytes
  );
}

function hexToBytes(hex: string): Uint8Array {
  const bytes = new Uint8Array(hex.length / 2);
  for (let i = 0; i < hex.length; i += 2) {
    bytes[i / 2] = parseInt(hex.substr(i, 2), 16);
  }
  return bytes;
}
```

---

## Input Validation

### URL Validation

```typescript
function isValidURL(url: string): boolean {
  try {
    const parsed = new URL(url);
    // Only allow HTTPS
    return parsed.protocol === 'https:';
  } catch {
    return false;
  }
}
```

### Email Validation

```typescript
function isValidEmail(email: string): boolean {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return emailRegex.test(email);
}
```

### Content Type Validation

```typescript
function validateContentType(request: Request, expected: string): boolean {
  const contentType = request.headers.get('Content-Type');
  return contentType?.startsWith(expected) ?? false;
}

export default {
  async fetch(request: Request): Promise<Response> {
    if (request.method === 'POST') {
      if (!validateContentType(request, 'application/json')) {
        return new Response('Invalid content type', { status: 400 });
      }
    }

    return new Response('OK');
  }
};
```

### SQL Injection Prevention

```typescript
// ALWAYS use parameterized queries with D1
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const userId = url.searchParams.get('userId');

    // ❌ NEVER do this (SQL injection risk)
    // const query = `SELECT * FROM users WHERE id = '${userId}'`;

    // ✅ ALWAYS use prepared statements
    const { results } = await env.DB.prepare(
      'SELECT * FROM users WHERE id = ?'
    ).bind(userId).all();

    return Response.json(results);
  }
};
```

---

## Security Headers

### Comprehensive Security Headers

```typescript
function addSecurityHeaders(response: Response): Response {
  const newHeaders = new Headers(response.headers);

  // Content Security Policy
  newHeaders.set(
    'Content-Security-Policy',
    "default-src 'self'; script-src 'self' 'unsafe-inline' https://challenges.cloudflare.com; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data:; connect-src 'self'; frame-ancestors 'none';"
  );

  // XSS Protection
  newHeaders.set('X-Content-Type-Options', 'nosniff');
  newHeaders.set('X-Frame-Options', 'DENY');
  newHeaders.set('X-XSS-Protection', '1; mode=block');

  // HSTS
  newHeaders.set(
    'Strict-Transport-Security',
    'max-age=31536000; includeSubDomains; preload'
  );

  // Referrer Policy
  newHeaders.set('Referrer-Policy', 'strict-origin-when-cross-origin');

  // Permissions Policy
  newHeaders.set(
    'Permissions-Policy',
    'geolocation=(), microphone=(), camera=()'
  );

  return new Response(response.body, {
    status: response.status,
    statusText: response.statusText,
    headers: newHeaders
  });
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const response = await handleRequest(request, env);
    return addSecurityHeaders(response);
  }
};
```

### CORS Headers

```typescript
function handleCORS(request: Request, response: Response): Response {
  const origin = request.headers.get('Origin');
  const allowedOrigins = ['https://example.com', 'https://app.example.com'];

  if (origin && allowedOrigins.includes(origin)) {
    const newHeaders = new Headers(response.headers);
    newHeaders.set('Access-Control-Allow-Origin', origin);
    newHeaders.set('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
    newHeaders.set('Access-Control-Allow-Headers', 'Content-Type, Authorization');
    newHeaders.set('Access-Control-Max-Age', '86400');

    return new Response(response.body, {
      status: response.status,
      headers: newHeaders
    });
  }

  return response;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Handle preflight
    if (request.method === 'OPTIONS') {
      return new Response(null, {
        headers: {
          'Access-Control-Allow-Origin': '*',
          'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
          'Access-Control-Allow-Headers': 'Content-Type, Authorization',
          'Access-Control-Max-Age': '86400'
        }
      });
    }

    const response = await handleRequest(request, env);
    return handleCORS(request, response);
  }
};
```

---

---

## Web Crypto API

### Overview

**Source**: https://developers.cloudflare.com/workers/runtime-apis/web-crypto/

Use Web Crypto API for all cryptographic operations. **Never implement custom crypto**.

### Hashing & Digesting

**Supported algorithms**: SHA-1, SHA-256, SHA-384, SHA-512, MD5 (legacy only)

```typescript
async function hashPassword(password: string): Promise<string> {
  const encoder = new TextEncoder();
  const data = encoder.encode(password);

  const hashBuffer = await crypto.subtle.digest('SHA-256', data);

  // Convert to hex string
  const hashArray = Array.from(new Uint8Array(hashBuffer));
  const hashHex = hashArray.map(b => b.toString(16).padStart(2, '0')).join('');

  return hashHex;
}
```

### HMAC Request Signing

```typescript
interface Env {
  SIGNING_SECRET: string;
}

async function signRequest(data: string, secret: string): Promise<string> {
  const encoder = new TextEncoder();

  // Import secret key
  const key = await crypto.subtle.importKey(
    'raw',
    encoder.encode(secret),
    { name: 'HMAC', hash: 'SHA-256' },
    false,
    ['sign']
  );

  // Sign data
  const signature = await crypto.subtle.sign(
    'HMAC',
    key,
    encoder.encode(data)
  );

  // Convert to hex
  return Array.from(new Uint8Array(signature))
    .map(b => b.toString(16).padStart(2, '0'))
    .join('');
}

async function verifyRequest(
  data: string,
  signature: string,
  secret: string
): Promise<boolean> {
  const encoder = new TextEncoder();

  // Import secret key
  const key = await crypto.subtle.importKey(
    'raw',
    encoder.encode(secret),
    { name: 'HMAC', hash: 'SHA-256' },
    false,
    ['verify']
  );

  // Convert hex signature to Uint8Array
  const signatureBytes = new Uint8Array(
    signature.match(/.{1,2}/g)!.map(byte => parseInt(byte, 16))
  );

  // Verify (timing-safe)
  return await crypto.subtle.verify(
    'HMAC',
    key,
    signatureBytes,
    encoder.encode(data)
  );
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const receivedSignature = request.headers.get('X-Signature');
    const body = await request.text();

    if (!receivedSignature) {
      return new Response('Missing signature', { status: 401 });
    }

    // ✅ Use crypto.subtle.verify() - timing-safe
    const isValid = await verifyRequest(body, receivedSignature, env.SIGNING_SECRET);

    if (!isValid) {
      return new Response('Invalid signature', { status: 403 });
    }

    return new Response('OK');
  }
};
```

**Security Best Practice**: Use `crypto.subtle.verify()` instead of string comparison to prevent timing attacks.

### Timing-Safe Comparison

```typescript
async function timingSafeEqual(a: string, b: string): Promise<boolean> {
  const encoder = new TextEncoder();
  const bufferA = encoder.encode(a);
  const bufferB = encoder.encode(b);

  if (bufferA.length !== bufferB.length) {
    return false;
  }

  // Constant-time comparison
  return crypto.subtle.timingSafeEqual(bufferA, bufferB);
}
```

### Encryption & Decryption

**Symmetric encryption (AES-GCM)**:

```typescript
async function encryptData(data: string, secret: string): Promise<string> {
  const encoder = new TextEncoder();

  // Generate random IV
  const iv = crypto.getRandomValues(new Uint8Array(12));

  // Import key
  const keyMaterial = await crypto.subtle.importKey(
    'raw',
    encoder.encode(secret.padEnd(32, '0').slice(0, 32)),
    'AES-GCM',
    false,
    ['encrypt']
  );

  // Encrypt
  const encrypted = await crypto.subtle.encrypt(
    { name: 'AES-GCM', iv },
    keyMaterial,
    encoder.encode(data)
  );

  // Combine IV + encrypted data
  const combined = new Uint8Array(iv.length + encrypted.byteLength);
  combined.set(iv, 0);
  combined.set(new Uint8Array(encrypted), iv.length);

  // Return as base64
  return btoa(String.fromCharCode(...combined));
}

async function decryptData(encrypted: string, secret: string): Promise<string> {
  const encoder = new TextEncoder();
  const decoder = new TextDecoder();

  // Decode base64
  const combined = Uint8Array.from(atob(encrypted), c => c.charCodeAt(0));

  // Extract IV and encrypted data
  const iv = combined.slice(0, 12);
  const data = combined.slice(12);

  // Import key
  const keyMaterial = await crypto.subtle.importKey(
    'raw',
    encoder.encode(secret.padEnd(32, '0').slice(0, 32)),
    'AES-GCM',
    false,
    ['decrypt']
  );

  // Decrypt
  const decrypted = await crypto.subtle.decrypt(
    { name: 'AES-GCM', iv },
    keyMaterial,
    data
  );

  return decoder.decode(decrypted);
}
```

### Secure Random Values

```typescript
// Generate cryptographically secure random values
const randomBytes = crypto.getRandomValues(new Uint8Array(32));

// Generate UUID (RFC 4122 v4)
const uuid = crypto.randomUUID(); // e.g., '123e4567-e89b-12d3-a456-426614174000'
```

---

## CORS Best Practices

### Origin-Specific CORS

**Source**: https://developers.cloudflare.com/workers/examples/cors-header-proxy/

```typescript
function handleCORS(request: Request, response: Response): Response {
  const origin = request.headers.get('Origin');

  // Whitelist of allowed origins
  const allowedOrigins = [
    'https://example.com',
    'https://app.example.com',
    'https://staging.example.com'
  ];

  if (origin && allowedOrigins.includes(origin)) {
    const headers = new Headers(response.headers);

    // ✅ Set origin-specific (not wildcard)
    headers.set('Access-Control-Allow-Origin', origin);

    // ✅ Include Vary header to prevent incorrect caching
    headers.append('Vary', 'Origin');

    headers.set('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
    headers.set('Access-Control-Allow-Headers', 'Content-Type, Authorization, X-API-Key');
    headers.set('Access-Control-Allow-Credentials', 'true');
    headers.set('Access-Control-Max-Age', '86400'); // 24 hours

    return new Response(response.body, {
      status: response.status,
      headers
    });
  }

  // Origin not allowed
  return response;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Handle preflight request
    if (request.method === 'OPTIONS') {
      return handlePreflight(request);
    }

    const response = await handleRequest(request, env);

    // Add CORS headers to response
    return handleCORS(request, response);
  }
};

function handlePreflight(request: Request): Response {
  const origin = request.headers.get('Origin');
  const method = request.headers.get('Access-Control-Request-Method');
  const headers = request.headers.get('Access-Control-Request-Headers');

  // Detect preflight request
  if (origin && method && headers) {
    return new Response(null, {
      headers: {
        'Access-Control-Allow-Origin': origin,
        'Access-Control-Allow-Methods': method,
        'Access-Control-Allow-Headers': headers,
        'Access-Control-Max-Age': '86400',
        'Vary': 'Origin'
      }
    });
  }

  return new Response('Invalid preflight request', { status: 400 });
}
```

**Security Considerations**:
- Use origin-specific responses, not wildcards (`*`) when using credentials
- Include `Vary: Origin` header to prevent cache poisoning
- Return request's `Access-Control-Request-Headers` in preflight response
- Validate origin against whitelist

---

## TLS Validation

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    // Validate TLS version 1.2 or higher
    const tlsVersion = request.cf?.tlsVersion;

    if (tlsVersion && !['TLSv1.2', 'TLSv1.3'].includes(tlsVersion)) {
      return new Response('TLS 1.2+ required', { status: 403 });
    }

    return new Response('OK');
  }
};
```

---

## Summary

### Security Layer Checklist

For production applications, implement:

1. ✅ **WAF Rules** - Basic attack protection
2. ✅ **Turnstile** - Bot protection for forms
3. ✅ **Rate Limiting** - Prevent abuse
4. ✅ **Secrets Management** - Protect credentials
5. ✅ **Input Validation** - Sanitize all inputs
6. ✅ **Security Headers** - Browser-level protection
7. ✅ **Authentication** - JWT or API keys
8. ✅ **Request Signing** - HMAC with Web Crypto API
9. ✅ **HTTPS Only** - Enforce TLS 1.2+
10. ✅ **CORS** - Origin-specific with Vary header
11. ✅ **Web Crypto API** - Use for all cryptographic operations
12. ✅ **Timing-Safe Comparisons** - Prevent timing attacks
13. ✅ **Logging** - Monitor security events

### Defense in Depth

Combine multiple security layers:
```
WAF → Rate Limiting → Turnstile → Input Validation → Web Crypto → Business Logic
```

Each layer provides redundancy if another fails.

### Critical Security Principles

1. **Never implement custom crypto** - Use Web Crypto API
2. **Use timing-safe comparison** - `crypto.subtle.verify()` or `timingSafeEqual()`
3. **Origin-specific CORS** - No wildcards with credentials
4. **Parameterized queries** - Prevent SQL injection
5. **Encrypted secrets** - Never plaintext environment variables
6. **TLS 1.2+ only** - Validate TLS version
7. **Security headers** - CSP, HSTS, X-Frame-Options, etc.

---

## See Also

- **Deployment**: `deployment-workflows.md` - Secrets management, environments
- **Integration Details**: `workers-integrations.md` - Binding configuration
- **Best Practices**: `workers-best-practices.md` - Code standards
- **Examples**: `workers-examples.md` - Complete implementations
- **Official Docs**:
  - WAF: https://developers.cloudflare.com/waf/
  - Turnstile: https://developers.cloudflare.com/turnstile/
  - Web Crypto: https://developers.cloudflare.com/workers/runtime-apis/web-crypto/
  - Security Headers: https://developers.cloudflare.com/workers/examples/security-headers/
