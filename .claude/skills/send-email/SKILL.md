---
name: send-email
description: >
  Envoyer des emails via un serveur SMTP. Prend en charge les emails simples
  (texte brut), HTML, et les pièces jointes. Triggers: "envoie un email",
  "envoie un mail à", "envoyer un message par email", "notifie par email",
  "send email", or any request to send an email or notification via email.
---

# Send Email Skill

Envoyer des emails via un serveur SMTP configuré par variables d'environnement.

---

## Setup

```bash
source .env.integrations
# Requis : SMTP_HOST, SMTP_PORT, SMTP_USER, SMTP_PASS, SMTP_FROM
# Optionnel : SMTP_SECURE (true/false, défaut true si port 465)
```

### Variables d'environnement

| Variable | Description | Exemple |
|---|---|---|
| `SMTP_HOST` | Hôte du serveur SMTP | `smtp.gmail.com` |
| `SMTP_PORT` | Port SMTP | `465` (SSL) ou `587` (STARTTLS) |
| `SMTP_USER` | Identifiant SMTP | `user@exemple.fr` |
| `SMTP_PASS` | Mot de passe ou App Password | `xxxx xxxx xxxx xxxx` |
| `SMTP_FROM` | Adresse expéditeur par défaut | `"Mon App" <noreply@exemple.fr>` |
| `SMTP_SECURE` | Forcer SSL (optionnel) | `true` ou `false` |

---

## Envoyer un email — Python (smtplib)

Script réutilisable `send_email.py` :

```python
#!/usr/bin/env python3
import os, smtplib, ssl
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.mime.base import MIMEBase
from email import encoders

def send_email(
    to: str | list[str],
    subject: str,
    body: str,
    html: str | None = None,
    attachments: list[str] | None = None,
    cc: str | list[str] | None = None,
    reply_to: str | None = None,
):
    host     = os.environ["SMTP_HOST"]
    port     = int(os.environ["SMTP_PORT"])
    user     = os.environ["SMTP_USER"]
    password = os.environ["SMTP_PASS"]
    from_    = os.environ.get("SMTP_FROM", user)
    secure   = os.environ.get("SMTP_SECURE", "true").lower() == "true" or port == 465

    recipients = [to] if isinstance(to, str) else to
    cc_list    = ([cc] if isinstance(cc, str) else cc) if cc else []

    msg = MIMEMultipart("alternative" if html else "mixed")
    msg["Subject"] = subject
    msg["From"]    = from_
    msg["To"]      = ", ".join(recipients)
    if cc_list:
        msg["Cc"] = ", ".join(cc_list)
    if reply_to:
        msg["Reply-To"] = reply_to

    msg.attach(MIMEText(body, "plain", "utf-8"))
    if html:
        msg.attach(MIMEText(html, "html", "utf-8"))

    for path in (attachments or []):
        with open(path, "rb") as f:
            part = MIMEBase("application", "octet-stream")
            part.set_payload(f.read())
        encoders.encode_base64(part)
        part.add_header("Content-Disposition", f'attachment; filename="{os.path.basename(path)}"')
        msg.attach(part)

    all_recipients = recipients + cc_list
    context = ssl.create_default_context()

    if secure:
        with smtplib.SMTP_SSL(host, port, context=context) as smtp:
            smtp.login(user, password)
            smtp.sendmail(from_, all_recipients, msg.as_string())
    else:
        with smtplib.SMTP(host, port) as smtp:
            smtp.ehlo()
            smtp.starttls(context=context)
            smtp.login(user, password)
            smtp.sendmail(from_, all_recipients, msg.as_string())

    print(f"Email envoyé à {', '.join(all_recipients)} — sujet : {subject}")
```

### Usage

```python
# Email simple texte
send_email(
    to="destinataire@exemple.fr",
    subject="Rapport hebdomadaire",
    body="Bonjour,\n\nVeuillez trouver ci-joint le rapport.\n\nCordialement",
)

# Email HTML avec CC
send_email(
    to=["alice@exemple.fr", "bob@exemple.fr"],
    cc="manager@exemple.fr",
    subject="Alerte système",
    body="Une anomalie a été détectée.",
    html="<h2>Alerte</h2><p>Une <strong>anomalie</strong> a été détectée.</p>",
)

# Email avec pièce jointe
send_email(
    to="client@exemple.fr",
    subject="Votre facture",
    body="Veuillez trouver votre facture en pièce jointe.",
    attachments=["/tmp/facture_2026_04.pdf"],
)
```

