# GitLab mirror and `glab` authentication

Mirrors live under `gitlab.com/dk-raas/dkai/game-servers/<repo-name>`.

GitHub Actions pushes with the `GITLAB_TOKEN` repository secret after CI passes on
`main`.

## Authenticate GitLab CLI (`glab`)

There is **no token stored in this workspace** by default.

`~/.config/glab-cli/config.yml` lists `hosts.gitlab.com.token` as empty until you
set it.

1. Create a [GitLab personal access token](https://docs.gitlab.com/user/profile/personal_access_tokens/)
   or [group/project access token](https://docs.gitlab.com/user/project/settings/project_access_tokens/)
   with at least **write_repository** (and scopes needed for your org policy).

2. Either run:

   ```bash
   glab auth login --hostname gitlab.com
   ```

   and paste the token, **or** set an environment variable for non-interactive
   use:

   ```bash
   export GITLAB_TOKEN="glpat-..."   # do not commit this value
   ```

   `glab` also honors `GITLAB_HOST` / `GL_HOST` for self-managed GitLab.

3. Optional: write the token into `~/.config/glab-cli/config.yml` under
   `hosts.gitlab.com.token` (file permissions should be user-read-only).

4. Verify:

   ```bash
   glab auth status
   ```

## Create the empty mirror project (once)

Replace `<repo>` with `satisfactory-server-k8s` or `windrose-server-k8s`:

```bash
glab repo create "dk-raas/dkai/game-servers/<repo>" \
  --private \
  --description "Mirror of DataKnifeAI/<repo> for CI to Harbor"
```

Use `--public` if your policy requires a public mirror.

The GitHub Actions user needs permission to **push** to this project.

## GitLab CI variables (Harbor)

On the GitLab project (or parent group), set **masked** variables:
`HARBOR_REGISTRY`, `HARBOR_PROJECT`, `HARBOR_USER`, `HARBOR_PASSWORD`.

See other DataKnife repos (for example `freya`) for the same pattern.
