# Self-host OSRM — אופציות חינמיות

> **למה?** היום `https://router.project-osrm.org` (public demo) מקבל כל waypoint של כל arrest plan. self-host מסיר את ההדלפה הזו.
> **דרישה:** VM חיצוני + Docker. **לא אוטומטי** — דורש לוגאין SSH.

---

## אופציות חינמיות (אין צורך לשלם)

| ספק | מה מקבלים חינם | מתאים? | תוקף |
|---|---|---|---|
| **Oracle Cloud Free Tier** | 4 ARM cores, 24GB RAM, 200GB disk, 10TB egress | ✓ הכי טוב | Forever |
| **Google Cloud Free Tier** | e2-micro (0.25 vCPU, 1GB RAM), 30GB | מצומצם, יספיק | Forever |
| **AWS Free Tier** | t2.micro 1GB RAM, 30GB EBS | מצומצם | **12 חודשים בלבד** |
| **Fly.io Free** | 3× 256MB VMs, 3GB volume | מצומצם | Forever (אם פעיל) |

**המלצה:** Oracle Free Tier — הכי הרבה משאבים, חינמי לעד, מתאים בקלות ל-OSRM של ישראל (~350MB OSM data, ~1GB RAM בזמן ריצה).

---

## אופציה A — Oracle Free Tier (מומלץ)

### A1 — צור חשבון + VM

1. https://www.oracle.com/cloud/free → **Start for free** (דורש כרטיס אשראי לאימות, **לא נחיוב**)
2. אחרי הרשמה → **Create a VM Instance**:
   - Shape: **Ampere A1** (ARM, חינמי)
   - OCPU: **4**, Memory: **24 GB**
   - Image: **Ubuntu 22.04**
   - Networking: **Assign a public IP**, save the SSH key
3. אחרי יצירה → רשום את ה-Public IP.

### A2 — פתח port 5000 ב-firewall

1. ב-Oracle Console → ה-VM → **Subnet** → **Security List Default** → **Add Ingress Rule**
2. Source CIDR: `0.0.0.0/0`, Protocol: `TCP`, Destination Port: `5000`
3. גם ב-VM: `sudo ufw allow 5000`

### A3 — התקן OSRM ב-VM

```bash
ssh ubuntu@<your-vm-ip>
sudo apt update && sudo apt install -y docker.io
sudo usermod -aG docker $USER && exit
ssh ubuntu@<your-vm-ip>  # re-login for docker group

mkdir ~/osrm && cd ~/osrm
wget https://download.geofabrik.de/asia/israel-and-palestine-latest.osm.pbf

# Build (~10 דק׳)
docker run --rm -v "${PWD}:/data" ghcr.io/project-osrm/osrm-backend osrm-extract -p /opt/foot.lua /data/israel-and-palestine-latest.osm.pbf
docker run --rm -v "${PWD}:/data" ghcr.io/project-osrm/osrm-backend osrm-partition /data/israel-and-palestine-latest.osrm
docker run --rm -v "${PWD}:/data" ghcr.io/project-osrm/osrm-backend osrm-customize /data/israel-and-palestine-latest.osrm

# Run (auto-restart on boot)
docker run -d --name osrm --restart=always -p 5000:5000 -v "${PWD}:/data" \
  ghcr.io/project-osrm/osrm-backend osrm-routed --algorithm mld /data/israel-and-palestine-latest.osrm

# בדיקה
curl 'http://localhost:5000/route/v1/foot/35.0,31.9;35.1,31.9?overview=false'
# צריך לחזור JSON עם distance/duration
```

### A4 — TLS חינמי דרך Caddy

```bash
# קנה דומיין? אם לא — אפשר להשתמש ב-DuckDNS חינמי:
# https://www.duckdns.org → צור subdomain (e.g. mabat443.duckdns.org)
# הצבע אותו לכתובת ה-VM

sudo apt install -y caddy
sudo tee /etc/caddy/Caddyfile <<EOF
mabat443.duckdns.org {
    reverse_proxy localhost:5000
    header {
        Access-Control-Allow-Origin "https://elchai.github.io"
        Access-Control-Allow-Methods "GET, OPTIONS"
    }
}
EOF
sudo systemctl reload caddy
# Caddy מטפל ב-Let's Encrypt אוטומטית
```

### A5 — החלף ב-`index.html`

מצא:
```javascript
const osrmUrl = `https://router.project-osrm.org/route/v1/${profile}/...`;
```

החלף ל:
```javascript
const osrmUrl = `https://mabat443.duckdns.org/route/v1/${profile}/...`;
```

עדכן את ה-CSP `connect-src` ב-`<head>`: הסר `https://router.project-osrm.org` והוסף `https://mabat443.duckdns.org`.

---

## אופציה B — Fly.io (אם Oracle לא מסתדר)

Fly.io נותן 3× 256MB VMs בחינם. 256MB דחוק ל-OSRM של ישראל אבל יכול לעבוד עם `--algorithm ch` (channeling) במקום `mld`. תוצאות: איטיות יותר אבל זוכרת פחות RAM.

```bash
# התקן flyctl
iwr https://fly.io/install.ps1 -useb | iex
flyctl auth signup

# צור app
mkdir osrm-fly && cd osrm-fly
flyctl launch --name mabat443-osrm --no-deploy
# ערוך fly.toml: image = "ghcr.io/project-osrm/osrm-backend", הוסף command osrm-routed
flyctl deploy
```

---

## עלות (לוודא שזה חינם)

- **Oracle**: 4 ARM cores + 24GB RAM הם **Always Free**. שום חיוב.
- **Fly.io**: 3 small VMs + 3GB volume = $0 (כל עוד אתה תחת המכסה).
- **OSM data**: חינם (Geofabrik).
- **Caddy + Let's Encrypt**: חינם.
- **DuckDNS**: חינם.

**אם פספסת משהו ועלה כסף — תוכל לבטל את ה-VM וזה $0 חיוב.**

---

## אופציה C — לא לעשות עכשיו

אם החלטת לדחות, הסיכון הוא:
- **OSRM ימשיך לקבל את כל ה-waypoints של arrest plans**. ה-public demo server ([project-osrm.org](https://project-osrm.org)) מתעד queries באקסס לוג.
- הסיכון הזה לא מסכן את **משתמשים** אחרים — רק מדליף את הגזרה ואת היעדים שלך לצד שלישי.
- **חלופה ביניים זולה:** השאר OSRM הציבורי כברירת מחדל, ותראה ידנית באמצעות Gmaps deep-link (האפשרות הקיימת בדשבורד).
