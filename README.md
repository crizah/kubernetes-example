# Kubernetes Sample: my-k8s-ingress-app

A small example showing how to expose a frontend and backend service through on k3s (or any Kubernetes cluster with an Ingress controller running on bare metal).  
This repo contains Kubernetes manifests (Ingress, Deployments/Services examples) and instructions so you can run the app locally or deploy it on your own cluster.

Important: everywhere the manifests used `myapp.local` in examples, replace that with your own host value. In these docs we use the placeholder `MY_APP_HOST` replace it with a real hostname (the extrenal ip of your cluster)

Note: If you are running this on minikube without a driver set to none, this will not work


- manifests:
  - ingress.yaml                      (single Ingress for frontend + API, references MY_APP_HOST)
  - server-deployment.yaml           (example backend deployment; sets COOKIE_SECURE via env)
  - frontend-deployment.yaml          (example frontend deployment; sets BACKEND_URL env)
  - server-service.yaml               (service for server)
  - frontend-service.yaml             (service for frontend)
  - server-secrets.yaml               (server secrets)


<!-- 
What this demonstrates
- Use one Ingress to route:
  - /api -> backend-service:8081  (server)
  - / -> frontend-service:80       (client)
- Strip the `/api` prefix for the backend using a Traefik middleware (so backend routes remain `/login`, `/profile`, etc.)
- Local dev and production HTTPS options
- How to configure cookies securely when behind an ingress (X-Forwarded-Proto or env var) -->

<!-- Before you start (choose one workflow)
- Local development (fast, no DNS): use a hostname you map in `/etc/hosts` or use nip.io and optional mkcert for local TLS.
- Real cluster / public domain: use cert-manager + Let's Encrypt or your normal production certificate provisioning. -->

Prerequisites
- kubectl configured to talk to your k8s cluster (any bare metal cluster)
- A running Ingress controller

Pick and replace the hostname
- Replace `MY_APP_HOST` in the manifests with the hostname you want to use.
  - Local example using hosts file: `MY_APP_HOST=myapp.local`
  - nip.io example (no hosts file edit): `MY_APP_HOST=myapp.<NODE_IP>.nip.io`
  - Public domain example: `MY_APP_HOST=app.example.com`

Quick commands to replace the placeholder locally (example):
- Using envsubst (POSIX shell):
  export MY_APP_HOST="myapp.local"
  envsubst < ingress-single.yaml.template > ingress-single.yaml
- Or a simple sed:
  sed "s/MY_APP_HOST/myapp.local/g" ingress-single.yaml.template > ingress-single.yaml

NOTE: The repo may contain YAMLs with the placeholder `MY_APP_HOST`. Replace it before applying.

Local dev (HTTP only, quick)
If you don't want to set up TLS and are OK with insecure cookies in dev:

1) Make cookies non-secure in dev (only for development)
- Either set an environment variable on the backend pod so the server uses Secure=false for cookies, or update the code to consider an env var (recommended).
  Example (patch deployment):
  kubectl set env deployment/backend-deployment COOKIE_SECURE=false

2) Ensure frontend calls send cookies
- If frontend is same origin (served by same host), BACKEND_URL="/api" works.
- For fetch: fetch('/api/login', { credentials: 'include', ... })
- For axios: axios.post('/api/login', data, { withCredentials: true })

3) Apply the manifests
- Replace the host placeholder as described above, then:
  kubectl apply -f middleware-strip-api.yaml
  kubectl apply -f ingress-single.yaml
  kubectl apply -f backend-deployment.yaml
  kubectl apply -f frontend-deployment.yaml

4) If using a custom local hostname (myapp.local), map it to the node IP in your /etc/hosts:
  sudo -- sh -c 'echo "192.168.1.100 myapp.local" >> /etc/hosts'
  Replace 192.168.1.100 with the node / loadbalancer IP reachable from your machine.

5) Test (HTTP)
- From your machine (after /etc/hosts or nip.io):
  curl -v -H "Host: MY_APP_HOST" http://<TRAefik-IP>/api/login
  or open: http://MY_APP_HOST/  (frontend)

Local dev (HTTPS, recommended for realistic testing)
Use mkcert to create a certificate trusted by your local machine and let Traefik terminate TLS.

1) Install mkcert (https://github.com/FiloSottile/mkcert) and create a cert:
  mkcert -install
  mkcert MY_APP_HOST
  → produces: MY_APP_HOST.pem and MY_APP_HOST-key.pem

2) Create a TLS secret in Kubernetes:
  kubectl create secret tls myapp-tls --cert=MY_APP_HOST.pem --key=MY_APP_HOST-key.pem -n default

