With dramatically increasing demand of container orchestration specifically Kubernetes, demand to templatise K8S manifests(Json/Yaml) also came to light. To handle increasing manifests, new CRDs(Custom resource definition), etc... it became obvious that we need a package manager somewhat like yum, apt, etc... However nature of Kubernetes manifest are very different than what one used to have with Yum and Apt. These manifests required a lot of templating which is now supported by Helm, a tool written in GoLang with custom helm functions and pipelines.
### Neutral background on templating
Templating has been a driver for configuration management for a long time. While it may seem trivial for users coming from Ansible, Chef, Puppet, Salt, etc..., it is not. Once one moves to Kubernetes, first realization is hard declarative approach that Kubernetes follows. It is difficult to make generic templating with declarative form since each application may have some unique feature and requirements. Users have been subjected to duplicate the manifests.

# Helm features ([Helm feature Credits](https://helm.sh/))
- Client side only, Helm currently support command `helm` which users can use in same manner as kubectl 
- Server side with tiller, may soon be removed with Helm-3 [Helm-3 without tiller](https://helm.sh/blog/helm-3-preview-pt2/)
- Manage Complexity, Complexity to provide different attributes to different services, repeatability and single point of authority 
- Easy Updates, Just one command to update the release
- Simple Sharing, One can package an entire complex application which can be provided as a Chart
- Rollbacks, Faster rollback with `helm rollback command` 


# Java Springboot
A lot of applications are built on top of JAVA's Springboot, a MVC framework designed for web applications and microservices architecture. It has a few components, 
1. An application.properties/.yaml file
2. Compiled jar file 
3. Other property files if any(Optional)
All of these components have to be understood for management. One can use spring configuration servers as well to manage the configuration, however in this excercise we are going to focus on helm based approach to manage all the secrets. 
# Helm approach (Stateless application)
Now we want to deploy an application using helm package manager - one common questions is, what we need to do to achieve that? Let's answer it:
1. Package the code using Dockerfile - Make sure that packaged code is environment agnostic, however it is not a requirement for Helm to work
2. Define application.property/yaml file - See if one wants to manage it with helm, in my opinion SRE/DevOps teams prefer to keep it separate for different environments in different repository.
3. Identify environment variables that needs to be injected into container
4. Identify volumes(secret volume) that needs to be injected 

Lets solve them one by one. How to containerise a Springboot application, This example is hoping that you are running this inside GitLab, however you can execute the same steps with Jenkins as well, 

## 1. Package the code 
I presume that you have your service name `my-boot-service` that you want to manage with Helm. I presume that one has some idea about Dockerfile since one is looking into solutions like Kubernetes and Helm. 
```Dockerfile
FROM openjdk:8u111-jdk-alpine
ENV MAIN_OPTS '' 
RUN mkdir -p /opt/my-boot-service/ /opt/appConfig
COPY target/my-boot-service.jar /opt/my-boot-service/
WORKDIR /opt/my-boot-service/
ENTRYPOINT java $JAVA_OPTS -jar ./my-boot-service.jar $MAIN_OPTS
EXPOSE 8080
```
## 2. Define application.properties/yaml
In my example service has structure mentioned below. I would recommend to keep applications inside a root directory like: `namespace/<ENV>/<application_name>/config/<files>`. So SRE/Devops teams can clone additional repository whose content can be used to deploy these changes. 

## 3. Identify environment variables
One may need additional environment variable for the application to perform. Due to Kubernetes limitation that volume is immutable one can not keep the Jar from Step-1 and Configuration files from step-2 together. You need either script to copy files from immutable location or refer it in some manner. Springboot allows us to use `-Dspring.config.location=<location of application.properties>` as parameter in JAVA_OPTS. Identify any such parameters. 

## 4. Identify volumes
Since we are looking for stateless applications this is very generic. We need to copy these application.yaml/properties file from step-2 to certain volume location that can be mounted inside container. Same mounted file location can further be referred by Step-3 environment injections. Now lets move to complete example:

# Templates:
We have kept the complete thing in our github page [helm-templates](https://github.com/briozing/helm-templates). Feel free to fork it and make it better. Contributions are welcome. Template also deploys nginx-ingress controller and ingress mapping of it. 
## namespace/<namespacename/ENV>/<application_name>/values.yaml 
```yml
# Default values for my-boot-service.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

# Generic properties
namespace: qa
replicaCount: 1
ingressEnabled: true
appName: my-boot-service
containerPort: 8080
hasSecretVolume: true
secretVolumeMountPath: /opt/appConfig

# Images
image:
  repository: my-spring-servics/my-boot-service
  tag: v1
  pullPolicy: Always
  pullSecrets: regcred

# Image creds
imageCredentials:
  secretName: regcred
  registry: hub.docker.com
  username: abcd
  password: password
  docker-email: kohitij.das@briozing.com
# ----------------------------------------

env:
  configMap:
    JAVA_OPTS: -Dspring.config.location=/opt/appConfig/application-qa.properties -Dspring.profiles.active=qa
  secrets:
    APPLICATION_NAME: my-boot-service

# Container liveness and readyness
springContainerHealthChecks:
  livenessProbe:
    httpGet:
      path: /my-boot-service/actuator/health
      port: 8080
    initialDelaySeconds: 180
    timeoutSeconds: 1
    periodSeconds: 15
  readinessProbe:
    httpGet:
      path: /my-boot-service/actuator/health
      port: 8080
    initialDelaySeconds: 180
    timeoutSeconds: 1
    periodSeconds: 15

# Nginx ingress settings
controller:
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
  service: 
    enableHttp: true
  stats:
    enabled: true
  metrics:
    enabled: true

# Ingress settigns
ingress:
  enabled: true
  name: my-boot-service
  rules:
    - host: my-boot-service
      http:
        paths:
        - path: /my-boot-service
          backend:
            serviceName: my-boot-service-entrypoint
            servicePort: 80
```
## _imagepullsecret_helper.tpl
> Required to create imagepullsecret
```tpt
/* image pull secret */
{{- define "imagePullSecret" }}
{{- printf "{\"auths\": {\"%s\": {\"auth\": \"%s\"}}}" .Values.imageCredentials.registry (printf "%s:%s" .Values.imageCredentials.username .Values.imageCredentials.password | b64enc) | b64enc }}
{{- end }}
```
## service.yaml
Kubernetes service definition for deployment. 
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.appName }}-entrypoint
  namespace: {{ .Values.namespace }}
