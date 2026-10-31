# bdtree-the-placeholders

Template **content repo** for [bdtree](https://github.com/batdevis/bdtree), the self-hosted link-in-bio page for bands.
It holds the demo band "The Placeholders" and a GitHub Actions workflow that builds your site with the bdtree code and deploys it to Firebase Hosting on every push.

bdtree keeps code and content apart:

```
my-band/
├── bdtree/                 the code (public, shared by every band)
│   └── content -> ../bdtree-my-band   (symlink)
└── bdtree-my-band/         your content: a copy of this template, usually private
```

## 1. Make your copy

Click **Use this template** → **Create a new repository**, name it e.g. `bdtree-my-band`, and make it **private**: your content (contacts, tech rider) stays yours.
Then clone it next to bdtree:

```sh
mkdir my-band && cd my-band
git clone https://github.com/batdevis/bdtree.git
git clone git@github.com:<you>/bdtree-my-band.git
cd bdtree
ln -s ../bdtree-my-band content
npm install
npm start          # http://localhost:8080, live reload, also on content edits
```

Replace The Placeholders with your band: the files and their format are described in the [bdtree README](https://github.com/batdevis/bdtree#make-it-yours).
Also replace `LICENSE`: the demo content is MIT, but your own texts and photos probably aren't (e.g. "All rights reserved").

Until the next step is done, the workflow runs on every push but deploys nothing (it ends with a notice).

## 2. Firebase Hosting

Create a Firebase project ([console](https://console.firebase.google.com/) or `npx firebase-tools projects:create <project-id>`), then set in `band.json`:

- `firebaseProject`: `<project-id>`
- `siteUrl`: `https://<project-id>.web.app` (or your custom domain)

You can already deploy from your computer: `npm run deploy` in `bdtree/` (after `npx firebase-tools login`).

## 3. Deploy on push (GitHub Actions)

The workflow needs a Google Cloud service account, with its key in the repo secret `FIREBASE_SERVICE_ACCOUNT`.
With [gcloud](https://cloud.google.com/sdk/docs/install) and [gh](https://cli.github.com/), from your content repo:

```sh
PROJECT=<project-id>
SA=github-deploy@$PROJECT.iam.gserviceaccount.com
gcloud iam service-accounts create github-deploy --project $PROJECT
for role in roles/firebasehosting.admin roles/serviceusage.apiKeysViewer roles/serviceusage.serviceUsageConsumer; do
  gcloud projects add-iam-policy-binding $PROJECT --member=serviceAccount:$SA --role=$role --condition=None
done
gcloud iam service-accounts keys create key.json --iam-account $SA
gh secret set FIREBASE_SERVICE_ACCOUNT < key.json && rm key.json
```

From now on every push to `main` rebuilds and deploys the site. To deploy without a push (e.g. after a change in bdtree), run it by hand: Actions → Deploy → Run workflow.

Gigs are filtered when the site is built, so after a show, redeploy to remove it from the list: a push or a manual run.

## 4. Admin editor (optional)

`/admin/` lets you and your bandmates sign in with Google and edit gigs, links, bio and texts with forms; each save is a commit to this repo, which then deploys as usual. It runs on Cloud Functions, so it needs the **Blaze** plan (a band site stays in the free tier; set a budget alert anyway).

1. Firebase console → Authentication → Sign-in method → enable **Google**.
2. GitHub → Settings → Developer settings → [Fine-grained tokens](https://github.com/settings/personal-access-tokens/new): access to **only this content repo**, permission **Contents: read and write**. Check it, then store it as a Firebase secret, piping it in (pasting into the hidden prompt can store a truncated value):
   ```sh
   export GITHUB_TOKEN=<the token>
   curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer $GITHUB_TOKEN" https://api.github.com/repos/<you>/bdtree-my-band   # 200
   printf %s "$GITHUB_TOKEN" | npx firebase-tools functions:secrets:set GITHUB_TOKEN --data-file=- --project $PROJECT
   ```
3. In `functions.env`, set `GITHUB_REPO=<you>/bdtree-my-band` and the Google accounts in `ADMIN_EMAILS`.
4. First deploy from your computer (it enables the Cloud APIs, which the CI account can't do), in `bdtree/`:
   ```sh
   npm run deploy:functions
   npx firebase-tools functions:artifacts:setpolicy --location europe-west1 --force --project $PROJECT   # delete old build images
   ```
5. Let the workflow deploy the functions from now on: give the CI account these roles too.
   ```sh
   for role in roles/cloudfunctions.admin roles/secretmanager.viewer roles/artifactregistry.reader; do
     gcloud projects add-iam-policy-binding $PROJECT --member=serviceAccount:$SA --role=$role --condition=None
   done
   NUMBER=$(gcloud projects describe $PROJECT --format='value(projectNumber)')
   # Service Account User only on the accounts the deploy acts as (the appspot one may not exist: skip it then).
   for account in $NUMBER-compute@developer.gserviceaccount.com $PROJECT@appspot.gserviceaccount.com; do
     gcloud iam service-accounts add-iam-policy-binding $account --project $PROJECT --member=serviceAccount:$SA --role=roles/iam.serviceAccountUser
   done
   ```

Then adding or removing an editor is just: edit `ADMIN_EMAILS`, commit, push. The workflow redeploys the functions when `functions.env` changes; after a change in `bdtree/functions`, run it by hand with **Also deploy the Cloud Functions** ticked.
When the token expires, the admin page says so: create a new one and run the `printf … secrets:set` line again with `--force` (which also redeploys the functions).

## 5. Analytics (optional)

Set `"analytics": true` in `band.json` to count visits with Firebase Analytics: pages show a consent banner (texts `consent*` in `i18n.json`), and Analytics loads only for visitors who accept.
Firebase console → Project settings → Integrations → enable **Google Analytics**, and under General → Your apps register a **Web** app if there is none. The measurement ID then reaches the site by itself, no config to copy. Details in the [bdtree README](https://github.com/batdevis/bdtree#analytics-optional).

## Using your own fork of bdtree

The workflow builds with `batdevis/bdtree` at `main`. To use a fork or pin a commit, change `BDTREE_REPO` / `BDTREE_REF` at the top of `.github/workflows/deploy.yml`.
