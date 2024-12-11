Commands List for CrowdSec

```
docker exec crowdsec cscli bouncers add crowdsecBouncer
```
# Generate API key, One-Time generateable only.Keep safe!

```
docker exec crowdsec cscli metrics
```

# For Metrics and log read checking

```
docker exec crowdsec cscli hub update && docker exec crowdsec cscli hub upgrade
```
# Security List update command

For Decisions : 
```
docker exec crowdsec cscli decisions list
```
# List crowdsec ip decisions
```
docker exec crowdsec cscli decisions add --ip 192.168.1.100
```
# add ip to crowdsec decisions block list
```
docker exec crowdsec cscli decisions delete --ip 192.168.1.100
```
# delete ip from crowdsec decisions block list