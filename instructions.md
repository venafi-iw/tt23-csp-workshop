# Q2 2023 Tech Training - Automating Code Signing with Enterprise Pipelines

This guide is specifically written to be used within the Venafi AWS CloudFormation POC/Demo environment.  your lab environment has been provisioned specifically for your use and is not shared with other people/organizations.

Your environment is here:

TPP instance: `tt23-<username>-tpp.se.venafi.com`

DevOps/CodeSigning Ubuntu instance: `tt23-<username>-devopslab.se.venafi.com`

Windows instance: `tt23-<username>-vsatworker.se.venafi.com`

For signing on the DevOps/CodeSigning Ubuntu and the VSAT Windows worker instance we will use the following account:

User: `venafilab`

Password: `Ven@fiPassw0rd`

This guide walks through automating code signing as part of several Jenkins pipelines using a CodeSign Protect Certificate environment.  We’ll walk through specific use cases around signing Jar files, a Windows executable, and finally sign a container image using the Sigstore cosign tool.

This guide assumes you have a working knowledge of Docker and Jenkins.

Jenkins is already installed and configured on the DevOps/CodeSigning Ubuntu instance with the same credentials as above.

Please use the following credentials for the TPP platform:

User: administrator
Password: Ven@fiPassw0rd

:blue_book: Instructions
Jenkins Configuration
Launch a browser tab and connect to https://tt23-<username>-devopslab.se.venafi.com:8002. Ignore any certificate errors/warnings since this is a demo environment.

Install the Venafi CodeSign Protect plugin via Dashboard → Manage Jenkins -> Plugin Manager -> Available Plugins → Venafi CodeSign Protect. Click on Download now and install after restart. You may also have to check the Restart Jenkins when installation is complete and no jobs are running option.



Configure Venafi CodeSign Protect plugin via Manage Jenkins → Configure System 

Go to Venafi Code Signing section, then click on Add button under Venafi TPPs:

Enter TPP Name → tpp

Enter Authorization URL → https://tt23-<username>-tpp.se.venafi.com/vedauth

Enter HSM URL → https://tt23-<username>-tpp.se.venafi.com/vedhsm 

Click on the Save button at the bottom.

Install the Role Strategy plugin via Manage Jenkins → Plugin Manager → Available Plugins → Role-based Authorization Strategy.  Click on Download now and install after restart .  You may also have to check the Restart Jenkins when installation is complete and no jobs are running option.



Create Folder via Dashboard → New Item → Folder

Provide a name for the folder → App Team 1

Optionally enter Display Name such as App Team 1 and then Save

Create a domain via The App Team 1 folder

Click on Credentials

Click on Stores scoped to App Team 1

Click on Add Domain and enter a sample domain such as venafilab.com, then Save

Create a Username/Password credential to represent the user account that will be signing

Click on the domain that you just created

Click on + Add Credentials

Make sure Kind is Username and password

Username = sample-cs-user

Password = NewPassw0rd! 

Note the ID, as we will need this when creating the groovy pipeline

Install Jenkins Agent on Ubuntu:
Let’s first start by setting up the Jenkins agent in your environment.  This will be used to sign a sample Jar file.

Browse to https://tt23-<username>-devopslab.se.venafi.com:8002/ and authenticate using the credential described above

Go To Dashboard -> Manage Jenkins -> Manage Nodes and Clouds and select New Node

Give it a name: linux_node and set it as a Permanent Agent

Set Remote root directory to /home/venafilab Set Labels to linux_node and Save

On the Nodes page select the agent you just created

SSH into the DevOps/Ubuntu instance using the credential provided above.  Follow the steps outlined in the Run from agent command line, with some slight modifications such as adding the -noCertificateCheck argument to the java command line.  Here is an example:


curl -sO -k https://tt23-<username>-devopslab.se.venafi.com:8002/jnlpJars/agent.jar
java -jar agent.jar -noCertificateCheck -jnlpUrl https://tt23-<username>-devopslab.se.venafi.com:8002/manage/computer/linux%5Fnode/jenkins-agent.jnlp -secret 192a683fd4bf5ad5017a1216696a8670935d446a389207a70762bdf0d59388f2 -workDir "/home/venafilab"
May 19, 2023 6:19:05 AM hudson.remoting.jnlp.Main$CuiListener status

INFO: Remote identity confirmed: 74:5f:ad:f1:08:ff:4f:4e:f9:9e:b2:77:a6:99:71:09

May 19, 2023 6:19:05 AM hudson.remoting.jnlp.Main$CuiListener status

INFO: Connected

Install Jenkins Agent on Windows:
Since we will be signing a sample executable we’ll need to have a Windows environment for the Jenkins pipeline to execute the signtool command:

Via the Jenkins Web Admin console Go To Dashboard -> Manage Jenkins -> Nodes and select New Node

Give it a name: windows_node and set it as a Permanent Agent

Set Remote root directory to c:\temp and Set Labels to windows_node and Save

On the Nodes page select the agent you just created

RDP into the VSAT Windows worker instance (tt23-<username>-vsatworker.se.venafi.com) using the credential provided above.  Follow the steps outlined in the Run from agent command line, with some slight modifications such as adding the -noCertificateCheck argument to the java command line.  Here is an example:


