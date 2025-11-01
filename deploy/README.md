# Flood Kubernetes Deployment

This directory contains Kubernetes deployment configurations for Flood using Helm and ArgoCD GitOps.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│ Internet                                                     │
│   ↓                                                          │
│ Cloudflare Tunnel (SSL/TLS termination)                     │
│   ↓ flood.hederium.com                                      │
│ Traefik Ingress Controller                                  │
│   ↓                                                          │
│ Flood Service (ClusterIP:8765)                              │
│   ↓                                                          │
│ Flood Pod                                                    │
│   - Port: 8765                                               │
│   - Volumes:                                                 │
│     • /downloads → /mnt/media/Plex (hostPath)               │
│     • /config → /opt/stack/flood (hostPath)                 │
└─────────────────────────────────────────────────────────────┘
```

## Directory Structure

```
deploy/
├── helm/
│   └── flood/
│       ├── Chart.yaml              # Helm chart metadata
│       ├── values.yaml             # Default configuration
│       └── templates/
│           ├── _helpers.tpl        # Template helpers
│           ├── deployment.yaml     # Flood deployment
│           ├── service.yaml        # ClusterIP service
│           ├── ingress.yaml        # Traefik ingress
│           └── configmap.yaml      # Configuration
├── argocd/
│   └── flood-application.yaml      # ArgoCD Application manifest
└── README.md                        # This file
```

## Prerequisites

### 1. Kubernetes Cluster
- k3s cluster running on Ubuntu server
- kubectl configured to access your cluster

### 2. ArgoCD
```bash
# Check if ArgoCD is installed
kubectl get pods -n argocd

# If not installed:
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 3. Storage Directories
Ensure these directories exist on your k3s host:
```bash
# On your Ubuntu server:
sudo mkdir -p /mnt/media/Plex/{Movies,TV,Other}
sudo mkdir -p /opt/stack/flood

# Set permissions (adjust UID/GID if needed)
sudo chown -R 1000:1000 /mnt/media/Plex
sudo chown -R 1000:1000 /opt/stack/flood
```

### 4. Traefik Ingress Controller
k3s comes with Traefik pre-installed. Verify:
```bash
kubectl get svc -n kube-system traefik
```

### 5. Cloudflare Tunnel
Configure your Cloudflare tunnel to point to your Traefik service:
```yaml
# In your Cloudflare tunnel configuration:
ingress:
  - hostname: flood.hederium.com
    service: http://traefik.kube-system.svc.cluster.local:80
  - service: http_status:404
```

## Deployment Steps

### Option 1: Deploy with ArgoCD (Recommended)

#### Step 1: Update Repository URL
Edit `deploy/argocd/flood-application.yaml`:
```yaml
spec:
  source:
    repoURL: https://github.com/YOUR-USERNAME/flood.git  # Update this!
```

#### Step 2: Apply ArgoCD Application
```bash
# Apply the ArgoCD application
kubectl apply -f deploy/argocd/flood-application.yaml

# Watch the deployment
kubectl get application -n argocd
kubectl get pods -n media -w
```

#### Step 3: Check Status in ArgoCD UI
```bash
# Get ArgoCD admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port-forward to access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Open: https://localhost:8080
# Login: admin / <password from above>
```

### Option 2: Deploy with Helm Directly

```bash
# Install from local chart
helm install flood ./deploy/helm/flood \
  --namespace media \
  --create-namespace

# Or upgrade if already installed
helm upgrade --install flood ./deploy/helm/flood \
  --namespace media

# Check status
helm status flood -n media
kubectl get pods -n media
```

## Configuration

### Default Configuration (values.yaml)

The default configuration uses:
- **Port**: 8765
- **Base URI**: /
- **Downloads**: /mnt/media/Plex (hostPath)
- **Config**: /opt/stack/flood (hostPath)
- **Ingress**: flood.hederium.com via Traefik
- **Resources**: 100m/256Mi (requests), 500m/512Mi (limits)

### Custom Configuration

Create a `values-custom.yaml` file to override defaults:

```yaml
# values-custom.yaml
flood:
  port: 9000  # Change port
  secret: "your-generated-secret"  # Generate with: openssl rand -hex 32

resources:
  limits:
    cpu: 1000m
    memory: 1Gi

nodeSelector:
  kubernetes.io/hostname: your-node-name
```

Apply with:
```bash
helm upgrade --install flood ./deploy/helm/flood \
  --namespace media \
  -f ./deploy/helm/flood/values.yaml \
  -f values-custom.yaml
```

## Accessing Flood

### First-Time Setup

1. Navigate to: https://flood.hederium.com
2. Create your admin account
3. Configure torrent client connection (see below)

### Initial Login
- You'll be prompted to create a username and password
- This creates your Flood user account (stored in `/opt/stack/flood/users.db`)

## Connecting qBittorrent (Future Step)

When you're ready to add qBittorrent:

### 1. Deploy qBittorrent
Create a similar Helm chart or use an existing one, ensuring:
```yaml
# qBittorrent must mount the SAME volume
storage:
  downloads:
    type: hostPath
    hostPath:
      path: /mnt/media/Plex  # Must match Flood!
    mountPath: /downloads
```

### 2. Configure in Flood UI
1. Login to Flood
2. Go to Settings → Client Connection
3. Configure:
   ```
   Client Type: qBittorrent
   Host: qbittorrent-service.media.svc.cluster.local
   Port: 8080
   Username: admin
   Password: <your-qbittorrent-password>
   ```

### 3. Set Up Download Categories
In qBittorrent settings (via web UI or config):
```
Categories:
  movies → /downloads/Movies
  tv → /downloads/TV
  other → /downloads/Other
```

