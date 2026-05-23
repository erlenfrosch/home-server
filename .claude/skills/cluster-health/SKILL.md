---
name: cluster-health
description: SSH into the home server and report full cluster health — node status, ArgoCD app sync/health, unhealthy pods, high-restart pods, and PVC status. Summarize findings and flag anything requiring attention.
---

Run all checks below via SSH (`ssh -i ~/.ssh/id_ed25519 erlenfrosch@192.168.1.109`) and produce a concise health report. Flag anything that is not Synced+Healthy, not Running/Completed, or showing ≥5 restarts.

## Checks

1. **Node**: `sudo kubectl get nodes -o wide`
2. **ArgoCD apps**: `sudo kubectl -n argocd get applications`
3. **Unhealthy pods**: `sudo kubectl get pods -A | grep -v -E '\s(Running|Completed|Succeeded)\s'`
4. **High-restart pods**: `sudo kubectl get pods -A --sort-by='.status.containerStatuses[0].restartCount' | tail -10`
5. **PVCs**: `sudo kubectl get pvc -A`

## Report format

Output a short markdown summary:

- One line per section (Node / Apps / Pods / PVCs)
- Use ✅ if everything in that section looks healthy, ⚠️ if something needs attention
- List any flagged items with their namespace and a one-line reason
- End with an overall status: **All healthy** or **Action required: <item>**
