# sweep-mcp (deploy)

Kustomize manifests + ImageUpdater CR for the [sweep-mcp][build] MCP server.
Phase 3 of [model-testing#36][epic] — wires the chat-trigger surface so the
operator can say "Run qwen3.7:35b through the sweep test" in OWUI chat and
have the existing kickoff → GHA → dual-summarize chain fire.

[build]: https://github.com/dvystrcil/sweep-mcp-docker
[epic]: https://github.com/dvystrcil/model-testing/issues/36

## Layout

| Path | Role |
|---|---|
| `base/namespace.yaml` | The `sweep-mcp` namespace + ambient-mode label |
| `base/deployment.yaml` | Single-replica Go MCP server, scratch image, /healthz probes |
| `base/service.yaml` | ClusterIP on port 8080, name `http`, targets the `http` container port |
| `base/harbor-pull-secret.yaml` | Harbor pull credentials via InfisicalSecret (`HB_*` keys) |
| `base/image-updater-rbac.yaml` | Role + RoleBinding so IU can read the pull secret for tag discovery |
| `base/kustomization.yaml` | Glues the above + pins the image tag (IU writes back here) |
| `image-updater/sweep-mcp-image-updater.yaml` | Enrolled by the image-updater-apps ApplicationSet in argocd-projects |

## How it fits

```
sweep-mcp-docker (build)
  └─► docker-release.yaml fires on GitHub release publish
        └─► retags Harbor :dev → :X.Y.Z
              └─► IU watches harbor.sirddail.net/ai/sweep-mcp:0.x
                    └─► writes new tag back to this repo's base/kustomization.yaml
                          └─► ArgoCD reconciles → rollout
```

## Memory hooks

- [[feedback_three_repo_split_for_new_services]] — this is the "deploy" layer
- [[feedback_image_updater_crd_not_annotations]] — IU lives as a CRD enrolled
  via the image-updater-apps ApplicationSet, never as annotations
- [[feedback_rbac_ownership]] — image-updater-rbac.yaml lives here, not in
  the IU install
- [[feedback_kustomize_source_of_truth]] — `base/kustomization.yaml` is the
  applied state; IU writes back to it

## Related

- Build: https://github.com/dvystrcil/sweep-mcp-docker
- ArgoCD App: https://github.com/dvystrcil/argocd-projects (sweep-mcp/sweep-mcp.yaml)
- Epic: https://github.com/dvystrcil/model-testing/issues/36 (Phase 3)
- Sibling deploy repos: `dvystrcil/tor-character-mcp`, `dvystrcil/wiki-mcp`
