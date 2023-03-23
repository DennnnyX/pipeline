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
            container('maven')
            {
                stage('mvn packaging')
                {
                    sh 'mvn clean package'
                }
            }
        }
        stage('Build Into Image')
        {
            container('kaniko')
            {
                stage('Build With Kaniko')
                {
                    echo 'Hello kaniko'
                    sh "/kaniko/executor --dockerfile `pwd`/Dockerfile --context `pwd` --destination dennnys/pipeline:v2"

                }
            }
        }

    }
}




# /kaniko/executor --force --cache=true --cache-dir=/cache -c `pwd` --dockerfile=./Dockerfile --destination=${image}:latest