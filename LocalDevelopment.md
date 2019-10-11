# Forward all `.kube` host for local development

Install Dnsmasq (or [DNSAgent](https://github.com/stackia/DNSAgent) on Windows) and configure it to match any request which ends in `.kube` and send `192.168.205.11` in response.

```bash
# Example for dnsmasq.conf
address=/kube/192.168.205.11
```
