# 12. (bonus) (Advanced) Istio Integration


## 12.1 How to Expose Jenkins Internally using Istio GW and VS (given that you have AWS VPN enabled)

Install istio using [istio_overrides.yaml](istio_overrides.yaml) which creates private ELB to expose Jenkins internally:
```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  components: # ref: https://istio.io/latest/docs/reference/config/istio.operator.v1alpha1/#IstioOperatorSpec
    egressGateways:
    - enabled: true
      k8s:
        env:
        - name: ISTIO_META_ROUTER_MODE
          value: standard
        hpaSpec:
          maxReplicas: 5
          metrics:
          - resource:
              name: cpu
              targetAverageUtilization: 80
            type: Resource
          minReplicas: 1
          scaleTargetRef:
            apiVersion: apps/v1
            kind: Deployment
            name: istio-egressgateway
        resources:
          limits:
            cpu: 2000m
            memory: 1024Mi
          requests:
            cpu: 10m
            memory: 40Mi
        service:
          ports:
          - name: http2
            port: 80
            protocol: TCP
            targetPort: 8080
          - name: https
            port: 443
            protocol: TCP
            targetPort: 8443
          - name: tls
            port: 15443
            protocol: TCP
            targetPort: 15443
        strategy:
          rollingUpdate:
            maxSurge: 100%
            maxUnavailable: 25%
      name: istio-egressgateway
    ingressGateways:
    - enabled: true
      name: istio-ingressgateway
      k8s:
        service:
          ports:
          - name: status-port
            port: 15021
            targetPort: 15021
          - name: http2
            port: 80
            protocol: TCP
            targetPort: 8080
          - name: https
            port: 443
            protocol: TCP
            targetPort: 8443
          - name: tls
            port: 15443
            protocol: TCP
            targetPort: 15443      
        serviceAnnotations:
          # if using NLB, then can't terminate HTTPS. 
          # Won't use GA with NLB as K8s service with NLB is still alpha feature and not for production cluster
          # instead, deploy NLB in front of CLB (K8s service) to provide two static IPs 
          #service.beta.kubernetes.io/aws-load-balancer-type: nlb
          # enable ELB access log
          # ref: https://www.giantswarm.io/blog/load-balancer-service-use-cases-on-aws
          service.beta.kubernetes.io/aws-load-balancer-access-log-enabled: "true"
          # enable TLS termination at AWS ELB level
          service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:us-east-1:xxxxxxx:certificate/xxxxxxx-7b9b-4540-93ee-xxxxxxx" # this cert is "*.YOUR_DOMAIN.co" and "YOUR_DOMAIN.co" with DNS validation
          service.beta.kubernetes.io/aws-load-balancer-backend-protocol: http
          service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"
          service.beta.kubernetes.io/aws-load-balancer-ssl-negotiation-policy: "ELBSecurityPolicy-TLS-1-2-2017-01" # disable TSLv1.0-1.1
    - enabled: true
      name: istio-private-ingressgateway
      label:
        istio: privateingressgateway # this will be needed as gateway will look for this selector
        app: istio-private-ingressgateway
      k8s:
        service:
          ports:
          - name: status-port
            port: 15021
            targetPort: 15021
          - name: http2
            port: 80
            targetPort: 8080
          - name: https
            port: 443
            targetPort: 8443
          - name: tcp
            port: 31400
            targetPort: 31400
          - name: tls
            port: 15443
            targetPort: 15443      
        serviceAnnotations:
          # ref: https://medium.com/swlh/public-and-private-istio-ingress-gateways-on-aws-f968783d62fe
          service.beta.kubernetes.io/aws-load-balancer-internal: "true" # make this CLB private. Refs for service annotations for AWS ELB: https://kubernetes.io/docs/concepts/cluster-administration/cloud-providers/#aws, https://docs.aws.amazon.com/eks/latest/userguide/load-balancing.html
          # enable ELB access log
          # ref: https://www.giantswarm.io/blog/load-balancer-service-use-cases-on-aws
          service.beta.kubernetes.io/aws-load-balancer-access-log-enabled: "true"
          # enable TLS termination at AWS ELB level
          service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:us-east-1:xxxxxxx:certificate/xxxxxxx-7b9b-4540-93ee-xxxxxxx" # this cert is "*.YOUR_DOMAIN.co" and "YOUR_DOMAIN.co" with DNS validation
          service.beta.kubernetes.io/aws-load-balancer-backend-protocol: http
          service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"
          service.beta.kubernetes.io/aws-load-balancer-ssl-negotiation-policy: "ELBSecurityPolicy-TLS-1-2-2017-01" # disable TSLv1.0-1.1
          # enable cross-zone load balancing to better handle instance loss in one AZ
          service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
  values:
    gateways: # configure gateways: https://istio.io/latest/docs/setup/install/istioctl/#configure-gateways
      istio-ingressgateway: # for internal ELB
        autoscaleEnabled: false
        env: {}
        # meshExpansionPorts:
        # - name: tcp-pilot-grpc-tls
        #   port: 15011
        #   targetPort: 15011
        # - name: tcp-istiod
        #   port: 15012
        #   targetPort: 15012
        # - name: tcp-citadel-grpc-tls
        #   port: 8060
        #   targetPort: 8060
        # - name: tcp-dns-tls
        #   port: 853
        #   targetPort: 8853
        name: istio-private-ingressgateway
        secretVolumes:
        - mountPath: /etc/istio/ingressgateway-certs
          name: ingressgateway-certs
          secretName: istio-ingressgateway-certs
        - mountPath: /etc/istio/ingressgateway-ca-certs
          name: ingressgateway-ca-certs
          secretName: istio-ingressgateway-ca-certs
        type: LoadBalancer
    global:
      hub: xxxxxxx.dkr.ecr.us-east-1.amazonaws.com # To workaround dockerhub rate limiting. Refs: https://istio.io/latest/docs/setup/install/helm/, https://istio.io/latest/docs/setup/install/helm/, https://istio.io/latest/docs/setup/install/istioctl/, https://istio.io/latest/docs/setup/install/operator/
      tag: 1.8.2
      imagePullPolicy: IfNotPresent # To workaround dockerhub rate limiting. Ref: https://medium.com/@kenneth.skertchly/fix-docker-pull-rate-limit-issues-in-istio-sidecars-9f7a410acefd
      proxy:
        tracer: datadog
    pilot:
      traceSampling: 100.0
      env:
        ENABLE_LEGACY_FSGROUP_INJECTION: true # for EKS 1.19, workaround for the issue "open ./var/run/secrets/tokens/istio-token: permission denied" refs: https://github.com/istio/istio/issues/31740#issuecomment-811292371, https://discuss.istio.io/t/istio-token-permissions-problem-on-eks-failed-to-fetch-token-from-file-open-var-run-secrets-tokens-istio-token-permission-denied/10028/8
```

