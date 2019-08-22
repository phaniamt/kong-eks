# kong-eks

# Create Kong Namespace
    apiVersion: v1
    kind: Namespace
    metadata:
      name: kong
# Create the CustomResourceDefinition for kong
    ---
    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: kongplugins.configuration.konghq.com
    spec:
      group: configuration.konghq.com
      version: v1
      scope: Namespaced
      names:
        kind: KongPlugin
        plural: kongplugins
        shortNames:
        - kp
      additionalPrinterColumns:
      - name: Plugin-Type
        type: string
        description: Name of the plugin
        JSONPath: .plugin
      - name: Age
        type: date
        description: Age
        JSONPath: .metadata.creationTimestamp
      - name: Disabled
        type: boolean
        description: Indicates if the plugin is disabled
        JSONPath: .disabled
        priority: 1
      - name: Config
        type: string
        description: Configuration of the plugin
        JSONPath: .config
        priority: 1
      validation:
        openAPIV3Schema:
          required:
          - plugin
          properties:
            plugin:
              type: string
            disabled:
              type: boolean
            config:
              type: object
            run_on:
              type: string
              enum:
              - first
              - second
              - all
            protocols:
              type: array
              items:
                type: string
                enum:
                - http
                - https
                - tcp
                - tls

    ---
    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: kongconsumers.configuration.konghq.com
    spec:
      group: configuration.konghq.com
      version: v1
      scope: Namespaced
      names:
        kind: KongConsumer
        plural: kongconsumers
        shortNames:
        - kc
      additionalPrinterColumns:
      - name: Username
        type: string
        description: Username of a Kong Consumer
        JSONPath: .username
      - name: Age
        type: date
        description: Age
        JSONPath: .metadata.creationTimestamp
      validation:
        openAPIV3Schema:
          properties:
            username:
              type: string
            custom_id:
              type: string

    ---
    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: kongcredentials.configuration.konghq.com
    spec:
      group: configuration.konghq.com
      version: v1
      scope: Namespaced
      names:
        kind: KongCredential
        plural: kongcredentials
      additionalPrinterColumns:
      - name: Credential-type
        type: string
        description: Type of credential
        JSONPath: .type
      - name: Age
        type: date
        description: Age
        JSONPath: .metadata.creationTimestamp
      - name: Consumer-Ref
        type: string
        description: Owner of the credential
        JSONPath: .consumerRef
      validation:
        openAPIV3Schema:
          required:
          - consumerRef
          - type
          properties:
            consumerRef:
              type: string
            type:
              type: string

    ---
    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: kongingresses.configuration.konghq.com
    spec:
      group: configuration.konghq.com
      version: v1
      scope: Namespaced
      names:
        kind: KongIngress
        plural: kongingresses
        shortNames:
        - ki
      validation:
        openAPIV3Schema:
          properties:
            upstream:
              type: object
            route:
              properties:
                methods:
                  type: array
                  items:
                    type: string
                regex_priority:
                  type: integer
                strip_path:
                  type: boolean
                preserve_host:
                  type: boolean
                protocols:
                  type: array
                  items:
                    type: string
                    enum:
                    - http
                    - https
                https_redirect_status_code:
                  type: integer
            proxy:
              type: object
              properties:
                protocol:
                  type: string
                  enum:
                  - http
                  - https
                path:
                  type: string
                  pattern: ^/.*$
                retries:
                  type: integer
                  minimum: 0
                connect_timeout:
                  type: integer
                  minimum: 0
                read_timeout:
                  type: integer
                  minimum: 0
                write_timeout:
                  type: integer
                  minimum: 0
            upstream:
              type: object
              properties:
                hash_on:
                  type: string
                hash_on_cookie:
                  type: string
                hash_on_cookie_path:
                  type: string
                hash_on_header:
                  type: string
                hash_fallback_header:
                  type: string
                hash_fallback:
                  type: string
                slots:
                  type: integer
                  minimum: 10
                healthchecks:
                  type: object
                  properties:
                    active:
                      type: object
                      properties:
                        concurrency:
                          type: integer
                          minimum: 1
                        timeout:
                          type: integer
                          minimum: 0
                        http_path:
                          type: string
                          pattern: ^/.*$
                        healthy: &healthy
                          type: object
                          properties:
                            http_statuses:
                              type: array
                              items:
                                type: integer
                            interval:
                              type: integer
                              minimum: 0
                            successes:
                              type: integer
                              minimum: 0
                        unhealthy: &unhealthy
                          type: object
                          properties:
                            http_failures:
                              type: integer
                              minimum: 0
                            http_statuses:
                              type: array
                              items:
                                type: integer
                            interval:
                              type: integer
                              minimum: 0
                            tcp_failures:
                              type: integer
                              minimum: 0
                            timeout:
                              type: integer
                              minimum: 0
                    passive:
                      type: object
                      properties:
                        healthy: *healthy
                        unhealthy: *unhealthy
   # Create Kong serviceaccount and create rbac for kong

    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: kong-serviceaccount
      namespace: kong

    ---
    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: ClusterRole
    metadata:
      name: kong-ingress-clusterrole
    rules:
    - apiGroups:
      - ""
      resources:
      - endpoints
      - nodes
      - pods
      - secrets
      verbs:
      - list
      - watch
    - apiGroups:
      - ""
      resources:
      - nodes
      verbs:
      - get
    - apiGroups:
      - ""
      resources:
      - services
      verbs:
      - get
      - list
      - watch
    - apiGroups:
      - "extensions"
      resources:
      - ingresses
      verbs:
      - get
      - list
      - watch
    - apiGroups:
      - ""
      resources:
      - events
      verbs:
      - create
      - patch
    - apiGroups:
      - "extensions"
      resources:
      - ingresses/status
      verbs:
      - update
    - apiGroups:
      - "configuration.konghq.com"
      resources:
      - kongplugins
      - kongcredentials
      - kongconsumers
      - kongingresses
      verbs:
      - get
      - list
      - watch
    - apiGroups:
      - ""
      resources:
      - configmaps
      resourceNames:
      #Defaults to "<election-id>-<ingress-class>"
      # Here: "<ingress-controller-leader>-<kong>"
      # This has to be adapted if you change either parameter
      # when launching the kong-ingress-controller.
      - "ingress-controller-leader-kong"
      verbs:
      - get
      - update
    - apiGroups:
      - ""
      resources:
      - configmaps
      verbs:
      - create

    ---
    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: ClusterRoleBinding
    metadata:
      name: kong-ingress-clusterrole-nisa-binding
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: kong-ingress-clusterrole
    subjects:
    - kind: ServiceAccount
      name: kong-serviceaccount
      namespace: kong

# Create Kong-proxy service as LoadBalancer
        ---
        apiVersion: v1
        kind: Service
        metadata:
          name: kong-proxy
          namespace: kong
          annotations:
            # Cloud-provider specific annotations
            # GKE
            # GKE creates a L4 LB for any service of type LoadBalancer
            # TODO figure out how to enable Proxy Protocol on an L4 LB for GKE
            # AWS
            # Use NLB over ELB
        #    service.beta.kubernetes.io/aws-load-balancer-type: nlb
            # Use L4 LB so that Kong can do TLS termination
            service.beta.kubernetes.io/aws-load-balancer-backend-protocol: tcp
            service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:eu-central-1:294387193228:certificate/ed86172e-ffda-46e0-881e-b2bdea07501d"
            # Enable Proxy Protocol when Kong is listening for proxy-protocol
            service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"
            #service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: '*'
        spec:
          externalTrafficPolicy: Local
          type: LoadBalancer
          ports:
          - name: kong-proxy
            port: 80
            targetPort: 80
        #    protocol: TCP
          - name: kong-proxy-ssl
            port: 443
            targetPort: 8000
            protocol: TCP
          selector:
            app: kong

