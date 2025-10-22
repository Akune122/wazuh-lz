# Ajouter cette partie dans le ossec.conf (à la fin du premier <ossec_global>)

  <integration>
    <name>custom-email</name>
    <hook_url>none</hook_url>
    <alert_format>json</alert_format>
  </integration>

# Accéder au manager sur Docker (docker exec -it single-node-wazuh.manager-1 bash)

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

# 🔒 On n’envoie que les alertes de niveau 8 à 12
if level < 8:
    sys.exit(0)

rule = alert['rule']
agent = alert['agent']
description = rule.get('description', 'No description')
rule_id = rule.get('id', 'Unknown')
hostname = agent.get('name', 'Unknown')
timestamp = alert.get('timestamp', 'Unknown')

# Couleur principale selon le niveau
if level < 9:
    color = "#f0ad4e"  # jaune/orange
elif level < 11:
    color = "#ff7043"  # orange soutenu
else:
    color = "#e53935"  # rouge fort

# HTML stylé
html = f"""
<html>
<head>
  <style>
    body {{
      font-family: 'Segoe UI', Roboto, sans-serif;
      background-color: #f5f6fa;
      padding: 30px;
    }}
    .card {{
      background: white;
      border-radius: 12px;
      box-shadow: 0 4px 15px rgba(0,0,0,0.08);
      max-width: 700px;
      margin: auto;
      overflow: hidden;
    }}
    .header {{
      background: {color};
      color: white;
      padding: 18px 25px;
      font-size: 22px;
      font-weight: 600;
      letter-spacing: 0.5px;
    }}
    .content {{
      padding: 20px 25px;
      color: #333;
    }}
    table {{
      width: 100%;
      border-collapse: collapse;
      margin-top: 10px;
    }}
    td {{
      padding: 8px;
      border-bottom: 1px solid #eee;
    }}
    td.label {{
      font-weight: 600;
      width: 30%;
      color: #444;
    }}
    pre {{
      background: #f4f4f4;
      padding: 10px;
      border-radius: 8px;
      overflow-x: auto;
      font-size: 13px;
    }}
    .footer {{
      text-align: center;
      color: #888;
      font-size: 12px;
      padding: 12px;
      background: #fafafa;
      border-top: 1px solid #eee;
    }}
  </style>
</head>
<body>
  <div class="card">
    <div class="header">🚨 Alerte Wazuh — Niveau {level}</div>
    <div class="content">
      <table>
        <tr><td class="label">📝 Description :</td><td>{description}</td></tr>
        <tr><td class="label">⚙️ Règle :</td><td>{rule_id}</td></tr>
        <tr><td class="label">💻 Agent :</td><td>{hostname}</td></tr>
        <tr><td class="label">🕒 Date :</td><td>{timestamp}</td></tr>
      </table>
      <br>
      <b>Détails techniques :</b>
      <pre>{json.dumps(alert, indent=2)}</pre>
    </div>
    <div class="footer">
      Rapport d’alerte généré automatiquement par Wazuh SIEM
    </div>
  </div>
</body>
</html>
"""

subject = f"[Wazuh 🚨] Alerte niveau {level} - {description}"

# Configuration email
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
