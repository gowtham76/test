cicd.grovvy=== def call(Map<String, Object> args = [:], Closure pipeline) {

    String javaVersion = args.getOrDefault('java', '21')
    String cdkVersion = args.getOrDefault('cdk', '2.1033.0')
    Boolean useCacheForce = args.getOrDefault('useCacheForce', null)

    String mvnCpuRequest = args.getOrDefault('mvnCpuRequest', '500m')
    String mvnCpuLimit = args.getOrDefault('mvnCpuLimit', '1000m')
    String mvnMemoryRequest = args.getOrDefault('mvnMemoryRequest', '500Mi')
    String mvnMemoryLimit = args.getOrDefault('mvnMemoryLimit', '1Gi')
    String mvnOpts = args.getOrDefault('mvnOpts', '')
    String ENV_arg = args.getOrDefault('env', "DEV")

    def AWS_ACCOUNT_ID = args.getOrDefault('AWS_ACCOUNT_ID', '')
    def AWS_REGION = 'eu-west-1'

    def POD_USER = args.getOrDefault('POD_USER', 1001220000)
    def AWS_ROLE_TO_ASSUME = 'secops/automation/ASSUMEROLE-IAM-AUTOMATION'
    def ROLE_ARN = "arn:aws:iam::${AWS_ACCOUNT_ID}:role/${AWS_ROLE_TO_ASSUME}"

    def KANIKO_MEMORY = args.getOrDefault('KANIKO_MEMORY', '2Gi')
    def AWSCLI_MEMORY = args.getOrDefault('AWSCLI_MEMORY', '800Mi')
    def JNLP_MEMORY = args.getOrDefault('JNLP_MEMORY', '500Mi')

    String workingPath = "/home/jenkins/agent"
    String configPath = "${workingPath}/config"
    String storesPath = "${workingPath}/stores"
    String cachePath = "${workingPath}/cache"

    def TRI = "RWS".toLowerCase()
    def ENV = ENV_arg.toLowerCase()

    def AWS_CONFIG_PATH = '/tmp/aws/config'
    def AWS_CONFIG_MAP = "aws-profiles"
    def AWS_PROFILE = "account_${TRI}_${ENV}"
    def WORKING_DIR = '/tmp/workspace/'


    def SERVICE_ACCOUNT_NAME = "jenkins-automation-rws"

    String findLabel = 'captain-findvol-' + UUID.randomUUID()
    String podLabel = 'captain-linux-' + UUID.randomUUID()

    String volume = 'none'
    String volumeBasePrefix = 'jenkins-home-teams-marvel'
    String volumeBaseSuffix = '-0'
    int maxVolume = Integer.valueOf("${env.CACHE_VOLUME_COUNT ?: 2}")
    echo "maxVolume: ${maxVolume}"
    int leaseTimeout = Integer.valueOf("${env.CACHE_VOLUME_LEASE_TIMEOUT ?: 1}")
    echo "leaseTimeout: ${leaseTimeout}"

    boolean useCache = useCacheForce == true || (useCacheForce == null && env.CACHE_VOLUME != 'false')
    echo("node: ${podLabel}, java: ${javaVersion}, cdk: $cdkVersion, useCacheForce: ${useCacheForce}, env_cache: ${env.CACHE_VOLUME}")
    echo("useCache : ${useCache}")
    echo("mvnOpts : ${mvnOpts}")

    def jdkImage = "harbor01.registry.eu-west-1.group.aws-socgen.com/ocf-a8567-prd-shared/build/openjdk${javaVersion}-cdk:${cdkVersion}"
    String image = args.getOrDefault('image', jdkImage)

    def volumes = [
            configMapVolume(mountPath: "${configPath}", configMapName: 'slave-settings'),
            secretVolume(mountPath: "${storesPath}", secretName: 'maven-stores')
    ]

    stage('Find Volume') {

        timeout(time: 60, unit: 'MINUTES') {

            if (useCache) {
                podTemplate(
                        label: findLabel,
                        serviceAccount: "jenkins",
                        namespace: 'bsc-marvel-ci',
                        containers: [
                                containerTemplate(
                                        name: 'jnlp',
                                        image: "harbor01.registry.eu-west-1.group.aws-socgen.com/dds-dpt1-prod/inbound-agent:3107.v665000b_51092-REV-02-kubectl",
                                        imagePullPolicy: 'Always',
                                        workingDir: '/tmp',
                                        command: 'cat',
                                        ttyEnabled: true,
                                        securityContext: [
                                                runAsNonRoot            : true,
                                                readOnlyRootFilesystem  : false,
                                                runAsUser               : 1000,
                                                runAsGroup              : 1000,
                                                fsGroup                 : 1000,
                                                privileged              : false,
                                                allowPrivilegeEscalation: false,
                                                capabilities            : [drop: ['ALL']]
                                        ],
                                        resourceRequestCpu: '80m',
                                        resourceLimitCpu: '500m',
                                        resourceRequestMemory: '50Mi',
                                        resourceLimitMemory: '1Gi'
                                )
                        ]
                ) {
                    withEnv(["NODE=${findLabel}"]) {
                        node(env.NODE) {
                            container('jnlp') {
                                def volumeNamesList = []
                                for (int i = 1; i <= maxVolume; i++) {
                                    volumeNamesList.add("${volumeBasePrefix}${i}${volumeBaseSuffix}")
                                }
                                def volumeNames = String.join(",", volumeNamesList)

                                volume = sh(
                                        script: """
                                    set +x
                                    mkdir -p /tmp/leases
                                    cd /tmp/leases
                                    first=1
                                    stop=0
                                    while (( stop == 0 )) ; do
                                        echo "Waiting for a volume to be available..." >&2
                                        (( first == 1 )) && first=0 || { sleep 2; }
                                        echo "${volumeNames}" | sed "s/,/\\n/g" | sort > allVolumes.txt
                                        kubectl get pods -o yaml | grep claimName: | awk '{print \$2}' | sort -u > usedVolumes.txt || continue
                                        grep -xvFf usedVolumes.txt allVolumes.txt | shuf > unusedVolumes.txt ||:
                                        kubectl get leases -o json | jq -r '.items[] | .metadata.name' > leases.txt || continue
                                        grep -xvFf leases.txt unusedVolumes.txt | shuf > availableVolumes.txt ||:
                                        while read -r line ; do
                                            cat > lease.yaml <<EOF
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: \$line
spec:
  acquireTime: \$( date -u +%FT%T ).000000Z
EOF
                                            kubectl create -f lease.yaml &>/dev/null || continue
                                            echo "Found available volume: \$line" >&2
                                            echo "\$line"
                                            stop=1
                                            break
                                        done < availableVolumes.txt
                                        kubectl get leases -o json | jq -r '.items[] | [.metadata.name,.spec.acquireTime] | join(",")' > leases.txt || continue
                                        limit=\$( date +%s -d "-${leaseTimeout} minutes" )
                                        while IFS=, read -r n d ; do d=\$(date -d "\$d" +%s) ; (( d < limit )) && echo "\$n" ; done < leases.txt > oldLeases.txt ||:
                                        while read -r line ; do kubectl delete lease \$line 1>&2 ||: ; done < oldLeases.txt
                                    done
                                    :
                                    """,
                                        returnStdout: true
                                ).trim()
                            }
                        }
                    }
                }

                // Add cache volume
                echo "Volume name to use out of ${maxVolume}: ${volume}"
                volumes.add(persistentVolumeClaim(mountPath: "${cachePath}", claimName: "${volume}"))
            }
        }
    }

    podTemplate(
            label: podLabel,
            serviceAccount: "jenkins",
            namespace: 'bsc-marvel-ci',
            yaml: """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: ${SERVICE_ACCOUNT_NAME}
  automountServiceAccountToken: false
  securityContext:
    runAsNonRoot: true
    readOnlyRootFilesystem: false
    privileged: false
    allowPrivilegeEscalation: false
    runAsUser: 1000
    runAsGroup: 1000
    capabilities:
        drop:
        - ALL
  volumes:
    - name: ${AWS_CONFIG_MAP}
      configMap:
        name: ${AWS_CONFIG_MAP}-${TRI}
  containers:
  - name: jnlp
    image: harbor01.registry.eu-west-1.group.aws-socgen.com/dds-dpt1-prod/inbound-agent:3107.v665000b_51092-REV-02-kubectl
    imagePullPolicy: Always
    workingDir: ${workingPath}
    resources:
      requests:
        cpu: 80m
        memory: 50Mi
        ephemeral-storage: 4Gi
      limits:
        cpu: 500m
        memory: ${JNLP_MEMORY}
        ephemeral-storage: 8Gi

  - name: maven
    image: harbor01.registry.eu-west-1.group.aws-socgen.com/ocf-a8567-prd-shared/build/openjdk21-cdk:2.1029.4
    imagePullPolicy: Always
    command:
    - cat
    tty: true
    workingDir: ${workingPath}
    env:
      - name: MAVEN_CONFIG
        value: ${configPath}
      - name: MAVEN_OPTS
        value: ${mvnOpts}
      - name: MVN_CPU_LIMIT
        value: ${mvnCpuLimit}
    resources:
      requests:
        cpu: ${mvnCpuRequest}
        memory: ${mvnMemoryRequest}
        ephemeral-storage: 4Gi
      limits:
        cpu: ${mvnCpuLimit}
        memory: ${mvnMemoryLimit}
        ephemeral-storage: 8Gi
  - name: cdk-awscli
    image: harbor01.registry.eu-west-1.group.aws-socgen.com/ocf-a8567-prd-shared/build/openjdk21-cdk:2.1029.4
    imagePullPolicy: Always
    workingDir: ${WORKING_DIR}
    command:
    - /bin/sh
    - -c
    - cat
    tty: true
    resources:
        requests:
        cpu: 80m
        memory: 5Mi
        limits:
        cpu: 500m
        memory: 800Mi
    env:
        - name: GIT_SSL_NO_VERIFY
          value: true
        - name: NO_PROXY
          value: "${NO_PROXY}"
        - name: HTTP_PROXY
          value: "${HTTP_PROXY}"
        - name: HTTPS_PROXY
          value: "${HTTP_PROXY}"
        - name: HOME
          value: /tmp
        - name: AWS_CONFIG_FILE
          value: "${AWS_CONFIG_PATH}"
        - name: AWS_DEFAULT_PROFILE
          value: "account_${TRI}_${ENV}"          
        - name: AWS_PROFILE
          value: "account_${TRI}_${ENV}"          
        - name: "${AWS_CONFIG_MAP}"
          mountPath: "${AWS_CONFIG_PATH}"
          subPath: "profiles-aws.conf"
  - name: kaniko
    image: harbor01.registry.eu-west-1.group.aws-socgen.com/dds-dpt1-prod/kaniko:sg-v1.23.2-debug
    imagePullPolicy: Always
    command:
    - cat
    tty: true
    workingDir: ${workingPath}
    env:
      - name: AWS_DEFAULT_REGION
        value: eu-west-1
      - name: no_proxy
        value: "*.amazonaws.com,172.31.0.1:443,*.docker.io,*.docker.com"
      - name: HTTP_PROXY
        value: "http://\$(HOST_IP):8181"
    securityContext:
      privileged: true
    resources:
      requests:
        cpu: 80m
        memory: 5Mi
        ephemeral-storage: 4Gi
      limits:
        cpu: 500m
        memory: ${KANIKO_MEMORY}
        ephemeral-storage: 8Gi
""",
            volumes: volumes
    ) {
        withEnv([
                "NODE=${podLabel}"
        ]) {
            pipeline()

            if (useCache) {
                node(env.NODE) {
                    container('jnlp') {
                        sh """
                            kubectl delete lease "${volume}" ||:
                        """
                    }
                }
            }
        }
    }
}


