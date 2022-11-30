## Table of contents
1. [Repo rules](#repo-rules)
1. [PostgreSQL](#postgresql)
   1. [What is PostgreSQL?](#what-is-postgresql)
   1. [How it works?](#how-it-works)
1. [Crossplane Schema High Level Explanation](#crossplane-schema-high-level-explanation)
1. [Amazon RDS](#amazon-rds)
1. [Amazon WAF](#amazon-waf)
1. [Configuration & Deployment](#configuration-&-deployment)
   1. [Prerequisites](#prerequisites)
   1. [Configuration](#configuration)
   1. [Deploy](#deploy)
   1. [Delete](#delete)

## Repo rules
- acn2az dev stage for RDS/WAF
- repo admin: Surya
- try anything but ideally not on main branches
- for minor improvements use [branches](https://docs.github.com/en/get-started/quickstart/github-flow)
- for major changes or code redo use [forks](https://docs.github.com/en/get-started/quickstart/fork-a-repo) and cloning
- comment commit changes
- if you are not sure ask
---


## PostgreSQL
### What is PostgreSQL?
PostgreSQL, also known as Postgres, is a free and open-source relational database management system emphasizing extensibility and SQL compliance. It was originally named POSTGRES, referring to its origins as a successor to the Ingres database developed at the University of California, Berkeley

### How it works?
PostgreSQL uses a client-server model where the client and the server can reside on different hosts in a networked environment. The server program manages the database files, accepts connections to the database from client applications.

---
## Crossplane Schema High Level Explanation

![Schema](https://user-images.githubusercontent.com/99032711/168852482-96ccc853-25cd-4657-8e1e-11d9075b7574.PNG)


## Amazon RDS
Amazon Relational Database Service (Amazon RDS) is a distributed relational database service by Amazon Web Services. 
It is a web service running "in the cloud" designed to simplify the setup, operation, and scaling of a relational database for use in applications.

## Amazon WAF 
AWS WAF is a web application firewall that helps protect your web applications or APIs against common web exploits and bots that may affect availability, compromise security, or consume excessive resources. AWS WAF gives you control over how traffic reaches your applications by enabling you to create security rules that control bot traffic and block common attack patterns, such as SQL injection or cross-site scripting.
You can also customize rules that filter out specific traffic patterns.
You can get started quickly using Managed Rules for AWS WAF, a pre-configured set of rules managed by AWS

## Configuration & Deployment

- composite is intended to run on local Kubernetes cluster like: minikube,rancher,kind,...
    - managed resources are created in the same cluster from which they are being managed

### Prerequisites
- local K8s cluster
- configured provider and credentials



## Configuration 
First we’ll apply a `CompositeResourceDefinition (XRD)` to define the schema of our `XPostgreSQLInstance` and its `PostgreSQLInstance` resource claim.
```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xpostgresqlinstances.database.example.org
spec:
  group: database.example.org
  names:
    kind: XPostgreSQLInstance
    plural: xpostgresqlinstances
  claimNames:
    kind: PostgreSQLInstance
    plural: postgresqlinstances
  connectionSecretKeys:
    - username
    - password
    - endpoint
    - port
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              parameters:
                type: object
                properties:
                  storageGB:
                    type: integer
                  rg-test: 
                    type: string  
                  dbInsClass: 
                    type: string
                  eng: 
                    type: string
                  engVersion: 
                    type: string
                  multiAZ: 
                    type: boolean
                  storageType: 
                    type: string
                required:
                  - storageGB
                  - rg-test
                  - dbInsClass
                  - eng
                  - engVersion
                  - storageType
            required:
              - parameters


```


- `kubectl apply -f /packages/rds/definition.yaml`

Now we’ll specify which managed resources our `XPostgreSQLInstance` XR and its claim could be composed of, and how they should be configured. We do this by defining a `Composition` that can satisfy the XR we defined above. In this case, our `Composition` will specify how to provision a public PostgreSQL instance on the chosen provider.

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: xpostgresqlinstances.aws.database.example.org
  labels:
    provider: aws
    guide: quickstart
    vpc: default
spec:
  writeConnectionSecretsToNamespace: crossplane-system
  compositeTypeRef:
    apiVersion: database.example.org/v1alpha1
    kind: XPostgreSQLInstance
  resources:
    - name: rdsinstance
      base:
        apiVersion: database.aws.crossplane.io/v1beta1
        kind: RDSInstance
        spec:
          forProvider:
            name: provider-aws
            dbSubnetGroupName: default
            autoMinorVersionUpgrade: true
            backupRetentionPeriod: 0
            caCertificateIdentifier: rds-ca-2019
            copyTagsToSnapshot: false
           ## dbInstanceClass: db.t3.medium
            deletionProtection: false
            enableIAMDatabaseAuthentication: false
            enablePerformanceInsights: false
            ##engine: postgres
            ##engineVersion: "13"
            finalDBSnapshotIdentifier: orange-test
            masterUsername: orange
            ##multiAZ: false
            port: 5432
            preferredBackupWindow: 06:15-06:45
            preferredMaintenanceWindow: sat:09:21-sat:09:51
            publiclyAccessible: false
            storageEncrypted: false
           ## storageType: gp2
          writeConnectionSecretToRef:
            namespace: crossplane-system
            name: default
      patches:
        - fromFieldPath: "spec.parameters.rg-test"
          toFieldPath: "spec.forProvider.region"
        
        - fromFieldPath: "spec.parameters.dbInsClass"
          toFieldPath: "spec.forProvider.dbInstanceClass"

        - fromFieldPath: "spec.parameters.eng"
          toFieldPath: "spec.forProvider.engine"

        - fromFieldPath: "spec.parameters.engVersion"
          toFieldPath: "spec.forProvider.engineVersion"
        
        - fromFieldPath: "spec.parameters.multiAZ"
          toFieldPath: "spec.forProvider.multiAZ"
        
        - fromFieldPath: "spec.parameters.storageType"
          toFieldPath: "spec.forProvider.storageType"
          
        - fromFieldPath: "metadata.uid"
          toFieldPath: "spec.writeConnectionSecretToRef.name"
          transforms:
            - type: string
              string:
                fmt: "%s-postgresql"
        - fromFieldPath: "spec.parameters.storageGB"
          toFieldPath: "spec.forProvider.allocatedStorage"
      connectionDetails:
        - fromConnectionSecretKey: username
        - fromConnectionSecretKey: password
        - fromConnectionSecretKey: endpoint
        - fromConnectionSecretKey: port

```
- `kubectl apply -f /packages/rds/composition.yaml`

### Deployment
  `PostgreSQLInstance` XRC could be created in the default namespace to provision a `PostgreSQL` instance and all the supporting infrastructure (VPCs, firewall rules, resource groups, etc) that it may need!

You can apply claim from `/example` folder by populating env var `$CLAIM_NAME` and applying the claim afterwards: `envsubst '$CLAIM_NAME' < example/claim.yaml | kubectl apply -f - ` or you can create `env1` claim: `kubectl apply -f env1/examples/PostgreSql/claim.yaml`

```yaml
apiVersion: database.example.org/v1alpha1
kind: PostgreSQLInstance
metadata:
  name: damyan-test
  namespace: default
spec:
  parameters:
    storageGB: 10
    rg-test: us-east-1
    dbInsClass: db.t3.small
    eng: postgres
    engVersion: "13"
    multiAZ: false
    storageType: gp2  
  compositionSelector:
    matchLabels:
      provider: aws
      vpc: default
  writeConnectionSecretToRef:
    name: db-conn
```
- `kubectl apply -f /examples/PostgreSql/claim.yaml`

After creating the `PostgreSQLInstance` Crossplane will begin provisioning a database instance on your provider of choice. Once provisioning is complete, you should see `READY: True` in the output when you run:
- `kubectl get claim` 
```
C:\Users\damyan.gochev\OneDrive - Accenture\Documents\Crossplane\repo\22_r2_waf\examples\PostgreSql>kubectl get claim
NAME          READY   CONNECTION-SECRET   AGE
damyan-test   True    db-conn             164m

```


Once your `PostgreSQLInstance` is ready, you should see a `Secret` in the `default` namespace named `db-conn` that contains keys that we defined in XRD. If they were filled by the composition, then they should appear:

- `kubectl describe secrets db-conn`

```
C:\Users\damyan.gochev\OneDrive - Accenture\Documents\Crossplane\repo\22_r2_waf\examples\PostgreSql>kubectl describe secrets db-conn
Name:         db-conn
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  connection.crossplane.io/v1alpha1

Data
====
endpoint:  64 bytes
password:  27 bytes
port:      4 bytes
username:  6 bytes

```

### Delete 
Depending on chosen path you can delete your claim by doing:
 - `envsubst '$CLAIM_NAME' < example/claim.yaml | kubectl delete -f - `
 - `kubectl delete -f environments/env1/claim.yaml`
 - or get claim's name and delete it with: `kubectl delete claim <CLAIM NAME>`

 ```
C:\Users\damyan.gochev\OneDrive - Accenture\Documents\Crossplane\repo\22_r2_waf\examples\PostgreSql>kubectl get claim
NAME          READY   CONNECTION-SECRET   AGE
damyan-test   True    db-conn             3h13m

C:\Users\damyan.gochev\OneDrive - Accenture\Documents\Crossplane\repo\22_r2_waf\examples\PostgreSql>kubectl delete claim damyan-test
postgresqlinstance.database.example.org "damyan-test" deleted

C:\Users\damyan.gochev\OneDrive - Accenture\Documents\Crossplane\repo\22_r2_waf\examples\PostgreSql>kubectl get claim
No resources found in default namespace.
 ```
