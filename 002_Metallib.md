# Install Metallib
[Metallib](https://metallb.universe.tf/concepts/) creates a LoadBalancer service type that lets me request a service of type LoadBalancer, which will be automatically assigned an IP from my configured pool.

1. Install the deployment:
  ```
  kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.7.3/manifests/metallb.yaml
  ```

1. Apply the configuration - In my file, setting up Layer 2 mode and reserving 192.168.1.40-192.168.1.49:
  ```
  kubectl apply -f ./002_Metallib/metallb-conf.yaml
  ```