Now when you add torrents in Flood:
- Select category: "movies" → Downloads to /mnt/media/Plex/Movies
- Select category: "tv" → Downloads to /mnt/media/Plex/TV

## Verification

### Check Pod Status
```bash
kubectl get pods -n media
kubectl logs -n media -l app.kubernetes.io/name=flood -f
```

### Check Service
```bash
kubectl get svc -n media
```

### Check Ingress
```bash
kubectl get ingress -n media
kubectl describe ingress -n media flood
```

### Test Internal Access
```bash
# Port-forward for testing
kubectl port-forward -n media svc/flood 8765:8765

# Open: http://localhost:8765
```

## Troubleshooting

### Pod Not Starting

**Check events:**
```bash
kubectl describe pod -n media -l app.kubernetes.io/name=flood
```

**Common issues:**
- Directory permissions: Ensure UID 1000 can write to hostPath directories
- Directory doesn't exist: Create `/mnt/media/Plex` and `/opt/stack/flood`
- Image pull issues: Check `kubectl get pods -n media` for ImagePullBackOff

### Can't Access via Ingress

**Check Ingress:**
```bash
kubectl describe ingress -n media flood
```

**Verify Traefik:**
```bash
kubectl get svc -n kube-system traefik
```

**Check Cloudflare tunnel:**
- Ensure tunnel is running and configured correctly
- Check Cloudflare dashboard for tunnel status

### Permission Denied Errors

```bash
# On your Ubuntu server:
sudo chown -R 1000:1000 /mnt/media/Plex
sudo chown -R 1000:1000 /opt/stack/flood
sudo chmod -R 755 /mnt/media/Plex
sudo chmod -R 755 /opt/stack/flood
```

### Database Issues

If Flood can't create its database:
```bash
# Check config volume
kubectl exec -it -n media deployment/flood -- ls -la /config

# Manual cleanup if needed (CAUTION: deletes user accounts)
sudo rm -rf /opt/stack/flood/*
kubectl rollout restart -n media deployment/flood
```

## GitOps Workflow

### Making Changes

1. **Edit configuration:**
   ```bash
   vim deploy/helm/flood/values.yaml
   ```

2. **Commit and push:**
   ```bash
   git add deploy/
   git commit -m "Update Flood configuration"
   git push origin master
   ```

3. **ArgoCD auto-syncs:**
   - Detects changes within ~3 minutes
   - Applies changes automatically
   - Performs rolling update (zero downtime)

4. **Monitor sync:**
   ```bash
   kubectl get application -n argocd flood -w
   ```

### Manual Sync

```bash
# Via ArgoCD CLI
argocd app sync flood

# Via kubectl
kubectl patch application flood -n argocd \
  -p '{"metadata": {"annotations": {"argocd.argoproj.io/refresh": "normal"}}}' \
  --type merge
```

## Uninstalling

### Via ArgoCD
```bash
kubectl delete application -n argocd flood
```

### Via Helm
```bash
helm uninstall flood -n media
```

### Clean Up Data (Optional)
```bash
# CAUTION: This deletes your database and settings
sudo rm -rf /opt/stack/flood
```

## Security Considerations

1. **Generate a secret:**
   ```bash
   openssl rand -hex 32
   ```
   Add to `values.yaml`:
   ```yaml
   flood:
     secret: "your-generated-secret"
   ```

2. **Restrict allowed paths:**
   ```yaml
   flood:
     allowedPaths:
       - /downloads
       - /downloads/Movies
       - /downloads/TV
   ```

3. **Use strong passwords** for Flood user accounts

4. **Consider additional authentication** if exposing publicly:
   - Cloudflare Access
   - OAuth proxy (e.g., oauth2-proxy)
   - Traefik ForwardAuth middleware

## Next Steps

1. ✅ Deploy Flood
2. ⏳ Deploy qBittorrent with shared storage
3. ⏳ Configure categories in qBittorrent
4. ⏳ Connect Flood to qBittorrent
5. ⏳ (Optional) Deploy Sonarr/Radarr for automation
6. ⏳ (Optional) Set up monitoring (Prometheus/Grafana)

## Support

- **Flood Documentation**: https://github.com/jesec/flood/wiki
- **Flood Discord**: https://discord.gg/Z7yR5Uf
- **ArgoCD Docs**: https://argo-cd.readthedocs.io/
- **k3s Docs**: https://docs.k3s.io/

## Additional Resources

### Example qBittorrent Helm Values

```yaml
# values-qbittorrent.yaml (for future use)
image:
  repository: linuxserver/qbittorrent
  tag: latest

service:
  port: 8080

ingress:
  enabled: false  # Access via Flood only

storage:
  downloads:
    type: hostPath
    hostPath:
      path: /mnt/media/Plex
    mountPath: /downloads
  config:
    type: hostPath
    hostPath:
      path: /opt/stack/qbittorrent
    mountPath: /config

env:
  - name: PUID
    value: "1000"
  - name: PGID
    value: "1000"
  - name: TZ
    value: "UTC"
  - name: WEBUI_PORT
    value: "8080"
```

### Useful Commands

```bash
# Watch pod logs
kubectl logs -n media -l app.kubernetes.io/name=flood -f

# Shell into pod
kubectl exec -it -n media deployment/flood -- /bin/sh

# Check resource usage
kubectl top pod -n media

# View Helm values
helm get values flood -n media

# Rollback to previous version
helm rollback flood -n media

# ArgoCD app details
kubectl describe application -n argocd flood
```
