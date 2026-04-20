def call(Map<String, Object> args = [:], Closure pipeline) {

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

    String workingPath = "/tmp"
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
    readOnlyRootFilesystem: true
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
