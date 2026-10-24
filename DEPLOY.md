# Deploying Primefield Property Services to Hostinger

One Node process serves the whole site: the public pages, the back office at `/admin`,
and the forms. There is no build step on the server and nothing to compile. A deploy is
an upload, a migration and a restart.

Everything below is done in hPanel unless it says otherwise.

**There are two routes onto Hostinger, and hPanel decides which one you get.** Sections 1
to 9 are the classic route: **Advanced > Node.js**, where you create the application
yourself, upload into a folder and run commands in a terminal. Newer accounts do not have
that screen at all. They have **Websites > Setup > Node.js web app**, which takes the zip
and runs everything itself, with no terminal anywhere. If that is what you see, skip to
**Deploying with Hostinger's Node.js web app (upload flow)** further down and follow that
instead. The zip is the same file either way.

---

## What you need before you start

- The release zip, `primefield-v<version>-<date>.zip`, from `npm run package`.
- The domain pointed at Hostinger.
- Twenty minutes for the first deploy. Five for every one after it.

---

## 1. Pre-flight on the database (first deploy only)

The schema uses `TIMESTAMP` columns that must stay empty until something writes to them.
MySQL only behaves that way when one server setting is on. **`node server/index.js --check`
in step 7 enforces this and refuses to pass without it**, so this section is the
explanation rather than a step you have to remember. To see the value for yourself,
open **Databases > phpMyAdmin** (or the hPanel SQL console) and run:

```sql
SELECT @@explicit_defaults_for_timestamp;
```

It must come back **1** (ON).

If it comes back **0**, stop. Do not migrate. Every nullable timestamp in the schema is
stamped with the current time the moment a row is written instead of staying empty, and
two things are broken from the first row, with nothing anywhere reporting an error:

- **Password reset stops working entirely.** Each reset link is recorded as already used
  as soon as it is created, so the link in the email is refused as expired before anyone
  opens it. There is no sign up page, so a lost password with no working reset is a back
  office nobody can get into.
- **Every lead from the website arrives marked as contacted**, because `leads.contacted_at`
  is stamped on insert. Real requests drop straight out of the follow-up list.

Raise a support ticket asking for `explicit_defaults_for_timestamp = ON`, or report it
back before going any further.

---

## 2. Create the database and its user

**Databases > Management > Create a new database**

- Database name: something like `u123456_upm`
- User: `u123456_upm`
- Password: generate a long one and keep it

hPanel prefixes both with your account number. Write down the three values exactly as
shown, they go into `DATABASE_URL` in a moment.

The host is `localhost` and the port is `3306` unless hPanel tells you otherwise on the
same screen.

---

## 3. Create the Node application

**Advanced > Node.js > Create application**

| Field | Value |
| --- | --- |
| Node version | 20 or newer. The build targets Node 20. |
| Application root | the folder the zip will be extracted into, for example `domains/primefieldpropertyservices.com/app` |
| Application URL | the domain itself, `primefieldpropertyservices.com` |
| Application startup file | `server.js`. It works whether the platform runs it or requires it; `server/index.js` only works when it is run. |

Leave it created and stopped for now. There is nothing in the folder yet.

**If the domain cannot be attached to the Node application.** Some hPanel plans will not
let a Node app take a domain that is already serving a website, and the Application URL
dropdown then shows no usable entry. Attach a subdomain instead, for example
`app.primefieldpropertyservices.com`, finish the whole deploy against it, and point the
domain at it afterwards. One rule while you do that: `SITE_URL` in the environment must
be the address people actually visit. It is what the reference links in the office
emails are built from, so a mismatch sends the owner to a page that does not exist.

---

## 4. Set the environment

Still on the Node.js screen, under **Environment variables**, add:

| Name | Value |
| --- | --- |
| `PORT` | whatever hPanel assigned the app. If it did not, `3000`. |
| `DATA_DIR` | an absolute path **outside** the application root, for example `/home/u123456/primefield-data` |

