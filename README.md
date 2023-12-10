# Deployment

The application is deployed into a prepared environment where ingress Controller, ArgoCD, monitoring utility (Grafana, Prometheus) are deployed.

Deployment occurs automatically. After the “git action” is completed, the values.yaml file will be automatically updated with the Docker image tag and all necessary files will be checked by Helm Lint. 

```
NOTE: 
Changes to variables should only be made in the values.yaml file.
```

# Slack Notification

Notifications in the slack channel "incoming-webhooks":
- completion of a Git Action
- creating a Docker image and changing the tag in values.yaml
- sync status

# Slack Notification ArgoCD install and setting

[https://dev.to/infracloud/integrating-argo-cd-and-slack-for-real-time-notifications-235f]

Argo CD Notifications triggers and templates installation
```
kubectl apply -n argocd -f \
https://raw.githubusercontent.com/argoproj-labs/argocd-notifications/release-1.0/catalog/install.yaml
```

For manifest-based or operator-based installation:
1. Let’s add your Slack app OAuth token to argocd-notifications-secretl Secret: Replace the placeholder “xoxb-xxxx” with your actual Slack token value before executing the following command:

```
kubectl patch secret argocd-notifications-secret -n argocd \
 --type merge --patch '{"stringData":{"slack-token": "xoxb-5996791930950-6321715852323-F6lBYmUlWdo5iRPM38QNc5WQ"}}'

```

2. Configure Slack integration in the argocd-notifications-cm ConfigMap:

Let’s add Slack Notifications Service along with reference to your Slack token in argocd-notifications-cm ConfigMap with below command:
```
kubectl patch configmap argocd-notifications-cm -n argocd --type merge -p '{"data": {"service.slack": "token: $slack-token"}}'
```


3. Update your local copy of the application specification:

```bash
kubectl get application myproject -n argocd -o yaml > application.yaml
```
Make the necessary changes by adding the required annotations for trigger subscriptions and recipients to the file `application.yaml`.

```yaml
annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: incoming-webhooks
    template.app-sync-status: |
      message: |
        Application {{.app.metadata.name}} sync is {{.app.status.sync.status}}.
        Application details: {{.context.argocdUrl}}/applications/{{.app.metadata.name}}.
      slack:
        attachments: |
          [{
            "title": "{{.app.metadata.name}}",
            "title_link": "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
            "color": "#18be52",
            "fields": [{
              "title": "Sync Status",
              "value": "{{.app.status.sync.status}}",
              "short": true
            }, {
              "title": "Repository",
              "value": "{{.app.spec.source.repoURL}}",
              "short": true
            }]
          }]
```
Remove the legacy application object:

```
kubectl delete application myproject -n argocd
```
4. Apply the updated application specification:

```
kubectl apply -f application.yaml
```

# Secrets

Database password added to secrets
