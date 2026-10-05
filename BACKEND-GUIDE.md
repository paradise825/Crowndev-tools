# Crown Dev: how to add payments, automatic access and an admin view

Your site is on GitHub Pages, which only serves files. It cannot take payments or remember who paid. To do that you add a small **backend**. The simplest, mostly free way:

| Job | Service | Cost |
|---|---|---|
| Take card and bank payments in Naira and Dollar | **Paystack** | % fee per payment, no monthly fee |
| Remember who paid, sign people in, show you a list | **Supabase** | Free plan is enough to start |
| Receive payment receipts in your email | **FormSubmit** (already in your pages) | Free |

How it fits together:

1. A customer pays on `pricing.html` (Paystack).
2. Paystack tells your Supabase function "this email paid for this plan".
3. The function saves a row in an `access` table.
4. The customer signs in on `tools.html` with that email. The page reads their row and unlocks the right tools.
5. You open the table in Supabase and see everyone who has access.

Bank-transfer customers: you check their receipt and add (or switch on) their row yourself. They get access the next time they sign in.

---

## Step 1. Create the accounts

1. **Supabase**: supabase.com, sign up, click **New project**, choose a region near your customers, set a database password and save it.
2. **Paystack**: paystack.com, create a business account and finish the verification so you can take live payments. Use the **test keys** first while you try everything.
   - Paystack takes Naira by default. **Dollar payments must be switched on for your Paystack account** (ask their support or check Settings, Preferences). If you cannot get Dollar enabled, ask me and I will adapt the page for another provider such as Flutterwave.
3. **FormSubmit**: no account. The first time anyone sends a receipt or message, FormSubmit emails you an activation link. Click it once. After that, receipts arrive in your inbox with the file attached.

## Step 2. Create the access table

In Supabase open **SQL Editor**, paste this and press **Run**:

```sql
create table public.access (
  id bigint generated always as identity primary key,
  email text not null,
  plan text not null check (plan in ('basic','standard','premium')),
  currency text,
  amount numeric,
  reference text unique,
  source text default 'paystack',        -- 'paystack' or 'transfer'
  status text not null default 'active', -- set to 'revoked' to remove access
  paid_at timestamptz default now(),
  expires_at timestamptz                 -- leave empty for lifetime access
);

alter table public.access enable row level security;

-- A signed-in customer can read only their own rows.
create policy "read own access" on public.access
  for select to authenticated
  using (lower(email) = lower(auth.jwt() ->> 'email'));
```

There is no insert policy on purpose. Nobody on the website can give themselves access. Only your function and you (in the dashboard) can write rows.

## Step 3. Turn on email sign-in

Supabase, **Authentication**, **Providers**: make sure **Email** is on (magic link / one-time code).
Then **Authentication**, **URL Configuration**:
- **Site URL**: `https://paradise825.github.io/CrownDev-tools/`
- **Redirect URLs**: add `https://paradise825.github.io/CrownDev-tools/tools.html`

## Step 4. The payment webhook (this is what grants access automatically)

Install the Supabase CLI (supabase.com/docs/guides/cli), then in a folder on your computer:

```
supabase login
supabase init
supabase link --project-ref YOUR_PROJECT_REF
supabase functions new paystack-webhook
```

Put this in `supabase/functions/paystack-webhook/index.ts`:

