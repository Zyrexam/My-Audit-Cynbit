# Recommended Improvements

**Also see:** [findings.md](./findings.md)

One recommendation per finding, in the same order — low severity first. For each one: what
changes, why, the code, the trade-off, and how to roll it out.

The code shows the approach rather than being ready to paste. I had no access to Cynbit's
source, so the imports, the Prisma client path and the email helper will all look different in
the real codebase. The stack I have based it on is the one the site runs on: Next.js 15, Prisma,
and a schema validation library like Zod.

## List

| # | Change | Fixes | Effort |
| - | ------ | ----- | ------ |
| [R-01](#r-01) | Canonical URLs and consistent metadata host | F-01 | ~2 hours |
| [R-02](#r-02) | Correct the cookie page and ask before loading analytics | F-02 | ~1 day |
| [R-03](#r-03) | Add the security headers | F-03 | ~2 hours |
| [R-04](#r-04) | Cache the database-backed pages instead of rebuilding them | F-04 | ~1 day |
| [R-05](#r-05) | One validation contract for all the forms | F-05 | ~1 day |
| [R-06](#r-06) | Ask for less personal data, and say so accurately | F-06 | ~2 days |
| [R-07](#r-07) | Rate limit, size limit and duplicate protection on the forms | F-07 | ~1 day |
| [R-08](#r-08) | Limit portal login attempts and keep the session server-side | F-08 | ~1 day |
| [R-09](#r-09) | Update Next.js to the patched release | F-09 | ~1 hour |

---

## R-01 — Canonical URLs and consistent metadata host

**Fixes [F-01](./findings.md#f-01)** · Effort: ~2 hours

**What to change.** Pick the host the site actually serves on (`www.cynbit.in`) and make the
sitemap, the `og:url` and the canonical link all agree with it. Add the missing
`rel="canonical"`, give each page its own `og:url` rather than the homepage URL, and use a PNG
for the social preview image instead of an SVG.

**Why.** Right now the site tells search engines and social scrapers that its address is a host
that redirects, and does not give them a canonical address at all. The pages are fine; the
signal is not, and it is a small config change.

**Code.**

```tsx
// app/layout.tsx
import type { Metadata } from 'next';

export const metadata: Metadata = {
  metadataBase: new URL('https://www.cynbit.in'),   // the host that actually serves
  title: { default: '…', template: '%s | Cynbit Technologies' },
  description: '…',
  alternates: {
    canonical: '/',               // default; individual pages override it
  },
  openGraph: {
    type: 'website',
    siteName: 'Cynbit Technologies',
    url: 'https://www.cynbit.in',
    images: [{ url: '/og.png', width: 1200, height: 630, alt: 'Cynbit Technologies' }],
  },
  twitter: { card: 'summary_large_image' },
};
```

```tsx
// app/blog/[slug]/page.tsx — each page gets its own canonical URL
export async function generateMetadata({ params }): Promise<Metadata> {
  const { slug } = await params;
  const post = await getPost(slug);
  if (!post) return {};

  return {
    title: post.title,
    description: post.excerpt,
    alternates: { canonical: `/blog/${slug}` },
    openGraph: { title: post.title, description: post.excerpt, url: `/blog/${slug}` },
  };
}
```

And the sitemap agrees with it:

```ts
// app/sitemap.ts
import type { MetadataRoute } from 'next';

export default function sitemap(): MetadataRoute.Sitemap {
  return routes.map((r) => ({
    url: `https://www.cynbit.in${r.path}`,
    lastModified: r.updatedAt,
    changeFrequency: 'weekly',
    priority: r.priority,
  }));
}
```

**Trade-off.** Whichever host is chosen becomes the real one, and links to the other get one
extra redirect until they are updated. Either choice is fine — what is not fine is the sitemap,
the metadata and the serving host disagreeing. If the company prefers the apex as the brand,
point it the other way; the code is the same with the hosts swapped.

**How to roll it out.** The metadata change is safe on its own. Change the redirect later, and
check it against the sitemap and the analytics settings first, because a canonical change moves
search signals and is easier to get right the first time.

---

## R-02 — Correct the cookie page and ask before loading analytics

**Fixes [F-02](./findings.md#f-02)** · Effort: ~1 day

**What to change.** Two parts:

1. **Rewrite the cookie page to match reality.** The site sets no cookies of its own, so the
   "Session tokens, CSRF protection" and "Theme selection, form persistence" lines should go.
   What is left is analytics, and how it is used.
2. **Ask before GA4 loads.** Nothing loads it now, and there is no banner.

**Why.** The cookie page is the document a visitor, a customer or a regulator would look at, so
it should describe what actually happens. Cynbit is Jaipur-based and takes applications from
Indian candidates, so consent for analytics and for application data are DPDP Act 2023
questions, and GDPR ones if any of it reaches people in the EU.

**Code.** The consent has to happen before the analytics tag loads, otherwise the first page
view is already recorded:

```tsx
// components/analytics-consent.tsx
'use client';
import { useEffect, useState } from 'react';

const CONSENT_KEY = 'cynbit_analytics_consent';

export function AnalyticsConsent() {
  const [visible, setVisible] = useState(false);

  useEffect(() => {
    // Deny by default, before the tag runs. This is the part that matters.
    window.gtag?.('consent', 'default', {
      analytics_storage: 'denied',
      ad_storage: 'denied',
    });

    if (localStorage.getItem(CONSENT_KEY) === 'granted') {
      window.gtag?.('consent', 'update', { analytics_storage: 'granted' });
    } else {
      setVisible(true);
    }
  }, []);

  function decide(granted: boolean) {
    localStorage.setItem(CONSENT_KEY, granted ? 'granted' : 'denied');
    window.gtag?.('consent', 'update', {
      analytics_storage: granted ? 'granted' : 'denied',
    });
    setVisible(false);
  }

  if (!visible) return null;

  return (
    <div className="fixed bottom-4 left-4 right-4 z-50 rounded-2xl border bg-surface p-6 shadow-xl">
      <p className="text-sm">
        We use Google Analytics to understand how the site is used. You can accept or decline.
        See our <a href="/cookies">cookie policy</a> and <a href="/privacy">privacy policy</a>.
      </p>
      <div className="mt-4 flex gap-3">
        <button onClick={() => decide(true)}>Accept</button>
        <button onClick={() => decide(false)}>Decline</button>
      </div>
    </div>
  );
}
```

The cookie page then needs to say something like: *this site sets no cookies of its own. Google
Analytics sets its own cookies to count visits; you can decline above, and we do not run
advertising or cross-site tracking.*

**Trade-off.** A banner is one more thing on the page, and declining analytics means losing
some of the traffic data. That is the point of asking, and the loss is smaller than it sounds
because most visitors will accept. A banner also has to be a real decision rather than a
dismissable popup, otherwise it does not count as consent.

**How to roll it out.** Rewrite the cookie page first — it is a copy change with no code risk.
Then the banner, with the analytics tag loading only after the answer.

---

## R-03 — Add the security headers

**Fixes [F-03](./findings.md#f-03)** · Effort: ~2 hours

**What to change.** One `headers()` block in the Next config, applied to every route, plus
turning off the `X-Powered-By` header so it stops appearing on some pages and not others.

**Why.** These are configuration, not a project. They are the layer that catches the things
that actually go wrong: a third-party script gets compromised, a stored value ends up in an
internal view, or an XSS slips past a sanitiser.

**Code.**

```js
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  poweredByHeader: false,        // removes "X-Powered-By: Next.js"

  async headers() {
    return [
      {
        source: '/:path*',
        headers: [
          {
            key: 'Content-Security-Policy',
            value: [
              "default-src 'self'",
              "script-src 'self' 'unsafe-inline' 'unsafe-eval' https://www.googletagmanager.com",
              "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
              "font-src 'self' https://fonts.gstatic.com data:",
              "img-src 'self' data: blob: https://images.unsplash.com https://i.pravatar.cc",
              "connect-src 'self' https://www.google-analytics.com",
              "frame-ancestors 'none'",
              "base-uri 'self'",
              "form-action 'self'",
              "object-src 'none'",
            ].join('; '),
          },
          { key: 'X-Frame-Options', value: 'DENY' },
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
          { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
          { key: 'Cross-Origin-Opener-Policy', value: 'same-origin' },
        ],
      },
    ];
  },
};

module.exports = nextConfig;
```

**Trade-off.** A CSP is the one change here that can break the site, because the browser
enforces it and this site uses inline styles and a Google tag manager. The way to avoid a broken
production page: add `Content-Security-Policy-Report-Only` first with the same value, watch what
it complains about for a few days, then switch to the enforcing header. The `'unsafe-inline'`
for scripts is a deliberate shortcut — removing it properly means using nonces, which is more
work than it looks and is better as a later change.

**How to roll it out.** Report-only first, then enforce. Keep `frame-ancestors 'none'` —
nothing on the site should be embeddable, and if that turns out to be wrong it is a one-line
change.

---

## R-04 — Cache the database-backed pages instead of rebuilding them

**Fixes [F-04](./findings.md#f-04)** · Effort: ~1 day

**What to change.** Move the 70 pages that are rebuilt on every request to incremental static
regeneration. Each page is then built at most once every few minutes instead of once per
visitor.

**Why.** It is the largest performance improvement available and it is a small change. The
content changes a few times a month, and right now it is rebuilt and re-read from the database
for every reader.

**Code.**

```ts
// app/blog/[slug]/page.tsx
import { notFound } from 'next/navigation';
import { prisma } from '@/lib/prisma';
import { getPost } from '@/lib/content';

// Rebuild this page at most once every 5 minutes, and serve it from the
// edge cache in between.
export const revalidate = 300;

// Build every post at deploy time too, so the first visitor after a release
// does not wait on a cold build.
export async function generateStaticParams() {
  const posts = await prisma.post.findMany({ select: { slug: true } });
  return posts.map((post) => ({ slug: post.slug }));
}

export default async function PostPage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const post = await getPost(slug);
  if (!post) notFound();
  return <Article post={post} />;
}
```

If a page needs genuinely fresh data, cache the query rather than the whole page:

```ts
// lib/content.ts
import { unstable_cache } from 'next/cache';

export const getPost = unstable_cache(
  async (slug: string) => prisma.post.findUnique({ where: { slug } }),
  ['post'],
  { revalidate: 300, tags: ['posts'] },
);
```

And clear the tag when something is published, so it appears straight away instead of waiting
out the five minutes:

```ts
// wherever a post is created or updated
import { revalidateTag } from 'next/cache';

export async function publishPost(input: PostInput) {
  const post = await prisma.post.create({ data: input });
  revalidateTag('posts');
  return post;
}
```

**Trade-off.** Content can be up to five minutes out of date, and an edit needs a
revalidation call to show up immediately. That is the right trade for a blog and a case study
library, and the wrong one for anything showing live stock. I would leave the careers pages
until the posting flow is confirmed to call revalidation, so a closed role disappears on time.

**How to roll it out.** Start with the case studies and the diaries — they change least often
and are the heaviest pages. The blog is the biggest win by visitor count and is the one to do
second.

---

## R-05 — One validation contract for all the forms

**Fixes [F-05](./findings.md#f-05)** · Effort: ~1 day

**What to change.** Put the form schemas in one module and give every action the same reply
shape. That fixes three things at once: the job application and diary forms stop showing their
internal text, the inquiry and service lead forms stop failing silently, and the next form
starts from a template instead of from whichever endpoint it sits next to.

**Why.** Several endpoints doing the same job with different reply shapes looks like a small
inconsistency and turns into a week of debugging later. This is also the change that pays off
most over time, because every form after this one gets it for free.

**Code.**

```ts
// lib/forms/contract.ts
import { z, type ZodTypeAny } from 'zod';

export type ActionResult<T = void> =
  | { success: true; data?: T }
  | { success: false; error: string; fieldErrors?: Record<string, string> };

/**
 * Validate, and on failure return something safe for the browser.
 * The raw validation issues never leave the server.
 */
export function parse<S extends ZodTypeAny>(
  schema: S,
  input: unknown,
): { ok: true; value: z.infer<S> } | { ok: false; result: ActionResult } {
  const parsed = schema.safeParse(input);

  if (parsed.success) return { ok: true, value: parsed.data };

  const fieldErrors: Record<string, string> = {};
  for (const issue of parsed.error.issues) {   // Zod v4: .issues   (v3: .errors)
    const key = issue.path.join('.') || '_';
    fieldErrors[key] ??= issue.message;
  }

  return {
    ok: false,
    result: {
      success: false,
      error: 'Some details need attention. Please check the highlighted fields.',
      fieldErrors,
    },
  };
}
```

```ts
// lib/forms/schemas.ts
import { z } from 'zod';

export const inquirySchema = z.object({
  fullName: z.string().trim().min(2, 'Please enter your full name.').max(120),
  email: z.string().trim().email('Please enter a valid email address.'),
  company: z.string().trim().max(120).optional().or(z.literal('')),
  industry: z.string().trim().max(120).optional().or(z.literal('')),
  message: z.string().trim().min(10, 'Please add a little more detail.').max(5000),
});
```

```ts
// app/contact/actions.ts
'use server';
import { inquirySchema } from '@/lib/forms/schemas';
import { parse, type ActionResult } from '@/lib/forms/contract';

export async function submitInquiry(
  _prev: ActionResult,
  formData: FormData,
): Promise<ActionResult> {
  const parsed = parse(inquirySchema, Object.fromEntries(formData));
  if (!parsed.ok) return parsed.result;

  try {
    await prisma.inquiry.create({ data: parsed.value });
    await sendInquiryNotification(parsed.value);
    return { success: true };
  } catch (error) {
    // Log the detail here, return something safe. The browser never sees a
    // stack trace or a database message.
    console.error('submitInquiry failed', error);
    return {
      success: false,
      error: 'We could not save your inquiry just now. Please try again, or email us directly.',
    };
  }
}
```

And the client shows the per-field messages instead of one generic string:

```jsx
const result = await submitInquiry(prevState, formData);

if (result.success) {
  setDone(true);
} else {
  setError(result.error);
  setFieldErrors(result.fieldErrors ?? {});
}
```

**Trade-off.** A shared contract means a validation change in one place changes the reply for
every form, so it is worth doing behind a flag for one release. It also means the schemas
become shared code that needs to be treated like an API, which is the point of it.

**How to roll it out.** Start with the inquiry and service lead forms, because those are the
ones failing silently and the difference is visible to users. Then the job application and
diary forms, which also stops the internal text appearing. The newsletter and the portal login
can follow, or stay as they are — their contracts are already reasonable.

---

## R-06 — Ask for less personal data, and say so accurately

**Fixes [F-06](./findings.md#f-06)** · Effort: ~2 days

**What to change.** Three parts, and the order matters:

1. **Stop asking for what is not needed.** Date of birth, gender and permanent address are not
   selection criteria for a software role. Remove them from the form and from the schema.
2. **Rewrite the privacy page to match reality** — the fields actually collected, how long
   data is kept, and a named contact for data requests, which DPDP Act 2023 asks for.
3. **State the processing in plain terms**, including that candidate data is passed to the
   transactional email provider.

**Why.** A privacy policy that understates what you collect is a trust problem before it is a
legal one, and the extra fields are not buying anything. The form also gets shorter and
quicker to complete, which is good for the applicant too.

**Code.**

```ts
// lib/forms/schemas.ts — the reduced job application schema
export const candidateApplicationSchema = z.object({
  fullName: z.string().trim().min(2, 'Please enter your full name.').max(120),
  email: z.string().trim().email('Please enter a valid email address.'),
  phone: z.string().trim().min(7, 'Please enter a contact number.').max(20),
  currentCity: z.string().trim().max(80),
  skills: z.string().trim().max(2000),
  noticePeriod: z.string().trim().max(60),
  linkedinUrl: z.string().trim().url('Please enter a valid URL.').optional().or(z.literal('')),
  portfolioUrl: z.string().trim().url('Please enter a valid URL.').optional().or(z.literal('')),
  message: z.string().trim().max(3000),
  // dob, gender and permanentAddress are gone, along with the columns behind them
  agreedToTerms: z.literal('on', { error: 'Please accept the privacy notice to continue.' }),
});
```

The privacy page then needs to list what is actually kept, for how long, and who to ask. A
structure that covers the main questions:

```md
## What we collect

- Name, email, phone and city
- Your CV, portfolio and LinkedIn profile, if you share them
- Skills and notice period
- Your message

## What we do not collect

We do not ask for date of birth, gender or home address. They are not used to assess
applications and we do not want to hold them.

## How long we keep it

Applications are kept for 12 months from the date of application, then deleted.

## Who processes it

Our transactional email provider (Resend) delivers the notification emails, and our database
provider (Prisma) hosts the records. We do not sell or share application data.

## Your rights

Write to opportunities@cynbit.in to ask what we hold about you, to correct it, or to have it
deleted.
```

**Trade-off.** The one thing to check first is whether any downstream process — a background
check, a statutory record, an HR system — currently reads those fields. That is why the rollout
drops the form fields first and the database columns later.

**How to roll it out.** In order: stop showing the three fields, rewrite the privacy page, add
the named contact, then drop the columns. Nothing needs to be backfilled.

---

## R-07 — Rate limit, size limit and duplicate protection on the forms

**Fixes [F-07](./findings.md#f-07)** · Effort: ~1 day

**What to change.** Four small things:

1. **Cap the request body.** A 1 MB message was accepted. Next.js lets you set the limit.
2. **Limit submissions** per IP address and per email address.
3. **Add a honeypot field** — one that visitors cannot see, which must arrive empty. If it is
   filled, the submission is quietly dropped.
4. **Stop duplicates.** Submitting the same inquiry twice created two records.

**Why.** The public forms are an open, unauthenticated way to make the server do work and to
write rows. These are the controls that stop that being free.

**Code.**

```js
// next.config.js — alongside the headers() block
const nextConfig = {
  experimental: {
    serverActions: {
      bodySizeLimit: '256kb',   // down from the 1 MB default
    },
  },
  // ... headers() as above
};
```

```ts
// lib/forms/guard.ts
import { headers } from 'next/headers';

const attempts = new Map<string, number[]>();

/**
 * A sliding window counter.
 *
 * On serverless each instance has its own memory, so this only limits per
 * instance. For a limit that covers every instance, keep the counter in Redis
 * — the rest of the code does not change.
 */
export async function withinRateLimit(
  key: string,
  limit: number,
  windowMs: number,
): Promise<boolean> {
  const now = Date.now();
  const recent = (attempts.get(key) ?? []).filter((t) => now - t < windowMs);

  if (recent.length >= limit) {
    attempts.set(key, recent);
    return false;
  }

  recent.push(now);
  attempts.set(key, recent);
  return true;
}

export async function clientIp(): Promise<string> {
  const h = await headers();
  return h.get('x-forwarded-for')?.split(',')[0]?.trim() ?? 'unknown';
}

/**
 * Honeypot: the field is hidden from people and must arrive empty.
 * A bot that fills everything gets a normal-looking reply and no record.
 */
export function isHoneypotTripped(formData: FormData): boolean {
  return String(formData.get('website') ?? '').length > 0;
}
```

```ts
// app/contact/actions.ts
export async function submitInquiry(_prev: ActionResult, formData: FormData) {
  if (isHoneypotTripped(formData)) {
    return { success: true };   // looks like it worked, stores nothing
  }

  const ip = await clientIp();
  if (!(await withinRateLimit(`inquiry:${ip}`, 5, 60 * 60 * 1000))) {
    return { success: false, error: 'Too many inquiries from this connection. Please try again later.' };
  }

  const email = String(formData.get('email') ?? '').toLowerCase();
  if (!(await withinRateLimit(`inquiry-email:${email}`, 3, 24 * 60 * 60 * 1000))) {
    return { success: false, error: 'We have already received several inquiries from this address.' };
  }

  // ... validation and saving
}
```

The hidden field on the form:

```jsx
<input
  type="text"
  name="website"
  tabIndex={-1}
  autoComplete="off"
  aria-hidden="true"
  className="absolute -left-[9999px] h-0 w-0 opacity-0"
/>
```

For duplicates, give each form session a key and let the database sort it out:

```prisma
// prisma/schema.prisma
model Inquiry {
  id             String   @id @default(cuid())
  idempotencyKey String   @unique          // <- stops the second row
  fullName       String
  email          String
  message        String
  createdAt      DateTime @default(now())
}
```

```ts
// the client makes the key once per page load, not once per click
const key = useRef(crypto.randomUUID()).current;
<input type="hidden" name="idempotencyKey" value={key} />
```

```ts
// app/contact/actions.ts
await prisma.inquiry.createMany({
  data: [{ ...parsed.value, idempotencyKey }],
  skipDuplicates: true,   // a repeated submission becomes a no-op
});
```

**Trade-off.** A body size limit will reject very long inquiries. 256 KB is far above a real
one — a genuine inquiry is a few hundred bytes — so this costs nothing in practice. Limiting
per email will occasionally turn away a second inquiry from the same person, which is fair for
a form that is already limited per IP. The honeypot is the only one with a false-positive
risk, so the field must be hidden with CSS rather than a `display: none` wrapper, and it must
never be `required`. The in-memory rate limiter only works per instance on serverless, so it
needs a shared store to be a real limit.

**How to roll it out.** The body size limit and the honeypot are same-day changes. The
idempotency key needs a column, added as optional first. The rate limiter is better done once
R-08 has introduced a shared counter.

---

## R-08 — Limit portal login attempts and keep the session server-side

**Fixes [F-08](./findings.md#f-08)** · Effort: ~1 day

**What to change.** Three things on the portal:

1. **Store the security key as a hash** instead of comparing the raw value. If the environment
   variable or the database ever leaks, the key itself does not come with it.
2. **Keep the session on the server.** The client currently writes a timestamp to `localStorage`,
   which the browser owns. A signed `httpOnly` cookie means the credential never touches
   JavaScript.
3. **Limit attempts** — five failures per IP per fifteen minutes, then a lockout.

**Why.** This is the only access-controlled part of the property. Everything else is public on
purpose; this one needs a real barrier.

**Code.**

```ts
// lib/auth/portal.ts
import { cookies } from 'next/headers';
import { scrypt, randomBytes, timingSafeEqual, createHmac } from 'node:crypto';
import { promisify } from 'node:util';

const scryptAsync = promisify(scrypt);

const COOKIE = 'cynbit_portal_session';
const SECRET = process.env.PORTAL_SESSION_SECRET!;
const KEY_HASH = process.env.PORTAL_KEY_HASH!;   // format: salt:hash (both hex)

// --- the security key -------------------------------------------------------

export async function hashKey(value: string) {
  const salt = randomBytes(16);
  const derived = (await scryptAsync(value, salt, 64)) as Buffer;
  return `${salt.toString('hex')}:${derived.toString('hex')}`;
}

export async function verifySecurityKey(candidate: string): Promise<boolean> {
  const [saltHex, hashHex] = KEY_HASH.split(':');
  if (!saltHex || !hashHex) return false;

  const expected = Buffer.from(hashHex, 'hex');
  const actual = (await scryptAsync(candidate, Buffer.from(saltHex, 'hex'), 64)) as Buffer;

  // Compare in constant time, so the answer does not depend on how far in
  // the strings matched.
  return expected.length === actual.length && timingSafeEqual(expected, actual);
}

// --- the session ------------------------------------------------------------

function sign(value: string) {
  return createHmac('sha256', SECRET).update(value).digest('hex');
}

export async function issueSession() {
  const payload = `${Date.now()}.${randomBytes(8).toString('hex')}`;
  const token = `${payload}.${sign(payload)}`;

  (await cookies()).set(COOKIE, token, {
    httpOnly: true,    // JavaScript cannot read it
    secure: true,      // HTTPS only
    sameSite: 'lax',   // not sent on cross-site POSTs
    path: '/',
    maxAge: 60 * 60 * 8,
  });
}

export async function hasValidSession(): Promise<boolean> {
  const token = (await cookies()).get(COOKIE)?.value;
  if (!token) return false;

  const parts = token.split('.');
  if (parts.length !== 3) return false;
  const [issued, nonce, signature] = parts;

  const expected = Buffer.from(sign(`${issued}.${nonce}`));
  const actual = Buffer.from(signature);
  if (expected.length !== actual.length || !timingSafeEqual(expected, actual)) return false;

  return Date.now() - Number(issued) < 8 * 60 * 60 * 1000;   // 8 hours
}
```

```ts
// app/portal/actions.ts
'use server';
import { headers } from 'next/headers';
import { verifySecurityKey, issueSession } from '@/lib/auth/portal';
import { tooManyAttempts } from '@/lib/auth/rate-limit';

export async function loginToPortal(prev: { error?: string }, formData: FormData) {
  const ip = (await headers()).get('x-forwarded-for')?.split(',')[0] ?? 'unknown';

  if (await tooManyAttempts(`portal:${ip}`)) {
    return { error: 'Too many attempts. Please try again in a few minutes.' };
  }

  const password = String(formData.get('password') ?? '');
  if (!(await verifySecurityKey(password))) {
    return { error: 'Incorrect password' };   // same message every time
  }

  await issueSession();
  return { error: undefined };
}
```

The client stops writing to `localStorage`:

```jsx
// Before
localStorage.setItem('cynbit_portal_last_activity', Date.now().toString());
window.location.href = '/dashboard';

// After — the server set the cookie, the client just navigates
window.location.href = '/dashboard';
```

**Trade-off.** A shared rate-limit counter needs a shared store, which means a dependency or
platform configuration. An in-memory counter is simpler but, on serverless, it only limits per
instance — so it is a useful first step and not the whole answer. `node:crypto` is used here so
that this recommendation does not need any new dependency; `argon2` or `bcrypt` would also be
fine.

**How to roll it out.** Do the hashing and the rate limit first. They are self-contained, need
no client work, and on their own they close F-08. The cookie change needs the client and the
dashboard route deployed together, since the two have to agree on how the session works.

---

## R-09 — Update Next.js to the patched release

**Fixes [F-09](./findings.md#f-09)** · Effort: ~1 hour

**What to change.** Move from 15.5.18 to 15.5.28, and make sure `sharp` is on a patched version.

**Why.** Same minor line, so no API changes and no migration. It closes the Server Action
denial-of-service advisory, which applies here specifically because the app uses Server
Actions.

**Code.**

```jsonc
// package.json
{
  "dependencies": {
    "next": "15.5.28",     // was 15.5.18
    "sharp": "^0.35.5"     // 15.5.28 still allows the old sharp, so pin it
  }
}
```

```bash
npm install next@15.5.28 sharp@latest
npm ls next sharp        # check both resolved as intended
npm run build && npm run deploy
```

**Trade-off.** I cannot see one. 15.5.28 is a patch release. The only thing to watch is `sharp`:
15.5.28 still permits the older version to stay installed, so it has to be updated explicitly.

**How to roll it out.** Deploy to a preview, run the seven forms end to end, then promote.
There is no database migration in this upgrade, so the risk is low.

---

## Notes on the code

- These examples use **Next.js 15, Prisma and Zod**, which matches the stack the site runs on.
  The validation library is an inference from the error wording the forms return, not something
  I could confirm from the bundle.
- They are **illustrations, not Cynbit's code.** The imports, the Prisma client path, the email
  helper and the rate limit store will all differ in the real codebase.
- The rate limit examples use an in-memory map because it is easy to read. On serverless that
  only limits per instance, so it needs a shared store (Redis) or a platform-level edge limit to
  work properly. The rest of the code does not change either way.
- The Zod examples use `.issues` from Zod v4. On Zod v3 it is `.errors`.
