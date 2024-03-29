## jenkins Pipeline

    Jenkins is, fundamentally, an automation engine which supports a number of automation patterns. Pipeline adds a powerful set of automation tools onto Jenkins, supporting use cases that span from simple continuous integration to comprehensive CD pipelines.

## scenario description and design

    We have a jave demo and compile it. We build it into docker image with dockerfile. After that, we push this image into a image repository. During this whole process, we should use a jenkins running on kubernetes. We use a dynamic pod generated by our pipeline to install our dependencies and build the image. After pipeline, the dynamibc pod will be deleted.

## jenkins installation and configuration

    First we need to deploy jenkins service in kubernetes using yaml file.

serviceAccount.yaml

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: jenkins-admin
rules:
  - apiGroups: [""]
    resources: ["*"]
    verbs: ["*"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-admin
  namespace: devops-tools
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins-admin
subjects:
- kind: ServiceAccount
  name: jenkins-admin
  namespace: devops-tools
```

volume.yaml

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-pv-volume
  labels:
    type: local
spec:
  storageClassName: local-storage
  claimRef:
    name: jenkins-pv-claim
    namespace: devops-tools
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  local:
    path: /mnt
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - docker-desktop       # k8s work node
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pv-claim
  namespace: devops-tools
spec:
  storageClassName: local-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```

deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: devops-tools
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins-server
  template:
    metadata:
      labels:
        app: jenkins-server
    spec:
      securityContext:
            fsGroup: 1000
            runAsUser: 0
      serviceAccountName: jenkins-admin
      containers:
        - name: jenkins
          image: jenkins/jenkins:lts
          resources:
            limits:
              memory: "2Gi"
              cpu: "1000m"
            requests:
              memory: "500Mi"
              cpu: "500m"
          ports:
            - name: httpport
              containerPort: 8080
            - name: jnlpport
              containerPort: 50000
          livenessProbe:
            httpGet:
              path: "/login"
              port: 8080
            initialDelaySeconds: 90
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 5
          readinessProbe:
            httpGet:
              path: "/login"
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          volumeMounts:
            - name: jenkins-data
              mountPath: /var/jenkins_home
      volumes:
        - name: jenkins-data
          persistentVolumeClaim:
              claimName: jenkins-pv-claim
```

service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins-service
  namespace: devops-tools
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/path:   /
      prometheus.io/port:   '8080'
spec:
  selector:
    app: jenkins-server
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 32000
```

    we can use some kubectl commands to check our jenkins service and pods. After deployment, we use some commands to get jenkins started and configured.

```shell
kubectl get pods -n devops-tools #get jenkins pod name
Kubectl exec -it podName cat /var/jenkins_home/secrets/initialAdminPassword -n devops-tools
#get jenkins initiallization password
```

### plugins

    After unlock jenkins, we need to install some basic plugins. These plugins help us pull code from github repo and deploy into a kubernetes pod with pipeline. Maven plugin is needed for java project.

![](file://C:\Users\I528814\AppData\Roaming\marktext\images\2023-04-30-08-05-06-image.png?msec=1683004126290)

    github plugin configuration. This plugin helps us configure connection between kubernetes and github.

![](file://C:\Users\I528814\AppData\Roaming\marktext\images\2023-04-30-08-07-18-image.png?msec=1683004126268)

    kubernetes plugin configuration. This plugin can set up kubernetes in jenkins.

![](file://C:\Users\I528814\AppData\Roaming\marktext\images\2023-04-30-08-07-50-image.png?msec=1683004126269)

![](file://C:\Users\I528814\AppData\Roaming\marktext\images\2023-04-30-08-08-06-image.png?msec=1683004126269)

![](file://C:\Users\I528814\AppData\Roaming\marktext\images\2023-04-30-08-08-18-image.png?msec=1683004126270)

    Here jenkins URL stands for the access point for Jenkins service. We use nodeport to set up kubernetes connection, so we use jenkins service IP for jenkins URL.

```shell
kubectl get svc -n devops-tools -o wide # get jenkins service IP
```

The jenkins tunnel helps the dynamic pod we create to connect with the jenkins pod. It uses TCP to communicate with our pods, so we use jenkins pod IP.

```shell
kubectl get pods -n devops-tools -o wide # get jenkins pod IP
```

![](file://C:\Users\I528814\AppData\Roaming\marktext\images\2023-04-30-08-12-12-image.png?msec=1683004126270)

## pipeline description

    pipeline is an importan part of jenkins. We use pipeline to build a process that pull the code and we build it into a image. After that, we use pipeline to deploy it into a kubernetes pod equipped with running environment. During this whole process, we need to implement a brand new concept called podTemplate. podTemplate is used under the circusmtance which we want to generate a new pod to run our code.

    some basic podTemplates

```shell
 podTemplate(containers: [
    containerTemplate(
     name: 'maven',
     image: 'sunrdocker/jdk17-git-maven-docker-focal',
     command: 'cat',
     ttyEnabled: 'true')
  ])  

# image stands for the image repo you want
# command stands for the command that you want the image to run
# ttyEnabled if true, the dynamic pod will be deleted after pipeline


 podTemplate(yaml: '''
    apiVersion: v1
    kind: Pod
    spec:
      containers:
      - name: maven
        image: sunrdocker/jdk17-git-maven-docker-focal
        command:
        - cat
        tty: true
''') 

# podTemplate in yaml 
```

### pipeline

```shell
podTemplate(containers: [
    containerTemplate(name: 'maven', image: 'sunrdocker/jdk17-git-maven-docker-focal', command: 'sleep', args: '99d')
  ],
  yaml: """\
apiVersion: v1
kind: Pod
metadata:
  name: kaniko
spec:
  containers:
  - name: kaniko
    image: uhub.service.ucloud.cn/uk8sdemo/executor:debug
    command:
    - cat
    tty: true
    volumeMounts:
      - name: kaniko-secret
        mountPath: /kaniko/.docker
  restartPolicy: Never
  volumes:
    - name: kaniko-secret
      secret:
        secretName: regcred
    """.stripIndent()

  )
  {
    node(POD_LABEL) {
        stage('Git Clone')
        {
            git credentialsId: 'a9245f23-3644-4bc1-8ff3-9edd068958c9', url: 'https://github.com/DennnnyX/pipeline.git'
        }
        stage('Compile')
        {
            container('maven') # pull code with maven
            {
                stage('mvn packaging')
                {
                    sh 'mvn clean package'
                }
            }
        }
        stage('Build Into Image') # build image with kaniko
        {
            container('kaniko')
            {
                stage('Build With Kaniko')
                {
                    echo 'Hello kaniko'
                    sh "/kaniko/executor --dockerfile `pwd`/Dockerfile --context `pwd` --destination dennnys/pipeline:v2"
                    # use kaniko to build image to your repo
                }
            }
        }
    }
}
```

    Kaniko is a tool to build images from Dockerfile inside a kubernetes pod. The kaniko executor image is responsible for building an image from a Dockerfile and pushing it to a registry. Within the executor image, we extract the filesystem of the base image (the FROM image in the Dockerfile). We then execute the commands in the Dockerfile, snapshotting the filesystem in userspace after each one. After each command, we append a layer of changed files to the base image (if there are any) and update image metadata.

Dockerfile

```dockerfile
FROM sunrdocker/jdk17-git-maven-docker-focal

COPY /target/*.jar /app.jar

CMD ["--server.port=8080"]

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "/app.jar"]
```