---

## Envoyer un email — Node.js (nodemailer)

```bash
npm install nodemailer
```

```js
const nodemailer = require("nodemailer");

async function sendEmail({ to, subject, text, html, attachments, cc, replyTo }) {
  const transporter = nodemailer.createTransport({
    host: process.env.SMTP_HOST,
    port: parseInt(process.env.SMTP_PORT),
    secure: process.env.SMTP_SECURE === "true" || process.env.SMTP_PORT === "465",
    auth: {
      user: process.env.SMTP_USER,
      pass: process.env.SMTP_PASS,
    },
  });

  const info = await transporter.sendMail({
    from: process.env.SMTP_FROM || process.env.SMTP_USER,
    to: Array.isArray(to) ? to.join(", ") : to,
    cc,
    replyTo,
    subject,
    text,
    html,
    attachments: (attachments || []).map((path) => ({ path })),
  });

  console.log(`Email envoyé : ${info.messageId}`);
  return info;
}

// Usage
await sendEmail({
  to: "destinataire@exemple.fr",
  subject: "Test",
  text: "Bonjour depuis nodemailer",
});
```

---

## Envoyer un email — curl (via API relay)

Si un relay HTTP est configuré (ex. Resend, Sendgrid, Brevo) :

```bash
# Exemple avec Brevo (ex-Sendinblue)
curl -X POST https://api.brevo.com/v3/smtp/email \
  -H "api-key: $BREVO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "sender": {"name": "Mon App", "email": "noreply@exemple.fr"},
    "to": [{"email": "dest@exemple.fr"}],
    "subject": "Mon sujet",
    "textContent": "Corps de l email en texte"
  }'
```

---

## Providers courants

| Provider | Host | Port SSL | Port STARTTLS | Notes |
|---|---|---|---|---|
| Gmail | `smtp.gmail.com` | 465 | 587 | Nécessite un App Password (2FA activée) |
| Outlook / Microsoft 365 | `smtp.office365.com` | — | 587 | STARTTLS uniquement |
| OVH | `ssl0.ovh.net` | 465 | 587 | |
| Infomaniak | `mail.infomaniak.com` | 465 | 587 | |
| Brevo (Sendinblue) | `smtp-relay.brevo.com` | 465 | 587 | Ou API REST |

### Gmail — obtenir un App Password

1. Activer la validation en 2 étapes sur le compte Google
2. Aller dans **Compte Google → Sécurité → Mots de passe des applications**
3. Créer un mot de passe pour "Autre (nom personnalisé)"
4. Utiliser ce mot de passe de 16 caractères comme `SMTP_PASS`

---

## Bonnes pratiques

- **Ne jamais hardcoder** les credentials — toujours utiliser des variables d'environnement
- **Vérifier** l'adresse destinataire avant d'envoyer (surtout en production)
- **Tester** d'abord avec une adresse de test (`+test` suffix si Gmail)
- **Limiter** le volume : les serveurs SMTP ont des quotas (Gmail : 500/jour, Brevo gratuit : 300/jour)
- Pour les emails en masse, préférer une API transactionnelle (Brevo, Sendgrid, Resend)

---

## Erreurs fréquentes

| Erreur | Cause | Fix |
|---|---|---|
| `535 Auth failed` | Mauvais user/pass | Vérifier les credentials, utiliser App Password pour Gmail |
| `Connection refused` | Mauvais host ou port bloqué | Vérifier `SMTP_HOST`, `SMTP_PORT`, firewall |
| `SSL handshake failed` | Port/mode SSL incorrect | Port 465 → `secure: true`, port 587 → STARTTLS |
| `Recipient rejected` | Adresse invalide ou bloquée | Vérifier l'adresse destinataire |
| `Daily limit exceeded` | Quota SMTP dépassé | Changer de provider ou attendre 24h |
