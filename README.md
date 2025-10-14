# Wazuh custom mail integration
## Fork of JCT-Wazuh


## ossec.conf
```
  <integration>
    <name>custom-email-alerts</name>
    <hook_url>recipient@domain.tld</hook_url>   <!-- recipient(s) -->
    <level>12</level>
    <alert_format>json</alert_format>
  <!-- optionale filter:
       <rule_id>5710,5711,4740</rule_id>
       <group>windows,authentication_failed</group>
       <event_location>^win</event_location>
  -->
  </integration>
```