`DATA_DIR` is where uploads, backups and the real configuration live. It sits outside the
app root on purpose: a deploy replaces the app root wholesale, and nothing anybody sent
through the website may be inside something that gets replaced.

Create that folder in **Files > File Manager** if it is not there.

---

## 5. Put `.env` inside `DATA_DIR`

Copy `.env.example` from the project, fill it in, and upload it into `DATA_DIR` as
`.env`. Nothing in the zip carries a secret; this file is the only place they live.

```
NODE_ENV=production
PORT=3000
SITE_URL=https://primefieldpropertyservices.com
DATA_DIR=/home/u123456/primefield-data
DATABASE_URL=mysql://u123456_upm:THE_PASSWORD@localhost:3306/u123456_upm
SESSION_SECRET=<64 random hex characters>
SMTP_HOST=smtp.hostinger.com
SMTP_PORT=465
SMTP_USER=noreply@primefieldpropertyservices.com
SMTP_PASS=<the mailbox password>
MAIL_FROM="Primefield Property Services <noreply@primefieldpropertyservices.com>"
NOTIFY_EMAIL=info@primefieldpropertyservices.com
PUBLIC_SITE_MODE=served
TRUST_PROXY=1
```

Notes on the three that are easy to get wrong:

- `SESSION_SECRET` must be at least 32 characters. Generate it, never type it. Changing
  it signs everyone out, which is the correct thing to do if it ever leaks.
- `DATABASE_URL` needs the password URL encoded if it contains `@`, `:`, `/` or `#`.
  Generating a password without those characters is easier than escaping one.
- `TRUST_PROXY=1` says there is exactly one proxy in front of the app, which is what
  Hostinger runs. It is how the rate limits see a real visitor's address instead of the
  proxy's. Do not raise it: a larger number lets a visitor claim any address they like.
- `NOTIFY_EMAIL` is the mailbox every request and every vendor application is emailed to
  as it arrives. It is set here and nowhere else; the back office shows it on the
  Settings screen, under Alerts, so the owner can check where the mail is going without
  opening a terminal, but it cannot be changed from there. Changing it means editing this
  file and restarting.

The rate limits themselves are not in this file. The production numbers are built in, and
`.env.example` lists the keys only so a test run can lift them. Leave that block out.

---

## 6. Upload and extract the zip

**Files > File Manager**, open the application root, upload the zip, then extract it
there. It unpacks straight into the folder: `server/`, `site/`, `admin/`, `drizzle/`,
`scripts/`, `node_modules/`, `package.json`, `DEPLOY.md`.

Delete the zip afterwards so the next upload is not confused with it.

`node_modules` is already inside. Do not run `npm install` on the host: the tree was
installed and audited on a machine with a toolchain, and nothing in it needs compiling.

---

## 7. Migrate, then seed the owner account

Open the app's terminal (the **Run command** box on the Node.js screen, or SSH into the
application root) and run, in this order:

```bash
node server/index.js --check
npm run db:migrate
```

`--check` is the pre-flight. It loads the configuration, opens the database, runs
`SELECT 1`, checks that `explicit_defaults_for_timestamp` is on (section 1), prints `OK`
and exits. It binds no port and writes nothing, so it is safe to run at any time. If it
prints anything other than `OK`, fix that before going on: the message says which key,
which connection or which server setting is the problem.

`npm run db:migrate` applies the SQL in `drizzle/`. It is safe to run again; it only
applies what has not been applied.

Then, on the first deploy only, create the one back office login:

```bash
npm run seed:admin
```

It asks for a name, an email address, a username and a password of at least 12
characters. There is no sign up page and no second account: this is the only way in.
The username is what gets typed on the sign in screen, and pressing return at that
prompt takes the one it suggests from the address. If the terminal cannot prompt, set
`SEED_NAME`, `SEED_EMAIL`, `SEED_USERNAME` and `SEED_PASSWORD` and run it again;
`SEED_USERNAME` is the only optional one of the four.

A username is 3 to 32 characters, lowercase letters, numbers or underscores. The email
address is still the account's address, and it is where a password reset link is sent.

