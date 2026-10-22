# Recommended Improvements

**Also see:** [findings.md](./findings.md)

One recommendation per finding, in the same order. For each one: what changes, why, the code,
the trade-off, and how to roll it out.

The code shows the approach rather than being ready to paste. I had no access to Cynbit's
source, so the imports, the Prisma client path and the email helper will all look different in
the real codebase. The stack I based it on is the one the site runs on: Next.js 15, Prisma, and
a schema validation library like Zod.

## List

| # | Change | Fixes | Effort |
| - | ------ | ----- | ------ |
| [R-01](#r-01) | One validation contract for all the forms | F-01 | ~1 day |
| [R-02](#r-02) | Cache the database-backed pages instead of rebuilding them | F-02 | ~1 day |
| [R-03](#r-03) | Add the security headers | F-03 | ~2 hours |
| [R-04](#r-04) | Ask for less personal data, and say so accurately | F-04 | ~2 days |
| [R-05](#r-05) | Update Next.js to the patched release | F-05 | ~1 hour |

---

## R-01 — One validation contract for all the forms

**Fixes [F-01](./findings.md#f-01)** · Effort: ~1 day

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
every form, so it is worth doing behind a flag for one release. It also means the schemas become
shared code that needs to be treated like an API, which is the point of it.

**How to roll it out.** Start with the inquiry and service lead forms, because those are the ones
failing silently and the difference is visible to users. Then the job application and diary
forms, which also stops the internal text appearing. The newsletter can follow, or stay as it is
— its contract is already reasonable.

---

## R-02 — Cache the database-backed pages instead of rebuilding them

**Fixes [F-02](./findings.md#f-02)** · Effort: ~1 day

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

**Trade-off.** Content can be up to five minutes out of date, and an edit needs a revalidation
call to show up immediately. That is the right trade for a blog and a case study library, and the
wrong one for anything showing live stock. I would leave the careers pages until the posting
flow is confirmed to call revalidation, so a closed role disappears on time.

**How to roll it out.** Start with the case studies and the diaries — they change least often and
are the heaviest pages. The blog is the biggest win by visitor count and is the one to do second.

---

## R-03 — Add the security headers

**Fixes [F-03](./findings.md#f-03)** · Effort: ~2 hours

**What to change.** One `headers()` block in the Next config, applied to every route, plus
turning off the `X-Powered-By` header so it stops appearing on some pages and not others.

**Why.** These are configuration, not a project. They are the layer that catches the things that
actually go wrong: a third-party script gets compromised, a stored value ends up in an internal
view, or an XSS slips past a sanitiser.

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

**Trade-off.** A CSP is the one change here that can break the site, because the browser enforces
it and this site uses inline styles and a Google tag manager. The way to avoid a broken
production page: add `Content-Security-Policy-Report-Only` first with the same value, watch what
it complains about for a few days, then switch to the enforcing header. The `'unsafe-inline'`
for scripts is a deliberate shortcut — removing it properly means using nonces, which is more
work than it looks and is better as a later change.

**How to roll it out.** Report-only first, then enforce. Keep `frame-ancestors 'none'` — nothing
on the site should be embeddable, and if that turns out to be wrong it is a one-line change.

---

## R-04 — Ask for less personal data, and say so accurately

**Fixes [F-04](./findings.md#f-04)** · Effort: ~2 days

**What to change.** Three parts, and the order matters:

1. **Stop asking for what is not needed.** Date of birth, gender and permanent address are not
   selection criteria for a software role. Remove them from the form and from the schema.
2. **Rewrite the privacy page to match reality** — the fields actually collected, how long data
   is kept, and a named contact for data requests, which DPDP Act 2023 asks for.
3. **State the processing in plain terms**, including that candidate data is passed to the
   transactional email provider.

**Why.** A privacy policy that understates what you collect is a trust problem before it is a
legal one, and the extra fields are not buying anything. The form also gets shorter and quicker
to complete, which is good for the applicant too.

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

**How to roll it out.** In order: stop showing the three fields, rewrite the privacy page, add the
named contact, then drop the columns. Nothing needs to be backfilled.

---

## R-05 — Update Next.js to the patched release

**Fixes [F-05](./findings.md#f-05)** · Effort: ~1 hour

**What to change.** Move from 15.5.18 to 15.5.28, and make sure `sharp` is on a patched version.

**Why.** Same minor line, so no API changes and no migration. It closes the Server Action
denial-of-service advisory, which applies here specifically because the app uses Server Actions
for every write path.

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

**How to roll it out.** Deploy to a preview, run the six forms end to end, then promote. There is
no database migration in this upgrade, so the risk is low.

---

## Notes on the code

- These examples use **Next.js 15, Prisma and Zod**, which matches the stack the site runs on.
  The validation library is an inference from the error wording the forms return, not something
  I could confirm from the bundle.
- They are **illustrations, not Cynbit's code.** The imports, the Prisma client path and the
  email helper will all differ in the real codebase.
- The Zod examples use `.issues` from Zod v4. On Zod v3 it is `.errors`.
