# 07 - Homarr

Dashboard aggregator — single page that shows all homelab services with live status.

## Access

- **URL:** http://10.2.7.105
- **Port:** 80 (default)

## Services Added

| Service | Widget Type | URL |
|---------|-------------|-----|
| Hermes Agent | Custom API | http://10.2.7.107:8642/health |
| Grafana | (add via URL widget) | http://10.2.7.108:3000 |

## Adding Widgets

1. Click edit (pencil icon, top right)
2. Click "+" to add a tile
3. Choose widget type
4. Enter the service URL
5. Save

## Future Additions

- [ ] Pi-hole stats widget
- [ ] Immich gallery widget
- [ ] Nextcloud files widget
- [ ] Grafana iframe embed
