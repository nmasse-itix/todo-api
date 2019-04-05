# API Automation Demo

## Deliver the demo

*In this demo, we will see how we can create very easily REST APIs using the iPaaS capabilities of Red Hat Integration.*
*We will also discover how this API can be automatically discovered and managed as any other API.*
*Last but not least, we will see how this API can be deployed in a TEST and a PROD environment in an automated manner using a CI/CD pipeline.*

*The whole demo runs on OpenShift in containers.*

*Let's start with a very simple REST API that expose our TODO List.*
*The TODO List is currently managed by an existing application and we will need to integrate with it.*

- Open the **TODO List** application
- Insert a new TODO

*Now, let's craft a REST API that integrates with this application to show our TODOs from the list.*

- Go on Fuse Online
- Create an integration based on a REST API

*We use a contract-first approach and since I have no pre-existing API Contract, I will create a new one.*

- Select **build from scratch**
- Create a resource named `TODO` with two fields: `id` and `message`
- Create the `/todo` GET method that returns a `TODO` resource with `200 OK`
- Save the API contract

*The list of our TODO is stored in a Database.*

- Add a Database connection that queries `SELECT * FROM todo`

*We now have to map the fields from the database to the fields defined in the API Contract.*

- Add a mapping step that maps `id` to `id` and `task` to `message`

*Let's publish the integration and wait for the task to complete.*

- Publish the integration

*Now that the integration is published, we can manage it as an API.*

- Go to 3scale and on the dashboard, select **New API**

*The integration is automatically discovered and promoted as an API.*

- Select **Import from OpenShift**. In the `fuse` project, select `i-todo`.
- Wait for 10 seconds and refresh the dashboard
- Select the `i-todo` api in the dropdown menu

*Let's create the mandatory configuration to realize and end-to-end test.*

- Go to **Applications** > **Application Plans**
- Click **Create Application Plan**
- Fill the name and system_name with `test`
- Click **Create Application Plan**
- Select **Audience** in the dropdown menu
- Click **Accounts** > **Listing**
- Click the `Developer` account
- Click **1 Applications**
- Click **Create Application**
- Select the `test` plan created earlier
- Fill in the name and description and click **Create Application**
- Go to **Integration** > **Configuration**
- Click **Edit APIcast configuration**
- Edit the **Staging Public Base URL** to insert `.api` just before `.apps`
- Edit the **Production Public Base URL** to insert `.api` just before `.apps`
- Set the **API test GET request** to `/todo`
- Click **Update & test in Staging Environment**

*Now, we can test our API is published and live!*

- Copy the curl command line, paste it in a terminal and add `-k` to the end

*Our API is live and working.*

*Now, let's deploy this API in a TEST and a PROD environment.*

- Go to the OpenShift Console
- Open the `TODO List (TEST)` project

*The TEST environment is empty.*

- Open the `TODO List (PROD)` project

*The PROD environment is empty.*

- Open the `TODO List (BUILD)` project
- Go to **Build** > **Pipelines**
- Click **Start pipeline**
- Wait for the Pipeline to complete
- Go to the 3scale Developer Portal
- Login with `john`
- Go to **Applications**
- Copy the API Key for the `TODO List API (PROD, v1.0.0)`
- Go to **Documentation** > `TODO List API (PROD, v1.0.0)`
- Expand the **GET /todo** method
- Click on the **red exclamation mark**
- Fill-in the API Key you copied earlier
- Click **Authorize**
- Click **Try it out!**

*Our API is live in the production environment and working.*

## Reset the demo

- Delete the `TODO` integration in Fuse Online
- Re-populate the test and prod environments with your "reset kit":

```sh
oc delete dc/i-todo svc/i-todo is/i-todo -n todo-test --ignore-not-found
oc delete dc/i-todo svc/i-todo is/i-todo -n todo-prod --ignore-not-found
oc create -f test-env-copy.yaml -n todo-test
oc create -f prod-env-copy.yaml -n todo-prod
```

- Delete all the `TODO List API` in 3scale
- Delete all TODOs in the **TODO List** application

## Setup

Create a build, a test and a prod environment:

```sh
oc new-project todo-test --display-name="TODO List (TEST)"
oc new-project todo-prod --display-name="TODO List (PROD)"
oc new-project todo-build --display-name="TODO List (BUILD)"
```

Deploy Jenkins in the build project and give it admin privileges:

```sh
oc project todo-build
oc new-app --template=jenkins-persistent -p MEMORY_LIMIT=1Gi
oc set env dc/jenkins JENKINS_OPTS=--sessionTimeout=86400
oc adm policy add-role-to-user system:build-strategy-custom -z jenkins
oc adm policy add-role-to-user admin "system:serviceaccount:$(oc project -q):jenkins" -n todo-test
oc adm policy add-role-to-user admin "system:serviceaccount:$(oc project -q):jenkins" -n todo-prod
oc adm policy add-role-to-user admin "system:serviceaccount:$(oc project -q):jenkins" -n fuse
```

Provision the 3scale APIs:

