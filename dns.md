# Local DNS Rewrite
 - Enter the domain name or wildcard you want to be rewritten.
 - gatus.lab

 - IP address
 - 192.168.0.50

# Cloudflare DNS Rewrite [make sure ingress.yaml have domain info]
Zero Trust -> Netwoks -> Connectors -> Published application routes -> 
Select Subdomain and Domain
Type  : HTTPS
URL   : 192.168.0.50:443