3) Update / replace Ingress to include TLS (example snippet in the repo), and ensure annotation uses `websecure` entrypoint:
  spec:
    tls:
    - hosts:
      - MY_APP_HOST
      secretName: myapp-tls
  metadata.annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.middlewares: default-strip-api@kubernetescrd

4) Apply the manifests:
  kubectl apply -f middleware-strip-api.yaml
  kubectl apply -f ingress-single.yaml
  ...

5) Access the site:
  https://MY_APP_HOST/
  Your browser will trust the mkcert cert and cookies set with Secure=true will work if your Go code sets Secure when X-Forwarded-Proto == "https" or if COOKIE_SECURE=true.

Production / public domain (Let's Encrypt + cert-manager)
1) Install cert-manager in the cluster (recommended by cert-manager docs).
2) Deploy an Issuer / ClusterIssuer for Let's Encrypt (staging first, then production).
3) Add the `tls` section to the Ingress and annotate for cert-manager if necessary (cert-manager will create and populate the TLS secret).
4) Ensure DNS for MY_APP_HOST points to your cluster's external IP / load balancer.
5) Configure your backend to always set Secure cookies (COOKIE_SECURE=true) or detect X-Forwarded-Proto.

Example cert-manager-enabled Ingress snippet:
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
  spec:
    tls:
    - hosts:
      - MY_APP_HOST
      secretName: myapp-tls

Make cookies work correctly (server-side)
- If Traefik terminates TLS and forwards to the pod over HTTP, r.TLS will be nil and your app must trust `X-Forwarded-Proto` to determine whether the original request was HTTPS.
- Recommended server behavior (pseudocode):
  - If env COOKIE_SECURE == "true" then set cookies Secure=true
  - Else if r.Header.Get("X-Forwarded-Proto") == "https" then Secure=true
  - Else Secure=false
- For production, set COOKIE_SECURE=true in the backend Deployment.

Example kubectl commands to set env in Deployment
- Set COOKIE_SECURE (production)
  kubectl set env deployment/backend-deployment COOKIE_SECURE=true

- Set BACKEND_URL in frontend (if using a deployment manifest instead of builtin config)
  kubectl set env deployment/frontend-deployment BACKEND_URL="/api"

Traefik notes
- The repo uses Traefik Middleware CRDs (traefik.containo.us/v1alpha1) for replacePathRegex. Verify CRDs exist:
  kubectl get crd | grep traefik.containo.us
- If you do not run Traefik or your controller doesn't support the same CRDs, adapt the rewrite behavior to your ingress controller (or modify the server to accept `/api` paths).

Testing & debugging
- List ingresses:
  kubectl get ingress -A
- Describe ingress:
  kubectl describe ingress my-ingress -n default
- Check Traefik service and external IP:
  kubectl -n kube-system get svc traefik
- Check nodes for IP:
  kubectl get nodes -o wide
- Inspect headers hitting your backend (temporary): log `r.Header` to ensure `X-Forwarded-Proto` is present.

Security notes
- Never run with Secure=false in production. Secure=false sends cookies over plain HTTP and is vulnerable to interception.
- Use SameSite settings appropriate to your case. For cross-site cookies you may need SameSite=None + Secure (HTTPS required).
- Prefer TLS termination at the ingress and enabling Secure cookie flag in production.

Frequently used commands (summary)
- Apply manifests:
  kubectl apply -f middleware-strip-api.yaml
  kubectl apply -f ingress-single.yaml
  kubectl apply -f backend-deployment.yaml
  kubectl apply -f frontend-deployment.yaml

- Patch environment for local dev:
  kubectl set env deployment/backend-deployment COOKIE_SECURE=false

- Create TLS secret from local certs:
  kubectl create secret tls myapp-tls --cert=MY_APP_HOST.pem --key=MY_APP_HOST-key.pem -n default

- Verify ingress:
  kubectl describe ingress my-ingress -n default

Need help customizing for your cluster?
- Tell me:
  - Which Ingress controller do you use (Traefik built-in k3s, nginx-ingress, other)?
  - Do you want local mkcert TLS or cert-manager/Let's Encrypt?
  - The hostname you plan to use (or if you'd prefer nip.io)
I can produce ready-to-apply manifests (with MY_APP_HOST replaced), exact kubectl commands, and the small Go cookie snippet to reliably detect HTTPS behind Traefik.

Enjoy — and keep Secure=true in production!