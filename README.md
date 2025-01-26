# Postfix relay running in Kubernetes

This repository has an example of a postfix relay running in Kubernetes using a helm chart via argocd. 
Fork of https://github.com/eldada/postfix-relay-kubernetes 


## Build Docker image
You can build the Docker image locally
```bash
# For local build
docker build -t anantharc/postfix:3.0.2 Docker/

# Multi arch build and push
docker buildx build --platform linux/amd64,linux/arm64 -t anantharc/postfix:3.0.2 Docker/ --push
```

## Run locally with Docker
Run the postfix relay locally for testing
```bash
# Need to set SMTP connection details
export SMTP="[smtp.mailgun.org]:587"
export USERNAME_TEST=<your smtp username>
export PASSWORD_TEST=<your smtp password>

# Optional custom configuration to add/override in /etc/postfix/main.cf (delimited by a ";")
export POSTFIX_CUSTOM_CONFIG="key1 = value1;key2 = value2;key3 = value3"

# Set list of allowed networks
export TX_SMTP_RELAY_NETWORKS='10.0.0.0/8,127.0.0.0/8,172.17.0.0/16,192.0.0.0/8'

docker run --rm -d --name postfix-relay -p 2525:25 \
	-e TX_SMTP_RELAY_HOST="${SMTP}" \
	-e TX_SMTP_RELAY_MYHOSTNAME=my.local \
	-e TX_SMTP_RELAY_USERNAME=${USERNAME_TEST} \
	-e TX_SMTP_RELAY_PASSWORD=${PASSWORD_TEST} \
	-e TX_SMTP_RELAY_NETWORKS=${TX_SMTP_RELAY_NETWORKS} \
	-e POSTFIX_CUSTOM_CONFIG="${POSTFIX_CUSTOM_CONFIG}" \
	eldada.jfrog.io/docker/postfix-relay:0.9
```

### Test sending mail
1. Connect to running container on port 2525
```bash
telnet localhost 2525
```

2. Edit the following with your details and paste in your terminal
```bash
helo localhost
mail from: noreply@yourhost.com
rcpt to: you@your.co
data
Subject: Subject here...
The true story of swans singing Pink Floyd. 
.
quit
```

3. You should see the following
```bash
220 tx-smtp-relay.yourhost.com ESMTP Postfix
helo localhost
250 tx-smtp-relay.yourhost.com
mail from: noreply@yourhost.com
250 2.1.0 Ok
rcpt to: you@your.co
250 2.1.5 Ok
data
354 End data with <CR><LF>.<CR><LF>
Subject: Subject here...
The true story of swans singing Pink Floyd. 
.
250 2.0.0 Ok: queued as 982FF53C
quit
221 2.0.0 Bye
Connection closed by foreign host
```

4. Check the inbox of `you@your.co` and see you got the email.


## Deploy Helm Chart
The Helm Chart in [helm/postfix](helm/postfix) directory can be used to deploy the postfix-relay into your Kubernetes cluster.

Create a `custom-values.yaml` with the configuration details
```yaml
smtp:
  relayHost: "[smtp.mailgun.org]:587"
  relayMyhostname: <your smtp hostname>
  relayUsername: <your smtp username>
  relayPassword: <your smtp password>
  relayNetworks: '10.0.0.0/8,127.0.0.0/8,172.17.0.0/16,192.0.0.0/8'
```

Deploy postfix
```bash
helm upgrade --install postfix-relay helm/postfix -f custom-values.yaml
```


To enable the exporter sidecar, update your `custom-values.yaml` file and **add**
```yaml
# Enable the postfix-exporter sidecar
exporter:
  enabled: true

# Enable a ServiceMonitor object for Prometheus scraping
serviceMonitor:
  enabled: true
```

Deploy postfix
```bash
helm upgrade --install postfix-relay helm/postfix -f custom-values.yaml
```


ArgoCD: 
deploying new application

```app create postfix --repo https://github.com/aa04arc/postfix-relay-kubernetes-master.git --path helm/postfix --dest-server https://kubernetes.default.svc --dest-namespace default```