Running it later with the same email changes that account's password, which is the way
back in if the owner is ever locked out.

---

## 8. Restart

**Advanced > Node.js > Restart application.**

**Restart after every deploy, without exception.** The app reads its configuration once,
runs the migrations once and caches the built index at boot. A new zip on disk changes
nothing at all until the process that is serving the old one is replaced.

---

## 9. Verify

In a browser, in this order:

| URL | What you should see |
| --- | --- |
| `/healthz` | `{"ok":true}`, with a `build` stamp (the short commit and the time this bundle was built) and `"storage":"persistent"`. If it says `"storage":"temporary"`, photos are being saved inside the app folder, which every upload replaces: fix `DATA_DIR` before anyone sends a photo. If it says `"storage":"unwritable"`, the site cannot write to its photo folder at all: see the note under `DATA_DIR` below. |
| `/` | the home page, with photographs and the red buttons |
| `/services` | the eight services, not a redirect |
| `/admin` | the sign in screen, navy with the logo on a white card, asking for a username |

Then sign in with the username and password from step 7, and send one request through
`/request-service` to prove the whole path: the form takes it, the thanks page gives a
reference number like `REQ-000001`, the office mailbox gets the notification, and it
appears in the Inbox while you are looking at it, without a refresh.

If `/admin` says **Admin build missing**, the zip was extracted incompletely. Extract it
again and restart.

**Confirm the app is seeing real visitor addresses.** `TRUST_PROXY=1` is what turns the
proxy's forwarded header into `req.ip`, and every rate limit, the lockout email and the
honeypot warning in the log carry that address. Get it wrong and it is silent: too low
and every visitor looks like the proxy, so one person's mistyped password throttles
everybody; too high and a visitor can claim any address they like and never be throttled
at all. To check it, sign in wrongly once from a phone on mobile data, then look at the
Node.js screen's log. Nothing is printed for a single failure, so the quicker proof is
the honeypot line: it is logged as `honeypot hit { kind: ..., ip: ... }` and that `ip`
must be the visitor's own address, not the proxy's and not one anybody sent in a header.
If it reads as a Hostinger internal address, the hop count is too low.

---

## Deploying with Hostinger's Node.js web app (upload flow)

This is the route for an hPanel that has no **Advanced > Node.js** screen. You upload the
zip, the platform installs it, builds it and starts it, and there is no terminal at any
point. Nothing is missing compared with the classic route; the two things the terminal
used to do, the migration and the first account, are both done by the app itself at boot.

### 1. Create the app and upload the zip

**Websites > Setup**, choose **Node.js web app**, and upload `primefield-v<version>-<date>.zip`.
Hostinger extracts it and shows **Review build settings**.

| Field | Value |
| --- | --- |
| Framework | **Express** |
| Node version | **24.x**. Not lower than 22.12 on this route: see the note below the table. |
| Root directory | `./`. The zip extracts straight into the app root; there is no folder inside it to point at. |
| Build settings | Leave them exactly as they arrive. The install and build commands the preset fills in are the right ones. |

The preset's build command is `npm run build`, and in this release that script prints
`already built` and exits. There is genuinely nothing to compile: the zip carries the
finished bundle and its `node_modules`, installed and audited on a machine with a
toolchain. The script exists because the preset insists on one, and a deployment that
cannot find it fails with an install log in which nothing looks wrong.

**How the platform starts the app.** It does not run `npm start`. The web server there
is LiteSpeed, and its `lsnode.js` loads the application by `require()`ing the entry file
at the root of the extracted zip. That is what `server.js` is for: it is four lines of
code, and it exists only so there is something at the root that can be required. If the
**Runtime logs** screen fills with `Error [ERR_REQUIRE_ASYNC_MODULE]: require() cannot
be used on an ESM graph with top-level await`, repeating every couple of seconds, the
uploaded zip is an older one. Upload `v0.2.3` or later and the message goes away. Note
that the **Node version** in the table above has to be 22.12 or newer for this to work
at all, which is why it says 24.x.

