# Deploying Primefield Property Services to Hostinger

One Node process serves the whole site: the public pages and the two forms. There is no
database, no data folder and no account to set up. Every form submission is an email to the
office, so a deploy is an upload, a few settings and a test request.

There is no build step on the server and nothing to compile. The zip carries the finished
bundle and its `node_modules`, installed and audited on a machine with a toolchain.

---

## What you need

- The release zip, `primefield-v<version>-<date>.zip`, from `npm run package`.
- The domain pointed at Hostinger.
- The `noreply@primefieldpropertyservices.com` mailbox, and its password.
- Ten minutes.

---

## 1. Create the app and upload the zip

**Websites > Setup**, choose **Node.js web app**, and upload the zip. Hostinger extracts it and
shows **Review build settings**.

| Field | Value |
| --- | --- |
| Framework | **Express** |
| Node version | **24.x**. Not lower than 22.12. |
| Root directory | `./`. The zip extracts straight into the app root; there is no folder inside it. |
| Build settings | Leave them exactly as they arrive. |

The preset's build command is `npm run build`, and in this release that script prints
`already built` and exits. There is genuinely nothing to compile. The script exists because the
preset insists on one, and a deployment that cannot find it fails with an install log in which
nothing looks wrong.

**How the platform starts the app.** It does not run `npm start`. The web server there is
LiteSpeed, and its `lsnode.js` loads the application by `require()`ing the entry file at the root
of the extracted zip. That is what `server.js` is for: four lines that exist so there is
something at the root that can be required. The app must open its port within about three
seconds of being started, and this one does nothing slow before it listens.

---

## 2. Environment variables

Same screen, **Environment variables**. One row per line:

| Name | Value |
| --- | --- |
| `NODE_ENV` | `production` |
| `SITE_URL` | `https://primefieldpropertyservices.com` |
| `SESSION_SECRET` | 64 random hex characters. It signs the form token and nothing else. |
| `SMTP_HOST` | `smtp.hostinger.com` |
| `SMTP_PORT` | `465` |
| `SMTP_USER` | `noreply@primefieldpropertyservices.com` |
| `SMTP_PASS` | the mailbox password |
| `MAIL_FROM` | `Primefield Property Services <noreply@primefieldpropertyservices.com>` |
| `NOTIFY_EMAIL` | `info@primefieldpropertyservices.com` |
| `PUBLIC_SITE_MODE` | `served` |
| `TRUST_PROXY` | `1` |

**Do not add `PORT`.** This flow injects it, and a value typed here would be the wrong one.

Notes on the ones that are easy to get wrong:

- `SESSION_SECRET` must be at least 32 characters. Generate it, never type it.
- `TRUST_PROXY=1` is the number of proxies in front of the app. It is what makes the visitor's own
  address the one the rate limit counts and the one the office's email reports. At `0` every visitor
  looks like the proxy, and the limit of five submissions an hour is shared by everyone.
- `NOTIFY_EMAIL` is where every request and every vendor application arrives, so it must be a
  mailbox somebody reads. **The email is the only record of a submission.**

**Old rows.** If this app was running the version with a back office, its list still has
`DATABASE_URL`, `DATA_DIR` and perhaps `SEED_*` rows. Leave them or delete them. They are not
read, and the app starts the same either way. There is no database to create and no folder to keep.

---

## 3. Deploy

Press **Deploy**. When it finishes, **Runtime logs** shows

```
upm listening on <port> (build <commit> <time>)
```

That line says the app is up. A deployment with a clean install log and no runtime log at all
never started the app: check that the zip was extracted into the root, and that `server.js` is
there.

If the log says `Bad config: ...`, it names the setting that is missing or wrong, in one line.

---

## 4. Verify

1. **`/healthz`** answers `{"ok":true,"at":"...","build":"<commit> <time>"}`. The `build` is how
   you confirm the upload is the one that is live and not the bundle that was already running.
2. **`/`**, **`/services`** and **`/request-service`** load. **`/admin`** shows the site's "That
   page is not here." page, with a 404: there is no back office.
