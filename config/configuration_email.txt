# Ajouter cette partie dans le ossec.conf (à la fin du premier <ossec_global>)

  <integration>
    <name>custom-email</name>
    <hook_url>none</hook_url>
    <alert_format>json</alert_format>
  </integration>


# Créer le fichier custom-email
nano /var/ossec/integrations/custom-email

# Script du mail d'alerte perso 

#!/usr/bin/env python3
import json
import sys
import smtplib
from email.mime.text import MIMEText

# Lecture du fichier d'alerte
alert_file = sys.argv[1]
with open(alert_file) as f:
    alert = json.load(f)

level = alert['rule']['level']

# 🔒 Filtrer uniquement les alertes de niveau 8 à 12
if level < 8:
    sys.exit(0)

# Données principales
rule = alert['rule']
agent = alert['agent']
description = rule.get('description', 'No description')
rule_id = rule.get('id', 'Unknown')
hostname = agent.get('name', 'Unknown')
timestamp = alert.get('timestamp', 'Unknown')

# Couleur selon le niveau
if level < 8:
    color = "#4CAF50"  # vert
elif level < 10:
    color = "#FFC107"  # orange
else:
    color = "#F44336"  # rouge

# Contenu HTML du mail
html = f"""
<html>
<head>
  <style>
    body {{
      font-family: 'Segoe UI', Arial, sans-serif;
      background-color: #f8f9fa;
      color: #333;
      padding: 20px;
    }}
    .card {{
      background-color: white;
      border-radius: 10px;
      box-shadow: 0 2px 8px rgba(0,0,0,0.1);
      padding: 20px;
      max-width: 600px;
      margin: auto;
    }}
    .header {{
      font-size: 20px;
      font-weight: bold;
      color: {color};
      margin-bottom: 10px;
    }}
    .section {{
      margin-bottom: 10px;
    }}
    .footer {{
      font-size: 12px;
      color: #777;
      text-align: center;
      margin-top: 20px;
    }}
  </style>
</head>
<body>
  <div class="card">
    <div class="header">🚨 Alerte Wazuh Niveau {level}</div>
    <div class="section">
      <b>Description :</b> {description}<br>
      <b>Règle :</b> {rule_id}<br>
      <b>Agent :</b> {hostname}<br>
      <b>Date :</b> {timestamp}
    </div>
    <div class="section">
      <b>Détails techniques :</b><br>
      <pre style="background:#f1f1f1;padding:10px;border-radius:8px;">{json.dumps(alert, indent=2)}</pre>
    </div>
    <div class="footer">
      Alerte générée automatiquement par Wazuh SIEM
    </div>
  </div>
</body>
</html>
"""

subject = f"[Wazuh ⚠️] Niveau {level} - {description}"

# Email
sender = "wazuh.jean23@gmail.com"
recipient = "loic.zaercher@jean23.org"
smtp_server = "postfix"
smtp_port = 25

msg = MIMEText(html, "html")
msg["Subject"] = subject
msg["From"] = sender
msg["To"] = recipient

try:
    with smtplib.SMTP(smtp_server, smtp_port) as server:
        server.sendmail(sender, [recipient], msg.as_string())
except Exception as e:
    print(f"Erreur d'envoi email: {e}")
    sys.exit(1)


# Donner les droits d'execution au script

chmod 750 /var/ossec/integrations/custom-email
chown root:wazuh /var/ossec/integrations/custom-email