### 2. Environment variables

Same screen, **Environment variables**. Add one row per line of section 5 above:

| Name | Value |
| --- | --- |
| `NODE_ENV` | `production` |
| `SITE_URL` | `https://primefieldpropertyservices.com` |
| `DATA_DIR` | `/home/<hosting user>/primefield-data`. Read the note below. |
| `DATABASE_URL` | left until step 4, when the database exists |
| `SESSION_SECRET` | 64 random hex characters |
| `SMTP_HOST` | `smtp.hostinger.com` |
| `SMTP_PORT` | `465` |
| `SMTP_USER` | `noreply@primefieldpropertyservices.com` |
| `SMTP_PASS` | the mailbox password |
| `MAIL_FROM` | `Primefield Property Services <noreply@primefieldpropertyservices.com>` |
| `NOTIFY_EMAIL` | `info@primefieldpropertyservices.com` |
| `PUBLIC_SITE_MODE` | `served` |
| `TRUST_PROXY` | `1` |

**Do not add `PORT`.** This flow injects it, and a value typed here would be the wrong
one.

There is no `.env` file on this route. These rows are the configuration, and the four
notes in section 5, on `SESSION_SECRET`, `DATABASE_URL`, `TRUST_PROXY` and
`NOTIFY_EMAIL`, apply to them word for word.

**`DATA_DIR` has to be a folder outside the application, and this is the one setting on
this route that fails quietly.** The application folder is replaced on every upload, so a
`DATA_DIR` inside it (`data` is the usual mistake) works perfectly: the site runs and every
photograph anyone sends is stored. And the next upload of the site erases all of them. Set
it to `/home/<hosting user>/primefield-data` instead, where `<hosting user>` is the account
name in the path hPanel shows for the site (`/home/u123456789/...`). The server creates the
folder itself the first time a photo is saved, so there is nothing to make by hand. The
database is untouched by this either way: a request or a work order never disappears, only
its photographs would.

The server checks this for you. While `DATA_DIR` is inside the application folder:

- the log prints `storage: temporary, photos will be lost on the next upload: set DATA_DIR to a folder outside the app` at every start;
- `/healthz` answers `"storage":"temporary"`;
- the back office shows an amber bar on every screen, and Settings explains the fix.

After you change the value and deploy, open `/healthz` and look for `"storage":"persistent"`.
That is the confirmation, and the bar disappears with it. Do it before the first real
photograph: photos saved in the old folder do not move to the new one.

The server also tries the folder the way a photograph would be saved: at every start it
makes `DATA_DIR/uploads`, writes a few bytes into it and removes them again. If it cannot,
because the host gives the site no rights to that folder or the disk is full:

- the log prints `storage: unwritable, photos cannot be saved: the uploads folder under DATA_DIR cannot be written to` followed by the reason in brackets, for example `(EACCES)`;
- `/healthz` answers `"storage":"unwritable"`, which is said before `temporary` because nothing is saved at all;
- the back office shows the amber bar "Photos cannot be saved right now: the photo folder cannot be written to. See Settings.", and Settings explains the fix.

Give the site's user rights to the folder (or point `DATA_DIR` at one it owns), then open
Settings: the folder is tried again each time that page is opened, so the bar goes the
moment it can be written to, with no restart.

### 3. Deploy

Press **Deploy**. The first deployment installs and starts. It will not serve anything
useful yet, because the database is not connected.

### 4. The database

On the site's dashboard, open **Database > Connect**. It shows the database name, the
user and the host. Build the URL from them:

```
mysql://USER:PASSWORD@localhost:3306/DATABASE
```

The host is `localhost` unless the **Connect** screen names a server, in which case use
the name it gives. URL encode the password if it contains `@`, `:`, `/` or `#`.

Put that value in `DATABASE_URL` under **Environment variables**, and deploy again.

**There is no migration step here.** The app applies every migration it has not applied
at boot, so the deploy that first reaches the database is the deploy that builds the
schema. That has always been true; on this route it is the only path, because
`npm run db:migrate` needs a terminal and there is not one.

