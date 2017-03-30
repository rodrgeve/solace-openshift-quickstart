cd origin
vagrant up --provider=virtualbox

cd ..

oc login https://10.2.2.2:8443
# Answer y to "Use insecure connections? (y/n):"
# use "admin" for username
# Enter any password.  This will become admin's password from this point forward, so remember what you typed.

# Expose the Docker registry
oc project default
oc delete route/docker-registry
oc create -f dev-route.yml

# Create the new project and enable privileged and anyuid SCCs.
oc new-project vmr-openshift-demo
cd origin
vagrant ssh
sudo -i
oadm policy add-scc-to-user privileged system:serviceaccount:vmr-openshift-demo:default
oadm policy add-scc-to-user anyuid system:serviceaccount:vmr-openshift-demo:default
exit
cd ..

# Create a new docker engine VM which allows insecure TLS connections to hub.10.2.2.2.xip.io
docker-machine create --driver virtualbox --engine-insecure-registry hub.10.2.2.2.xip.io dev
eval $(docker-machine env dev)

# Load the docker image and push it to the Openshift Origin VM
docker load -i <image>.tar.gz
docker login --username=admin --password=`oc whoami -t` hub.10.2.2.2.xip.io
docker tag solace-app:<version-tag> hub.10.2.2.2.xip.io/vmr-openshift-demo/solace-app:latest
docker push hub.10.2.2.2.xip.io/vmr-openshift-demo/solace-app:latest

oc create -f https://raw.githubusercontent.com/jorgemoralespou/s2i-java/master/ose3/s2i-java-imagestream.json

oc create -f solace-messaging-demo-template.yml
oc secrets new-sshauth gitsshsecret --ssh-privatekey=openshift_demo_deploy.key
oc process solace-springboot-messaging-sample APPLICATION_SUBDOMAIN=10.2.2.2.xip.io VMR_IMAGE=172.30.200.195:5000/vmr-openshift-demo/solace-app  | oc create -f -