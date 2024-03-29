# kong-eks

 #Ref: https://github.com/Kong/kubernetes-ingress-controller/blob/master/docs/deployment/minikube.md
 
 #Ref: https://github.com/Kong/kubernetes-ingress-controller/blob/master/docs/guides/getting-started.md
 
 #Ref: https://blog.baeke.info/2019/06/15/api-management-with-kong-ingress-controller-on-kubernetes/
 
 #Ref: https://raw.githubusercontent.com/Kong/kubernetes-ingress-controller/master/deploy/single/all-in-one-postgres.yaml
 
 #Ref: https://gist.githubusercontent.com/lalyos/4553081032b1f75eb18e66788a76d2a2/raw/k8s-echo-server.yaml
 
 #Ref: https://github.com/phaniamt/ssl-redirect/blob/master/README.md

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
      - name: kong-proxy-ssl
        port: 443
        targetPort: 8000
        protocol: TCP
      selector:
        app: kong
# Create the postgres service for db
    apiVersion: v1
    kind: Service
    metadata:
      name: postgres
      namespace: kong
    spec:
      ports:
      - name: pgql
        port: 5432
        targetPort: 5432
        protocol: TCP
      selector:
        app: postgres
 # Create the postgres statefulset for db
    apiVersion: apps/v1  #  for k8s versions before 1.9.0 use apps/v1beta2  and before 1.8.0 use extensions/v1beta1
    kind: StatefulSet
    metadata:
      name: postgres
      namespace: kong
    spec:
      serviceName: "postgres"
      replicas: 1
      selector:
        matchLabels:
          app: postgres
      template:
        metadata:
          labels:
            app: postgres
        spec:
          containers:
          - name: postgres
            image: postgres:9.5
            volumeMounts:
            - name: datadir
              mountPath: /var/lib/postgresql/data
              subPath: pgdata
            env:
            - name: POSTGRES_USER
              value: kong
            - name: POSTGRES_PASSWORD
              value: kong
            - name: POSTGRES_DB
              value: kong
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
            ports:
            - containerPort: 5432
          # No pre-stop hook is required, a SIGTERM plus some time is all that's
          # needed for graceful shutdown of a node.
          terminationGracePeriodSeconds: 60
      volumeClaimTemplates:
      - metadata:
          name: datadir
        spec:
          accessModes:
          - "ReadWriteOnce"
          resources:
            requests:
              storage: 1Gi
# Create job for kong migrations
    apiVersion: batch/v1
    kind: Job
    metadata:
      name: kong-migrations
      namespace: kong
    spec:
      template:
        metadata:
          name: kong-migrations
        spec:
          initContainers:
          - name: wait-for-postgres
            image: busybox
            env:
            - name: KONG_PG_HOST
              value: postgres
            - name: KONG_PG_PORT
              value: "5432"
            command: [ "/bin/sh", "-c", "until nc -zv $KONG_PG_HOST $KONG_PG_PORT -w1; do echo 'waiting for db'; sleep 1; done" ]
          containers:
          - name: kong-migrations
            image: kong:1.2
            env:
            - name: KONG_PG_PASSWORD
              value: kong
            - name: KONG_PG_HOST
              value: postgres
            - name: KONG_PG_PORT
              value: "5432"
            command: [ "/bin/sh", "-c", "kong migrations bootstrap" ]
          restartPolicy: OnFailure
# Create a service for kong ingress-controller
    apiVersion: v1
    kind: Service
    metadata:
      name: kong-ingress-controller
      namespace: kong
    spec:
      type: NodePort
      ports:
      - name: kong-admin
        port: 8001
        targetPort: 8001
        protocol: TCP
      selector:
        app: ingress-kong
# Create a deployment for kong ingress-controller
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      labels:
        app: ingress-kong
      name: kong-ingress-controller
      namespace: kong
    spec:
      selector:
        matchLabels:
          app: ingress-kong
      strategy:
        rollingUpdate:
          maxSurge: 1
          maxUnavailable: 0
        type: RollingUpdate
      template:
        metadata:
          annotations:
            # the returned metrics are related to the kong ingress controller not kong itself
            prometheus.io/port: "10254"
            prometheus.io/scrape: "true"
          labels:
            app: ingress-kong
        spec:
          serviceAccountName: kong-serviceaccount
          initContainers:
          - name: wait-for-migrations
            image: kong:1.2
            command: [ "/bin/sh", "-c", "kong migrations list" ]
            env:
            - name: KONG_ADMIN_LISTEN
              value: 'off'
            - name: KONG_PROXY_LISTEN
              value: 'off'
            - name: KONG_PROXY_ACCESS_LOG
              value: "/dev/stdout"
            - name: KONG_ADMIN_ACCESS_LOG
              value: "/dev/stdout"
            - name: KONG_PROXY_ERROR_LOG
              value: "/dev/stderr"
            - name: KONG_ADMIN_ERROR_LOG
              value: "/dev/stderr"
            - name: KONG_PG_HOST
              value: postgres
            - name: KONG_PG_PASSWORD
              value: kong
          containers:
          - name: admin-api
            image: kong:1.2
            env:
            - name: KONG_PG_PASSWORD
              value: kong
            - name: KONG_PG_HOST
              value: postgres
            - name: KONG_ADMIN_ACCESS_LOG
              value: /dev/stdout
            - name: KONG_ADMIN_ERROR_LOG
              value: /dev/stderr
            - name: KONG_ADMIN_LISTEN
              value: 0.0.0.0:8001, 0.0.0.0:8444 ssl
            - name: KONG_PROXY_LISTEN
              value: 'off'
            ports:
            - name: kong-admin
              containerPort: 8001
            livenessProbe:
              failureThreshold: 3
              httpGet:
                path: /status
                port: 8001
                scheme: HTTP
              initialDelaySeconds: 30
              periodSeconds: 10
              successThreshold: 1
              timeoutSeconds: 1
            readinessProbe:
              failureThreshold: 3
              httpGet:
                path: /status
                port: 8001
                scheme: HTTP
              periodSeconds: 10
              successThreshold: 1
              timeoutSeconds: 1
          - name: ingress-controller
            args:
            - /kong-ingress-controller
            # the kong URL points to the kong admin api server
            - --kong-url=https://localhost:8444
            - --admin-tls-skip-verify
            # Service from were we extract the IP address/es to use in Ingress status
            - --publish-service=kong/kong-proxy
            env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.namespace
            image: kong-docker-kubernetes-ingress-controller.bintray.io/kong-ingress-controller:0.5.0
            imagePullPolicy: IfNotPresent
            livenessProbe:
              failureThreshold: 3
              httpGet:
                path: /healthz
                port: 10254
                scheme: HTTP
              initialDelaySeconds: 30
              periodSeconds: 10
              successThreshold: 1
              timeoutSeconds: 1
            readinessProbe:
              failureThreshold: 3
              httpGet:
                path: /healthz
                port: 10254
                scheme: HTTP
              periodSeconds: 10
              successThreshold: 1
              timeoutSeconds: 1
