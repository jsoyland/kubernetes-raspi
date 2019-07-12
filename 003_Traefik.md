# Install Traefik
[Traefik](https://traefik.io) will handle the load balancing on the service level, allowing any deployment to be exposed by subdomain.  Traefik will be deployed with a LoadBalancer service type (with a static IP), then mapped as a wildcard on my internal DNS.  This way any lookups into \*.my.domain.com will go to that IP and Traefik will forward to the appropriate pod based on it's rules.  

1. The config files can be modified to suit the domain/IPs it's being configured for, and then deployed thusly:
  ```
  kubectl apply -f 003_Traefik/traefik-internal-configmap.yaml
  kubectl apply -f 003_Traefik/traefik-internal-deployment.yaml
  kubectl apply -f 003_Traefik/traefik-internal-service.yaml
  kubectl apply -f 003_Traefik/traefik-rbac.yaml
  ```
