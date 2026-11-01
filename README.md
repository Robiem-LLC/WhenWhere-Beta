# WhenWhere beta

WhenWhere puts your family on one map, remembers every trip, and tells you when someone gets home. It runs on a server you own, and every location is encrypted on the phone before it leaves, and stays encrypted on your server. Nothing passes through us.

This repository is the home of the public beta: how to set it up, and where to report problems. The app's source code is not published here.

**Website:** https://whenwhere.robiem.com · **Privacy policy:** https://whenwhere.robiem.com/privacy.html

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
10. [Troubleshooting](#troubleshooting)
11. [Reporting a problem](#reporting-a-problem)

Allow about 30 minutes, once.

---

## 1. What you need

- **A computer that stays on, with Docker** and an Intel or AMD (x86-64) processor: a Linux box, a NAS such as Unraid, TrueNAS or an Intel-based Synology, or Windows with Docker Desktop. The server image isn't built for ARM boards such as the Raspberry Pi yet. [Portainer](https://www.portainer.io/) makes it point-and-click; plain Docker Compose works too.
- **A domain on Cloudflare** (the free plan is fine) for a secure HTTPS address without opening ports on your router. Any other HTTPS reverse proxy also works.
- **An email account that can send mail by SMTP**, such as your email provider's SMTP server with an app password. WhenWhere emails invitations and the codes that register each phone.
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

3. Click **Deploy the stack**.

Your data is kept in `/opt/docker/whenwhere` on that computer (set `WHENWHERE_DIR` to use another folder). The server listens on port `8186` with plain HTTP, so **don't forward that port on your router**; step 3 puts HTTPS in front of it.

<details>
<summary>Using Docker Compose instead of Portainer</summary>

Save `whenwhere-stack.yaml` as `compose.yaml` in a new folder, create a `.env` file beside it with the three variables above (one `NAME=value` per line), then run:

```sh
docker compose up -d
```

</details>

## 3. Give it a secure address

The phone apps only connect over HTTPS. A Cloudflare Tunnel gives you that without opening any ports or revealing your home IP address.

1. In the Cloudflare dashboard, open **Zero Trust → Networks → Tunnels** and create a tunnel. Cloudflare shows a command to install its connector; run it on the same computer (the Docker option works well in Portainer too).
2. Add a **public hostname** to the tunnel, such as `family.example.com`. Set the service to **HTTP** and the URL to `localhost:8186`, or `<that computer's IP>:8186` if the connector runs in Docker.
3. Make sure the hostname exactly matches `WHENWHERE_PUBLIC_URL`. If it doesn't, update the variable and redeploy the stack.

## 4. Create your household

1. Open your address in a browser. You'll see **Make this server yours.**
2. Find the one-time setup token: in Portainer, open the stack's **whenwhere** container and choose **Logs**. Look for the line `WHENWHERE FIRST-RUN SETUP TOKEN`.
3. Enter the token, a household name, your email and a password of at least 15 characters. You become the household's administrator, and setup closes for good.

## 5. Connect email

1. Sign in. On **Connect your mail server.**, enter your provider's SMTP host, port, security (STARTTLS or TLS), username, password and the sender address.
2. Send the test email, then confirm your own address from the email you receive.

Many providers require an app password rather than your normal password for SMTP. The server stores these settings encrypted.

## 6. Trust your browser

Locations are encrypted for specific devices, so your browser needs its own keys before it can show the map.

1. On **Approve your first device.**, the page asks for a device bootstrap token.
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

- **iPhone:** join the beta through TestFlight, Apple's beta app. The public TestFlight link will be posted here and on the website as soon as Apple approves the beta.
- **Android:** download the latest APK from this repository's [Releases](../../releases) page, open it on the phone and allow the install when Android asks.

**Sign in and register the phone**

1. Enter your server address (for example `https://family.example.com`), your email and password, and sign in.
2. Request a code: **Email my six-digit code** on iPhone; on Android, open **Settings** and choose **Email verification code**.
3. Enter the six-digit code from the email. The phone joins the household right away; no other approval is needed.
4. Allow location **Always**, so the phone can share while the app is closed, and allow notifications for arrival alerts and Find My Phone.
5. Under **Who can see this phone?**, pick who in the household sees your location.

That's it. Everyone who shares with you appears on your map, and trips start recording automatically.

## 9. Keeping it running

- **Updates:** in Portainer, open the stack, choose **Update the stack** and turn on **Re-pull image**. Your data stays in place.
- **Backups:** back up the whole `/opt/docker/whenwhere` folder. The `server-data` folder holds the server's encryption key, which is needed to restore; back it up together with `database-data`.
- **Never** change `WHENWHERE_DB_PASSWORD` after the first start, and never delete `server-data`.

---

## Troubleshooting

**The app says "Enter your household's HTTPS server address without a path."**
Enter just the address, starting with `https://` and without anything after the domain, for example `https://family.example.com`.

**The app says the server address could not be found.**
Check the address for typos, then open it in the phone's browser. If the browser can't reach it either, check the tunnel's public hostname in Cloudflare and that the stack is running.

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

**Please don't post** your server address, email addresses, tokens, passwords or screenshots that show real locations. If something involves private details, email dev@robiem.com instead.

---

WhenWhere by Robiem.
