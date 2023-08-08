pipeline {
  agent { // 后续的 stages 执行指令会部署在哪个节点运行
    kubernetes {
      label 'maven' // 通过标签进行匹配
    }
  }

  // 在构建的过程中可以选择哪些参数进行构建的
  parameters { // 参数化构建
    string(name: 'BRANCH_NAME', defaultValue: 'master', description: '请选择要发布的分支')
    string(name: 'TAG_NAME', defaultValue: 'snapshot', description: '标签名称，必须以 v 开头，例如：v1、v1.0.0')
    // choice('name': 'NAMESPACE', choices: ['devops-dev', 'devops-test', 'devops-prod'], description: '命名空间')
  }

  environment { // 环境变量
    REGISTRY = '10.0.0.51:8858' // Harbor 镜像仓库的用户名和密码
    DOCKER_CREDENTIAL_ID = 'harbor-user-pass' // Harbor 镜像仓库的用户名和密码
    GIT_REPO_URL = '10.0.0.52:28080' // Gitlab 的仓库URL地址
    GIT_CREDENTIAL_ID = 'gitlab-user-pass' // Gitlab 代码仓库的用户名和密码
    GIT_ACCOUNT = 'kubesphere' // 根据实际环境进行需要修改
    KUBECONFIG_CREDENTIAL_ID = 'e9ebfe57-1e23-4ca6-b47c-df265bdf9814' // kubeconfig-id 配置文件，需要kubernetes集群的权限
    DOCKERHUB_NAMESPACE = 'wolfcode' // 根据实际环境进行需要修改，Harbor的项目名称
    GITHUB_ACCOUNT = 'root' // 弃用
    APP_NAME = 'k8s-cicd-demo' // 项目的名称
    SONAR_SERVER_URL = 'http://10.0.0.50:30090' // Sonarqube 的连接地址URL
    SONAR_CREDENTIAL_ID = 'sonarqube-token' // Sonarqube 的 token 信息
  }

  stages { // 执行的各种阶段步骤
    stage('clone code') { // 代码拉取
      steps {
        // container('maven') {
          git(url: 'http://10.0.0.52:28080/kubesphere/k8s-cicd-demo.git', credentialsId: 'gitlab-user-pass', branch: '$BRANCH_NAME', changelog: true, poll: false) // Jenkins 环境中必须要用 git 的环境命令
        // }
      }
    }

    stage('unit test') { // 单元测试
      steps {
        // container('maven') {
          sh 'mvn clean test' // Jenkins 环境中必须要用 mvn 的命令
        // }
      }
    }

    stage('sonarqube analysis') { // Sonarqube的分析
      agent none
      steps {
        withCredentials([string(credentialsId : 'sonarqube-token' ,variable : 'SONAR_TOKEN')]) {
          withSonarQubeEnv('sonarqube') { // 在系统配置中的Sonarqube的NAME
            // container('maven') {
              sh '''mvn sonar:sonar -Dsonar.projectKey=$APP_NAME
echo "mvn sonar:sonar -Dsonar.projectKey=$APP_NAME"''' // Sonarqube 会创建 $APP_NAME 的项目
            // }
          }

          // Jenkins在分析代码审查分析的结果发给服务端进行验证是否通过的时间
          timeout(unit: 'MINUTES', activity: true, time: 5) { // 设置超时时间
            waitForQualityGate 'true'
          }
        }
      }
    }

//    stage('unit test') { // 单元测试
//      steps {
//        container('maven') {
//          sh 'mvn clean test'
//        }
//      }
//    }

    stage('build & push') { // 镜像构建和镜像推送
      steps {
        // container('maven') {
          sh 'mvn clean package -DskipTests' // 进行打包，跳过单元测试
          sh 'docker build -f Dockerfile -t $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BUILD_NUMBER .' // BUILD_NUMBER 是Jenkins 内部的参数
          withCredentials([usernamePassword(credentialsId : 'harbor-user-pass' ,passwordVariable : 'DOCKER_PASSWORD' ,usernameVariable : 'DOCKER_USERNAME' ,)]) {
            sh '''echo "$DOCKER_PASSWORD" | docker login $REGISTRY -u "$DOCKER_USERNAME" --password-stdin
                  docker push $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BUILD_NUMBER'''
          }
        // }
      }
    }

    stage('push latest') { // 将镜像打上新的标签，保证镜像构建过程中就会有一个最新版本  并进行推送
      // when { // 进行条件判断，需要满足分支是 master 分支
      //   branch 'master'
      // }
      steps {
        // container('maven') {
          sh 'docker tag $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BUILD_NUMBER $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:latest'
          sh 'docker push $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:latest'
        // }
      }
    }

    stage('deploy to dev') { 
      // when { // 进行条件判断，需要满足分支是 master 分支
      //   branch 'master'
      // }
      steps {
        // container('maven') {
          input(id: 'deploy-to-dev', message: 'deploy to dev?')

          // withCredentials([kubeconfigContent(credentialsId : '$KUBECONFIG_CREDENTIAL_ID' ,variable : 'ADMIN_KUBECONFIG' ,)]) { // KUBECONFIG_CREDENTIAL_ID
            sh 'mkdir -p ~/.kube/'
            sh 'echo "$ADMIN_KUBECONFIG" > ~/.kube/config'
            sh '''sed -i\'\' "s#REGISTRY#$REGISTRY#"                        deploy/cicd-demo-dev.yaml
                  sed -i\'\' "s#DOCKERHUB_NAMESPACE#$DOCKERHUB_NAMESPACE#"  deploy/cicd-demo-dev.yaml
                  sed -i\'\' "s#APP_NAME#$APP_NAME#"                        deploy/cicd-demo-dev.yaml
                  sed -i\'\' "s#BUILD_NUMBER#$BUILD_NUMBER#"                deploy/cicd-demo-dev.yaml

                  kubectl apply -f deploy/cicd-demo-dev.yaml'''
          // }
        // }
      }
    }

    stage('push with tag') {
      // agent none
      when { // 条件的匹配
        expression {
          params.TAG_NAME =~ /v.*/ // 必须需要TAG名字要有 v 开头
        }
      }
      steps {
        // input(id: 'release-image-with-tag', message: "release image with tag?")
        input(message: 'release image with tag?', submitter: '') // 接受一个输入值，必须是 true 才能部署
        withCredentials([usernamePassword(credentialsId : 'gitlab-user-pass' ,passwordVariable : 'GIT_PASSWORD' ,usernameVariable : 'GIT_USERNAME' ,)]) {
          sh 'git config --global user.email "zhongzhiwei@kubesphere.io" '
          sh 'git config --global user.name "zhongzhiwei" '
          sh 'git tag -a $TAG_NAME -m "$TAG_NAME" '
          sh 'git push http://$GIT_USERNAME:$GIT_PASSWORD@$GIT_REPO_URL/$GIT_ACCOUNT/k8s-cicd-demo.git --tags --ipv4'
        }

        // container('maven') {
          // 重新给镜像打标签
          sh 'docker tag $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BUILD_NUMBER $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:$TAG_NAME'
          // 推送镜像
          sh 'docker push $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:$TAG_NAME'
        // }
      }
    }

    stage('deploy to production') {
      // agent none
      when { // 条件判断
        expression {
          params.TAG_NAME =~ /v.*/ // 必须需要TAG名字要有 v 开头
        }
      }
      steps {
        // input(id: 'deploy-to-production', message: "deploy to production?")
        input(message: 'deploy to production?', submitter: '')
        // container('maven') {
          sh '''sed -i\'\' "s#REGISTRY#$REGISTRY#"                        deploy/cicd-demo.yaml
                sed -i\'\' "s#DOCKERHUB_NAMESPACE#$DOCKERHUB_NAMESPACE#"  deploy/cicd-demo.yaml
                sed -i\'\' "s#APP_NAME#$APP_NAME#"                        deploy/cicd-demo.yaml
                sed -i\'\' "s#TAG_NAME#$TAG_NAME#"                        deploy/cicd-demo.yaml

                kubectl apply -f deploy/cicd-demo.yaml'''
        // }
      }
    }
  }
}