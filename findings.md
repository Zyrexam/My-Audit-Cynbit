# Cynbit — Backend Technical Audit and Findings

**Target:** `cynbit.in` (served from `www.cynbit.in`)
**Date:** 2026-10-21
**Method:** passive and authorised only — no exploitation, no load testing, no password guessing.

Part 1 is the technical audit: what the site is built on, and how its backend actually works.
Part 2 is the findings, ordered from low severity to high.

Each finding states what I saw, why it matters, and what would change my conclusion. Every claim
can be checked from outside with `curl`.

---

# Part 1 — Technical audit

## 1.1 Where I focused, and why

The brief asked me to pick the areas that mattered most rather than cover everything thinly.
These are the five I chose, with the findings each one produced:

| Area | Why I looked at it | Findings |
| ---- | ------------------ | -------- |
| **API behaviour and error handling** | Every write on the site goes through one mechanism, and that mechanism is not documented anywhere. If the contracts are inconsistent, every future form inherits the problem | F-01 |
| **Performance and backend architecture** | Three quarters of the site is rebuilt from the database on every visitor, for content that changes a few times a month | F-02 |
| **Security configuration** | Response headers are the layer that catches the things that actually go wrong in production, and all of it is configuration | F-03 |
| **Data handling** | The careers form asks for more than the privacy page admits to, which is a trust problem before it is a technical one | F-04 |
| **Dependency currency** | The cheapest security improvement available, and it needs a decision rather than a project | F-05 |

**Scope.** This review covers `cynbit.in` and `www.cynbit.in`. The site also links to a separate
portal on its own hostname, which is a different deployment with its own access model. That is
outside what this review covers and would be a separate piece of work.

## 1.2 What the site is built on

Everything here came from the site itself: the JavaScript the browser downloads, the response
headers, the public policy pages, and the pages as a browser sees them. Where Cynbit states
something publicly I have said so.

| Layer | What I found | How I know |
| ----- | ------------- | ---------- |
| Hosting | Vercel, edge region `bom1` (Mumbai) | `X-Vercel-Id` header, `Server: Vercel` |
| Framework | Next.js 15.5.18, App Router | version string in the client bundle, and the `X-Nextjs-Prerender` / `X-Nextjs-Stale-Time` headers |
| Rendering | React Server Components | the payload format in the HTML, `text/x-component` responses |
| Styling | Tailwind CSS | the `__variable_*` class names in the rendered HTML |
| Database | Prisma | stated on `/privacy`, and consistent with how the pages render |
| Transactional email | Resend | stated on `/privacy` |
| Analytics | Google Analytics 4, property `G-LK6CQWCWX0` | the `gtag/js?id=G-LK6CQWCWX0` script tag |
| Fonts | Google Fonts plus two self-hosted `woff2` files | the `<link>` tags on the homepage |
| Images | Next's image optimizer proxying `images.unsplash.com` | `/_next/image?url=…` returns `200 image/jpeg` |
| CDN | Vercel edge network; HTTP to HTTPS is a `308` | redirect behaviour, `X-Vercel-Cache` |
| TLS | TLS 1.2 and 1.3 both work | tested with `--tlsv1.2` and `--tlsv1.3` |

## 1.3 The backend has no REST API

This is the first thing worth knowing, because it is not obvious from the outside. **Every write
in this application is a Next.js Server Action** — a server-side function the browser calls
directly, with a build-time id in a `Next-Action` header.

There is no `/api` directory, no REST or GraphQL endpoint, and no admin panel. Reading is
ordinary page rendering; there is no read API either, because the content is rendered into the
HTML.

I found six write paths. The identifiers are in the JavaScript every visitor downloads, so they
are readable by anyone:

| Purpose | Page | Action name | Takes |
| ------- | ---- | ----------- | ----- |
| Inquiry | `/contact` | `submitInquiry` | `FormData` |
| Service lead | `/services` | `submitServiceLead` | `FormData` + `serviceType` |
| Job application | `/careers/[slug]` | `submitCandidateApplication` | `FormData` |
| Diary entry | `/diaries` | `submitDiaryEntry` | `FormData` |
| BitElevate application | `/bitelevate` | `submitBitelevateApplication` | `FormData` |
| Newsletter signup | layout form, every page | `subscribeToNewsletter` | a plain email string |

