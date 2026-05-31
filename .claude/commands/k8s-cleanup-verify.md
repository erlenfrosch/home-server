# Cleanup nach Merge und Deployment verifizieren

Führe folgende Schritte aus:

1. **Lokale Branches aufräumen** (merged branches löschen):
   ```
   git fetch --prune
   git branch -vv | grep '\[origin/.*: gone\]' | awk '{print $1}' | xargs -r git branch -d
   ```

2. **ArgoCD Sync-Status prüfen**:
   ```
   kubectl get application claude-agent -n argocd -o jsonpath='{.status.sync.status} {.status.health.status}' 2>/dev/null \
     || kubectl get application -n argocd | grep claude-agent
   ```
   Falls nicht Synced: `kubectl -n argocd patch application claude-agent -p '{"operation":{"initiatedBy":{"username":"manual"},"sync":{}}}' --type=merge`

3. **Neues Secret verifizieren** – prüfe ob CLAUDE_CODE_OAUTH_TOKEN korrekt gesetzt ist:
   ```
   kubectl get secret claude-agent-secrets -n claude-agent -o json \
     | python3 -c "import json,sys,base64; s=json.load(sys.stdin); d=s['data']; [print(k,':', base64.b64decode(v).decode()[:25]+'...') for k,v in d.items()]"
   ```
   Erwartet: `CLAUDE_CODE_OAUTH_TOKEN: sk-ant-oat01-...`

4. Berichte Ergebnis: Sync-Status, welche Keys im Secret vorhanden sind, ob CLAUDE_CODE_OAUTH_TOKEN korrekt befüllt ist.
