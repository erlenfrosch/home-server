# Gotify push notifications

[Gotify](https://gotify.net) läuft als ArgoCD-verwaltete App im k3s-Cluster
(`argocd/apps/gotify/`) und liefert Self-hosted Push-Notifications an dein
Handy (Android/iOS) oder jeden anderen Gotify-Client.

Typische Nutzung im Home-Lab:
- Monitoring-Alerts (VictoriaMetrics → Gotify)
- Eigene Scripts oder Cron-Jobs senden Nachrichten per HTTP POST
- Automatisierungen signalisieren Fertigstellung / Fehler

## 1. Erstmaligen Deploy abschließen

### 1.1 Admin-Passwort vault-verschlüsseln

```bash
ansible-vault encrypt_string 'DEIN_STARKES_PW' \
  --name 'gotify_admin_password'
```

Den `!vault |`-Block in `ansible/group_vars/all.yml` unter
`gotify_admin_password` eintragen (als Dokumentation / Backup — Ansible liest
diesen Wert nicht direkt, er wird nur per SealedSecret in den Cluster gebracht).

### 1.2 SealedSecret-Cipher erzeugen

Der SealedSecrets-Controller (in `argocd/apps/sealed-secrets/` deployt)
entschlüsselt nur Ciphers, die mit seinem öffentlichen Schlüssel erstellt wurden.
Einfachster Weg: Web-UI unter <http://kubeseal-webgui.homeserver>:

1. Felder ausfüllen:
   - **Namespace**: `gotify`
   - **Secret name**: `gotify-admin`
   - **Key**: `password`
   - **Value**: das Plaintext-Passwort aus 1.1
2. **Encrypt** klicken, den langen base64-String kopieren.

Alternativ per CLI (von einem Rechner mit `kubeseal`):

```bash
echo -n 'DEIN_STARKES_PW' \
  | kubeseal --raw \
      --namespace gotify \
      --name gotify-admin \
      --from-file=/dev/stdin
```

### 1.3 Cipher in `values.yaml` eintragen

```yaml
# argocd/apps/gotify/values.yaml
adminSecret:
  enabled: true
  username: admin
  secretName: gotify-admin
  encryptedPassword: "AgB...langer-base64-String..."
```

Committen und pushen:

```bash
git add argocd/apps/gotify/values.yaml
git commit -m "feat(gotify): set sealed admin password"
git push
```

ArgoCD übernimmt die Änderung innerhalb von ~3 Minuten (oder in der ArgoCD-UI
auf **Refresh** klicken).

### 1.4 Verify

```bash
ssh erlenfrosch@192.168.1.109 \
  'sudo kubectl -n gotify get pods,svc,ingress,pvc,sealedsecret,secret'
curl -sS http://gotify.homeserver/health
# Expected: {"health":"green",...}
```

Login auf `http://gotify.homeserver` mit `admin` + Passwort aus 1.1.

## 2. Anwendung registrieren und Nachrichten senden

### 2.1 App-Token erstellen

Im Gotify Web-UI: **Apps → CREATE APPLICATION**
- Name: z.B. `monitoring` oder `scripts`
- Token kopieren (langer opaker String).

### 2.2 Nachricht per HTTP POST schicken

```bash
curl -fsS -X POST "http://gotify.homeserver/message" \
  -H "X-Gotify-Key: DEIN_APP_TOKEN" \
  -F "title=Test" \
  -F "message=Hallo vom Home-Server!" \
  -F "priority=5"
```

Das reicht, um Gotify von jedem Script oder Cron-Job aus anzusprechen.

## 3. Admin-Passwort / App-Token rotieren

- **Admin-Passwort**: neuen Cipher per `kubeseal` erzeugen, in `values.yaml`
  eintragen, committen + pushen. Altes `gotify-admin` Secret im Cluster löschen
  falls ArgoCD es nicht automatisch pruned, dann Gotify Pod neu starten.
- **App-Token**: alten Token im Gotify Web-UI widerrufen, neuen erstellen und
  in den eigenen Scripts ersetzen.

## 4. Troubleshooting

| Symptom | Hinweis |
|---|---|
| Pod CrashLoopBackOff nach erstem Deploy | `encryptedPassword` ist noch Placeholder — Schritt 1.3 abschließen |
| `gotify-admin` Secret fehlt | `kubectl -n gotify describe sealedsecret gotify-admin` — Cipher muss gegen den Cluster-Public-Key erzeugt worden sein |
| `gotify.homeserver` löst nicht auf | `gotify` in `dnsmasq_hosts` in `group_vars/all.yml` eintragen, dann `make dnsmasq` |