# Create a deployment for kong-proxy
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: kong
      namespace: kong
    spec:
      template:
        metadata:
          labels:
            name: kong
            app: kong
        spec:
          initContainers:
          # hack to verify that the DB is up to date or not
          # TODO remove this for Kong >= 0.15.0
          - name: wait-for-migrations
            image: kong:1.2
            command: [ "/bin/sh", "-c", "kong migrations list" ]
            env:
            - name: KONG_ADMIN_LISTEN
              value: 'off'
            - name: KONG_PROXY_LISTEN
              value: 'off'
            - name: KONG_PROXY_ACCESS_LOG
              value: "/dev/stdout"
            - name: KONG_ADMIN_ACCESS_LOG
              value: "/dev/stdout"
            - name: KONG_PROXY_ERROR_LOG
              value: "/dev/stderr"
            - name: KONG_ADMIN_ERROR_LOG
              value: "/dev/stderr"
            - name: KONG_PG_HOST
              value: postgres
            - name: KONG_PG_PASSWORD
              value: kong
          containers:
          - name: kong-proxy
            image: kong:1.2
            env:
            - name: KONG_PG_PASSWORD
              value: kong
            - name: KONG_PG_HOST
              value: postgres
            - name: KONG_PROXY_ACCESS_LOG
              value: "/dev/stdout"
            - name: KONG_PROXY_ERROR_LOG
              value: "/dev/stderr"
            - name: KONG_ADMIN_LISTEN
              value: 'off'
            ports:
            - name: proxy
              containerPort: 8000
              protocol: TCP
            - name: proxy-ssl
              containerPort: 8443
              protocol: TCP
            lifecycle:
              preStop:
                exec:
                  command: [ "/bin/sh", "-c", "kong quit" ]
          - name: kong-https
            image: yphani/kong-ssl-redirect
            ports:
            - name: nginx
              containerPort: 80
# Deploy a Demo application
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: echo
      name: echo
    spec:
      ports:
      - port: 8080
        name: high
        protocol: TCP
        targetPort: 8080
      - port: 80
        name: low
        protocol: TCP
        targetPort: 8080
      selector:
        app: echo
    ---
    apiVersion: apps/v1beta1
    kind: Deployment
    metadata:
      labels:
        app: echo
      name: echo
    spec:
      replicas: 4
      selector:
        matchLabels:
          app: echo
      strategy: {}
      template:
        metadata:
          creationTimestamp: null
          labels:
            app: echo
        spec:
          containers:
          - image: gcr.io/kubernetes-e2e-test-images/echoserver:2.2
            name: echo
            ports:
            - containerPort: 8080
            env:
              - name: NODE_NAME
                valueFrom:
                  fieldRef:
                    fieldPath: spec.nodeName
              - name: POD_NAME
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: POD_IP
            valueFrom:
              fieldRef:
                fieldPath: status.podIP
        resources: {}
# Create a kong consumer
    apiVersion: configuration.konghq.com/v1
    kind: KongConsumer
    metadata:
      name: top
    username: topuser 
# Create a KongCredential for topuser
    apiVersion: configuration.konghq.com/v1
    kind: KongCredential
    metadata:
      name: topcred
    consumerRef: top
    type: key-auth
    config:
      key: yourverysecretkeyhere
 # Create a KongPlugin for key-authentication
    apiVersion: configuration.konghq.com/v1
    kind: KongPlugin
    metadata:
      name: http-auth
      namespace: default
    plugin: key-auth 
 # Create a ingress rule for Demo application
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: demo
      annotations:
        kubernetes.io/ingress.class: kong
        plugins.konghq.com: http-auth
    spec:
      rules:
      - host: api.yphanikumar.xyz
        http:
          paths:
          - path: /*
            backend:
              serviceName: echo
              servicePort: 80
# 
    # Create a record set for the api domain add the loadbalancer url as cname
    
    # Check from the browser
     
     https://api.yphanikumar.xyz/?apikey=yourverysecretkeyhere
     
    #  Check from command line
    
    curl -i https://api.yphanikumar.xyz/?apikey=yourverysecretkeyhere