**The one thing section 1 asks you to check by hand is still on you.** The pre-flight
that enforces it, `node server/index.js --check`, is a command, and commands need a
terminal. Run the `SELECT @@explicit_defaults_for_timestamp;` from section 1 in
**Databases > phpMyAdmin** before the first deploy and confirm it comes back **1**. It
takes ten seconds, and section 1 explains what is silently broken if it comes back 0.

### 5. Create the owner account (first deploy only)

There is no sign up page and no terminal to run `npm run seed:admin` on, so the server
creates the first account itself, from the environment. Under **Environment variables**
add:

| Name | Value |
| --- | --- |
| `SEED_EMAIL` | the account's email address, where a password reset link would be sent |
| `SEED_PASSWORD` | a password of at least 12 characters |
| `SEED_USERNAME` | the name you will type to sign in, for example `vinc9nt` |
| `SEED_NAME` | optional, the name shown in the back office. `Owner` if you leave it out. |

`SEED_USERNAME` is optional as well: left out, the part of the address before the `@` is
used, lowered, with anything a username may not hold removed. Set it whenever that is
not a name you would want to type. A username is 3 to 32 characters, lowercase letters,
numbers or underscores.

Deploy again. The runtime log prints `seeded owner <your username> (<your address>)`.
Sign in at `/admin` with that username, then **remove those variables and deploy once
more**. They have done their job and a password does not belong in a settings screen a
moment longer than it must.

Three things this deliberately will not do:

- It only ever runs when there is no account at all. Once one exists, the variables are
  read and ignored, and the log says so. It is not a way to change a password, so leaving
  the variables set by accident cannot overwrite the account, and neither can anyone who
  reaches that screen later.
- It refuses a password shorter than 12 characters or longer than 200, an address that
  is not one, and a username outside the rule above, and it says which in the runtime
  log. It does not stop the site coming up: correct the value and deploy again.
- It creates one account. If you are ever locked out afterwards, the way back in is the
  **Forgot your password?** link on the sign in screen, which emails a reset link through
  the same mailbox as everything else.

### 6. Reading the logs

The site's **Runtime logs** menu is where the running app talks. The deployment log,
which is the install and build output, is a different screen and a different thing.

`upm listening on <port> (build <commit> <time>)` is the line that says the app is up.

**A deployment that shows a clean install log and no runtime log at all never started
the app.** That is the symptom this release exists to fix: nothing had gone wrong with
the install, the platform simply never got as far as running anything.

### 7. Redeploying

Upload the zip again. There is no retry button that reuses the last upload, and after a
failed deployment there is nothing on the server to retry: the file has to go up again.

Changing an environment variable and deploying is enough on its own; that path does not
need a new zip.

### 8. Verify

Exactly as section 9 above: `/healthz`, `/`, `/services`, `/admin`, then one real request
through `/request-service`. Then the two header checks under **Security headers on Hostinger**
further down, which show what a visitor's browser is actually given.

---

## Upgrading from v0.1.0

v0.2.0 adds the Inbox to the back office. Nothing on the public site changed, no
configuration key was added or renamed, and no data is thrown away: this is an ordinary
deploy with one migration in it.

1. **Upload and extract `primefield-v<version>-<date>.zip`.** Either over the top of the existing
   folder, or into a new folder that you then point the Node application's Application
   root at. Both work. If you extract over the top, the new files replace the old ones
   and nothing has to be deleted first. Extracting does not put the new version live:
   the back office shell and its assets are read from disk on every request, so a
   browser opening `/admin` now gets v0.2.0's screens, while the old process is still
   the one answering `/admin/api`. Until the restart in step 5 those screens ask for
   addresses the running process does not have, and a 404 from `/admin/api/inbox/*` in
   that window is the old process answering rather than a bad upload.
2. **Leave `DATA_DIR` exactly as it is.** It is outside the application root, it holds
   `.env` and every photograph anyone has ever sent, and no step here touches it. Do not
   move it and do not recreate it.
