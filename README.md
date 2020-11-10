# Deployment Procedure

- [Deployment Procedure](#deployment-procedure)
  * [Requirements](#requirements)
  * [Deployment config generation](#deployment-config-generation)
  * [Deploy Fabric-services](#deploy-fabric-services)
  * [Nginxplus Internal Loadbalancer](#nginxplus-internal-loadbalancer)
  * [Cronicle-Master Deployment](#cronicle-master-deployment)
    + [Reference](#reference)
    + [Setup](#setup)
  * [Keys service Deployment](#keys-service-deployment)
  * [Gateway Service Deployment](#gateway-service-deployment)
  * [Catalog service deployment](#catalog-service-deployment)
  * [Fulfillment service deployment](#fulfillment-service-deployment)
  * [Pricing Engine](#pricing-engine)
  * [Notifier](#notifier)
  * [Microservices](#microservices)
    + [notifier-pol](#notifier-pol)
    + [mailer-pol](#mailer-pol)
    + [partner-api-pol](#partner-api-pol)
    + [shopping-carts-pol](#shopping-carts-pol)
    + [payment-service-pol](#payment-service-pol)
    + [configurator-service-pol](#configurator-service-pol)
    + [pricing-cron-pol](#pricing-cron-pol)
    + [pwa-pol service](#pwa-pol-service)
    + [ops-audit-pol service](#ops-audit-pol-service)

## Overall Step

![](./docs/imgs/MicroservicesDeployment.png)

## Requirements

Kubernetes namespace is needed first

## Kubernetes tenant label

- Tenant label check

        $ kubectl get nodes -o=json|jq -r '.items[] | .metadata| .name as $name | "\($name) \(.labels.vlan)"'
        ip-10-111-10-41.ec2.internal null  null null null null
        ip-10-111-10-42.ec2.internal null  null null null null
        ip-10-111-10-43.ec2.internal null  null null null null
        ip-10-111-20-41.ec2.internal pol  pol lva null 20
        ip-10-111-20-42.ec2.internal null  pol lva null 20
        ip-10-111-30-156.ec2.internal pol  pol lva worker worker
        ip-10-111-30-157.ec2.internal pol  pol lva worker worker
        ip-10-111-30-231.ec2.internal pol  pol lva null data
        ip-10-111-30-41.ec2.internal pol  pol lva null 30
        ip-10-111-30-42.ec2.internal pol  pol lva null 30
        ip-10-111-55-11.ec2.internal pol  pol lva null 55
        ip-10-111-55-12.ec2.internal pol  pol lva null 55

- Tenant labeling example

        $ kubectl label nodes ip-10-111-20-41.ec2.internal tenant_ltu=ltu
        node "ip-10-111-20-41.ec2.internal" labeled

- Tenant automatic labeling

        TENANTS="est ltu pol gbr lva"
        for TENANT in $TENANTS
        do
            NODES=`kubectl get nodes -o=json|jq -r '.items[] | .metadata| .name as $name | "\($name) \(.labels.vlan)"'|grep -v null|awk '{print $1}'`
            for NODE in $NODES; do kubectl label nodes $NODE tenant_${TENANT}=${TENANT}; done
        done



## Deployment config generation

- kubernetes chart values should be generated first

- deprecated) Files will be generated at subdirectory

        cd  ~/devops-global-eu/deploy/config/template/values/env
        export REGION=eu
        export TENANT=est
        export COUNTRY=ee
        export envs="dev qa1 stg prod"
        for env in $envs
        do
            export ENV=$env
            bash ./run.sh
        done

- copy poland config and change value

        export TENANT=est
        cd devops-global-eu/deploy/config/svk/values/prod
        sed -i 's/pol/svk/g' ./*
        cd ../stg
        sed -i 's/pol/svk/g' ./*
        cd ../qa1
        sed -i 's/pol/svk/g' ./*


## Deploy Fabric-services

- Fabric image is used from global. So don't need to build image

    **Fabric image has fixed endpoint with cockroachdb**

- **boot-config-[tenant]** repo is needed also to deploy other microservices with **Fabric**


## Nginxplus Internal Loadbalancer

- other services connect fabric with nginxplus endpoint

> To continuously update we need different approach

- create config

        cd deploy/helm/nginx-plus/files/templates

        TENANTS="gbr"
        files="dev.conf prod.conf qa1.conf stg.conf"
        for tenant in $TENANTS
        do
            export TENANT=$tenant
            mkdir $tenant
            for file in $files
            do
                erb tenant-${file}.erb > ${tenant}/${tenant}-${file}
            done
            mv $tenant ../
        done


- apply

        cd deploy/helm/nginx-plus  
        tenant=rou
        env=qa1
        helm template . -f values.yaml --set service.name=${tenant}-${env}-nginx \
        --set service.nginxFile=files/${tenant}/${tenant}-${env}.conf \
        --set service.tenant=${tenant}| kubectl apply -n ${tenant}-${env} -f -

- delete

        cd deploy/helm/nginx-plus 
        tenant=pol
        env=qa1
        helm template . -f values.yaml --set service.name=${tenant}-${env}-nginx \
        --set service.nginxFile=files/${tenant}/${tenant}-${env}.conf \
        --set service.tenant=${tenant}| kubectl delete -n ${tenant}-${env} -f -


## K8S Tenant Dependency configuration

- ngw

    backend system connect

    tenants value should be udpated at defaults directory

- prx

    backend system connect

    tenants value should be udpated at defaults directory

- k8s

    tenants configuration to create cronicle directory

    tenants value should be udpated at defaults directory

## Cronicle-Master Deployment

### Reference

https://confluence.samsungmtv.com/display/EC/Cronicle+Setup

### Setup

- host node label setup

- cronicle

```
cd deploy/helm/micro-service
tenant=est
env=prod
helm template . -f ../../config/${tenant}/values/${env}/cronicle.yml \
--set microService.image.tag=v1.1.6 \
--set microService.autoscale.Maxreplicas=1 \
--set microService.autoscale.Cputhreshold=100| kubectl apply -n ${tenant}-${env} -f -
```

## Seed data

When deploying a service for the first time in an environment, please execute the queries under /seed-data/{tenant}/ as part of the deployment. This has to be done only the first time as this is not repeatable like db-scripts. If you could document this, it would be great.


Choonghwan Ryu [3 hours ago]
Thank you for guiding us, and please could you tell me which services need seeding data?


Karthik P [1 hour ago]
We would need it for Payment, Orders, Shopping Cart, exchange


Prasanna Kumar [12 minutes ago]
keys, sso, epp as well

## Keys service Deployment

- Guide 

    config.json 

    Have the tenantconfigs and service configs in cockroach

    Keys Database created on postgres secure node

    .sql run from the repo to create the relations.

    All the key-pairs stored in keys.keys table were re-created for each Env.

    appids rows at pgs should be generated

- cdb tenantsconfig.configs

- cdb pol_qa1_configs.configs 

    - example

            # pol_keys.appids
            insert into appids (id, data) values ('c8e753b3-cedb-43c2-880c-b4cfc78fa58e','{"app_id": "order-service-pol", "app_secret": "order-service-global-secret-code"}');
            insert into appids (id, data) values ('8dc9c4e9-23fa-4e85-9b33-10036b8869c4','{"app_id": "catalog-service", "app_secret": "catalog-service-secret-code"}');
            insert into appids (id, data) values ('c25eb0f0-8350-41b8-abe8-57052fd2c0a5','{"app_id": "shopping-carts-pol", "app_secret": "shopping-carts-ind-service-secret-code"}');
            insert into appids (id, data) values ('04d49af0-c6a5-4120-b638-9331d7c0dd91','{"app_id": "payment-service-pol", "app_secret": "payment-service-ind-secret-code"}');
            insert into appids (id, data) values ('f3716540-c725-11e8-a8d5-f2801f1b9fd1','{"app_id": "sso", "app_secret": "sso-service-ind-secret-code"}');
            insert into appids (id, data) values ('1d165a81-0c4b-480d-bbfa-c649a0624c14','{"app_id": "billing-record-pol", "app_secret": "billing-record-ind-secret-code"}');
            insert into appids (id, data) values ('520e01ea-be39-4c16-9720-39b56994de58', '{"app_id": "gateway-pol", "app_secret": "gateway-secret-code"}');
            insert into appids (id, data) values ('c25bb0f0-8350-41b8-abe8-57052fd2c0a5', '{"app_id": "keys-service-pol", "app_secret": "keys-service-secret-code"}');

---
## Jenkins cli to build job

        #login to jenkins master as jenkins user
        cd /var/cache/jenkins/war/WEB-INF
        export JENKINS_URL=http://127.0.0.1:8080

        # list job
        java -jar jenkins-cli.jar list-jobs
        
        # run job
        REGION=eu
        TENANT=ltu
        ENV=prod
        JOB=ecom-service-deploy
        SERVICE=mailer
        TAG=v4.0.2
        java -jar jenkins-cli.jar build $JOB -p region=$REGION -p tenant=$TENANT -p namespace=$ENV -p service=$SERVICE -p tag=$TAG -v

---
## Gateway Service Deployment

- Check cockroachdb

    insert into configs (id,tenant_id,data) values 
    ('4cfac176-0b4d-4600-adb2-9d0e8aa86e00', 'POL', '{"app_config": {"appId": "gateway-pol", "keys_auth_secret": "gateway-secret-code"}, "schema_version": "v4.0.0", "service_name": "gateway-pol"}');

- Deploy

- Check status

    ![](./docs/imgs/CropperCapture[14].png)


---
## Catalog service deployment

- Catalog worker pod is using cronicle

- Check label **role=worker** at k8s wrk nodes 

    ![](./docs/imgs/CropperCapture[13].png)


---
## Fulfillment service deployment

- Cockroachdb's tenant configs are needed

    ![](./docs/imgs/CropperCapture[12].png)


- Rabbitmq connection

        root@qa-rmq-01  -= EU =- /var/lib/rabbitmq/mnesia/rabbit@qa-rmq-01 # rabbitmqadmin list connections
        +-------------------------------------------+---------+----------+
        |                   name                    |  user   | channels |
        +-------------------------------------------+---------+----------+
        | 10.111.30.156:43548 -> 10.111.30.141:5672 | qa_user | 1        |
        | 10.111.30.156:44468 -> 10.111.30.141:5672 | qa_user | 1        |
        | 10.111.30.156:44798 -> 10.111.30.141:5672 | qa_user | 1        |
        | 10.111.30.156:46498 -> 10.111.30.141:5672 | qa_user | 1        |
        | 10.111.30.156:47402 -> 10.111.30.141:5672 | qa_user | 1        |
        | 10.111.30.156:56700 -> 10.111.30.141:5672 | qa_user | 1        |
        | 10.111.30.156:58300 -> 10.111.30.141:5672 | qa_user | 1        |
        | 10.111.30.156:58990 -> 10.111.30.141:5672 | qa_user | 1        |
        | 10.111.30.42:33874 -> 10.111.30.141:5672  | qa_user | 0        |
        +-------------------------------------------+---------+----------+

- pm2 name is needed at deploy config

    ![](./docs/imgs/fulfillment.png)

---
## Pricing Engine


---
## Notifier

1. put config data to [tenant]_qa1_configs.configs

2. config.yml naming check

pol-qa1/qa1-fabric-service-9b4544c94-vwk5c[qa1-fabric-service]: 03:01:14 0|fabric-s | 181017/030113.947, (1539745273947:qa1-fabric-service-9b4544c94-vwk5c:19:jn5hjzqw:71913) [response,api,fabric] http://qa1-fabric-service-9b4544c94-vwk5c:5099: get /v1/_fabric/tenants/POL/services/notifier/service-config {} 404 (53ms)

3. package.json check

---
## Microservices

1. put config data to [tenant]_qa1_configs.configs

2. check config.yml

3. check package.json

4. check log with kail

5. status check at rancher or k8s

### notifier-pol

```
 tenantconfig is added
 config.yml is updated
 package.json is updated (dev should check)
 notification.js is updated
 
 messenger-lib-pol repo is not existing
 
-    "mailer-client-lib": "git+ssh://git@bitbucket.org:seaecom/mailer-client-lib.git",
-    "messenger-lib": "git+ssh://git@bitbucket.org:seaecom/messenger-lib.git",
-    "notifier-lib": "git+ssh://git@bitbucket.org/seaecom/notifier-lib.git",
+    "mailer-client-lib-pol": "git+ssh://git@bitbucket.org:seaecom/mailer-client-lib-pol.git",
+    "messenger-lib-pol": "git+ssh://git@bitbucket.org:seaecom/messenger-lib-pol.git",
+    "notifier-lib-pol": "git+ssh://git@bitbucket.org/seaecom/notifier-lib-pol.git",
```

### mailer-pol

```
    - Fixes
    -- app.js
        : fixed error in service name when checking health (publicRoutePrefix, privateRoutePrefix)
    -- messaging-worker.js
        : modify workerName messaging-worker to mailer-worker (13 line)

    - Current Status : deploy done (v1.44.6)
```

- mail send s3 template bucket

    I created s3 bucket and iam user for email images in eu-prod. (I sent iam access and secret key through slack dm.)

    Please check those resource and give me feedback if you have any problem.

```
[S3 Bucket]

*s3 bucket name: estore-svc-eu-email-images
*s3 bucket policy

{
    "Version": "2012-10-17",
    "Id": "Policy1535666059112",
    "Statement": [
        {
            "Sid": "Stmt1535665978559",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::estore-svc-eu-email-images/*"
        },
        {
            "Sid": "Stmt1535666057532",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::522741338650:user/s3-write-eu"
            },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::estore-svc-eu-email-images/*"
        }
    ]
}
 
[IAM]
* iam user: s3-write-eu
* attached policy

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:ListAllMyBuckets",
                "s3:HeadBucket"
            ],
            "Resource": "*"
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": "arn:aws:s3:::estore-svc-eu-email-images"
        }
    ]
}
```

### partner-api-pol

```
-> updated
 config.yml is updated
-> status (deploy fail) 
-> issue (dev should check, please compare package.json with india)
06:05:19 0|partner- | Error: Cannot find module './order-shipping-status-convertor'
06:05:19 0|partner- |     at Function.Module._resolveFilename (module.js:547:15)
06:05:19 0|partner- |     at Function.Module._load (module.js:474:25)
06:05:19 0|partner- |     at Module.require (module.js:596:17)
06:05:19 0|partner- |     at require (internal/module.js:11:18)
06:05:19 0|partner- |     at Object.<anonymous> (/opt/services/dev/partner-api/node_modules/fulfillment-adapter-plugin-iterra/lib/message-convertors/message-convertor-factory.js:4:34)
```

### shopping-carts-pol

```
-> updated
 config.yml bug fix
-> status (deployed)
```

### payment-service-pol

```
-> updated
 tenantconfig is added
 config.yml is updated
-> status (build fail)
 tenantconfig includes payment auth information and needed to check by dev
-> issue
Could not install from "../payment-adapter-lib-pol" as it does not contain a package.json file
```

- payment worker add

```
{
            "name": "payment-service-pol",
            "server": {
                "instances": [
                    {
                        "label": "public",
                        "port": "4975"
                    },
                    {
                        "label": "private",
                        "port": "5075"
                    }
                ],
                "logging_file_path": "/var/log/services/dev/",
                "worker": {
                    "messaging-worker": {
                        "http": {
                            "enabled": true,
                            "port": 4500
                        }
                    }
                }
            }
        },
```

### order-service



- Order service messaging worker issue

LTU's messaging worker memory is increased from 2Gi to 4Gi 

Also with requests 64mi to 2Gi. and message worker worker successfully.


### configurator-service-pol

```
-> updated
 tenantconfig is added (keys and secret should be udpated, default currency config is needed)
-> status (build fail, no repo, dev check is required)
npm ERR! /usr/bin/git ls-remote -h -t ssh://git@bitbucket.org/seaecom/epp-lib-pol.git
```

### pricing-cron-pol

```
-> updated
 config/qa1.json
 credential/qa1.json (detailed credential check is needed)
-> status (deployed)
```

### ops-gateway

- tenant config has domain info whith opstools

Also, since the domain is changed we have to add a new key in tenant config of ops-gateway.

```{
"domain": ".seasam.org",
"ecom-ops-gateway": {...}
"name": "ops-gateway-ind",
"server": {...}
}```

- Domain policy is like below

    opstools-pl.seasam.org

    opstools-qa-pl.seasam.org

    opstools-stage-pl.seasam.org

- Database

    opstool database setup is also needed to login
    


### ops-audit-pol service

```
 - Fixes
   -- create config.yml
   -- app.js
      : fixed error in service name when checking health (publicRoutePrefix, privateRoutePrefix)

- Current status : deploy done (v2.0.2)
```

### pwa-pol service

```
ERROR in ./src/assets/images/sp-footer-ie8.png
Module build failed: Error: Cannot find module 'file-loader'
    at Function.Module._resolveFilename (module.js:547:15)
    at Function.Module._load (module.js:474:25)
    at Module.require (module.js:596:17)
    at require (internal/module.js:11:18)
    at Object.loader (/var/lib/jenkins/workspace/ecom-service-build/pwa/node_modules/url-loader/dist/index.js:59:20)
 @ ./node_modules/css-loader?{"minimize":true,"sourceMap":true}!./node_modules/vue-loader/lib/style-compiler?{"optionsId":"0","vue":true,"scoped":false,"sourceMap":false}!./node_modules/sass-loader/lib/loader.js?{"sourceMap":true}!./node_modules/vue-loader/lib/selector.js?type=styles&index=0!./src/shared/components/B2eLogin.vue 8:10118-10160 8:10277-10319 8:10438-10480 8:10612-10654 8:10792-10834 8:11056-11098 8:11223-11265 8:13405-13447 8:15823-15865 8:15977-16019 8:16127-16169
 @ ./node_modules/extract-text-webpack-plugin/dist/loader.js?{"omit":1,"remove":true}!./node_modules/vue-style-loader!./node_modules/css-loader?{"minimize":true,"sourceMap":true}!./node_modules/vue-loader/lib/style-compiler?{"optionsId":"0","vue":true,"scoped":false,"sourceMap":false}!./node_modules/sass-loader/lib/loader.js?{"sourceMap":true}!./node_modules/vue-loader/lib/selector.js?type=styles&index=0!./src/shared/components/B2eLogin.vue
 @ ./src/shared/components/B2eLogin.vue
 @ ./src/router/index.js
 @ ./src/app.js
 @ ./src/entry-server.js
 @ multi ./src/entry-server.js
```


### identity store poland

https://bitbucket.org/seaecom/identity-store-pol/commits/all

2 sources are fixed. 

    - config.yml

    - package.json


### mpos-gateway