3. **One test request.** Fill in `/request-service` for real, with your own email address as the
   sender, tick the box and send it. Within a minute:
   - `info@primefieldpropertyservices.com` has an email "Service request REQ-...", with every
     field, the time in Eastern time, your address, and the sentence you agreed to;
   - your own address has the receipt "We have your request REQ-...".

   If the page says "We could not send this just now" instead, mail is not working: see **When
   something is wrong**. Do not go live until a test request arrives.
4. **Clear the CDN cache once** (hPanel > CDN). Hostinger's CDN keeps its own copy of each page,
   and a copy it took before this release still has the old privacy page.
5. The two checks under **Security headers on Hostinger** below, which show what a visitor's
   browser is actually given.

---

## Security headers on Hostinger

**What the host replaces.** With every page, the server sends the browser a set of rules called
the content policy: load scripts, styles, fonts and pictures from this site only, never run a
script written into a page, never send a form anywhere else. It travels as a response header
named `Content-Security-Policy`. Hostinger's web server sets a header of the same name by itself,
with the single value `upgrade-insecure-requests`, and that replaces the one the server sent. The
server's other security headers arrive untouched.

**What the site does about it.** The same rules are also written into each page, as one line near
the top of the page source. The host cannot replace that line, because it is part of the page. One
rule cannot be written into a page: the one that stops another website from showing this site
inside a frame. That one is carried by `X-Frame-Options: DENY`, which Hostinger does pass through.

**How to check after a deploy.**

1. The headers a visitor receives:

   ```
   curl -sI https://primefieldpropertyservices.com/ | grep -i -E "content-security-policy|x-frame-options"
   ```

   `x-frame-options: DENY` is the line to look for. The other line will read
   `content-security-policy: upgrade-insecure-requests`. That is Hostinger's header, it is
   expected, and it is the reason the rules are inside the page.

2. The rules inside the page:

   ```
   curl -s https://primefieldpropertyservices.com/ | head -c 600
   ```

   Straight after `<meta charset="utf-8">` there should be a line that starts
   `<meta http-equiv="Content-Security-Policy" content="default-src 'self';`.

3. If the page has no such line, or the header does not read `DENY`, the CDN is answering from
   an old copy. Clear its cache again, wait a minute and ask once more. `/healthz` shows which
   build the server itself is running.

There is nothing to set in hPanel for any of this.

---

## Every deploy after the first

1. `npm run build && npm run package` on the development machine.
2. Upload the new zip. There is no retry button that reuses the last upload: the file has to go
   up again. Changing only an environment variable needs no new zip, just **Deploy**.
3. Check `/healthz`: the `build` must be the new one.
4. Send one test request (step 3 above) whenever the SMTP settings, the mailbox or the host's mail
   changed.

---

## When something is wrong

**The app will not start.** Read **Runtime logs**. A configuration problem says so in one line,
naming the key: `Bad config: SESSION_SECRET: SESSION_SECRET must be at least 32 characters`.

**Visitors see "We could not send this just now."** The site could not hand the email to the mail
server, twice. Nothing was lost on the visitor's side (what they typed is still in the form), but
nothing reached the office either. The runtime log has one line per failed submission:

```
office alert not sent ref=REQ-20261031K7QX code=EAUTH
```

The code says why: `EAUTH` is a wrong `SMTP_USER` or `SMTP_PASS`, `ECONNECTION` or `ETIMEDOUT`
is the mail server not answering at `SMTP_HOST`/`SMTP_PORT`, and a number such as `550` is the
mail server refusing the message (a full or missing mailbox is the usual cause). Fix the setting,
deploy, and send a test request. The line carries no name, address or message text on purpose.

**A receipt never arrives but the office's email does.** The sender's address bounced or went to
spam. The log has `receipt not sent ref=... code=...` when the mail server refused it outright.
The submission itself is fine.

**Requests stop arriving but no visitor complains.** Look at the mailbox's own spam folder, then
at `NOTIFY_EMAIL`. A visitor sees a success only after the mail server accepted the message, so
the site's side of it is done by then.

**Every visitor is told they sent too many.** `TRUST_PROXY` is `0` or missing, so everyone shares
one address. Set it to `1`.
