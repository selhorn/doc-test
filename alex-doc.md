# Auto scaling the BIG-IP VE Web Application Firewall in AWS
[![Slack Status](https://f5cloudsolutions.herokuapp.com/badge.svg)](https://f5cloudsolutions.herokuapp.com)

## Introduction
This solution implements auto scaling of BIG-IP Virtual Edition Web Application Firewall (WAF) systems in Amazon Web Services. The BIG-IP Virtual Editions have the <a href="https://f5.com/products/big-ip/local-traffic-manager-ltm">Local Traffic Manager</a> (LTM) and <a href="https://f5.com/products/big-ip/application-security-manager-asm">Application Security Manager</a> (ASM) modules enabled to provide advanced traffic management and web application security functionality.  As traffic increases or decreases, the number of BIG-IP VE WAF instances automatically increases or decreases accordingly. 

Maybe some more elaborate solution overview/value prop here...?


## BIG-IP deployment and configuration

To launch this solution, you will use a <a href="https://aws.amazon.com/cloudformation/aws-cloudformation-templates/">Cloudformation Template</a>. The template is used to collect some initial deployment input parameters and create an autoscale group of BIG-IP Virtual Editions. The instances parameters and configurations are defined by the Autoscale group's *launch configuration*. The launch configuration is used to:

  - Set the BIG-IP system information: hostname, NTP, DNS settings, etc.
  - Provision the WAF module - Application Security Manager (ASM)
  - Join the autoscaled cluster
  - Deploy integration with EC2 Auto Scale and CloudWatch services for scaling of the BIG-IP tier.
  - Create an initial HTTP virtual server with a basic Web Application Firewall policy (Low, Medium, High)
    - See the [Security Blocking Levels](#blocking) section for a description of the blocking levels for the Web Application Firewall presented in the template.

Once the first instance is deployed, it becomes the cluster primary and all subsequent instances launched will join a cluster primary to pull the latest configuration from the cluster. In this respect, you can make changes to the running configuration of this solution and not have to manage the lifecycle of the config strictly through the "Launch Configuration".


## Configuration Example <a name="config"></a>

The following is a simple configuration diagram deployment. 

![Single NIC configuration example](images/config-diagram-autoscale-waf.png)


## Prerequisites Summary:
The following are prerequisites for this solution:

  - Make sure that you have accepted the EULA for all BIG-IP images used in the AWS marketplace.
  - An existing AWS VPC with Internet Gateway, an ELB in front of the BIG-IP(s), and a DNS name for the application pool (which can be also be the DNS name of an ELB if using one behind the BIG-IP(s))
  - Permissions to launch Cloudformation templates which creates the following resources:
      - Autoscale Groups
      - S3 Buckets
      - Instances
      - IAM Instance Profiles
  - An AWS Security Group for allowing access the BIG-IP management and Virtual Service
  - Key pair for SSH access to BIG-IP VE (you can create or import in AWS)
      - This solution uses the SSH key to enable access to the BIG-IP system(s). To access the BIG-IP web-based Configuration utility, you must first SSH into the BIG-IP VE using the SSH key you provided in the template.  You can then create a custom user account with admin-level permissions on the BIG-IP VE to allow GUI access.


## Prerequisit Steps: 

1) Create a VPC with subnet for the BIG-IPs WAF tier (see diagram). The Marketplace template requires an Internet Gateway and assigning Public IPs to each of the autoscaled instances to access the Internet ( for licensing purposes and to download additional packages ). 

  - If creating an Internet Gateway and assigning a Public IP to allow Intenet access is not allowed in your environment but you can provide access through other means (ex. NAT instance, NAT Gateway, Proxy), you will need to customize the template to remove the

  ```
    "AssociatePublicIpAddress": true,
  ```

  line and [launch manually](#manual-launch). 

2) Create a Security Group for the BIG-IPs with the following rules:

  - Management Access:

     - **TCP Port 22** for SSH access to the BIG-IP VE  *(Source = Intra-VPC and/or mgmt networks)*
     - **TCP Port 8443**  for accessing the BIG-IP web-based Configuration utility  *(Source = Intra-VPC and/or mgmt networks)*

  - Cluster Config Sync:

     - **TCP Port 4353**  for cluster communication *(Source = Intra-VPC or Peer’s Public subnet )*
     - **TCP Ports 6123-6128**  for cluster communication *(Source = Intra-VPC or Peer’s Public subnet )*

  - WAF Traffic:
  
     - **TCP Port 80** for accessing your applications via the BIG-IP virtual server *(Source = Any)*


3) The provided solution requires a pre-configured ElasticLoadbalancer that performs SSL offload for the BIG-IP WAF autoscaled tier. 

http://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/load-balancer-getting-started.html


ex. of ELB configuration used in example deployment

```
    "bigipElasticLoadBalancer": {
      "Type": "AWS::ElasticLoadBalancing::LoadBalancer",
      "DependsOn": "internetGatewayAttachment",
      "Properties": {
        "LoadBalancerName": "<YOUR-ELB-NAME>",
        "HealthCheck": {
          "HealthyThreshold": "2",
          "Interval": "10",
          "Target": "HTTP:80/",
          "Timeout": "5",
          "UnhealthyThreshold": "10"
        },
        "subnets" : [ 
            { "Ref": "az1ExternalSubnet" },
            { "Ref": "az2ExternalSubnet" }
        ],
        "CrossZone" : true,
        "Listeners" : [ {
            "LoadBalancerPort" : "443",
            "InstancePort" : "80",
            "Protocol" : "HTTPS",
            "InstanceProtocol" : "HTTP",
            "SSLCertificateId" : { "Ref" : "certificateId" },
            "PolicyNames" : [
                "ELBSecurityPolicy-2016-08",
                "MyAppCookieStickinessPolicy"
            ]
        } ],
        "Policies" : [
            {
                "PolicyName" : "MyAppCookieStickinessPolicy",
                "PolicyType" : "AppCookieStickinessPolicyType",
                "Attributes" : [
                    { "Name" : "CookieName", "Value" : "MyCookie"}
                ]
            }
        ],
        "SecurityGroups": [
          {
            "Ref": "bigipSecurityGroup"
          }
        ],
        "Tags": [
          {
            "Key": "application",
            "Value": {
              "Ref": "AWS::StackId"
            }
          }
        ]
      }
    },

```
3) DNS Name & Port of the application. 

 - *ex.* a dns name of the internal elastic loadbalancer in WAF sandwich. 

