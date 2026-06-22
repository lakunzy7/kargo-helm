# Kargo Helm Example

This is a GitOps repository of a Kargo Helm example for getting started.

### Features:

* A Warehouse which monitors a container repository for new images
* Three Stage (dev, staging, prod) deploy pipeline
* Image tag promotion
* Direct Git commits to dev, staging
* Feature flag promotion
* Multi-cluster: each stage deploys to two clusters (`cluster1` = in-cluster,
  `cluster2` = a second registered ArgoCD cluster)

## Requirements

* Kargo v1.3 (for older Kargo versions, switch to the release-X.Y branch)
* GitHub and a container registry (GHCR.io)
* `git` and `docker` installed

## Instructions

1. Fork this repo, then clone it locally (from your fork).
2. Run the `personalize.sh` to customize the manifests to use your GitHub
   username:

   ```shell
   ./personalize.sh <yourgithubusername>
   ```
3. `git commit` the personalized changes:

   ```shell
   git commit -a -m "personalize manifests"
   git push
   ```
4. Create a guestbook container image repository in your GitHub account. 

   The easiest way to create a new ghcr.io image repository, is by retagging and 
   pushing an existing image with your GitHub username:

   ```shell
   docker login ghcr.io

   docker buildx imagetools create \
     ghcr.io/akuity/guestbook:latest \
     -t ghcr.io/<yourgithubusername>/guestbook:v0.0.1
   ```

   You will now have a `guestbook` container image repository. e.g.:
   https://github.com/yourgithubusername/guestbook/pkgs/container/guestbook

5. Change guestbook container image repository to public.

   In the GitHub UI, navigate to the "guestbook" container repository, Package
   settings, and change the visibility of the package to public. This will allow
   Kargo to monitor this repository for new images, without requiring you to 
   configuring Kargo with container image repository credentials.

   ![change-package-visibility](docs/change-package-visibility.png)

6. Download and install the latest CLI from [Kargo Releases](https://github.com/akuity/kargo/releases/latest)

   ```shell
   ./download-cli.sh /usr/local/bin/kargo
   ```

7. Login to Kargo:

   ```shell
   kargo login --admin https://<kargo-url>
   ```

8. Apply the ArgoCD manifests (AppProject + ApplicationSet). The
   ApplicationSet reads the repo's `env/*` directories from GitHub `HEAD` and
   generates one Application per stage, per cluster
   (`kargo-helm-<stage>-cluster1` / `-cluster2`). The AppProject must exist
   first, or the ApplicationSet fails generation with
   `AppProject "kargo-helm" not found`.

   ```shell
   kubectl apply -f ./argocd
   ```

9. Apply the Kargo manifests:

   ```shell
   kargo apply -f ./kargo
   ```

10. Add the Git repository credentials to Kargo. This can also be done in the UI
   in the `kargo-helm` Project.

   > Note: on recent Kargo CLI versions (v1.10+) the subcommand is
   > `create repo-credentials` and flags use `=` syntax.

   ```shell
   kargo create repo-credentials github-creds \
     --project=kargo-helm \
     --git \
     --username=<yourgithubusername> \
     --repo-url=https://github.com/<yourgithubusername>/kargo-helm.git \
     --password=<your-github-pat>
   ```

   As part of the promotion process, Kargo requires privileges to commit changes
   to your Git repository, as well as the ability to create pull requests. Ensure
   that the given token has these privileges (Contents: read & write, Pull
   requests: write). A classic GitHub PAT begins with `ghp_` and is 40
   characters; an invalid/truncated token causes the `git-push` step to fail
   with `Invalid username or token`.

11. Promote the image!

    You now have a Kargo Pipeline which promotes images from the guestbook
    container image repository, through a three-stage deploy pipeline. Visit
    the `kargo-helm` Project in the Kargo UI to see the deploy pipeline.

    ![pipeline](docs/pipeline.png)

    To promote, click the target icon to the left of the `dev` Stage, select
    the detected Freight, and click `Yes` to promote. Once promoted, the Freight
    will be qualified to be promoted to downstream Stages (`staging`, `prod`).


## Simulating a release

To simulate a release, simply retag an image with a newer semantic version. e.g.:

```shell
docker buildx imagetools create \
  ghcr.io/akuity/guestbook:latest \
  -t ghcr.io/<yourgithubusername>/guestbook:v0.0.2
```

Then refresh the Warehouse in the UI to detect the new Freight.


## Promoting a feature

Edit the `base/feature-flags.yaml` with a new setting. This will be detected
by the `features` Warehouse as a promotable configuration.
