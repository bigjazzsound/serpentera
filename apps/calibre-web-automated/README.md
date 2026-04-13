# Calibre-Web-Automated

Automated e-book management system for the Serpentera cluster, deployed via FluxCD and bjw-s app-template.

## Overview

Calibre-Web-Automated is a comprehensive e-book management solution that provides:

- **Automatic Ingest Service** - Processes 27+ e-book formats automatically
- **Format Conversion** - Converts between EPUB, MOBI, AZW3, KEPUB, PDF
- **Metadata Management** - Automatic fetching and enforcement
- **Network Share Support** - Optimized for NFS/SMB with `NETWORK_SHARE_MODE=true`
- **KOReader Sync** - Built-in reading progress synchronization
- **Send-to-eReader** - Email books directly to devices

## Configuration

### Default Setup (Local ZFS Storage)

The app is configured with:
- **Config**: 5Gi on `zfs-dataset` storage class
- **Library**: 100Gi on `zfs-media` storage class  
- **Ingest**: 20Gi on `zfs-media` storage class
- **Access**: https://books.cjf.sh
- **Network Share Mode**: Enabled for compatibility

### Network Share Configuration

To use NFS network shares instead of local storage:

1. **Edit `values-config.yml`**:
   ```yaml
   persistence:
     library:
       type: nfs
       server: "192.168.1.100"  # Your NFS server
       path: "/volume1/calibre/library"  # Your NFS path
       globalMounts:
         - path: /calibre-library
   
     ingest:
       type: nfs  
       server: "192.168.1.100"  # Your NFS server
       path: "/volume1/calibre/ingest"  # Your NFS path
       globalMounts:
         - path: /cwa-book-ingest
   ```

2. **Environment Variables**:
   - `NETWORK_SHARE_MODE: "true"` - Already enabled
   - Disables SQLite WAL for NFS compatibility
   - Uses polling-based file watchers

### Deployment

This app uses FluxCD GitOps deployment:

1. **Edit Configuration**: Modify `values-config.yml` as needed
2. **Commit Changes**: Push to the repository 
3. **FluxCD Sync**: Changes deploy automatically
4. **Monitor**: Check FluxCD and pod status

### Manual Sync (if needed)

```bash
flux reconcile source git flux-system
flux reconcile kustomization calibre-web-automated
```

## Access

- **URL**: https://books.cjf.sh
- **Default Credentials**: 
  - Username: `admin`
  - Password: `admin123`
- **⚠️ Change default credentials immediately after first login!**

## Storage Requirements

| Volume | Purpose | Default Size | Storage Class |
|--------|---------|--------------|---------------|
| Config | App settings & DB | 5Gi | zfs-dataset |
| Library | E-book collection | 100Gi | zfs-media |
| Ingest | Temporary processing | 20Gi | zfs-media |

## Features

### Network Share Optimizations

When `NETWORK_SHARE_MODE=true`:
- SQLite WAL mode disabled
- Polling-based file watchers vs inotify
- Optimized for network filesystem latency
- Recursive ownership changes disabled

### Automatic Processing

- **File Formats**: 27+ supported formats
- **Auto-Convert**: Configurable target formats
- **Metadata**: Automatic fetching from multiple sources
- **EPUB Fixing**: Repairs common compatibility issues
- **Backup**: Original files preserved during processing

### Reading Integration

- **KOReader Sync**: Progress sync with e-readers
- **Send-to-eReader**: Email books to devices
- **OPDS Feed**: Compatible with e-reader apps
- **Web Reader**: Built-in browser reading

## Troubleshooting

### Common Issues

1. **Database Locked**: Ensure `NETWORK_SHARE_MODE=true`
2. **Permission Errors**: Check NFS export settings and PUID/PGID
3. **Books Not Ingesting**: Verify ingest folder permissions
4. **Slow Performance**: Expected with network shares vs local storage

### Monitoring

```bash
# Check app status
kubectl get pods -n calibre-web-automated

# View app logs  
kubectl logs -n calibre-web-automated deployment/calibre-web-automated

# Check FluxCD status
kubectl get kustomizations -n flux-system calibre-web-automated
kubectl describe kustomization -n flux-system calibre-web-automated

# Check HelmRelease
kubectl get helmreleases -n flux-system calibre-web-automated
```

### Manual Restart

```bash
kubectl rollout restart deployment/calibre-web-automated -n calibre-web-automated
```

## Security Notes

- Change default admin credentials immediately
- Consider enabling authentication providers (OAuth2/OIDC)
- Review file permissions on network shares
- Monitor access logs for unusual activity

## Links

- [Calibre-Web-Automated GitHub](https://github.com/crocodilestick/Calibre-Web-Automated)
- [bjw-s app-template docs](https://bjw-s.github.io/helm-charts/docs/app-template/)
- [Discord Community](https://discord.gg/EjgSeek94R)