```ts
import { createClient } from "npm:@supabase/supabase-js@2";

const secret = Deno.env.get("PAYSTACK_SECRET_KEY")!;
const db = createClient(Deno.env.get("SUPABASE_URL")!, Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!);
const enc = new TextEncoder();

// Smallest amount (in kobo / cents) each plan must be paid. Keep in step with pricing.html.
const PRICE: Record<string, Record<string, number>> = {
  basic:    { NGN: 100000, USD: 150 },
  standard: { NGN: 200000, USD: 300 },
  premium:  { NGN: 350000, USD: 500 },
};

async function sign(body: string) {
  const key = await crypto.subtle.importKey("raw", enc.encode(secret),
    { name: "HMAC", hash: "SHA-512" }, false, ["sign"]);
  const sig = await crypto.subtle.sign("HMAC", key, enc.encode(body));
  return [...new Uint8Array(sig)].map(b => b.toString(16).padStart(2, "0")).join("");
}

Deno.serve(async (req) => {
  const body = await req.text();
  if (req.headers.get("x-paystack-signature") !== await sign(body))
    return new Response("bad signature", { status: 401 });

  const ev = JSON.parse(body);
  if (ev.event !== "charge.success") return new Response("ignored");

  const d = ev.data, plan = d.metadata?.plan;
  const need = PRICE[plan]?.[d.currency];
  if (!need || d.amount < need) return new Response("amount does not match plan", { status: 400 });

  const { error } = await db.from("access").upsert({
    email: String(d.customer.email).toLowerCase(),
    plan, currency: d.currency, amount: d.amount / 100,
    reference: d.reference, source: "paystack", status: "active",
  }, { onConflict: "reference" });

  return error ? new Response(error.message, { status: 500 }) : new Response("ok");
});
```

The amount check matters: without it, someone could edit the page in their browser and pay ₦1,000 for Premium.

Deploy it and store your Paystack **secret** key (never put the secret key in a web page):

```
supabase secrets set PAYSTACK_SECRET_KEY=sk_test_xxxxxxxx
supabase functions deploy paystack-webhook --no-verify-jwt
```

In the Paystack dashboard, **Settings, API Keys & Webhooks**, set the **Webhook URL** to:

```
https://YOUR_PROJECT_REF.supabase.co/functions/v1/paystack-webhook
```

## Step 5. Connect the website

1. **pricing.html**, top of the script, in `CONFIG`:
   - `PAYSTACK_PUBLIC_KEY`: your `pk_test_...` key (switch to `pk_live_...` when you go live)
   - `BANK`: your Naira account details, and how to pay in Dollars
2. **tools.html**, top of the script, in `CONFIG`:
   - `PAYWALL: true`
   - `SUPABASE_URL`: Supabase, Project Settings, API, Project URL
   - `SUPABASE_ANON_KEY`: the `anon public` key on the same page (it is meant to be public because of the row rules in Step 2)
3. Upload both files to GitHub again.

## Step 6. Test before you go live

1. With `pk_test_` and `sk_test_` keys, buy Basic on your site using Paystack's test card (listed in their docs).
2. Open Supabase, **Table Editor**, `access`. A row for that email should appear within seconds.
3. On `tools.html` click **Email me a sign-in link**, open the link, and check Image compressor and Unit converter are unlocked but QR and Background remover are not.
4. Swap in your live keys and repeat once with a real small payment.

## Seeing who has access (your admin view)

- Supabase, **Table Editor**, `access`: every customer, plan, amount, date and where they paid. Use the filter bar to search by email. **Export** gives you a CSV.
- **Give access after a bank transfer**: click **Insert row**, fill `email`, `plan`, `source = transfer`, leave `status = active`.
- **Remove access**: change `status` to `revoked`.
- **Time-limited plans**: fill `expires_at`. The page already ignores expired rows.

## About WhatsApp and receipts

- The receipt form emails the file to you automatically, with the customer's name, plan and amount.
- WhatsApp **cannot** receive a file automatically from a website. The "Message us on WhatsApp" button opens a chat with the details typed in, and the customer attaches the receipt themselves. Receiving files automatically needs the WhatsApp Business API, which is paid and needs Meta approval. Email is the reliable channel.

## Honest limits

- The tools run inside the visitor's browser, so a technical person could read the page's code and use a locked tool without paying. The unlock check stops normal visitors, and the payment and access list are secure, but it is not a vault. If paid tools become valuable, the next step is to serve their code only to signed-in paying users.
- Supabase free projects can pause after a week with no activity. Visit the dashboard or upgrade if that happens.
- The plans are set up as **one payment per plan with no expiry**. If you want monthly or yearly plans, the table already has `expires_at`, and the webhook needs a small change.
