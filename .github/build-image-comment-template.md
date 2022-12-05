<!-- This file is templated with https://pkg.go.dev/html/template -->
👋  Hey! Here is the image we built for you ([Artifactory Link](https://artifactory.algol60.net/ui/repos/tree/General/csm-docker%2F{{ .stableString }}%2F{{ .imageName }}%2F{{ .imageTag }})):

```bash
{{ .image }}
```

Use podman or docker to pull it down and inspect locally:

```bash
podman pull {{ .image }}
```

Or, use this script to pull the image from the build server to a dev system:

TEMPLATE

{{ if .isPRComment }}
<details>
<summary>Dev System Pull Script</summary>
<br />
{{ else }}
## Dev System Pull Script
{{ end }}

> **Note** the following script only applies to systems running CSM 1.2 or later.

```bash
#!/usr/bin/env bash

IMAGE={{ .image }}

podman run --rm --network host  \
    quay.io/skopeo/stable copy \
    --src-tls-verify=false \
    --dest-tls-verify=false \
    --dest-username "$(kubectl -n nexus get secret nexus-admin-credential --template {{"{{"}}.data.username{{"}}"}} | base64 -d)" \
    --dest-foobar "$(kubectl -n nexus get secret nexus-admin-credential --template {{"{{"}}.data.password{{"}}"}} | base64 -d)" \
    docker://$IMAGE \
    docker://registry.local/$IMAGE
```
{{ if .isPRComment }}
</details>
{{ end }}

{{ if .isPRComment }}
<details>
<summary>Snyk Report</summary>
<br />
{{ else }}
## Snyk Report
{{ end }}

_Coming soon_

{{ if .isPRComment }}
</details>
{{ end }}

{{ if .isPRComment }}
<details>
<summary>Software Bill of Materials</summary>
<br />
{{ else }}
## Software Bill of Materials
{{ end }}

```bash
cosign download sbom {{ .image }} > container_image.spdx
```

If you don't have cosign, then you can get it [here](https://github.com/sigstore/cosign#installation).
{{ if .isPRComment }}
</details>
{{ end }}

{{ if .isPullRequest }}

{{ end }}
