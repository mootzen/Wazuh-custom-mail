# Wazuh custom mail integration

Fork of JCT-Wazuh

Wazuh's default alert mails have a very generic subject that doesn't give much information on whats actually happening.

e.g. default subject: Wazuh notification - (DC-4) any - Alert level 12

To change that I forked JCT's custom mail integration and built on top of it.

The main focus of this integration is to highlight ONLY relevant information of the alert and to create a subject line that gives you an overview of what happened on the first glance

## Changes

- Subject format: HOST: Description
  - Clean and scannable; avoids noisy rule levels/IDs in the subject.

- Plain-text body (German) with key fields:
  - Host, Level, Rule-ID, EventID, Benutzer, Quelle-IP, Quelle-Host, Pfad → one-glance context without JSON walls.

- Robust Windows field extraction:
  - Picks the correct Benutzer depending on event family:
- Account-management (e.g. 4720/4722/4725/4738/4740/4767…): actor (SubjectDomainName\SubjectUserName)
  - Logon family (e.g. 4624/4625/4634/4672/4776/4768/4769/4771…): target (TargetDomainName\TargetUserName)
- Source IP resolution across multiple fields (IpAddress, SourceNetworkAddress, ClientAddress, …) with loopback filtering and reverse DNS fallback from WorkstationName.
  - → Fixes the common “::1 / 127.0.0.1” problem.
- JSON in, plain body out:
  - parse <alert_format>json</alert_format> for correctness, but send plain text. Optional env switch to append the pretty JSON when needed.
- Multi-recipient support:
  - Comma/semicolon separated recipients in <hook_url>.
- No duplicate emails:
  - Clear guidance to disable stock maild for the same levels.

## Install / Setup
```
/var/ossec/integrations/custom-email-alerts
# ensure executable & owned by wazuh group
chown root:wazuh /var/ossec/integrations/custom-email-alerts
chmod 0750 /var/ossec/integrations/custom-email-alerts
```
### ossec.conf
```
<integration>
  <name>custom-email-alerts</name>
  <hook_url>secops@domain.tld,netops@domain.tld</hook_url>  <!-- recipients -->
  <level>12</level>
  <alert_format>json</alert_format>                          <!-- important -->
  <!-- optional filters:
       <rule_id>5710,5711,4740</rule_id>
       <group>windows,authentication_failed</group>
  -->
</integration>

<alerts>
  <log_alert_level>3</log_alert_level>
  <email_alert_level>0</email_alert_level>  <!-- disable stock mailer -->
</alerts>

<global>
  <email_notification>no</email_notification>                <!-- also disable -->
</global>
```
> [!IMPORTANT]
> Set <email_alert_level>0</email_alert_level> and <email_notification>no</email_notification> to stop the stock maild.
Also remove or comment any <email_alerts> blocks.

To include the full JSON at the end (for debugging or threading rules), set:
```
INCLUDE_JSON_IN_BODY=1
```
### set server & sender
Default SMTP and sender can be set via env or by editing the top of the script.

Environment (preferred): create a systemd drop-in (optional):
```
/etc/systemd/system/wazuh-manager.service.d/email.env.conf
```
```
[Service]
Environment=EMAIL_SERVER=localhost
Environment=EMAIL_FROM=email-notifications@domain.tld
# Optional toggles:
# Environment=INCLUDE_JSON_IN_BODY=1
# Environment=RESOLVE_WORKSTATION_IP=0
```
reload & restart:
```
systemctl daemon-reload
systemctl restart wazuh-manager
```

## Formatting
### Subject
```<agent.name>: <rule.description>```

### Body
```
Automatische Benachrichtigung.

Am 13.10.2025 12:11:12 hat ein Ereignis auf "dc" folgende Regel ausgelöst: "Domänen-Admin eingeloggt: ...".

Details:
  Host:        dc
  Level:       12
  Rule-ID:     100096
  EventID:     4624
  Benutzer:    DOMAIN\alice           <-- smart selection per event family
  Quelle-IP:   10.0.0.25              <-- multi-field + reverse DNS fallback
  Quelle-Host: WS123
  Pfad:        -
```
## Troubleshooting

### Still get duplicate mails?
```
grep -RIn '<email_alerts>|email_alert_level|email_notification' /var/ossec/etc
ls -l /var/ossec/integrations/
```
Ensure only one custom-email-alerts script exists and only one <integration> block references it

### Only one recipient gets mail?
- Use comma or semicolon in <hook_url>, e.g. secops@d.tld,netops@d.tld
