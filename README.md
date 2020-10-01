# jira-cachecar
A Sidecar/NGinx Caching solution for Jira Data Center to cache static objects/offload tomcat and java services.

# Customizing this repo
* Configs can easily be used and applied to Nginx running infront of Tomcat/Jira locally. This solutuon is desinged to be a local/node specific proxy behind a load balancer. LB > Nginx > Jira.
* Customize the nginx-test.conf or use this in your reference design/testing 
* Build your own docker image and use with Docker/Kubernetes and proxy traffic to your local jira node

# How does this work? 
Nginx basically plays with HTTP headers and caches incoming requests based on Jira's paths. Traffic from your load balancer is terminated at Nginx and then upstream/proxied to Tomcat/Jira on the localhost. 

Jira may need some configration adjustment to make this work.

# Sample deployment using a sidecar/additional pod in Kubernetes
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{appname}}
  namespace: {{namespace}}
spec:
  serviceName: {{appname}}
  replicas: 1
  selector:
    matchLabels:
      app: {{appname}}
  template:
    metadata:
      labels:
        app: {{appname}}
    spec:
      subdomain: {{appname}}
      terminationGracePeriodSeconds: 300
      initContainers:
        - name: jirahomeperms
          securityContext:
          image: busybox
          command: ["chown", "-R", "2001", "/var/atlassian/application-data/jira/"]
          volumeMounts:
            - mountPath: "/var/atlassian/application-data/jira/"
              name: home
        - name: jirasharedperms
          image: busybox
          #command: ["ls", "-la"]
          command: ["chown", "-R", "2001:2001", "/var/atlassian/shared"]
          volumeMounts:
            - mountPath: "/var/atlassian/shared"
              name: shared-home
      containers:
        - name: cachecar
          image: mbern/nginx-cachecar:devel
          ports:
            - containerPort: 8080
              name: cache
          imagePullPolicy: Always
          volumeMounts:
          - mountPath: /opt/atlassian/jira/atlassian-jira/static-assets
            name: home
        - name: {{appname}}
          image: atlassian/jira-software:{{version}}
          resources:
            requests:
              cpu: {{podcpu}}
              memory: {{podmemory}}
            limits:
              cpu: {{podcpu}}
              memory: {{podmemory}}
          ports:
            - containerPort: 8081
              name: jiraport
            - containerPort: 40001
          env:
            - name: ATL_TOMCAT_PORT
              value: "8081"
            - name: ATL_PROXY_PORT
              value: "8080"
            - name: JVM_MINIMUM_MEMORY
              value: {{jvm}}
              # 1536m
            - name: JVM_MAXIMUM_MEMORY
              value: {{jvm}}
              # 1536m
            - name: CATALINA_CONNECTOR_PROXYNAME
              value: {{proxyname}}
              # jira.company.com
            - name: CATALINA_CONNECTOR_SCHEME
              value: {{proxyscheme}}
              # https
            - name: CATALINA_CONNECTOR_SECURE
              value: "false"
              # Value = "true
            - name: CLUSTERED
              value: "true"
            - name: JIRA_SHARED_HOME
              value: "/var/atlassian/shared"
            - name: SDOMAIN
              value: .{{appname}}
            - name: NODE_ID
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: JIRA_NODE_ID
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: EHCACHE_LISTENER_HOSTNAME
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: EHCACHE_LISTENER_PORT
              value: "40001"
            # These should be secrets in best practice
            - name: ATL_JDBC_URL
              value: jdbc:postgresql://{{postgresserver}}.{{namespace}}:5432/{{databasename}}
            - name: ATL_JDBC_USER
              value: supersecret
            - name: ATL_JDBC_PASSWORD
              value: supersecret
            # Above should be secrets to manage credentials securely! 
            - name: ATL_DB_DRIVER
              value: org.postgresql.Driver
            - name: ATL_DB_TYPE
              value: postgres72
          readinessProbe:
            httpGet:
              port: 8081
              path: /status
            initialDelaySeconds: 180
            periodSeconds: 15
            successThreshold: 2
            failureThreshold: 2
            timeoutSeconds: 5
          volumeMounts:
            - mountPath: "/var/atlassian/application-data/jira"
              name: home
            - mountPath: "/var/atlassian/shared"
              name: shared-home


      volumes:
        - name: home
          emptyDir:
            {}
        - name: shared-home
          persistentVolumeClaim:
            claimName: jira
        - name: jira-tomcat-volume
          configMap:
            name: jira-tomcat-xml
            defaultMode: 0777
```

# What's cached? What's the deal?
* /s/ contains a bunch of static resources, they're fairly persistent and are long lived including Javascript and CSS
* /secure/attachment is your jira's attachements, Nginx caches these if there is a HIT or HTTP 200 for 60s, reducing the requirement to retrieve attachements from NFS/Shared home on a per node basis. This should be adjusted and reduced based on use-case of attachements in Jira. In this scenario, a user may view an issue several times and the attachement is cached in the users browser/nginx cache for a duration.
* /images/icons/emoticons is just a bunch of other assets like .gifs and images servived like buttons and UI things, these can be cached for a longer duration


# Caveats? 
* Plugins/Changes to CSS may not be reflected quickly as they're cached. Commercial solutions like Nginx+ have cache expiry features that can help here. 

# Recommendations
Caches should be short, long lived caches could create a inconsistent experience or lead to end user frustration using older objects. Less than 8 hours would be recommended. Once Jira the user logs in and receives most of the assets for the day, this should offload jira enough to make this solution valuable for a large deployment. 

Caches can be blown away as per lifecycle of pods/AMIs or deployments.

Caches should live on local disks and have access to fast IO. Tuning Nginx is out of scope of this repo. Working with your nginx specialists and network teams on how to optimize and leverage Nginx is beyond the scope of this repo. 