test.grrovy==import com.socgen.marvel.captainamerica.bean.PipelineBranchConfiguration
import com.socgen.marvel.captainamerica.bean.PipelineConfiguration


def call(def params = []) {
    def isEmpty = { def value -> return value == null || (value.class == String.class && value.trim().length() == 0) }

    def IS_SECURE = false
    def IS_DEBUG = params['IS_DEBUG'] ?: false
    def IS_GRAVITON = params['IS_GRAVITON'] ?: false

    def TRI = params['TRI']
    def IRT = params['IRT']
    def COMPONENT_NAME = params['COMPONENT_NAME'] // TODO: check format
    def BRANCH_NAME = params['BRANCH_NAME']
    def POM_FILE = params['POM_FILE'] ?: 'crr/pom.xml'
    def CLONE_URL= params['CLONE_URL']
    def INFRA_VERSION = params['INFRA_VERSION']
    def SYSTEM_VERSION = params['SYSTEM_VERSION']
    def ENV = params['ENV']
    def CDK_PROFILE = params['CDK_PROFILE'] ?: ENV
    // to be used for any future extra context args
    def CDK_EXTRA_ARGS = params['CDK_EXTRA_ARGS']

    def JFROG_REPO_ID = params['JFROG_REPO_ID'] ?: env.JFROG_REPO_ID
    def JFROG_CREDENTIAL_ID = params['JFROG_CREDENTIAL_ID'] ?: env.JFROG_CREDENTIAL_ID
    def GROUP_ID = params['GROUP_ID']
    def MODULES = params['MODULES']
    def CDK = params['CDK']

    def AWS_ACCOUNT_ID = params['AWS_ACCOUNT_ID']

    if (isEmpty(TRI) ||
            isEmpty(IRT) ||
            isEmpty(COMPONENT_NAME) ||
            isEmpty(BRANCH_NAME) ||
            isEmpty(INFRA_VERSION) ||
            isEmpty(SYSTEM_VERSION) ||
            isEmpty(ENV) ||
            isEmpty(CDK_PROFILE) ||
            isEmpty(GROUP_ID) ||
            isEmpty(MODULES) ||
            isEmpty(CDK) ||
            isEmpty(AWS_ACCOUNT_ID)
    ) {
        currentBuild.result = 'FAILURE'
        println 'Some required args are empty'
        return
    }

    def OCF_PRD_ACCOUNT= "976495800378"
    def OCF_ECR_HOST = "${OCF_PRD_ACCOUNT}.dkr.ecr.eu-west-1.amazonaws.com"

    def FRONTAPP_COMPONENT_NAME_SUFFIX = '-frontapp' // Should be the same in ocf-shared-cdk-stacks

    ENV = ENV.toLowerCase()
    IRT = IRT.toLowerCase()
    TRI = TRI.toLowerCase()
    BATCH_COMPONENT_NAME = COMPONENT_NAME.toLowerCase()
    FRONTAPP_COMPONENT_NAME = BATCH_COMPONENT_NAME + FRONTAPP_COMPONENT_NAME_SUFFIX

    def AWS_REGION = 'eu-west-1'

    def OCF_FOUNDATION_STACK_NAME = 'OCFFoundationStack'
    def BATCH_COMPONENT_STACK_NAME = null
    def FRONTAPP_COMPONENT_STACK_NAME = null
    def BATCH_COMPONENT_STACK_IDENTIFIER = "${BATCH_COMPONENT_NAME}ComponentStack"
    def FRONTAPP_COMPONENT_STACK_IDENTIFIER = "${FRONTAPP_COMPONENT_NAME}ComponentStack"
    def BATCH_COMPONENT_VERSION_STACK_NAME = "${BATCH_COMPONENT_NAME}BatchCVStack-${INFRA_VERSION}-${SYSTEM_VERSION}"
    def FRONTAPP_COMPONENT_VERSION_STACK_NAME = "${FRONTAPP_COMPONENT_NAME}RealtimeCVStack-${INFRA_VERSION}-${SYSTEM_VERSION}"

    def DEFAULT_SOFA_BUCKET_NAME = "${IRT}-${TRI}-${ENV}"

    def AWS_ROLE_TO_ASSUME = 'secops/automation/ASSUMEROLE-IAM-AUTOMATION'
    def ROLE_ARN = "arn:aws:iam::${AWS_ACCOUNT_ID}:role/${AWS_ROLE_TO_ASSUME}"
    def CDK_QUALIFIER = 'main'
    def CFN_EXEC_ROLE = "arn:aws:iam::$AWS_ACCOUNT_ID:role/project/servicerole/cloudformation/cdk-${CDK_QUALIFIER}-cfn-exec-role-$AWS_ACCOUNT_ID-eu-west-1"

    def WORKING_DIR = '/tmp/workspace/'

    def REQUIRED_MODULE_KEYS = [
            'ARTIFACT_ID',
            'DOCKER_FILE_PATH',
            'DOCKER_BUILD_ARGS'
    ]

    def REQUIRED_CDK_KEYS = [
            'ARTIFACT_ID',
            'BATCH_MAIN_CLASS',
            'FRONTAPP_MAIN_CLASS'
    ]

    def FRONTAPP_MODULE_KEY = 'frontapp'
    def frontAppExists = false

    try {
        // Verify frontApp module exists, verify each module keys & set moduleName to lower case
        MODULES = MODULES.collectEntries { key, value ->
            REQUIRED_MODULE_KEYS.each { requiredKey ->
                if (!value.containsKey(requiredKey)) {
                    throw new Exception("The key $requiredKey is required for MODULES[$key], the required keys are : $REQUIRED_MODULE_KEYS")
                }
            }
            def b = [:]
            b[key.toLowerCase()] = value
            return b
        }

        frontAppExists = MODULES.containsKey(FRONTAPP_MODULE_KEY)

        if (!frontAppExists) {
            print("Deploying without $FRONTAPP_MODULE_KEY")
        }

        // Verify cdk attributes
        REQUIRED_CDK_KEYS.each { requiredKey ->
            if (!CDK.containsKey(requiredKey)) {
                throw new Exception("The key $requiredKey is required for CDK, the required keys are : $REQUIRED_CDK_KEYS")
            }
        }
    } catch (Exception e) {
        currentBuild.result = 'FAILURE'
        println e.getMessage()
        return
    }

    def BATCH_MAIN_CDK = CDK['BATCH_MAIN_CLASS']
    def FRONTAPP_MAIN_CLASS = CDK['FRONTAPP_MAIN_CLASS']

    // define branch version and display name
    def branch = PipelineBranchConfiguration.getInstance(this,
            BRANCH_NAME,
            "24.3",
            env.BUILD_NUMBER as Integer,
            true)

    node(env.NODE) {
        def infraDir = "${CDK['DIR']}/classes"
        def infraJar = "${CDK['ARTIFACT_ID']}-${branch.revision}.jar"

        currentBuild.displayName = "${BATCH_COMPONENT_NAME}&frontapp-${branch.revision}@${INFRA_VERSION}/${SYSTEM_VERSION}:${ENV}"
        def mvnConfig = "-f ${POM_FILE} -Drevision=${branch.revision} -Dmaven.test.skip -Dmaven.javadoc.skip".toString()

        def buildConfiguration = new PipelineConfiguration()
        buildConfiguration.jfrogCredentials = "jfrog-" + JFROG_REPO_ID
        buildConfiguration.mavenSettings = "jfrog-" + JFROG_REPO_ID
        int minVolumeFreeSize = Integer.valueOf("${env.CACHE_VOLUME_MIN_FREE ?: 1*1024*1024}")

        try {
            stage('Checkout Project') {
                git branch: "${BRANCH_NAME}",
                        credentialsId: 'jenkins-marvel',
                        url: "${CLONE_URL}"
                sh "ls -a"
            }
            stage('package JARs') {
                sh """
                                [ -d /tmp/cache ] && {
                                    echo "Show cache size..."
                                    df -h /tmp/cache
                                    echo "Cleaning cache..."
                                    if [ -d /tmp/cache/mvn ] ; then
                                        find /tmp/cache/mvn -name "*.lastUpdated" -delete
                                        find /tmp/cache/mvn -name "*-SNAPSHOT" -exec echo rm -rf "{}" \\;
                                    fi
                                    echo "Computing cache size..."
                                    read size used < <( df /tmp/cache --output=size,used | tail -n +2 | awk '{print \$1" "\$2}' )
                                    echo "Cache total size is \$size KB"
                                    echo "Cache used size is \$used KB"
                                    echo "Cache minimum free size is ${minVolumeFreeSize} KB"
                                    free=\$(( size - used ))
                                    echo "Cache free size is \$free KB"
                                    echo "Cache minimum free size is ${minVolumeFreeSize} KB"
                                    (( free >= ${minVolumeFreeSize} )) || {
                                        echo "Cache size is too big, deleting..."
                                        rm -rf /tmp/cache/*
                                    }
                                }
                                :
                            """
                mvn("package ${mvnConfig} -pl !crr-fitnesse", buildConfiguration)

            }

            stage('Copy JARs') {
                container('maven') {
                    //copy infra jar in classes folder
                    sh """
                                    ls -l ${infraDir}
                                    cp ${CDK['DIR']}/${infraJar} ${infraDir}/
                                    ls -a ${infraDir}
                                """

                    //copy module jar in classes folder
                    MODULES.each { moduleName, o ->
                        def moduleDir = o['DIR']
                        def jarName= "${o['ARTIFACT_ID']}-${branch.revision}.jar"
                        sh """
                                    cd ${moduleDir}
                                    if [ ! -f classes/${jarName} ]; then
                                        cp ${jarName} classes/
                                        echo "jar ${jarName} copied to classes/"
                                    else
                                        echo "file ${jarName} already exist/"
                                    fi
                                    ls -a classes/
                                """
                    }
                }
            }

            stage('Init AWS credentials') {
                container('cdk-awscli') {
                    sh "aws sts assume-role --role-arn ${ROLE_ARN} --role-session-name ${TRI}-ci-session --duration-seconds 3600 > ~/Assumerole.json"
                    def aws_access_key_id = sh(
                            script: "cat ~/Assumerole.json | jq -r '.Credentials.AccessKeyId'",
                            returnStdout: true
                    ).trim()
                    def aws_secret_access_key = sh(
                            script: "cat ~/Assumerole.json | jq -r '.Credentials.SecretAccessKey'",
                            returnStdout: true
                    ).trim()
                    def session_token = sh(
                            script: "cat ~/Assumerole.json | jq -r '.Credentials.SessionToken'",
                            returnStdout: true
                    ).trim()

                    sh 'mkdir ~/.aws'
                    sh "echo '[default]' >> ~/.aws/credentials"
                    sh "echo 'aws_access_key_id=${aws_access_key_id}' >> ~/.aws/credentials"
                    sh "echo 'aws_secret_access_key=${aws_secret_access_key}' >> ~/.aws/credentials"
                    sh "echo 'aws_session_token=${session_token}' >> ~/.aws/credentials"
                    sh "echo '[default]' >> ~/.aws/config"
                    sh "echo 'region=${AWS_REGION}' >> ~/.aws/config"
                    sh "echo 'output=json' >> ~/.aws/config"
                }
            }

            def tagsToPush = [:]
            def frontAppTag
            MODULES.each { key, value ->
                if (key.equals(FRONTAPP_MODULE_KEY)) {
                    def timestamp = new Date().getTime()
                    frontAppTag = "${FRONTAPP_COMPONENT_NAME}-${branch.revision}-${timestamp}"
                    tagsToPush[FRONTAPP_MODULE_KEY] = frontAppTag
                } else {
                    tagsToPush[key] = "${BATCH_COMPONENT_NAME}-${value['ARTIFACT_ID']}-${branch.revision}"
                }
            }

            def batchTagsToJSON = groovy.json.JsonOutput.toJson(tagsToPush).replace('\"', "\\\"")

            def commonContextArgs = "-c @aws-cdk/core:bootstrapQualifier=${CDK_QUALIFIER} \
                                    -c tri=${TRI} \
                                    -c irt=${IRT} \
                                    -c env=${ENV} \
                                    -c isSecure=${IS_SECURE} \
                                    -c isGraviton=${IS_GRAVITON} \
                                    -c cdkProfile=${CDK_PROFILE} \
                                    -c accountId=${AWS_ACCOUNT_ID} \
                                    -c region=${AWS_REGION} \
                                    -c isDebug=${IS_DEBUG} \
                                    ${CDK_EXTRA_ARGS ?: ''} "

            def batchCdkContextArgs = "${commonContextArgs} \
                                    -c componentName=${BATCH_COMPONENT_NAME} \
                                    -c currentSystemVersion=${SYSTEM_VERSION} \
                                    -c currentInfraVersion=${INFRA_VERSION} \
                                    -c tags=$batchTagsToJSON"

            def realtimeCdkContextArgs = "${commonContextArgs} \
                                    -c currentSystemVersion=${SYSTEM_VERSION} \
                                    -c currentInfraVersion=${INFRA_VERSION} \
                                    -c componentName=${FRONTAPP_COMPONENT_NAME} \
                                    -c imageTag=${frontAppTag}"

            stage('Generate infrastructure CloudFormation code') {
                container('cdk-awscli') {
                    dir(infraDir) {
                        if(frontAppExists){
                            sh "cdk --app 'java -cp ${infraJar} ${FRONTAPP_MAIN_CLASS}' synth ${realtimeCdkContextArgs} --require-approval=never"
                        }
                        sh "cdk --app 'java -cp ${infraJar} ${BATCH_MAIN_CDK}' synth ${batchCdkContextArgs} --require-approval=never"
                    }
                }
            }

            stage('Deploy component infrastructure if exists') {
                container('cdk-awscli') {
                    dir(infraDir) {
                        // Batch component stack
                        isComponentStackExists = false
                        try {
                            BATCH_COMPONENT_STACK_NAME = sh(
                                    script: "cdk --app 'java -cp ${infraJar} ${BATCH_MAIN_CDK}' ls ${batchCdkContextArgs} --require-approval=never | grep '${BATCH_COMPONENT_STACK_IDENTIFIER}'",
                                    returnStdout: true
                            ).trim()
                            isComponentStackExists = true
                        } catch (e) {
                            println "Stack: ${BATCH_COMPONENT_STACK_IDENTIFIER} does not exist."
                        }
                        if (isComponentStackExists)
                            sh "cdk --app 'java -cp ${infraJar} ${BATCH_MAIN_CDK}' deploy -r ${CFN_EXEC_ROLE} -e ${BATCH_COMPONENT_STACK_NAME} ${batchCdkContextArgs} --no-previous-parameters --parameters ${BATCH_COMPONENT_STACK_NAME}:infraVersion=${INFRA_VERSION} --require-approval=never"

                        if(frontAppExists){
                            // Frontapp component stack
                            isComponentStackExists = false
                            try {
                                FRONTAPP_COMPONENT_STACK_NAME = sh(
                                        script: "cdk --app 'java -cp ${infraJar} ${FRONTAPP_MAIN_CLASS}' ls ${realtimeCdkContextArgs} --require-approval=never | grep '${FRONTAPP_COMPONENT_STACK_IDENTIFIER}'",
                                        returnStdout: true
                                ).trim()
                                isComponentStackExists = true
                            } catch (e) {
                                println "Stack: ${FRONTAPP_COMPONENT_STACK_IDENTIFIER} does not exist."
                            }
                            if (isComponentStackExists)
                                sh "cdk --app 'java -cp ${infraJar} ${FRONTAPP_MAIN_CLASS}' deploy -r ${CFN_EXEC_ROLE} -e ${FRONTAPP_COMPONENT_STACK_NAME} ${realtimeCdkContextArgs} --no-previous-parameters --parameters ${FRONTAPP_COMPONENT_STACK_NAME}:infraVersion=${INFRA_VERSION} --require-approval=never"
                        }
                    }
                }
            }
            stage('Build & push Docker images to ECR') {
                def repoUri
                container('cdk-awscli') {
                    repoUri = sh(
                            script: "aws cloudformation describe-stacks --stack-name ${OCF_FOUNDATION_STACK_NAME} --query 'Stacks[0].Outputs[?OutputKey==`ECRREPOSITORYURI`].OutputValue' --output text",
                            returnStdout: true
                    ).trim()
                }
                MODULES.each { moduleName, o ->
                    def dockerFilePath = "${o['DOCKER_FILE_PATH']}"
                    def dockerBuildArgs = "${o['DOCKER_BUILD_ARGS']}"
                    def moduleDir= "${o['DIR']}/classes"
                    container('kaniko') {
                        dir(moduleDir) {
                            def imageTag = tagsToPush[moduleName]
                            def imageDest = "${repoUri}:${imageTag}"
                            sh "/kaniko/executor  --context `pwd` \
                                                            --dockerfile ${dockerFilePath} \
                                                            --destination=${imageDest} \
                                                            ${dockerBuildArgs}"
                        }
                    }
                }
            }
            stage('Deploy component version infrastructure') {
                container('cdk-awscli') {
                    dir(infraDir) {
                        // Deploy batch componentVersionStack
                        sh "cdk --app 'java -cp ${infraJar} ${BATCH_MAIN_CDK}' deploy -r ${CFN_EXEC_ROLE} -e '${BATCH_COMPONENT_VERSION_STACK_NAME}' ${batchCdkContextArgs} --no-previous-parameters --parameters ${BATCH_COMPONENT_VERSION_STACK_NAME}:systemVersion=${SYSTEM_VERSION} --parameters ${BATCH_COMPONENT_VERSION_STACK_NAME}:infraVersion=${INFRA_VERSION} --require-approval=never"
                        if(frontAppExists){
                            // Deploy frontapp componentVersionStack
                            sh "cdk --app 'java -cp ${infraJar} ${FRONTAPP_MAIN_CLASS}' deploy -r ${CFN_EXEC_ROLE} -e '${FRONTAPP_COMPONENT_VERSION_STACK_NAME}' ${realtimeCdkContextArgs} --no-previous-parameters --parameters ${FRONTAPP_COMPONENT_VERSION_STACK_NAME}:systemVersion=${SYSTEM_VERSION} --parameters ${FRONTAPP_COMPONENT_VERSION_STACK_NAME}:infraVersion=${INFRA_VERSION} --require-approval=never"
                        }
                    }
                }
            }
        } catch (err) {
            currentBuild.result = 'FAILURE'
            throw err
        }
    }

}



