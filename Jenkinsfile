def deployments = []
def childJob = ''

pipeline {
    agent none
    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 90, unit: 'MINUTES')
    }
    parameters {
        string(name: 'CONFIG_FILE', defaultValue: 'config/boards.yaml', description: 'Repository YAML path')
    }
    stages {
        stage('Read configuration') {
            agent any
            steps {
                script {
                    def config = readYaml(file: params.CONFIG_FILE)
                    if (!(config instanceof Map) || !(config.defaults instanceof Map) ||
                        !(config.boards instanceof List) || config.boards.isEmpty()) {
                        error('Expected defaults mapping and nonempty boards list')
                    }
                    if (!(config.child_job instanceof String) || !config.child_job.trim()) {
                        error('child_job is required')
                    }
                    childJob = config.child_job.trim()
                    def strings = ['SCRIPT_IMPL','PRODUCT','IMAGE_TYPE','VERSION','BOARD_ID','BOARD_IP',
                        'SERVER_IP','GATEWAY_IP','NETMASK','BOOT_PARTITION','ROOT_PARTITION',
                        'KERNEL_PATH','DTB_PATH','BOARD_BOOT_MOUNT']
                    def booleans = ['PUBLISH_IF_MISSING','REBOOT_BOARD','HEALTH_CHECK']
                    def ids = []
                    def ips = []
                    config.boards.each { board ->
                        if (!(board instanceof Map)) { error('Every board must be a mapping') }
                        def merged = config.defaults + board
                        ['PRODUCT','IMAGE_TYPE','VERSION','BOARD_ID','BOARD_IP'].each { name ->
                            if (!merged[name]?.toString()?.trim()) { error("Missing ${name}") }
                        }
                        def id = merged.BOARD_ID.toString().trim()
                        def ip = merged.BOARD_IP.toString().trim()
                        if (!(id ==~ /^[A-Za-z0-9][A-Za-z0-9._-]*$/) || id in ['.', '..']) {
                            error("Invalid BOARD_ID: ${id}")
                        }
                        if (id in ids || ip in ips) { error("Duplicate board ID or IP: ${id}") }
                        ids.add(id)
                        ips.add(ip)
                        def inputs = []
                        merged.each { name, value ->
                            if (value == null) { error("Null value for ${name}") }
                            if (name in booleans) {
                                if (!(value instanceof Boolean)) { error("${name} must be true or false") }
                                inputs.add(booleanParam(name: name, value: value))
                            } else if (name in strings) {
                                inputs.add(string(name: name, value: value.toString()))
                            } else {
                                error("Unknown parameter: ${name}")
                            }
                        }
                        deployments.add([id: id, inputs: inputs])
                    }
                    echo "Launching ${deployments.size()} boards using ${childJob}"
                }
            }
        }
        stage('NetBoot fleet') {
            steps {
                script {
                    def branches = [:]
                    deployments.each { deployment ->
                        def id = deployment.id
                        def inputs = deployment.inputs
                        branches[id] = {
                            stage("NetBoot ${id}") {
                                build(job: childJob, parameters: inputs,
                                    wait: true, propagate: true, quietPeriod: 0)
                            }
                        }
                    }
                    parallel branches
                }
            }
        }
    }
    post {
        success { echo 'All boards completed successfully.' }
        failure { echo 'Fleet deployment failed. Inspect the child builds.' }
    }
}