## 1.4 What one of those requests looks like

A Server Action is not a documented endpoint, so the request format is worth writing down. This
is a real request the browser makes when you submit the contact form:

```
POST /contact HTTP/1.1
Host: www.cynbit.in
Next-Action: 40abb1a98c0be9294b518c6ca84ffcc8f63f3a5849
Accept: text/x-component
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary…

------WebKitFormBoundary…
Content-Disposition: form-data; name="_1_fullName"

Security Assessment Probe
------WebKitFormBoundary…
Content-Disposition: form-data; name="_1_email"

probe@example.com
------WebKitFormBoundary…
Content-Disposition: form-data; name="0"

["$K1"]
------WebKitFormBoundary…--
```

Two details are easy to get wrong: the field names have a prefix (`_1_email`, not `email`), and
there is an extra part named `0` carrying `["$K1"]`, which is the argument list pointing at the
form data.

The reply is not JSON either, it is a React payload:

```
0:{"a":"$@1","f":"","b":"ztuwP7dezhtorZo2Dg72F"}
1:{"success":false}
```

**The status code tells you very little.** A validation failure is an `HTTP 200` with
`{"success":false}`. A real error is a `500` with an error digest. An action id that does not
exist is a `404 Server action not found.` You have to read the body to know what happened.

## 1.5 Two rendering models, and the site mostly uses the slower one

The response headers say which model a page uses.

**Pre-built pages** carry `X-Nextjs-Prerender: 1` and are served from the Vercel edge with
`X-Vercel-Cache: HIT`. There are **12** of them: the homepage, `/story`, `/team`, `/contact`,
`/services`, `/products`, `/impact`, `/bitelevate`, `/bitelevate/paths`, and the three legal
pages. None of them show live data.

**Pages rebuilt on every request** carry `Cache-Control: private, no-cache, no-store` and no
`X-Nextjs-Prerender` header. There are **70** of them — every blog post, every job role, every
case study and every diary entry, plus their index pages. The content comes from the database,
so each visit triggers a database read and a render.