4) Accept EULA

If launching this template from:

  - AWS Marketplace:

    - If launching the template from the Marketplace, you will be prompted to Accept EULA for Big-IP product and can proceed to filling out the input parameters according to [Usage instructions](#usage) below. 

  - This repository:

    * Sign in to your account
    * Go to [product page](https://aws.amazon.com/marketplace/search/results/ref=srh_navgno_search_box?page=1&searchTerms=f5+big-ip) for Big-IP selected
    * Select "Continue"
    * Hit "Accept Software Terms"
  Otherwise, the template will appear to hang, failing to deploy the first instance.



### Quick Start

Once you have completed the above pre-requisits, you can launch the template itself.

**From the Marketplace:** 

  - Select the Cloudformation Template radio button.
  - Accept EULA


**One Click:**

  - Sign in to your AWS account
  - Click "Launch Stack" button below

    - autoscale-bigip.template<br><a href="https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=autoscale-bigip.template&templateURL=https://s3.amazonaws.com/f5-downloads/dev/autoscale-bigip.template">
    <img src="https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png"/>
    </a>

**Manual:** <a name="manual-launch"></a>

  - Download the CloudFormation template from https://github.com/f5networks and use it to create a stack in AWS CloudFormation either using the AWS Console or AWS CLI

    - **AWS Console**

       From the AWS Console main page: 
         1. Under AWS Services, click **CloudFormation**.
         2. Click the **Create Stack** button 
         3. In the Choose a template area, click **Upload a template to Amazon S3**.
         4. Click **Choose File** and then browse to the **f5-autoscale-bigip.template** file.
       
       <br>

    - **AWS CLI**
   
       Fill in the parameters file f5-autoscale-bigip-parameters.json according to "Usage" section below. From the AWS CLI, use the following command syntax:
       ```
       aws cloudformation create-stack --stack-name Acme-autoscale-bigip --template-body file:///fullfilepath/f5-autoscale-bigip.template --parameters file:///fullfilepath/f5-autoscale-bigip-parameters.json --capabilities CAPABILITY_NAMED_IAM`
      ```
      <br>

### Usage ### <a name="usage"></a>

You can use and/or change the default parameter values, which are defined in the AWS CloudFormation template on the AWS Console.  


| Parameter | Required | Description |
| --- | --- | --- |
| deploymentName | x | Name the template uses to create object names |
| vpc | x | VPC where you want to deploy the BIG-IP VEs |
| availabilityZones | x | Availability Zones where you want to deploy BIG-IP VEs (we recommend at least 2) |
| subnets | x | Public or External Subnet for the Availability Zones |
| bigipSecurityGroup | x | AWS Security Group for BIG-IP VEs |
| bigipElasticLoadBalancer | x | AWS Elastic Load Balancer group for BIG-IPs |
| sshKey | x | EC2 KeyPair to enable SSH access to the BIG-IP instance |
| instanceType | x | AWS Instance Type (Default m3.2xlarge) |
| throughput | x | Maximum amount of throughput for BIG-IP VE (Default 1000Mbps) |
| adminUsername | x | BIG-IP Admin Username for clustering. Note that the user name can contain only alphanumeric characters, periods ( . ), underscores ( _ ), or hyphens ( - ). Note also that the user name cannot be any of the following: adm, apache, bin, daemon, guest, lp, mail, manager, mysql, named, nobody, ntp, operator, partition, password, pcap, postfix, radvd, root, rpc, rpm, sshd, syscheck, tomcat, uucp, or vcsa. |
| managementGuiPort | x | Port of BIG-IP management Configuration utility (Default 8443) |
| timezone | x | Olson timezone string from /usr/share/zoneinfo (Default UTC) |
| ntpServer | x | NTP server for this implementation (Default 0.pool.ntp.org) |
| scalingMinSize | x | Minimum number of BIG-IP instances (1-8) to be available in the AutoScale Group (Default 1) |
| scalingMaxSize | x | Maximum number of BIG-IP instances (2-8) that can be created in the AutoScale Group (Default 3) |
| scaleDownBytesThreshold | x | Incoming Bytes Threshold to begin scaling down BIG-IP Instances (Default 10000) |
| scaleUpBytesThreshold | x | Incoming Bytes Threshold to begin scaling up BIG-IP Instances (Default 35000) |
| notificationEmail |  | Valid email address to send Auto Scaling Event Notifications |
| virtualServicePort | x | Port on BIG-IP (Default 80) |
| applicationPort | x | Application Pool Member Port on BIG-IP (Default 80) |
| appInternalDnsName | x | DNS name for the application pool |
| policyLevel | x | WAF Policy Level to protect the application (Default high) |
| application |  | Application Tag (Default f5app) |
| environment |  | Environment Name Tag (Default f5env) |
| group |  | Group Tag (Default f5group) |
| owner |  | Owner Tag (Default f5owner) |
| costcenter |  | Cost Center Tag (Default f5costcenter) |
<br>


*NOTE:*

  - All instances deploy with a single interface (NIC) with a Public IP. This single interface processes both management and data plane traffic. In this deployment, the Management GUI port is bumped up to 8443 by default. 
  - The CloudFormation template uses the default **Best 200Mbs** image available in the AWS marketplace to license these modules.
  - Scaling Up/Down Thresholds:

    ```
    OPTIONAL (Default is set artificially low for testing. Recommended to adjust as ratio to utility size)
    ex. 80% of Throughput
    Scale Up Bytes Threshold = 
    25 Mbps   = 3276800 bytes   * .80 =   2621440
    200 Mbps  = 26214400 bytes  * .80 =  20971520
    1000 Mbps = 131072000 bytes * .80 = 104857600
    5000 Mbps = 655360000 bytes * .80 = 524288000
    ```

If using the AWS CLI, use the following JSON parameter file.

ex. minimum **autoscale-bigip-parameters.json** using default values for unlisted parameters
```json
[
  {
    "ParameterKey":"deploymentName",
    "ParameterValue":"abc"
  },
  {
    "ParameterKey":"vpc",
    "ParameterValue":"vpc-abcd1234"
  },
  {
    "ParameterKey":"availabilityZones",
    "ParameterValue":"us-east-1a,us-east-1b"
  },
  {
    "ParameterKey":"subnets",
    "ParameterValue":"subnet-abcd1234,subnet-abcd1234"
  },
  {
    "ParameterKey":"bigipSecurityGroup",
    "ParameterValue":"sg-abcd1234"
  },
  {
    "ParameterKey":"bigipElasticLoadBalancer",
    "ParameterValue":"abc-BigipElb"
  },
  {
    "ParameterKey":"sshKey",
    "ParameterValue":"awskeypair"
  },
  {
    "ParameterKey":"notificationEmail",
    "ParameterValue":"user@company.com"
  },
  {
    "ParameterKey":"appInternalDnsName",
    "ParameterValue":"poolapp.example.com"
  },
  {
    "ParameterKey":"policyLevel",
    "ParameterValue":"high"
  }
]
```

### Logging In

FURTHER CONFIGURATION CUSTOMIZATION INSTRUCTIONS HERE...

  - Log in via SSH, add user/password.

    - Navigate the Cloudformation Output Tab. Note the name of the Autoscale Group created.
    - Navigate to the Autoscale Group Tab. Click on the Instances Tab of the Autoscale Group. Click on the Instance ID link of the first member of the Autoscale Group. 
    - Note the IP or Public IP.
    - SSH to the IP or Public IP
``` 
  >ssh -i ~/.ssh/<YOUR-PRIVATE-SSH-KEY-HERE> admin@<INSTANCE IP>
```

    - Create custom-admin user.

``` 
  #tmsh create auth user custom-user shell advanced password 

  ex.
  admin@(ip-10-0-0-17)(Standalone)(Active)(/Common)(tmos)# create auth user my-custom-admin partition-access add { all-partitions { role admin } } prompt-for-password
  changing password for my-custom-admin
  new password:
  confirm password:
  admin@(ip-10-0-0-17)(Standalone)(Active)(/Common)(tmos)#

```

  - configure logging
  - Import custom policy, 
  - etc.


### Security blocking levels <a name="blocking"></a>
The security blocking level you choose when you configure the template determines how much traffic is blocked and alerted by the F5 WAF.

Attack signatures are rules that identify attacks on a web application and its components. The WAF has at least 2600 attack signatures available. The higher the security level you choose, the more traffic that is blocked by these signatures.

| Level | Details |
| --- | --- | --- |
| Low | The fewest attack signatures enabled. There is a greater chance of possible security violations making it through to the web applications, but a lesser chance of false positives. |
| Medium | A balance between logging too many violations and too many false positives. |
| High | The most attack signatures enabled. A large number of false positives may be recorded; you must correct these alerts for your application to function correctly. |

All traffic that is not being blocked is being used by the WAF for learning. Over time, if the WAF determines that traffic is safe, it allows it through to the application. Alternately, the WAF can determine that traffic is unsafe and block it from the application.


## HOW THIS WORKS HERE


## LIFECYCLE STUFF HERE

Alex to work on (Updgrade/ Recovery). Investigation needed.

  - Make UCS backup
  - Maintainence Window
    - Update Stack using Template with new AMI IDs
    - Or Blue/Green, create another deployment and ELB


## TROUBLESHOOTING

1) SSH 
2) /var/log/cloudlibs-install.log
3) Create a qkview. Open a case with Support.

etc.

## Security
This CloudFormation template downloads helper code to configure the BIG-IP system. To verify the integrity of the template, you can open the CFT and ensure the following lines are present. See [Security Detail](#securitydetail) for the exact code in each of the following sections.
  - In the */config/verifyHash* section: **script-signature** and then a hashed signature
  - In the */config/installCloudLibs.sh* section **"tmsh load sys config merge file /config/verifyHash"**
  
Additionally, F5 provides checksums for all of our supported Amazon Web Services CloudFormation templates. For instructions and the checksums to compare against, see https://devcentral.f5.com/codeshare/checksums-for-f5-supported-cft-and-arm-templates-on-github-1014.

In order to form a cluster of devices, a secure trust must be established between BIG-IP systems. To establish this trust, we generate and store credentials in an Amazon S3 bucket. You must not delete these credentials from the S3 bucket.


## Security Details <a name="securitydetail"></a>
This section has the entire code snippets for each of the lines you should ensure are present in your template file if you want to verify the integrity of the helper code in the template.

**/config/verifyHash section**

Note the hashes and script-signature may be different in your template. It is important to check there is a script-signature line present in the location shown.<br>


```json
"/config/verifyHash": {
                "content": {
                  "Fn::Join": [
                    "\n",
                    [
                      "cli script /Common/verifyHash {",
                      "proc script::run {} {",
                      "        if {[catch {",
                      "            set hashes(f5-cloud-libs.tar.gz) a6a9db3b89bbd014413706f22fa619c3717fac41fc99ffe875589c90e9b85a05cea227c134ea6e5b519c8fee0d12f2175368e75917f31f447ece3d92f31814af",
                      "            set hashes(f5-cloud-libs-aws.tar.gz) 90058095cc536a057378a90ed19c3afe0cecd9034e1d1816745bd5ad837939623fad034ebd2ee9bdf594f33358b50c50f49a18c2ee7588ba89645142f2217330",
                      "            set hashes(asm-policy-linux.tar.gz) 63b5c2a51ca09c43bd89af3773bbab87c71a6e7f6ad9410b229b4e0a1c483d46f1a9fff39d9944041b02ee9260724027414de592e99f4c2475415323e18a72e0",
                      "            set hashes(f5.http.v1.2.0rc4.tmpl) 47c19a83ebfc7bd1e9e9c35f3424945ef8694aa437eedd17b6a387788d4db1396fefe445199b497064d76967b0d50238154190ca0bd73941298fc257df4dc034",
                      "            set hashes(f5.http.v1.2.0rc6.tmpl) 811b14bffaab5ed0365f0106bb5ce5e4ec22385655ea3ac04de2a39bd9944f51e3714619dae7ca43662c956b5212228858f0592672a2579d4a87769186e2cbfe",
                      "",
                      "            set file_path [lindex $tmsh::argv 1]",
                      "            set file_name [file tail $file_path]",
                      "",
                      "            if {![info exists hashes($file_name)]} {",
                      "                tmsh::log err \"No hash found for $file_name\"",
                      "                exit 1",
                      "            }",
                      "",
                      "            set expected_hash $hashes($file_name)",
                      "            set computed_hash [lindex [exec /usr/bin/openssl dgst -r -sha512 $file_path] 0]",
                      "            if { $expected_hash eq $computed_hash } {",
                      "                exit 0",
                      "            }",
                      "            tmsh::log err \"Hash does not match for $file_path\"",
                      "            exit 1",
                      "        }]} {",
                      "            tmsh::log err {Unexpected error in verifyHash}",
                      "            exit 1",
                      "        }",
                      "    }",
                      "    script-signature OmyfJKVQkBj+Ks6SdIc2+UNxM2xFCK4MGizGysivShzeRof0EFlEUTQiZveZ4v2SElofUp5DMVKiTIIkM00kZ7LnwqvLYIOztDFNAtMGwO6/B/zA8jLhkfnA2xzxu9fFgFn3OEsc8QwbfFS1AqCMyyacbbiczJycHtu3z0a/8sqCgiZtcQ4iXqBP4fz+8HKLA36U0jpmW+z0gQQUwpiC+AfFWcAarXMtmpwLzScldnaZ5RLo0MG8EGrHmXiWjndSR/Ii9b3+vnHnceD6+sw7e7LXPvz+jV9/rFyEQOA1QNpv0Cy4SJcuY9NRjV9KNdBobJ5N+h2PZBlgaIdLMACAVQ==",
                      "}"
                    ]
                  ]
                },
```
<br><br>
**/config/installCloudLibs.sh section**


```json
 "/config/installCloudLibs.sh": {
                "content": {
                  "Fn::Join": [
                    "\n",
                    [
                      "#!/bin/bash",
                      "echo about to execute",
                      "checks=0",
                      "while [ $checks -lt 120 ]; do echo checking mcpd",
                      "    tmsh -a show sys mcp-state field-fmt | grep -q running",
                      "    if [ $? == 0 ]; then",
                      "        echo mcpd ready",
                      "        break",
                      "    fi",
                      "    echo mcpd not ready yet",
                      "    let checks=checks+1",
                      "    sleep 10",
                      "done",
                      "echo loading verifyHash script",
                      "if ! tmsh load sys config merge file /config/verifyHash; then",
                      "    echo cannot validate signature of /config/verifyHash",
                      "    exit",
                      "fi",
                      "echo loaded verifyHash",
                      "declare -a filesToVerify=(\"/config/cloud/f5-cloud-libs.tar.gz\" \"/config/cloud/f5-cloud-libs-aws.tar.gz\")",
                      "for fileToVerify in \"${filesToVerify[@]}\"",
                      "do",
                      "    echo verifying \"$fileToVerify\"",
                      "    if ! tmsh run cli script verifyHash \"$fileToVerify\"; then",
                      "        echo \"$fileToVerify\" is not valid",
                      "        exit 1",
                      "    fi",
                      "    echo verified \"$fileToVerify\"",
                      "done",
                      "mkdir -p /config/cloud/aws/node_modules",
                      "echo expanding f5-cloud-libs.tar.gz",
                      "tar xvfz /config/cloud/f5-cloud-libs.tar.gz -C /config/cloud/aws/node_modules",
                      "tar xvfz /config/cloud/asm-policy-linux.tar.gz -C /config/cloud",
                      "cd /config/cloud/aws/node_modules/f5-cloud-libs",
                      "echo installing dependencies",
                      "npm install --production /config/cloud/f5-cloud-libs-aws.tar.gz",
                      "echo cloud libs install complete",
                      "touch /config/cloud/cloudLibsReady"
                    ]
                  ]
                },
                "mode": "000755",
                "owner": "root",
                "group": "root"
              }
```

## Filing Issues
If you find an issue, we would love to hear about it. 
You have a choice when it comes to filing issues:
  - Use the **Issues** link on the GitHub menu bar in this repository for items such as enhancement or feature requests and non-urgent bug fixes. Tell us as much as you can about what you found and how you found it.
  - Contact F5 Technical support via your typical method for more time sensitive changes and other issues requiring immediate support.

### Help 
Because this template has been created and fully tested by F5 Networks, it is fully supported by F5. This means you can get assistance if necessary from F5 Technical Support via your typical methods.

We encourage you to use our [Slack channel](https://f5cloudsolutions.herokuapp.com) for discussion and assistance on F5 CloudFormation templates.  This channel is typically monitored Monday-Friday 9-5 PST by F5 employees who will offer best-effort support. 


## Copyright

Copyright 2014-2017 F5 Networks Inc.


## License


Apache V2.0
~~~~~~~~~~~
Licensed under the Apache License, Version 2.0 (the "License"); you may not use
this file except in compliance with the License. You may obtain a copy of the
License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and limitations
under the License.

Contributor License Agreement
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Individuals or business entities who contribute to this project must have
completed and submitted the `F5 Contributor License Agreement`

