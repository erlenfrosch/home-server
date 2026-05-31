# Manuellen Agent-Job triggern

Führe folgende Schritte aus:

1. Erstelle einen manuellen Job aus dem CronJob:
   ```
   kubectl create job --from=cronjob/issue-controller \
     issue-controller-manual-$(date +%s) \
     -n claude-agent
   ```

2. Warte bis der Pod läuft und zeige die Logs:
   ```
   kubectl get pods -n claude-agent -l job-name --sort-by=.metadata.creationTimestamp | tail -2
   ```
   Dann logs des neuesten Pods taillen:
   ```
   kubectl logs -n claude-agent -l job-name --tail=50 -f
   ```

3. Berichte: War der Auth gegen Claude erfolgreich? Gibt es Fehler in den Logs?