Out of the 82 URLs in the sitemap, that is 12 pre-built and 70 rebuilt per request. This is
[F-02](#f-02).

## 1.6 What the site already does well

Worth writing down, because these are decisions that should not get undone while fixing the
rest.

- **Every write path validates on the server.** I could not find an endpoint that accepted a
  malformed value. The `required` attributes in the browser are a convenience, not the control.
- **The framework's origin check is on.** A Server Action request with a foreign `Origin` is
  rejected before the action runs.
- **Nothing sensitive is published.** Source maps return `403`. `/.env`, `/.env.local`,
  `/.git/config`, `/package.json` and `/next.config.js` all return `404`. There is no `/api` and
  no `/admin`.
- **Static file caching is correct** — hashed files served `immutable` for a year.
- **Compression and TLS are both correct.** The homepage is 100,498 bytes uncompressed and
  15,392 bytes with brotli. HTTP to HTTPS is a `308`, not a `302`.
- **The privacy policy names its processors** (Prisma, Resend). A lot of sites do not. The
  problem is that it then understates what is collected.

---

## 2. How to check these yourself

Every finding below came from ordinary requests against the public site. These four commands
cover most of them:

```bash
# F-03 — the security headers that are missing
curl -sSI https://www.cynbit.in/ | grep -iE 'content-security|x-frame|x-content-type|referrer-policy|permissions-policy'

# F-02 — which pages are pre-built and which are rebuilt on every request
curl -sSI https://www.cynbit.in/blog    | grep -iE 'x-nextjs-prerender|cache-control'
curl -sSI https://www.cynbit.in/contact | grep -iE 'x-nextjs-prerender|cache-control'

# F-01 — the forms do not agree on how they report errors
curl -sSI https://www.cynbit.in/careers/backend-developer | grep -i 'cache-control'

# F-05 — the framework version is in the JavaScript the browser downloads
curl -sSL https://www.cynbit.in/ | grep -o '/_next/static/[^"]*\.js' | head -5
```

---

# Part 2 — Findings

Ordered from low severity to high. Each one states what I saw, why it matters, and what would
change my conclusion.

| ID | Area | Severity | Summary |
| -- | ---- | -------- | ------- |
| [F-01](#f-01) | API behaviour and error handling | Medium | The forms report errors in different ways, and two of them show internal validation text |
| [F-02](#f-02) | Performance and architecture | Medium | 70 of the site's 82 pages are rebuilt on every request |
| [F-03](#f-03) | Security configuration | Medium | No Content-Security-Policy, framing, MIME or referrer headers |
| [F-04](#f-04) | Data handling | Medium | The privacy policy does not match what the careers form actually asks for |
| [F-05](#f-05) | Dependencies | High | Next.js 15.5.18 is behind the patched release, and one published advisory applies to apps built this way |

---

## F-01 — The forms report errors in different ways, and two of them show internal validation text

**Severity: Medium** · **Area: API behaviour and error handling**

**What I saw.** The six write paths do not agree on how to report a bad request:

| Endpoint | What comes back for bad input |
| -------- | ------------------------------ |
| Inquiry (`/contact`) | `{"success":false}` — no reason given |
| Service lead (`/services`) | `{"success":false}` — no reason given |
| Job application (`/careers/[slug]`) | Internal validation text, e.g. `Invalid input: expected string, received undefined • Invalid input: …` |
| Diary entry (`/diaries`) | Internal validation text, same wording |
| BitElevate application (`/bitelevate`) | Proper messages, e.g. `ambitions must be at least 10 characters.` |
| Newsletter signup | `{"success":false,"error":"Invalid email format."}` |

Two problems come out of that:

1. **Two forms return their internal validation text to anyone who submits one.** The phrase
   `expected string, received undefined` describes the shape of the code's own data model to an
   unauthenticated visitor. The wording is the kind produced by a schema validation library.
2. **The inquiry and service lead forms fail with no explanation.** The page can only show
   "Something went wrong", so somebody who mistypes their email gets the same message as
   somebody whose submission was rejected. That pushes people to email instead of using the
   form.

One more thing I noticed on the same endpoint: the **newsletter checks the email format only**.
It accepted `a@b.co` and `probe@example.invalid` — valid-looking but undeliverable — and there
is no confirmation step, so the list can fill up with addresses that will never receive mail. It
does check for duplicates correctly, so the pattern already exists elsewhere in the code.

**Why it matters.** When several endpoints doing the same job return different shapes, every new
form starts by copying whichever one it happens to sit next to. It also makes the API hard to
test, and it is the kind of inconsistency that is much cheaper to fix now than in a year.

**Fix:** [R-01](./recommendations.md#r-01).

---

## F-02 — 70 of the site's 82 pages are rebuilt on every request

**Severity: Medium** · **Area: performance and backend architecture**

**What I saw.** 12 pages are pre-built and served from the edge. The other 70 are rebuilt from
the database on every single visit. The split is visible in the response headers
(`X-Nextjs-Prerender: 1` for pre-built, `Cache-Control: private, no-cache, no-store` for the
rest), and it covers all 82 URLs in the sitemap.

| Pages | Count | Time to first byte | Uncompressed size |
| ----- | ----- | ------------------- | ----------------- |
| Pre-built (homepage, contact, services, products, legal pages and a few others) | 12 | 0.28–0.39 s | 52–100 KB |
| `/blog` + 25 posts | 26 | 0.74–1.49 s | ~90 KB each |
| `/careers` + 20 roles | 21 | 0.69–1.79 s | ~78 KB each |
| `/case-studies` + 13 detail pages | 14 | 0.73–0.85 s | 67–157 KB |
| `/diaries` + 8 entries | 9 | 0.82–0.93 s | ~62–71 KB |

The heaviest page on the site, `/case-studies` at 157 KB, is one of the ones that is rebuilt
every time.

**Why it matters.** Each of those 70 pages triggers a database read and a render for every
visitor, with nothing cached in front of it. The content changes a few times a month, so all of
that work is repeated for no reason. This is the normal case for incremental static
regeneration, which Next.js supports with a single line per route.

**What would change this.** If the content genuinely changes constantly, or a small part of each
page has to be fresh, some of this is correct. The blog and the case studies do not look like
they need that.

**Fix:** [R-02](./recommendations.md#r-02).

---

## F-03 — No security headers on the responses

**Severity: Medium** · **Area: security configuration**

**What I saw.** I captured the full response headers for the homepage, `/contact`, the 404 page,
a static file and a form submission. The same set was missing from all of them:

| Header | What it does |
| ------ | ------------ |
| `Content-Security-Policy` | Stops a script that gets injected into a page from running |
| `X-Frame-Options` | Stops the site being put in an iframe on another site (clickjacking) |
| `X-Content-Type-Options` | Stops the browser guessing a file type |
| `Referrer-Policy` | Stops full URLs leaking to other sites in the `Referer` header |
| `Permissions-Policy` | Stops browser features like camera and location being used without permission |

Two smaller things in the same capture:

- `X-Powered-By: Next.js` shows up on some responses but not others — for example on `/contact`
  and `/blog`, but not on the homepage or `/services`.
- `Access-Control-Allow-Origin: *` is on some responses and not others, and the pattern is
  roughly the opposite of the one above.

Neither of those is a problem on its own, but headers that come and go depending on how a
response was cached are headers nobody is managing.

HSTS is set for two years, which is good, but **without `includeSubDomains` and without
`preload`**, so other subdomains of `cynbit.in` are not covered by it.

**Why it matters.** These are configuration, not code. They are the layer that catches the things
that actually go wrong: a third-party script gets compromised, a stored value ends up in an
internal view, or an XSS slips past a sanitiser.

**Fix:** [R-03](./recommendations.md#r-03).

---

## F-04 — The privacy policy does not match what the careers form actually asks for

**Severity: Medium** · **Area: data handling**

**What I saw.** The privacy page says:

> "We only collect the minimum information necessary to provide our services — typically your
> name, email, and professional context when you reach out via our forms."

The careers form asks for `fullName`, `email`, `phone`, `whatsapp`, `dob` (date of birth),
`gender`, `permanentAddress`, `currentCity`, `linkedinUrl`, `portfolioUrl`,
`professionalProfileUrl`, `skills`, `noticePeriod`, `message` and `agreedToTerms`, plus a set of
education fields.

Date of birth, gender and permanent home address are not "professional context". They are also
the fields that identify someone most strongly, and none of them help decide whether somebody is
right for a backend internship.

**Why it matters.** A privacy policy that understates what you collect is a trust problem before
it is a legal one, and the extra fields are not buying anything. Cynbit is Jaipur-based and takes
applications from Indian candidates, so what is collected and how long it is kept are DPDP Act
2023 questions, and GDPR ones if any of it reaches people in the EU. The fix is mostly a schema
change.

**What would change this.** The published text may be out of date rather than reflecting intent,
and I could not see the database, so I cannot confirm what is kept after a submission or whether
these fields are used for something legitimate.

**Fix:** [R-04](./recommendations.md#r-04).

---

## F-05 — The site runs a Next.js version that is behind the patched release

**Severity: High** · **Area: dependencies**

**What I saw.** The deployed application runs **Next.js 15.5.18**. I got the version from the
JavaScript the browser downloads, so that is the version a visitor actually runs.

The current patched versions are **15.5.28** and **16.4.1**. 15.5.18 is below both, so it does
not include the fixes published in the September and October 2026 security releases.

One of those matters here in particular. The denial-of-service issue published in the October
release applies to App Router applications that define at least one Server Action — and this
application defines six, with every write path running through them. I also checked the
image-optimizer advisory from the same release, which involves SVG handling: that code path is
**not** reachable on this site, because `/_next/image` returns `400` for `.svg` sources and the
site does not use `next/og`. So I am flagging the version, not an open hole.

**Why it matters.** Staying current with dependencies is the cheapest security work there is,
and 15.5.28 is a patch on the same minor version, so there is no migration to do and no API
change to plan around.

**What would change this.** If the deployed build is newer than the bundle suggests, this would
not apply. The bundle is what the browser runs, so I went with 15.5.18.

**Fix:** [R-05](./recommendations.md#r-05).

---

## What I could not check

These need access I do not have, so I did not guess at them and they are not findings:

1. **What is actually stored after a submission**, how long it is kept, and who can see it. I
   could not see the database.
2. **Whether a rate limit or WAF sits in front of the origin.** That would only show up as a
   `429`, and I did not get one — but I only made a small number of requests, so this is not
   something I can rule out.
3. **The separate portal deployment** linked from the main site, which is outside the scope of
   this review.

---

*Recommendations and code: [recommendations.md](./recommendations.md)*
