# Custom AMQ on OpenShift with s2i

You can build your own custom AMQ image on top of the official one with a s2i build.
This way you can easily add your own configuration files or apply any patch.
```sh
PROJECT_NAME="broker"
IMAGE_STREAM="https://raw.githubusercontent.com/jboss-container-images/jboss-amq-7-broker-openshift-image/76-7.6.0.GA/amq-broker-7-image-streams.yaml"
TEMPLATE="https://raw.githubusercontent.com/jboss-container-images/jboss-amq-7-broker-openshift-image/76-7.6.0.GA/templates/amq-broker-76-basic.yaml"

REG_SECRET="registry-secret"
REG_USER="my-user"
REG_PASS="my-pass"

oc new-project $PROJECT_NAME

# create an authentication secret with your own credentials
oc create secret docker-registry $REG_SECRET \
    --docker-server="registry.redhat.io" \
    --docker-username="$REG_USER" \
    --docker-password="$REG_PASS"

# clone this repo and start a new build from source
oc new-build registry.redhat.io/amq7/amq-broker:7.6~https://github.com/fvaleri/amq-s2i.git
oc set build-secret --pull bc/amq-s2i $REG_SECRET
oc start-build amq-s2i

# follow the build process (should end with: Push successful)
oc logs -f bc/amq-s2i
oc get is

# create the service account
echo '{"kind": "ServiceAccount", "apiVersion": "v1", "metadata": {"name": "amq-service-account"}}' | oc create -f -
oc policy add-role-to-user view system:serviceaccount:$PROJECT_NAME:amq-service-account

# deploy the broker
curl -o /tmp/broker.yaml $TEMPLATE
oc process -f /tmp/broker.yaml \
    -p APPLICATION_NAME=$PROJECT_NAME \
    -p AMQ_USER=admin \
    -p AMQ_PASSWORD=admin \
    -p IMAGE_STREAM_NAMESPACE=$PROJECT_NAME \
    -p IMAGE=image-registry.openshift-image-registry.svc:5000/$PROJECT_NAME/amq-s2i \
    | oc create -f -
```

## Configuration updates

Set the GitHub webhook URL in order to trigger a new build after each commit.

You can also do it manually:
```
git commit -am "Broker config update"
git push
oc start-build amq-s2i -n broker
oc deploy broker-amq --latest -n broker
```
