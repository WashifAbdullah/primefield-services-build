# PrimeField Services: website and back office build

This page holds the finished build for upload to Hostinger. It contains no source code.

## Download

Open **Releases** on the right, choose the newest one, and download the zip under **Assets**.

## Upload

Follow `DEPLOY.md` in this repository (the same guide is inside the zip). The short version:

1. Upload the zip to the Hostinger **Node.js web app** as it is (Express, Node 24.x, root `./`, build settings on default). Do not rebuild it.
2. Set `DATA_DIR` to a folder outside the app, for example `/home/<hosting user>/primefield-data`, or photos are erased on the next upload.
3. The database updates itself when the app starts.
4. Clear the Hostinger CDN cache once after the upload.
5. Open `https://<domain>/healthz`: it must show `"storage":"persistent"` and the build stamp from the release notes.
