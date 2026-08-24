# my-board

A forum, built on [Meith](https://github.com/meith-dev/meith).

## Deploy

Nothing here builds on your own server — a 2 GB VPS OOMs on a Next.js build,
which is the whole reason `Dockerfile`, `docker-compose.yml` and
`.github/workflows/build.yml` exist: something else builds the image, the
server only ever pulls one. Three steps, nothing to configure by hand beyond
one value only you know:

1. **Push this repository to GitHub.** `.github/workflows/build.yml` builds
   `Dockerfile` on every push to `main` and pushes the result to your own
   GitHub Container Registry, `ghcr.io/<you>/my-board` — using only the
   `GITHUB_TOKEN` every GitHub Actions run already carries. No secret to
   add, no registry account beyond the GitHub account you already have.

   Open the run under the repository's **Actions** tab once it finishes —
   its **Summary** prints the two things left: the exact image to paste
   into step 2 below, and a direct link to the one-time step of making the
   package public. It starts **private**, and Coolify's pull fails with an
   authentication error no operator can act on until that is done.

2. **Point [Coolify](https://coolify.io) at `docker-compose.yml`** — a Docker
   Compose resource, this repository as its source. `docker-compose.yml` already
   carries Coolify's own "magic variables" for `AUTH_SECRET`,
   `TICK_SECRET` and the database password, generated on the first deploy
   and never typed in. The one thing Coolify cannot generate is the image
   step 1 just pushed: set `MEITH_IMAGE` in the resource's own environment
   to the value that run's Summary printed — `ghcr.io/<you>/my-board:latest`
   (or a commit sha, once you want a pin that only moves when you say so —
   `docker-compose.yml` refuses to start without this set, with a message saying
   why).

3. **Deploy, then `/install` on your own domain.** Coolify issues the
   certificate; the installer from there is the one
   [docs/quickstart.md](https://github.com/meith-dev/meith/blob/main/docs/quickstart.md#4-run-the-installer)
   walks through, screen for screen. It seals itself when it finishes, and
   `/install` answers 404 from then on — run it **against the database you
   are going to keep**. Every push to `main` after this rebuilds the
   image; Coolify's own **Redeploy** button is what actually pulls it —
   pushing alone does not.

No Docker Hub, no paid CI: GitHub Actions' free tier and GHCR are the whole
build side of this, for a board of any size.

**Building it yourself**: works on any machine with Docker, if you would
rather not use GitHub Actions for the build — push the result wherever
`docker-compose.yml`'s `MEITH_IMAGE` can reach.

```sh
docker build --build-arg MEITH_VERSION=$(node -p "require('./package.json').dependencies['@meith/web']") -t my-board .
```

**Without a panel**: [docs/self-hosting.md](https://github.com/meith-dev/meith/blob/main/docs/self-hosting.md)
is the same four containers by hand — your own `.env`, a reverse proxy you
already run, no Coolify. `Dockerfile` and `docker-compose.yml` here are this
board's own version of exactly that shape.

Two things nothing configures for you:

- **Mail.** Until `MAIL_DRIVER` and its three settings exist, every message is
  written to the log and delivered to nobody, so password reset fails silently.
- **The tick.** `docker-compose.yml`'s `worker` service drives it here — a small
  loop calling `/api/system/tick` once a minute, since `@meith/web`'s own
  worker package is not something a board outside the meith monorepo can
  depend on yet. Deploy some other way and something still has to call that
  route (or run `community task:run`) every minute, or nothing catches up
  and nothing errors.

## Local

```sh
npm install
cp .env.example .env.local
npm run dev
```

With no `DATABASE_URL`, the board runs on deterministic in-memory sample data —
enough to click through every reading surface. Posting needs a database:

```sh
npm run forum -- migrate
npm run forum -- user:create --admin
```

## Configuring

- **`community.config.ts`** — installed themes and plugins. Everything installable
  is named here so the bundler can see it; nothing is found by scanning a
  directory at runtime.
- **`/admin`** — settings, forums, groups, members, themes, maintenance. An
  administrator re-enters their password to get in, and again for anything
  destructive.
- **`npm run forum -- --help`** — the operator CLI. Everything the panel does
  and a few things it cannot, without a browser.

## Upgrading

```sh
npm install @meith/web@latest @meith/cli@latest
git commit -am "Upgrade @meith/web and @meith/cli"
git push
```

That one `package.json` change is the whole pin: `Dockerfile`'s own
`FROM` line takes the version as a build argument, and
`.github/workflows/build.yml` reads it straight out of `package.json`'s
own `@meith/web` dependency when it rebuilds — nothing in `Dockerfile`
itself to keep in sync by hand. Once the rebuilt image is deployed, run
`npm run forum -- upgrade` against it for the plugin migrations — see
[the operator CLI](https://github.com/meith-dev/meith/blob/main/docs/operating.md#the-operator-cli)
for running it against this deployment.

Migrations are forward-only. Recovery is by restore, so take a backup first —
there is no down migration to undo a destructive one, and a button that pretended
otherwise would be worse than its absence.