```sh
istioctl install \
  -f overrides.yaml \
  --set values.pilot.env.ENABLE_LEGACY_FSGROUP_INJECTION=true
```


[istio_gateway.yaml](istio_gateway.yaml):
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: jenkins-private-gateway
  namespace: jenkins
spec:
  selector:
    istio: privateingressgateway # use private one, not the istio default controller
  servers: # defines L7 host, port, and protocol
  - port:
      number: 80 # jenkins listens to HTTP only
      name: http
      protocol: HTTP
    hosts: # host in http header
    - jenkins.YOUR_DOMAIN.co
    - jenkins/*.us-east-1.elb.amazonaws.com # using AWS internal ELB  # limit VirtualService in specified namespace to bind to this gateway host. Ref: https://archive.istio.io/v1.5/pt-br/docs/reference/config/networking/gateway/
    tls: 
      httpsRedirect: true # sends 301 redirect for http requests
  - port:
      number: 443
      name: https-to-http
      protocol: HTTP # NOTE: HTTPS traffic ends on AWS ELB, so select HTTP not HTTPS
    hosts:
    - jenkins.YOUR_DOMAIN.co
    - jenkins/*.us-east-1.elb.amazonaws.com # using AWS internal ELB  # limit VirtualService in specified namespace to bind to this gateway host. Ref: https://archive.istio.io/v1.5/pt-br/docs/reference/config/networking/gateway/

--- 
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: jenkins-public-gateway
  namespace: jenkins
spec:
  selector:
    istio: ingressgateway # use the istio default controller
  servers: # defines L7 host, port, and protocol
  - port:
      number: 80 # jenkins listens to HTTP only
      name: http
      protocol: HTTP
    hosts: # host in http header
    - public.jenkins.YOUR_DOMAIN.co
    - jenkins/*.us-east-1.elb.amazonaws.com
    # tls:
    #   mode: SIMPLE # enables HTTPS on this port
    #   credentialName: gateway-cert-aws-elb-dns # must be the same as secret
  - port:
      number: 443
      name: https-to-http
      protocol: HTTP # NOTE: HTTPS traffic ends on AWS ELB, so select HTTP not HTTPS
    hosts:
    - public.jenkins.YOUR_DOMAIN.co
    - jenkins/*.us-east-1.elb.amazonaws.com
```


[istio_virtual_service.yaml](istio_virtual_service.yaml):
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: jenkins
  namespace: jenkins
spec:
  hosts:
  - jenkins.YOUR_DOMAIN.co
  #- internal-a64095ec1a0da4d7d9a83f8253bb79c5-1484243758.us-east-1.elb.amazonaws.com # using AWS internal ELB
  gateways: # using gateways field, it'll be exposed externally
  # WARNING: this is legacy format (i.e. gateway.namespace.svc.cluster.local), which breaks in v1.6.7-1.6.8
  #- jenkins-private-gateway.jenkins.svc.cluster.local # Don't prepend jenkins/ as istio complains "configuration is invalid: invalid value for gateway name:"
  - jenkins/jenkins-private-gateway
  http:
  - route:
    - destination:
        host: jenkins.jenkins.svc.cluster.local # specify service name
        port:
          number: 8080

---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: jenkins-public-webhook
  namespace: jenkins
spec:
  hosts:
  - public.jenkins.YOUR_DOMAIN.co
  - 12345.us-east-1.elb.amazonaws.com # use public NLB
  gateways: # using gateways field, it'll be exposed externally
  # this is legacy format (i.e. gateway.namespace.svc.cluster.local), which breaks in v1.6.7-1.6.8
  # - jenkins-public-gateway.jenkins.svc.cluster.local
  - jenkins/jenkins-public-gateway
  http:
  - match:
    - uri:
        exact: /bitbucket-hook/ # only expose this endpoint for Jenkins
      ignoreUriCase: true
    route:
    - destination:
        host: jenkins.jenkins.svc.cluster.local # specify service name
        port:
          number: 8080

---
# WARNING: from istio v1.7+, disabling mtls might cause error "istio 503 upstream connect error or disconnect/reset before headers. reset reason: connection termination"
# refs: https://stackoverflow.com/questions/53103984/istio-destinationrule-gives-upstream-connect-error-or-disconnect-reset-before-he
# ref: https://istio.io/latest/docs/ops/common-problems/network-issues/#503-errors-after-setting-destination-rule

# apiVersion: networking.istio.io/v1alpha3
# kind: DestinationRule
# metadata:
#   name: jenkins
#   namespace: jenkins
# spec:
#   host: jenkins.jenkins.svc.cluster.local
#   trafficPolicy:
#     tls:
#       mode: DISABLE
```



Deploy Gateway and Virtual Service
```sh
kubectl apply -f istio_gateway.yaml
kubectl apply -f istio_virtual_service.yaml
```



You also need:
- R53 CNAME for `jenkins` subdomain to route to internal AWS ELB DNS
- AWS Client VPN to private subnet
