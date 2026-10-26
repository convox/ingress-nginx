# convox/ingress-nginx

Based on [kubernetes/ingress-nginx](https://github.com/kubernetes/ingress-nginx) controller v1.15.1, the final upstream release

Rebuilt with current Alpine packages and with the controller binaries compiled on current Go from the same upstream source. The nginx, Lua and controller code is unchanged

| Directory | ingress-nginx | Go | Image |
| --- | --- | --- | --- |
| `1.15` | v1.15.1 (`0a5901f`) | 1.26.10, `golang.org/x/net` v0.61.0, `golang.org/x/text` v0.43.0 | `convox/ingress-nginx:1.15.1-<n>` |

To release, push a git tag `1.15.1-<n>`, one higher than the last. It builds the directory matching the tag's major.minor and publishes `<tag>-amd64`, `<tag>-arm64` and the multi-arch `<tag>`. Tags are never moved or reused, and there is no `latest`.

Rebuild with a new tag when the `scan` workflow shows a fixable Alpine or Go finding (no file change for Alpine), on a Go security release for the builder's Go line (update the builder image in `1.15/Dockerfile`; move to the next Go line when the current one stops getting fixes), or on an advisory in any Go module the controller uses (add or bump it on the `go get` line and in the table above). GitHub disables the monthly schedule when the repository is quiet, so enable and run `scan` by hand before each Convox release.

Convox Racks pick up a new tag through a Convox release that changes the pinned digest. This image is retired together with the nginx router.
