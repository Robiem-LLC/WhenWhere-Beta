# WhenWhere beta

WhenWhere puts your family on one map, remembers every trip, and tells you when someone gets home. It runs on a server you own, and every location is encrypted on the phone before it leaves, and stays encrypted on your server. Nothing passes through us.

This repository is the home of the public beta: how to set it up, and where to report problems. The app's source code is not published here yet. It will be open source shortly; it just needs a good scrub first (profanity-laden comments included).

---

## Contents

1. [What you need](#1-what-you-need)
2. [Start the server](#2-start-the-server)
3. [Give it a secure address](#3-give-it-a-secure-address)
4. [Create your household](#4-create-your-household)
5. [Connect email](#5-connect-email)
6. [Trust your browser](#6-trust-your-browser)
7. [Invite your family](#7-invite-your-family)
8. [Connect the phones](#8-connect-the-phones)
9. [Keeping it running](#9-keeping-it-running)
10. [Optional: Google Maps in the web app](#optional-google-maps-in-the-web-app)
11. [Troubleshooting](#troubleshooting)
12. [Reporting a problem](#reporting-a-problem)

Allow about 30 minutes, once.

---

## 1. What you need

- **Docker** on a computer that stays on, with an Intel or AMD (x86-64) processor. ARM boards such as the Raspberry Pi aren't supported yet.
- **A domain to serve on is a must.**
- **An email account that can send mail by SMTP**: WhenWhere emails invitations and the codes that register each phone.
- **Phones:** iPhone with iOS 17 or later, or Android 8.0 or later.

## 2. Start the server

The server is two containers: WhenWhere and its PostgreSQL database, described in [`whenwhere-stack.yaml`](whenwhere-stack.yaml).

1. In Portainer, open **Stacks → Add stack → Web editor** and paste the contents of `whenwhere-stack.yaml`.
2. Under **Environment variables**, add:

   | Name | Value |
   | --- | --- |
   | `WHENWHERE_IMAGE` | `ghcr.io/robiem-llc/whenwhere` |
   | `WHENWHERE_PUBLIC_URL` | The exact address you'll use in step 3, such as `https://family.example.com` |
   | `WHENWHERE_DB_PASSWORD` | A long random password. Never change it after the first start. |
   | `WHENWHERE_GOOGLE_MAPS_API_KEY` | Optional!! See [the section about creating one](#optional-google-maps-in-the-web-app). |

3. Click **Deploy the stack**.

The server listens on port `8186` with plain HTTP, so **don't forward that port on your router**; put your tunnel or reverse proxy in front of it (step 3).

<details>
<summary>Using Docker Compose instead of Portainer</summary>

Save `whenwhere-stack.yaml` as `compose.yaml` in a new folder, create a `.env` file beside it with the three variables above (one `NAME=value` per line), then run:

```sh
docker compose up -d
```

</details>

## 3. Give it a secure address

The phone apps only connect over HTTPS. A Cloudflare Tunnel gives you that without opening any ports or revealing your home IP address.
Reverse proxy or Cloudflare Tunnel your heart out. Tunnels are pretty idiot proof though.

Whichever you pick, point it at port `8186`, and make sure the address exactly matches `WHENWHERE_PUBLIC_URL`.

## 4. Create your household

1. Open your address in a browser. You'll see **Make this server yours.**
2. Find the one-time setup token: in Portainer, open the stack's **whenwhere** container and choose **Logs**. Look for the line `WHENWHERE FIRST-RUN SETUP TOKEN`.
   If you lose this token it can also be obtained by running this in the container's **Console** (connect with `/bin/sh`):

   ```sh
   python manage.py bootstrap-token
   ```

   With Docker Compose: `docker compose exec whenwhere python manage.py bootstrap-token`.
3. Enter the token, a household name, your email and a password of at least 15 characters. You become the household's administrator, and setup closes for good.

## 5. Connect email

1. Sign in. On **Connect your mail server.**, enter your provider's SMTP host, port, security (STARTTLS or TLS), username, password and the sender address.
2. Send the test email, then confirm your own address from the email you receive.

Many providers require an app password rather than your normal password for SMTP. The server stores these settings encrypted.

## 6. Trust your browser

Locations are encrypted for specific devices, so your browser needs its own keys before it can show the map. This browser becomes the household's first trusted device; phones join later by emailed code, and other browsers are approved from a trusted one.

1. On **Approve your first device.**, the page asks for a device bootstrap token (a different token from step 4).
2. In Portainer, open the **whenwhere** container, choose **Console**, connect with `/bin/sh` and run:

   ```sh
   python manage.py device-bootstrap-token
   ```

   With Docker Compose: `docker compose exec whenwhere python manage.py device-bootstrap-token`.
3. Paste the token and choose **Trust this first device**. Use a browser that only you use.

## 7. Invite your family

1. Open **Manage household** and enter each person's email address to invite them.
2. They get an email to choose a username and password. They can also finish this in the app with **Set up invited account**.

## 8. Connect the phones

**Install the app**

- **iPhone:** [join the beta on TestFlight](https://testflight.apple.com/join/1MBkvkUK). Install Apple's TestFlight app if asked, then install WhenWhere from it.
- **Android:** [download the APK](https://github.com/Robiem-LLC/WhenWhere-Beta/raw/main/android/WhenWhere-Android-beta.apk), open it on the phone and allow the install when Android asks. Build details and checksum are in [`android/`](android/README.md).

**Sign in and register the phone**

1. Enter your server address (for example `https://family.example.com`), your email and password, and sign in.
2. Request a code: **Email my six-digit code** on iPhone; on Android, open **Settings** and choose **Email verification code**.
3. Enter the six-digit code from the email. The phone joins the household right away; no other approval is needed.
4. Allow location **Always**, so the phone can share while the app is closed, and allow notifications for arrival alerts and Find My Phone.
5. Under **Who can see this phone?**, pick who in the household sees your location.

That's it. Everyone who shares with you appears on your map, and trips start recording automatically.

## 9. Keeping it running

- **Updates:** pull the latest image and update your container (in Portainer: **Update the stack** with **Re-pull image** on). Your data stays in place.
- **Backups:** your data lives in `/opt/docker/whenwhere` (or your `WHENWHERE_DIR`). Back up `server-data` and `database-data` together: `server-data` holds the server's encryption key, and without it the database can't be restored.
- **Never** change `WHENWHERE_DB_PASSWORD` after the first start, and never delete `server-data`.

---

## Optional: Google Maps in the web app

The web app uses OpenStreetMap unless you give the server a Google Maps key. The iPhone and Android apps already include their own maps; this only changes the web app.

**1. Create the key**

1. In the [Google Cloud console](https://console.cloud.google.com/), create a project (or pick one) and turn on billing. Google charges only above its monthly free allowance; see [Google Maps pricing](https://developers.google.com/maps/billing-and-pricing/pricing).
2. Under **APIs & Services → Library**, enable:
   - **Maps JavaScript API**: the map itself.
   - **Geocoding API**: address search when saving a place, and street names.
   - **Maps Static API** (optional): map backgrounds on trip cards; without it they use OpenStreetMap.
3. Under **APIs & Services → Credentials**, choose **Create credentials → API key**.

**2. Restrict it**

Edit the new key:

- **Application restrictions:** choose **Websites** and add your address with `/*` on the end, for example `https://family.example.com/*`.
- **API restrictions:** choose **Restrict key** and select the APIs you enabled above.

Browsers can see this key, so these restrictions are what stop anyone else from using it. Setting a daily quota under **Quotas** adds a spending cap.

**3. Give it to the server**

1. In Portainer, open the stack and add the environment variable `WHENWHERE_GOOGLE_MAPS_API_KEY` with your key.
2. Update your container
3. Open the web app's **Household map** and choose **Load Google Maps**.

To go back to OpenStreetMap, clear the variable and update the stack again.

---

## Troubleshooting

**The app says the server address could not be found.**
Check the address for typos, then open it in the phone's browser. If the browser can't reach it either, check your chosen path out.

**The browser shows an error about the host or origin.**
`WHENWHERE_PUBLIC_URL` must match the address in the browser exactly, including `https://` and without a trailing slash.

**The six-digit code or an invitation never arrives.**
Check spam, then **Admin settings → Email** and send a test. Codes expire after ten minutes; request a new one if it's late.

**Someone isn't on the map.**
On their phone, check that sharing is on, that location is allowed **Always**, and that **Who can see this phone?** includes you. Parked phones check in about every 15 minutes; a location older than 20 minutes is shown as stale.

**A phone was lost or replaced.**
In **Manage household**, revoke the old phone from that person's card, then register the new phone with a new emailed code.

---

## Reporting a problem

[Open an issue](../../issues/new/choose) in this repository and include the app version (shown in Settings), your phone model and OS version, and what you did and what you expected.
Feel free to make improvement suggestions!!

**Please don't post** your server address, email addresses, tokens, passwords or screenshots that show real locations. If something involves private details, email dev@robiem.com instead.

Or always feel free to hit me up at dev@robiem.com or where we first linked up. (Reddit for most, Cell for the rest.)

---

WhenWhere by Robiem.

Side note. WhenWhere is a working name, so the first name I found that I liked is not going to stay. I will let everyone know what the final name is, and am open to suggestions!
