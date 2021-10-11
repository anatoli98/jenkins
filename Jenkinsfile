#!/usr/bin/env groovy

ansiColor('xterm') {
    timestamps {
        properties([
            parameters([
                string(
                    name: 'MVM_IP',
                    defaultValue: '',
                    description: 'External IPv4 address of Management VM'),
                string(
                    name: 'DEPLOYMENT_ID',
                    defaultValue: '',
                    description: 'ID of OpenStack deployment. If there is only one ' +
                                 'deployment on MVM it could be empty.'),
                credentials(
                    credentialType: 'com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl',
                    defaultValue: 'default-softi-ops-credentials',
                    description: 'Credentials of the MVM',
                    name: 'MVM_CREDENTIALS_ID',
                    required: true),
                credentials(
                    credentialType: 'com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl',
                    defaultValue: 'default-softi-ops-credentials',
                    description: 'Credentials of the OSCs and computes',
                    name: 'OSC_COMPUTE_CREDENTIALS_ID',
                    required: true),
            ])
        ])

        node('common'){
            deleteDir()
            def HOSTS_INFO = [:]

            if(DEPLOYMENT_ID == '') {
                stage ('Get deployment ID'){
                    def deployments_list = sh (script: "curl http://" + MVM_IP + ':5001/deployment',
                                               returnStdout: true).trim()
                    deployments_list = readJSON text: deployments_list
                    DEPLOYMENT_ID = deployments_list[0]
                }
            }

            stage ('Collect node information'){
                def deployment_info = sh (
                        script: "curl http://" + MVM_IP + ':5001/deployment/' + DEPLOYMENT_ID,
                        returnStdout: true).trim()
                deployment_info = readJSON text: deployment_info
                def hosts = deployment_info['context']['hosts']
                hosts.each { host ->
                    HOSTS_INFO[host['hostname']] = host['ip']
                }
            }

            stage ('Collect logs'){
                def mvm = [:]
                withCredentials([usernamePassword(credentialsId: MVM_CREDENTIALS_ID, passwordVariable: 'MVM_PASSWORD', usernameVariable: 'MVM_USER')]) {
                    mvm.name = 'MVM'
                    mvm.host = MVM_IP
                    mvm.user = MVM_USER
                    mvm.password = MVM_PASSWORD
                    mvm.allowAnyHosts = true
                    mvm.pty = true
                    mvm.knownHosts = '/dev/null'
                    mvm.timeoutSec = 30
                    mvm.retryCount = 3

                }
                withCredentials([usernamePassword(credentialsId: OSC_COMPUTE_CREDENTIALS_ID, passwordVariable: 'OSC_PASSWORD', usernameVariable: 'OSC_USER')]) {
                    HOSTS_INFO.each { hostname, host_ip ->
                        try {
                            def HOST_SSH_PREFIX = 'sshpass -p ' + OSC_PASSWORD + ' ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -tty ' + OSC_USER + '@' + host_ip + ' '

                            host_ip = (host_ip.contains(':'))?'[' + host_ip + ']':host_ip

                            sshCommand remote: mvm, command: HOST_SSH_PREFIX + '"sudo rm -rf /tmp/_data"'
                            sshCommand remote: mvm, command: HOST_SSH_PREFIX + '"sudo cp -r /var/lib/docker/volumes/kolla_logs/_data /tmp/"'
                            sshCommand remote: mvm, command: HOST_SSH_PREFIX + '"sudo tar cvfJ /tmp/kolla_logs_' + hostname + '.txz -C /tmp/ _data"'
                            sshCommand remote: mvm, command: HOST_SSH_PREFIX + '"sudo chown softi-ops:softi-ops /tmp/kolla_logs_' + hostname + '.txz"'
                            sshCommand remote: mvm, command: HOST_SSH_PREFIX + '"sudo rm -rf /tmp/_data"'
                            sshCommand remote: mvm, command: 'sshpass -p ' + OSC_PASSWORD + ' scp -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no ' + OSC_USER + '@' + host_ip + ':/tmp/kolla_logs_' + hostname + '.txz /tmp/'
                            sshGet remote: mvm, from: '/tmp/kolla_logs_' + hostname + '.txz' , into: '.', override: true
                            sshCommand remote: mvm, command: HOST_SSH_PREFIX + '"sudo cp -f /var/log/messages /tmp/_messages"'
                            sshCommand remote: mvm, command: HOST_SSH_PREFIX + '"sudo tar cvfJ /tmp/messages_' + hostname + '.txz /tmp/_messages"'
                            sshCommand remote: mvm, command: HOST_SSH_PREFIX + '"sudo chown softi-ops:softi-ops /tmp/messages_' + hostname + '.txz"'
                            sshCommand remote: mvm, command: 'sshpass -p ' + OSC_PASSWORD + ' scp -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no ' + OSC_USER + '@' + host_ip + ':/tmp/messages_' + hostname + '.txz /tmp/'
                            sshGet remote: mvm, from: '/tmp/messages_' + hostname + '.txz', into: '.', override: true

                        } catch(all) {
                            echo '\u001B[31m Unable to gather logs from node: ' + hostname + '! \u001B[0m'
                            echo '\u001B[31m Possible reason: the machine may be shut down or unreachable from MVM. \u001B[0m'
                            echo '\u001B[34m ... ignoring \u001B[0m'
                        }
                    }
                }
                sh 'tar cvfJ OpenStack_logs.txz --wildcards kolla_logs_*.txz --wildcards messages*.txz'
            }

            stage ('Archive logs'){
                archiveArtifacts allowEmptyArchive: true, artifacts: 'kolla_logs_*.txz'
                archiveArtifacts allowEmptyArchive: true, artifacts: 'messages_*.txz'
                archiveArtifacts allowEmptyArchive: true, artifacts: 'OpenStack_logs.txz'
            }
        }
    }
}