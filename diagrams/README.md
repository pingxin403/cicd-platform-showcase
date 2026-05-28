# Diagrams

Standalone diagram for this CI/CD showcase. Mermaid source + 1600px PNG + transparent SVG.

| File | Topic |
|---|---|
| `cicd-flow` | End-to-end CI/CD: detect-changes → matrix build → ArgoCD → Flagger canary + SLO gate |

Cross-shared with [`cuckoo-echo-showcase`'s diagram-03](https://github.com/pingxin403/cuckoo-echo-showcase/blob/main/diagrams/03-cicd-flow.png) — keep them in sync if either source `.mmd` is edited.

## Re-render

```bash
./render.sh   # requires @mermaid-js/mermaid-cli (pnpm i -g, npm i -g)
```

On macOS the script auto-falls-back to system Chrome to skip puppeteer's chromium download. Set `PUPPETEER_EXECUTABLE_PATH` manually if the auto-detection misses.