```sh
oc create -f https://raw.githubusercontent.com/nmasse-itix/threescale-cicd/master/support/openshift/openshift-template.yaml
oc new-app --template=deploy-3scale-api -p THREESCALE_CICD_VERSION=master -p THREESCALE_ADMIN_PORTAL_ACCESS_TOKEN="$ACCESS_TOKEN" -p THREESCALE_ADMIN_PORTAL_HOSTNAME="$TENANT-admin.3scale.net" -p API_NAME=todo-api-test -p THREESCALE_CICD_PRIVATE_BASE_URL=http://i-todo.todo-test.svc:8080 -p API_GIT_URI=https://github.com/nmasse-itix/todo-api.git -p THREESCALE_CICD_API_SYSTEM_NAME=test_todo_api -p THREESCALE_CICD_API_ENVIRONMENT_NAME=test -p THREESCALE_CICD_APICAST_SANDBOX_ENDPOINT="https://test-todo-api-staging.$WILDCARD" -p THREESCALE_CICD_APICAST_PRODUCTION_ENDPOINT="https://test-todo-api-production.$WILDCARD" -p THREESCALE_CICD_APICAST_POLICIES_CORS=true
oc new-app --template=deploy-3scale-api -p THREESCALE_CICD_VERSION=master -p THREESCALE_ADMIN_PORTAL_ACCESS_TOKEN="$ACCESS_TOKEN" -p THREESCALE_ADMIN_PORTAL_HOSTNAME="$TENANT-admin.3scale.net" -p API_NAME=todo-api-prod -p THREESCALE_CICD_PRIVATE_BASE_URL=http://i-todo.todo-prod.svc:8080 -p API_GIT_URI=https://github.com/nmasse-itix/todo-api.git -p THREESCALE_CICD_API_SYSTEM_NAME=prod_todo_api -p THREESCALE_CICD_API_ENVIRONMENT_NAME=prod -p THREESCALE_CICD_APICAST_SANDBOX_ENDPOINT="https://prod-todo-api-staging.$WILDCARD" -p THREESCALE_CICD_APICAST_PRODUCTION_ENDPOINT="https://prod-todo-api-production.$WILDCARD" -p THREESCALE_CICD_APICAST_POLICIES_CORS=true
```

Create and deploy the `TODO` integration as explained in the **Deliver the demo** section.

Populate the test and prod environments:

```sh
oc delete dc/i-todo svc/i-todo is/i-todo -n todo-test --ignore-not-found
oc delete dc/i-todo svc/i-todo is/i-todo -n todo-prod --ignore-not-found
oc get dc/i-todo svc/i-todo secret/i-todo -n fuse -o yaml --export |oc create -f - -n todo-test
oc set triggers dc/i-todo -n todo-test --remove-all
oc set triggers dc/i-todo -n todo-test --from-image=i-todo:test --containers i-todo --manual
oc get dc/i-todo svc/i-todo secret/i-todo -n fuse -o yaml --export |oc create -f - -n todo-prod
oc set triggers dc/i-todo -n todo-prod --remove-all
oc set triggers dc/i-todo -n todo-prod --from-image=i-todo:prod --containers i-todo --manual
```

Prepare your "reset kit": 

```sh
oc get dc/i-todo svc/i-todo secret/i-todo -n todo-test -o yaml --export > test-env-copy.yaml
oc get dc/i-todo svc/i-todo secret/i-todo -n todo-prod -o yaml --export > prod-env-copy.yaml
```

Create the Jenkins pipeline:

```sh
oc create -f - <<EOF
kind: "BuildConfig"
apiVersion: "v1"
metadata:
  name: "deploy-api"
spec:
  strategy:
    jenkinsPipelineStrategy:
      jenkinsfile: |-
        pipeline {
            agent none
            stages {
                stage('Deploy API Implementation in TEST') {
                    steps {
                        script {
                            openshift.withCluster() {
                                openshift.withProject('fuse') {
                                    def dc = openshift.selector('dc/i-todo')
                                    def obj = dc.objects(exportable:true)
                                    def version = obj[0].metadata.labels["syndesis.io/deployment-version"]
                                    echo "version = $version"
                                    openshift.tag("fuse/i-todo:" + version, "todo-test/i-todo:test")
                                    openshift.withProject('todo-test') {
                                        def dc2 = openshift.selector('dc', 'i-todo');
                                        dc2.rollout().latest();
                                        dc2.rollout().status();
                                    }
                                }
                            }
                        }
                    }
                }
                stage('Deploy API Contract in TEST') {
                    steps {
                        script {
                            openshift.withCluster() {
                                openshift.withProject('todo-build') {
                                    def bc = openshift.selector('bc', "deploy-3scale-api-todo-api-test");
                                    bc.startBuild("--wait=true");
                                }
                            }
                        }
                    }
                }
                stage('Deploy API Implementation in PROD') {
                    steps {
                        script {
                            openshift.withCluster() {
                                openshift.withProject('todo-prod') {
                                    openshift.tag("todo-test/i-todo:test", "todo-prod/i-todo:prod")
                                    def dc = openshift.selector('dc', 'i-todo');
                                    dc.rollout().latest();
                                    dc.rollout().status();
                                }
                            }
                        }
                    }
                }
                stage('Deploy API Contract in PROD') {
                    steps {
                        script {
                            openshift.withCluster() {
                                openshift.withProject('todo-build') {
                                    def bc = openshift.selector('bc', "deploy-3scale-api-todo-api-prod");
                                    bc.startBuild("--wait=true");
                                }
                            }
                        }
                    }
                }
            }
        }
EOF
```

Optionally, set a trigger on the jenkins Pipeline:

```sh
oc set triggers bc/deploy-api -n todo-build --from-image=fuse/i-todo:1
```

Go to your 3scale Admin Portal and add the API Catalog page as explained here:
TODO