3. **Check before you migrate.**
   ```bash
   node server/index.js --check
   ```
   It must print `OK`. It reads the configuration, opens the database and checks the one
   server setting the schema depends on, and it writes nothing, so a failure here has
   cost nothing.
4. **Migrate.**
   ```bash
   npm run db:migrate
   ```
   This applies `0001_inbox`, which is four statements: it adds `Done` to the list of
   values a request can hold, adds `Contacted` and `Done` to the list a vendor
   application can hold, and adds a `note` column to each of the two tables for the
   owner's private notes. Nothing is dropped and nothing is rewritten, so every request
   and application already in the database keeps exactly what it had. Running it again
   does nothing: it only applies what has not been applied.
5. **Restart the application.** As always. The app reads its configuration and applies
   any migration it has not applied at boot, so the code answering `/admin/api` and
   `/api` is the old code until the old process is replaced. This is the step that
   closes the window step 1 opened.
6. **Verify.** `/healthz` should show the new `build` stamp. Sign in at `/admin`: the
   screen after signing in is the **Inbox**, with two tabs, Requests and Vendor
   applications, and everything the website has ever collected already in them. Open one
   and the buttons at the bottom mark it New, Contacted or Done.

Step 4 is not the only thing that applies `0001`. Migrations also run when the app
boots, so the restart in step 5 would apply it on its own. Step 4 is here so that a
migration that cannot run is seen while the old process is still serving and the
database is still untouched, rather than as a server that does not come back up. Run
it anyway.

If a database were somehow left on `0000`, the Inbox would not merely lose two buttons.
Every list, every item and both spreadsheets read the `note` column by name, so all of
them would fail, and the Inbox would show the line it shows when a list does not load.
The fix is the same one either way: `npm run db:migrate`, then restart.

---

## Upgrading to v0.2.2: signing in with a username

v0.2.2 changes what gets typed in the first box on the sign in screen. It was the email
address; it is now a username. This is an ordinary deploy with one migration in it,
`0002_username`, and it takes nothing away: the email address is still the account's
address and still where a password reset link is sent, and typing it on the sign in
screen still works.

**Your username after the upgrade is the part of your email address before the `@`,
lowered, with anything a username may not hold removed.** For
`owner@primefieldpropertyservices.com` that is **`owner`**. The migration works it out
and fills it in for every account already in the database; nothing has to be set by hand
and no environment variable is needed for an account that already exists.

The steps are the ordinary ones: upload and extract the zip, `node server/index.js
--check`, `npm run db:migrate`, restart, then sign in at `/admin` with the username. On
the Node.js web app route there is no terminal, so the deploy and the restart are the
whole of it and the migration runs at boot.

To change the username afterwards, sign in and open **Settings**. The Username card
shows the one in use, and **Change** asks for a new one and for the current password.

---

## Upgrading to v0.6.0: the calls and texts box

v0.6.0 adds one required checkbox to both public forms and keeps the answer. This is an ordinary
deploy with one migration in it, `0003_contact_consent`, and it takes nothing away: it adds three
empty columns to the requests table and three to the vendor applications table, and every row
already there is left exactly as it was.

On the Node.js web app route there is no terminal, so the upload and the restart are the whole of
it: the app applies `0003` by itself at boot, before it answers a single request. Where there is a
terminal the steps are the ordinary ones below.

Afterwards, send one test request from the website with the box ticked, open it in `/admin`, and
the last line of its details reads **Calls and texts: Agreed on** today's date. A request from
before the upgrade reads **Not recorded. Sent before this box was added.**

---

## Upgrading to v0.6.1: lighter photographs on phones

v0.6.1 changes one thing: the photograph beside the title of each service page, and the same
photographs on the service cards, download at about half the weight on a phone. Nothing looks
different, nothing on any page moved, and a desktop is sent exactly the files it was sent before.
There is no migration and nothing to set. Upload, extract, restart, as for any deploy.

The new photograph files have new names and the old ones are still in the zip under their old
names, so a browser or a CDN holding yesterday's pages keeps working until it fetches today's.
Nothing needs to be purged.

