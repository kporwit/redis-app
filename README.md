# Infrastructure
Kubernetes resources created in this repository allows for creation of simple PHP application with Redis OSS 
(with Sentinel) as cache layer. External connectivity and secrets resources were created for AWS Cloud Provider.
Infrastructure is set up as HA and all workload resources are security hardened. 
External users can reach the application over HTTPS through ALB application load balancer where TLS is terminated,
next the traffic is passed through `php-app` service to the `php-app` Pods. Moreover, The application can connect to 
the external Database and Redis Cluster. Simple diagram of the architecture can be found below:
```
Internet
    |
    |
AWS ALB Ingress Controler HTTPS (ACM)
    |
    |
php-app service
    |
    |
php-app Deployment Pods
    |
    |
    -- Redis Sentinel -> Redis Master -> Redis Replicas
    |
    |
    -- External Database
```
## Assumptions
- External Secrets Operator (ESO)
    - is installed and have access to AWS Secrets Manager,
    - Cluster Secret Store `vault-backend` is created.
- AWS Load Balancer Controller
    - is installed and configured to properly provision Load Balancers within private/public Subnets,
    - IngressClass `alb` is configured,
    - subdomain wildcard certificate for `crabdance.com` is set up in ACM.

# Deployment
Two separate ArgoCD applications should be created. Assuming the app-of-apps approach both of them should
be placed in the folder monitored by the Root ArgoCD application. The ArgoCD application for Redis need to
be set up in a recursive directory mode (as it contains two separate directories). This can be done with
`source.directory.recursive` option set up to `true`. The `source.path` should be set up as the `php-app` and `redis-cache` 
respectively and the `source.repoURL` should contain the repository link to the remote repository where the
files will be pushed. With this setup ArgoCD should create two Applications with resources defined in respective
directories.

# Comments
- Plain text passwords for secrets were removed, External Secrets Operator resources were created instead.
- Network Policies were set up to block non-allowed ingress by default.
- The images keywords uses images SHA digest to make sure that the proper image is downloaded.
- Secrets (created by ESO) for Redis are separated as different teams will be using/rotating them.
- All workloads created in HA setting (3 replicas) with TopologySpreadConstrains.
- `php-app` Deployment configuration allows the pods to reach Redis cluster in `redis` namespace.
- egress on `php-app` is not restricted, thus it is possible to connect to the external DB.

# ToDo
- Redis `redis-0` pod is always the master and the rest are replicas. This setup works well until there is a failover
  and Sentinetl chooses new master from replicas. Init command should be modified in a way that Sentinel chooses 
  roles for the redis pods.
- Consider stakater/Reloader for Secrets,ConfigMap rotation.
- Consider packing the manifests into reusable Helm Chart.
- TopologySpreadConstrains for the workloads is set to `hostname`. Consider changing for appropriate Cloud Provider `zone` label.

# Versions
Applications versions used for the infrastructure can be found below:
- Kubernetes: 1.33.1
- Docker: 29.2.1
- AWS Load Balancer Controller: 3.0
- External Secrets Operator: 2.4.0
- Redis: 7.4
- Bash: 5.2
- PHP: 8.3 (alpine)
