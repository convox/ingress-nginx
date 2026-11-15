# convox/ingress-nginx

Based on [kubernetes/ingress-nginx](https://github.com/kubernetes/ingress-nginx) controller v1.15.1, the final upstream release

nginx 1.27.1 is compiled in the image with the security backports in `1.15/nginx/patches`, on current Alpine packages. The controller binaries are compiled on current Go from the same upstream source. The Lua and controller code is unchanged

| Directory | ingress-nginx | nginx | Go | Image |
| --- | --- | --- | --- | --- |
| `1.15` | v1.15.1 (`0a5901f`) | 1.27.1 with `1.15/nginx/patches` | 1.26.11, `golang.org/x/net` v0.62.0, `golang.org/x/text` v0.43.0 | `convox/ingress-nginx:1.15.1-<n>` |

## nginx

`1.15/nginx` is the nginx build of [chainguard-forks/ingress-nginx](https://github.com/chainguard-forks/ingress-nginx), which continues upstream's build and backports nginx fixes to 1.27.1. `1.15/NOTICE` names the commit it was taken from and what is changed. Patches 35 and up are the security backports, and each names its CVE and nginx commit. The fork is a source of patches, not a build dependency.

Every source the build downloads is pinned by sha256 or by git commit, and the build fails on a mismatch. If a download is gone or a hash no longer matches, compare the new archive with the project's git tag before recording a new hash in `1.15/nginx/build.sh`.

The version string stays `nginx/1.27.1`, so a scanner that matches on it reports advisories this build has patched.

## Releasing

A push to any branch except `master` builds `1.15` for amd64 and arm64 and publishes `convox/ingress-nginx:<commit sha>`, with `<commit sha>-amd64` and `<commit sha>-arm64`. Test that image.

To release, push a git tag `1.15.1-<n>`, one higher than the last, on the commit that was built and tested. After a merge that is the branch head, not the merge commit, so merge with a merge commit: a squash or rebase leaves the tested commit outside `master`. The tag publishes that commit's image as `convox/ingress-nginx:1.15.1-<n>` with the same digest, and nothing is built again. Tags are never moved or reused, and there is no `latest`.

A tag on a commit that has no image fails and publishes nothing. Delete the git tag and push it on the built commit. A tag whose image is already published is refused.

Before tagging, run the `scan` workflow by hand with the commit sha as `tag`. GitHub disables its monthly schedule when the repository is quiet. A fixable finding means a new commit and a new test, not a tag.

## When to rebuild

| Trigger | Change |
| --- | --- |
| `scan` shows a fixable Alpine finding | none: push an empty commit on a branch |
| `scan` shows a fixable Go finding, or a Go security release for the builder's Go line | the builder image in `1.15/Dockerfile`; move to the next Go line when the current one stops getting fixes |
| an advisory in a Go module the controller uses | add or bump it on the `go get` line and in the table above |
| `scan` lists a patch the fork has and `1.15/nginx/patches` does not | add the file and update `1.15/NOTICE` |
| [nginx.org](https://nginx.org/en/security_advisories.html) lists an advisory whose range includes 1.27.1, for a module this build compiles, that no patch covers | wait for the fork, or backport the nginx commit as the next patch file |

`scan` reads Alpine packages and Go binaries. It does not see the nginx compiled here, so check the nginx.org page with it.

Every build compiles nginx and its modules again, so a rebuild for any reason gets the full test: the image runs as the router of a Rack, on both architectures, before its commit is tagged.

Convox Racks pick up a new tag through a Convox release that changes the pinned digest. This image is retired together with the nginx router.