To see it, open a service page on a phone: it looks as it did. The check that matters was made
before the zip was built, and it is in `docs/DECISIONS.md` under 2026-10-05.

---

## Upgrading to v0.6.2: photographs named correctly, and one cache to clear

v0.6.2 changes one header. Every AVIF photograph used to leave the server labeled as a plain
download (`application/octet-stream`); it is now labeled `image/avif`. Nothing looks different and
no file changed. There is no migration and nothing to set. Upload, extract, restart.

**Then clear the CDN cache once.** Hostinger's CDN keeps its own copy of each photograph together
with the label it first arrived under, and it keeps it for up to a year. The file names are the
same as in v0.6.1, so the CDN goes on serving its old copy with the old label until its cache is
cleared from the CDN page in hPanel.

To check, after the restart and the clear:

```
curl -I https://primefieldpropertyservices.com/_astro/<any file ending in .avif>
```

The answer should carry `Content-Type: image/avif`. A file name to try is in the page source of
any service page. If it still says `application/octet-stream`, the CDN is answering from its old
copy: clear it again, or wait a minute and ask once more.

---

## Upgrading to v0.6.3: the security rules inside every page

v0.6.3 changes nothing you can see. Every page now carries its security rules inside the page
itself, because Hostinger replaces the copy the server sends alongside it. The section **Security
headers on Hostinger** below says what that means.

This zip also carries v0.6.2, which was packaged and never uploaded, so one upload delivers both.
There is no migration and nothing to set. Upload, extract, restart, as for any deploy.

**Then clear the CDN cache once.** This is the same single clear v0.6.2 asks for, and one clear
covers both releases. Hostinger's CDN keeps its own copy of each page as well as of each
photograph, and a copy of a page it took before this release has no rules inside it. Clear the
cache from the CDN page in hPanel and it fetches the new pages.

To check, after the restart and the clear, follow **How to check after a deploy** in the next
section. The v0.6.2 check above still applies too.

---

## Upgrading to v0.7.0: the work orders

This is the largest upgrade since the first one: the back office gains work orders, clients,
properties, vendors, a spreadsheet import and the sample data. The zip is deployed like any other.
What is different:

1. **The database changes by itself.** Migration `0004_work_orders` creates the new tables and adds a few
   columns to three old ones (nothing is dropped, renamed or rewritten, and it is safe on a database that
   has requests, applications and photographs in it). It runs at boot, and the twelve starting work types
   are put in the first time. There is nothing to run by hand on the Node.js web app route.
2. **Set `DATA_DIR` before anyone adds a photograph.** Photographs of work orders are kept in
   `DATA_DIR/uploads`, and the app folder is replaced by every upload of the site. If `DATA_DIR` is not
   outside the app folder, a bar across every screen of the back office says so, and **Settings** has a
   **Photo storage** card that says what to set. The same is readable without signing in at
   `/healthz`: `"storage":"persistent"` is right, `"temporary"` means the photographs will be erased by
   the next upload, and `"unwritable"` means the site cannot save a photograph at all.
3. **Check it after the restart.** Sign in. The app opens on **Work orders**. Open **Settings**: the
   **Photo storage** card should say photographs are in a safe folder outside the site. Press **Show sample
   data** to see every screen with something on it, and **Remove sample data** when you are done.
4. **The old name.** Nothing in this version says the old company name. If a proof is wanted, search the
   extracted zip for it: there should be no match.

If it goes wrong, the previous zip still works with the new database: the new tables are simply unused.

---

## Security headers on Hostinger

**What the host replaces.** With every page, the server sends the browser a set of rules called
the content policy: load scripts, styles, fonts and pictures from this site only, never run a
script written into a page, never send a form anywhere else. It travels as a response header
named `Content-Security-Policy`. Hostinger's web server sets a header of the same name by itself,
with the single value `upgrade-insecure-requests`, and that replaces the one the server sent. The
server's other security headers arrive untouched.