spec:
  selector:
    app: {{ .Values.appName }}
  ports:
    - protocol: TCP
      port: 80
      targetPort: {{ .Values.containerPort }}
      name: http
```
## secrets.yaml
You can see `{{- $path := printf "namespaces/%s/%s/configs/*" .Values.namespace .Values.appName  }}` line allows us to define the variable for Glob and create a secret file for all the files in the directory. 

```yaml
{{- if .Values.imageCredentials}}
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Values.imageCredentials.secretName }}
  namespace: {{ .Values.namespace}}
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: {{ template "imagePullSecret" . }}
{{- end}}

--- 
# Based on namespace from values.yaml we can define what files to put into secrets.
# Adding volume secret to avoid any confusion with envsecrets (IF any)
{{- $path := printf "namespaces/%s/%s/configs/*" .Values.namespace .Values.appName  }}
{{- if $path }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Values.appName }}-volume-sec
  namespace: {{ .Values.namespace}}
  labels:
    app: {{ .Values.appName }}
    chart: "{{ .Values.appName }}-{{ .Chart.Version | replace "+" "_" }}"
    release: "{{ .Release.Name }}"
    heritage: "{{ .Release.Service }}"
type: Opaque
data:
{{ (.Files.Glob $path).AsSecrets | indent 2 }}
{{- end }}
--- 
# Setup env secrets 
{{- if .Values.env.secrets }}
{{- $root := .Values.env.secrets }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Values.appName }}-env-secret
  namespace: {{ .Values.namespace}}
  labels:
    app: {{ .Values.appName }}
    chart: "{{ .Values.appName }}-{{ .Chart.Version | replace "+" "_" }}"
    release: "{{ .Release.Name }}"
    heritage: "{{ .Release.Service }}"
type: Opaque
data:
{{- range $k,$v := $root }}
  {{$k }}: {{ default "" $v | b64enc | quote }}
{{- end}}
{{- end }}
```
## Deployment
An attempt to make generic deployment. This works well for normal applications without sidecar. 
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.appName }}
  namespace: {{ .Values.namespace }}
spec:
  selector:
    matchLabels:
      app: {{ .Values.appName }}
  replicas: {{ .Values.replicaCount }}
  template:
    metadata:
      labels:
        app: {{ .Values.appName }}
    spec:
      containers:
      - name: {{ .Values.appName }}
        image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        {{- if .Values.hasSecretVolume }}
        volumeMounts:
        - name: {{ .Values.appName }}-volume-sec
          mountPath: {{ .Values.secretVolumeMountPath }}
        {{- end}}
        {{- if or .Values.env.configMap .Values.env.secrets }}
        envFrom:
        {{- if .Values.env.configMap }}
        - configMapRef:
            name: {{ .Values.appName }}-env-configmap
        {{- end }}
        {{- if .Values.env.secrets }}
        - secretRef:
            name: {{ .Values.appName }}-env-secret
        {{- end }}
        {{- end }}
        ports:
        - containerPort: {{ .Values.containerPort }}
          protocol: TCP
{{- if .Values.springContainerHealthChecks}}
{{ toYaml .Values.springContainerHealthChecks | indent 8 }}
{{- end}}
      {{- if .Values.hasSecretVolume }}
      volumes:
      - name: {{ .Values.appName }}-volume-sec
        secret:
          secretName: {{ .Values.appName }}-volume-sec
      {{- end}}
      {{- if .Values.imageCredentials}}
      imagePullSecrets:
      - name: {{.Values.imageCredentials.secretName}}
      {{- end}}
```
## Ingress resource
Nginx ingress controller and resource.
```yaml
{{- if .Values.ingress.enabled }}
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: {{ .Values.ingress.name }}
  namespace: {{ .Values.namespace }}
  annotations:
    kubernetes.io/ingress.class: nginx
    ingress.kubernetes.io/rewrite-target: /
    #nginx.org/server-snippet: "proxy_ssl_verify off;"
spec:
  rules:
{{ toYaml .Values.ingress.rules | indent 2 }}
{{- end}}
```
