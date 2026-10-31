# Primefield Property Services: website build

This page holds the finished build for upload to Hostinger. It contains no source code.

## Download

Open **Releases** on the right, choose the newest one, and download the zip under **Assets**.

## Upload

Follow `DEPLOY.md` in this repository (the same guide is inside the zip). The short version:

1. Upload the zip to the Hostinger **Node.js web app** as it is (Express, Node 24.x, root `./`, build settings on default). Do not rebuild it.
2. Add the environment variables listed in `DEPLOY.md`. There is no database and no data folder.
3. Open `https://primefieldpropertyservices.com/healthz`: it must show the build stamp from the release notes.
4. Send one test request from `/request-service` and check it reaches `info@primefieldpropertyservices.com`.
5. Clear the Hostinger CDN cache once.
