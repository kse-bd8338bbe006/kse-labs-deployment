# image-policy-provider

In-cluster deployment of the [image-policy-provider](https://github.com/kse-bd8338bbe006/image-policy-provider)
service -- an OPA Gatekeeper external-data provider that verifies
container image signatures with Cosign.

## Pieces

| File | What it creates |
| --- | --- |
| `certificate.yaml` | `Certificate` (cert-manager) issued by the in-cluster Vault PKI `ClusterIssuer`. cert-manager writes the keypair + chain into the `image-policy-provider-tls` Secret. |
| `deployment.yaml` | The Cosign public-key Secret (lab placeholder), the `Deployment` running the FastAPI service on `:8443` with TLS terminated by uvicorn, and the `Service` exposing port `443`. |
| `provider.yaml` | The Gatekeeper `Provider` CRD that Rego references via `external_data({"provider": "image-policy-provider", ...})`. `caBundle` starts empty and is populated by the patcher Job. |
| `cabundle-job.yaml` | RBAC + a `Job` that reads `ca.crt` from the cert-manager Secret and patches `spec.caBundle` on the `Provider`. Decouples the manifest from whatever CA Vault is currently using. |

## Sync-wave layout

| wave | what runs |
| --- | --- |
| 20 | `Certificate`, `Secret/cosign-key`, RBAC for the patcher |
| 25 | `Deployment` + `Service` |
| 30 | `Provider` (caBundle empty) |
| 35 | `Job` that fills caBundle |

## Trust material

The Cosign public key is the only piece that is not committed -- replace
the empty `cosign.pub` in `deployment.yaml` (or supply it via a sealed
secret / external secrets operator) before the verifier can return
`verified` for any image. The verifier returns a clear error string when
no key is configured, which is what we want until students bring their
own.