**What the site does about it.** From v0.6.3 the same rules are also written into each page, as
one line near the top of the page source. The host cannot replace that line, because it is part
of the page. The server still sends its header as before, so if Hostinger ever lets it through,
a browser simply receives the same rules twice and nothing needs changing. One rule cannot be
written into a page: the one that stops another website from showing this site inside a frame.
That one is carried by a different header, `X-Frame-Options`, which Hostinger does pass through,
and it is now set to `DENY`.

**How to check after a deploy.**

1. The headers a visitor receives:

   ```
   curl -sI https://primefieldpropertyservices.com/ | grep -i -E "content-security-policy|x-frame-options"
   ```

   `x-frame-options: DENY` is the line to look for. Before v0.6.3 it read `SAMEORIGIN`, so `DENY`
   also tells you the new release is the one answering. The other line will still read
   `content-security-policy: upgrade-insecure-requests`. That is Hostinger's header, it is
   expected, and it is the reason the rules are now inside the page.

2. The rules inside the page:

   ```
   curl -s https://primefieldpropertyservices.com/ | head -c 600
   ```

   Straight after `<meta charset="utf-8">` there should be a line that starts
   `<meta http-equiv="Content-Security-Policy" content="default-src 'self';`.

3. The same check without a terminal: open the site in a browser, right click the page and
   choose **View page source**. The line is the second one inside `<head>`. It is there on
   `/admin` too.

4. If the page has no such line, or the header still reads `SAMEORIGIN`, the CDN is answering
   from an old copy. Clear its cache again, wait a minute and ask once more. `/healthz` shows
   which build the server itself is running.

**There is nothing to set in hPanel.** Hostinger offers no setting for this. The CDN page has no
option for response headers, and the one file that could carry such a rule is written by
Hostinger and rewritten on every deploy, so an edit to it would not last. The site does not
depend on one: the rules inside the page do the work.

If you would still like the header itself put right on their side, this is the question to send
Hostinger support. It is optional.

> My site primefieldpropertyservices.com is a Node.js web app on your hosting. Every response
> carries the header `Content-Security-Policy: upgrade-insecure-requests`, and it replaces the
> `Content-Security-Policy` header my application sends. Can you turn that header off for this
> domain, or make the server keep my application's own header? If so, will the change survive a
> redeploy of the app?

---

## Every deploy after the first

1. `npm run build && npm run package` on the development machine.
2. Upload and extract the new zip into the application root.
3. `node server/index.js --check`, then `npm run db:migrate`. The check refuses to pass
   on a database server whose `explicit_defaults_for_timestamp` is off, so a host that
   quietly changes that setting is caught before the restart rather than months later.
4. Restart the application.
5. Check `/healthz`, `/` and `/admin`. The `build` on `/healthz` is how you confirm the
   restart picked up the new upload rather than the bundle it was already running.
6. Run the two header checks under **Security headers on Hostinger** above. They take a minute
   and show what a visitor's browser is actually given.

`DATA_DIR` is never touched by a deploy, so uploads and the configuration survive it.
`npm run seed:admin` is a first deploy step and a password reset, never a routine one.

---

## When something is wrong

**The app will not start.** Read the Node.js screen's log. A configuration problem says
so in one line, naming the key: `Bad config: SESSION_SECRET: SESSION_SECRET must be at
least 32 characters`. A database problem shows as a connection error from the migration
that runs at boot. `node server/index.js --check` gives you the same answer without
restarting anything.

**The site loads but the forms do not send.** Check `SITE_URL` matches the address in the
browser bar, and that the mailbox in `SMTP_USER` exists and its password is current. A
form that is accepted but sends no email still stores the request: it is in the back
office either way.

**Photographs do not appear on a request.** `DATA_DIR` is not writable, or it is wrong. `/healthz` says `"storage":"unwritable"` while it is not.
The request itself is never lost to this: the server stores the lead first and the photos
after, on purpose.

**Everyone is signed out after a deploy.** `SESSION_SECRET` changed. Put the old value
back if you still have it.