curl -sO -k https://tt23-<username>-devopslab.se.venafi.com:8002/jnlpJars/agent.jar
java -jar agent.jar -noCertificateCheck -jnlpUrl https://<customer>-devopslab.se.venafi.com:8002/manage/computer/linux%5Fnode/jenkins-agent.jnlp -secret 192a683fd4bf5ad5017a1216696a8670935d446a389207a70762bdf0d59388f2 -workDir "/home/venafilab"
May 19, 2023 6:19:05 AM hudson.remoting.jnlp.Main$CuiListener status

INFO: Remote identity confirmed: 74:5f:ad:f1:08:ff:4f:4e:f9:9e:b2:77:a6:99:71:09

May 19, 2023 6:19:05 AM hudson.remoting.jnlp.Main$CuiListener status

INFO: Connected

Install Venafi CodeSign Protect Clients on Jenkins Agent Systems:
Now install the Venafi CodeSign Protect client for Ubuntu using the following commands.  Make sure to replace the <customer> with your assigned POC customer name.


curl -o "venafi-codesigningclients-23.1.0-linux-x86_64.deb" https://tt23-<username>-tpp.se.venafi.com/csc/clients/venafi-csc-latest-x86_64.deb
sudo dpkg -i "venafi-codesigningclients-23.1.0-linux-x86_64.deb"
Install the Venafi CodeSign Protect client for Windows if it is not already available on the Windows node:
Make sure to run with elevated privileges (e.g. run as administrator )


curl -o "VenafiCodeSigningClients-23.1.0-x64.msi" ^https://tt23-<username>-tpp.se.venafi.com/csc/clients/venafi-csc-latest-x86_64.msi
start /wait msiexec /qn /i "VenafiCodeSigningClients-23.1.0-x64.msi"
Configure Jenkins Pipeline for Windows Signing:
Now that we have the Venafi CodeSign Protect client installed as well as the Jenkins agent up and running, let’s configure a Jenkins pipeline to leverage signtool to sign a sample executable.

Within the Folder you created earlier, Select + New Item

Provide the item name such as signtool-test and select Pipeline

Enter the pipeline script as follows:



node {
    agent 'windows_node'
    venafiCodeSignWithSignTool tppName: 'tpp',
    fileOrGlob: 'c:\\temp\\sample.exe',
    subjectName: 'Sample Code Signers Are Us, LLC',
    credential: [credentialsId: '<credential_id_goes_here>'],
    timestampingServers: [[address: 'http://timestamp.digicert.com']]
}
 

Run the job by clicking on Build Now

Assuming a successful run your executable should now be signed.  You can view the output of the run by clicking on the job # in the Build History.

Configure Jenkins Pipeline for Jar Signing
Now that we have the Venafi CodeSign Protect client installed as well as the Jenkins agent up and running, let’s configure a Jenkins pipeline to leverage jarsigner to sign a sample jar file.

Within the Folder you created earlier, Select + New Item

Provide the item name such as jarsigner-test and select Pipeline

Enter the pipeline script as follows:


pipeline {
    agent { label 'windows_node' }
    stages {
        stage("Venafi") {
            steps {
                venafiCodeSignWithJarSigner tppName: 'tpp',
                file: 'c:\\temp\\test.jar',
                certLabel: 'Sample-Development-Environment',
                credential: [credentialsId: '<credential_id_goes_here>'],
                timestampingServers: [[address: 'http://timestamp.digicert.com']]
            }
        
        }
    }
}
 

Run the job by clicking on Build Now

Assuming a successful run your jar file should now be signed.  You can view the output of the run by clicking on the job # in the Build History.

Configure Jenkins Pipeline for Signing a Container Image
In this part of the lab we will now seup a local stateless Docker-based registry so that we can store as well as distribute signed and unsigned container images that we’ve obtained from Docker hub.  From there we’ll run the cosign tool to sign and publish the signed container image tag.

Within the Folder you created earlier, Select the + New Item

Provide the item name such as cosign-test and select Pipeline

Enter the pipeline script as follows (make sure to modify URLs specific to your environment):


pipeline {
    agent { label 'linux_node' }
    environment {
        CSP_HSM_URL = 'https://tt23-<username>-tpp.se.venafi.com/vedhsm'
        CSP_AUTH_URL = 'https://tt23-<username>-tpp.se.venafi.com/vedauth'
    }
    stages {
        stage("Venafi") {
            steps {
                withCredentials([usernamePassword(credentialsId: '<credential_id_goes_here>', 
                                                  usernameVariable: "USERNAME", passwordVariable: "PASSWORD")]) {
                    sh '/opt/venafi/codesign/bin/pkcs11config getgrant --force --authurl=$CSP_AUTH_URL --hsmurl=$CSP_HSM_URL --username=$USERNAME --password=$PASSWORD'
                    sh 'docker run -d -p 5000:5000 --name registry registry:2'
                    sh 'docker pull alpine:latest'
                    sh 'docker image tag alpine:latest localhost:5000/alpine:signed'
                    sh 'docker image push localhost:5000/alpine:signed'
                    sh 'cosign sign --tlog-upload=false --key "pkcs11:slot-id=0;object=Sample-Development-Environment?module-path=/opt/venafi/codesign/lib/venafipkcs11.so&pin-value=1234" localhost:5000/alpine:signed'
                    sh 'docker container rm registry --force'
                }
            }
        
        }
    }
}
Run the job by clicking on Build Now

Assuming a successful run your container image should be signed.  You can view the output of the run by clicking on the job # in the Build History
