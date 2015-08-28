## The Registry
[//]: # (TODO: What is it for in AE??)
[//]: # (TODO: Write some example how to use it since we don't have STI)

Atomic Enterprise provides a Docker registry that administrators may run inside
the Atomic environment that will manage images "locally". Let's take a moment
to set that up.

### Storage for the registry
The registry stores docker images and metadata. If you simply deploy a pod
with the registry, it will use an ephemeral volume that is destroyed once the
pod exits. Any images anyone has built or pushed into the registry would
disappear. That would be bad.

What we will do for this demo is use a directory on the master host for
persistent storage. In production, this directory could be backed by an NFS
mount supplied from the HA storage solution of your choice. That NFS mount
could then be shared between multiple hosts for multiple replicas of the
registry to make the registry HA.

For now we will just show how to specify the directory and leave the NFS
configuration as an exercise. On the master, as `root`, create the storage
directory with:

    mkdir -p /mnt/registry

### Creating the registry
`oadm` again comes to our rescue with a handy installer for the
registry. As the `root` user, run the following:

[//]: # (TODO: fix the ca path)
[//]: # (TODO: fix the image path)

    oadm registry --create \
    --credentials=/etc/origin/master/openshift-registry.kubeconfig \
    --images='registry.access.redhat.com/openshift3/ose-${component}:latest' \
    --selector="region=infra" --mount-host=/mnt/registry

You'll get output like:

    services/docker-registry
    deploymentConfigs/docker-registry

You can use `oc get pods`, `oc get services`, and `oc get deploymentconfig`
to see what happened. This would also be a good time to try out `oc status`
as root:

[//]: # (TODO: fix image paths)

    oc status
    In project default

    service docker-registry (172.30.53.223:5000)
      docker-registry deploys registry.access.redhat.com/openshift3/ose-docker-registry:latest
        #1 deployed 4 hours ago - 1 pod

    service kubernetes (172.30.0.2:443)

    service kubernetes-ro (172.30.0.1:80)

    service router (172.30.74.178:80)
      router deploys registry.access.redhat.com/openshift3/ose-haproxy-router:latest
        #1 deployed 7 minutes ago - 1 pod

The project we have been working in when using the `root` user is called
"default". This is a special project that always exists (you can delete it, but
AE will re-create it) and that the administrative user uses by default.
One interesting feature of `oc status` is that it lists recent deployments.
When we created the router and registry, each created one deployment We will
talk more about deployments when we get into builds.

Anyway, you will ultimately have a Docker registry that is being hosted by AE
and that is running on the master (because we specified "region=infra" as the
registry's node selector).

To quickly test your Docker registry, you can do the following:

    curl -v `oc get services | grep registry | awk '{print $4":"$5}/v2/' | sed 's,/[^/]\+$,/v2/,'`

And you should see [a 200
response](https://docs.docker.com/registry/spec/api/#api-version-check) and a
mostly empty body. Your IP addresses will almost certainly be different.

    * About to connect() to 172.30.53.223 port 5000 (#0)
    *   Trying 172.30.53.223...
    * Connected to 172.30.53.223 (172.30.53.223) port 5000 (#0)
    > GET /v2/ HTTP/1.1
    > User-Agent: curl/7.29.0
    > Host: 172.30.53.223:5000
    > Accept: */*
    >
    < HTTP/1.1 200 OK
    < Content-Length: 2
    < Content-Type: application/json; charset=utf-8
    < Docker-Distribution-Api-Version: registry/2.0
    < Date: Thu, 11 Jun 2015 13:07:11 GMT
    <
    * Connection #0 to host 172.30.53.223 left intact
    {}

If you get "connection reset by peer" you may have to wait a few more moments
after the pod is running for the service proxy to update the endpoints necessary
to fulfill your request. You can check if your service has finished updating its
endpoints with:

    oc describe service docker-registry

And you will eventually see something like:

    Name:                   docker-registry
    Labels:                 docker-registry=default
    Selector:               docker-registry=default
    Type:                   ClusterIP
    IP:                     172.30.239.41
    Port:                   <unnamed>       5000/TCP
    Endpoints:              <unnamed>       10.1.0.4:5000
    Session Affinity:       None
    No events.

Once there is an endpoint listed, the curl should work and the registry is available.

Highly available, actually. You should be able to delete the registry pod at any
point in this training and have it return shortly after with all data intact.

## Creating and Wiring Disparate Components
This example involves a build of another application and a service that has two
pods -- a "front-end" web tier and a "back-end" database tier. This application
also makes use of auto-generated parameters and other neat features of Atomic
Enterprise.

[OpenShift Enterprise](https://docs.openshift.com/enterprise/3.0/welcome/index.html)
provides a [Source-to-Image (S2I)
framework](https://docs.openshift.com/enterprise/3.0/creating_images/s2i.html)
which makes it easy to produce an image from an application source code and
store it in local registry service. In Atomic Enterprise, we have to do the
crude work ourselves.

### Create a New Project
Open a terminal as `alice`:

    # su - alice

Then, create a project for this example:

    oc new-project wiring --display-name="Exploring Parameters" \
    --description='An exploration of wiring using parameters'

Before continuing, `alice` will also need the training repository:

    cd
    git clone https://github.com/projectatomic/atomic-enterprise-training.git training

### A Quick Aside on Templates
From the [OpenShift
documentation](http://docs.openshift.org/latest/dev_guide/templates.html):

    A template describes a set of resources intended to be used together that
    can be customized and processed to produce a configuration. Each template
    can define a list of parameters that can be modified for consumption by
    containers.

As we mentioned previously, this template has some auto-generated parameters.
For example, take a look at the following JSON:

    "parameters": [
      {
        "name": "ADMIN_USERNAME",
        "description": "administrator username",
        "generate": "expression",
        "from": "admin[A-Z0-9]{3}"
      },

This portion of the template's JSON tells OpenShift to generate an expression
using a regex-like string that will be presented as ADMIN_USERNAME.

### Stand Up the Frontend
The first step will be to stand up the frontend of our application. For
argument's sake, this could have just as easily been brand new vanilla code.
However, to make things faster, we'll start with an application that already is
looking for a DB, but won't fail spectacularly if one isn't found.

#### Building of the Frontend
You'll need to manually fetch the source and build an image out of it using docker.
First, log in as `root` and checkout the application:

    cd
    git clone -b early-access https://github.com/projectatomic/ruby-hello-world
    cd ruby-hello-world

The image needs to be available for download for your nodes. Registry, you've
set up earlier, is an ideal place for hosting it. In order to push the built
image to the registry, you need to know its URL:

    REGISTRY=`oc get services | grep registry | awk '{print $4":"$5}' | sed 's,/[^/]\+$,,'`

There's a `Dockerfile` prepared for you in the repository. Let's use it to
build the image.

    docker build -t $REGISTRY/openshift/ruby-hello-world .

Note the `$REGISTRY` prefix. It's the destination registry, where the image
will be pushed. Let's wait with that until we set up an ImageStream.

#### Wait, What's an ImageStream?
If you think about one of the important things that Atomic Enterprise needs to
do, it's to be able to deploy newer versions of user applications into Docker
containers quickly. In OpenShift Enterprise, an "application" is really two
pieces -- the starting image (the S2I builder) and the application code. While
it's "obvious" that the deployed Docker containers need to be updated when
application code changes, it may not have been so obvious that the deployed
container needs to be updated as well if the **builder** image changes.

For example, what if a security vulnerability in the Ruby runtime is
discovered? It would be nice if we could automatically know this and take
action. Triggers of deployment configuration let you define exactly that with
particular image stream like below:

    cat eap-latest/frontend-template.json | sed -n '/"triggers":/,+18p
            "triggers": [
              {
                "type": "ImageChange",
                "imageChangeParams": {
                  "automatic": true,
                  "containerNames": [
                    "ruby-hello-world"
                  ],
                  "from": {
                    "kind": "ImageStreamTag",
                    "name": "ruby-hello-world:latest",
                    "namespace": "openshift"
                  },
                  "lastTriggeredImage": ""
                }
              },
              {
                "type": "ConfigChange"
              }

Above can be translated as "launch a new deployment when its configuration is
updated or `latest` tag in openshift/ruby-hello-world ImageStream gets
updated".

The `ImageStream` resource is, somewhat unsurprisingly, a definition for a
stream of Docker images that might need to be paid attention to. By defining an
`ImageStream` on "ruby-hello-world", for example, and then building an
application against it, we have the ability with OpenShift to "know" when that
`ImageStream` changes and take action based on that change. In our example from
the previous paragraph, if the "ruby-hello-world" image changed in the
repository defined by the `ImageStream`, we might automatically trigger a new
build of our application code.

#### Adding the ImageStreams
Perform the following command as `root` in the `eap-latest`folder in order to
add all of the images:

    oc create -f image-streams.json -n openshift

You will see the following:

    imageStreams/mysql
    imageStreams/ruby-hello-world

Try to guess, which one belongs to the frontend and backend. If you inspect
them, you'll notice that the latter doesn't specify a repository, which causes
AE to look in its private registry for `openshift/ruby-hello-world`.

What is the `openshift` project where we added these builders? This is a
special project that can contain various elements that should be available to
all users of the Atomic Enterprise environment.

Let's inspect them:

    oc get imagestreams -n openshift

After several seconds, you'll see:

    NAME               DOCKER REPO                                            TAGS                       UPDATED
    mysql              registry.access.redhat.com/openshift3/mysql-55-rhel7   latest,v0.4.3.2,v0.5.2.2   5 minutes ago
    ruby-hello-world   172.30.53.223:5000/openshift/ruby-hello-world

Note that no tags are available for `ruby-hello-world`. Why? You haven't pushed
it yet:
    
    docker push $REGISTRY/openshift/ruby-hello-world

Once pushed, tags will be updated automatically:

    oc get is -n openshift
    NAME               DOCKER REPO                                            TAGS                       UPDATED
    mysql              registry.access.redhat.com/openshift3/mysql-55-rhel7   latest,v0.4.3.2,v0.5.2.2   6 minutes ago
    ruby-hello-world   172.30.53.223:5000/openshift/ruby-hello-world          latest                     5 seconds ago

The image is now accessible from all the nodes via `openshift/ruby-hello-world`
image stream. And we can finally proceed to frontend's deployment.

#### Frontend's deployment
Return to Alice's training directory and instantiate objects stored in the
frontend's template. Since we know that we want to talk to a database
eventually, you'll want to pass the right environment variables to a *process*
command:

    [alice]$ cd ~/training/eap-latest
    [alice]$ # Don't forget to set $REGISTRY variable
    [alice]$ oc process -f frontend-template.json \
        -v=MYSQL_USER=root,MYSQL_PASSWORD=redhat,MYSQL_DATABASE=mydb \
        > frontend-config.json

Above command parsed the `frontend-template.json`, replaced parameters with the
values given, generated new values for those unspecified and saved it to
`frontend-config.json`. `MYSQL_*` parameters may safely be omitted from the
command. They would have been auto-generated for us.

Now go ahead and instantiate generated objects:

    [alice]$ oc create -f frontend-config.json

You should see:

    services/ruby-hello-world
    deploymentConfigs/frontend

Shortly after that, a new pod should be available. If you want to double-check
that it is using right environment variables, just list them:

    [alice]$ oc env --list dc/frontend
    # deploymentconfigs frontend, container ruby-hello-world
    ADMIN_USERNAME=adminATH
    ADMIN_PASSWORD=XgjIFBoR
    MYSQL_USER=root
    MYSQL_PASSWORD=redhat
    MYSQL_DATABASE=mydb

If we'd omitted `MYSQL_*` parameters in the call to `oc process` above, we
would have got random values similar to `ADMIN_*` parameters. 

### Expose the Service
The `oc` command has a nifty subcommand called `expose` that will take a
service and automatically create a route for us. It will do this in the defined
cloud domain and in the current project as an additional "namespace" of sorts.
For example, the steps above resulted in a service called "ruby-hello-world".
We can use `expose` against it:

    [alice]$ oc expose service ruby-hello-world

After a few moments:

    [alice]$ oc get route
    NAME               HOST/PORT                                       PATH      SERVICE            LABELS
    ruby-hello-world   ruby-hello-world.wiring.cloudapps.example.com             ruby-hello-world 

Take a look at that hostname. It is

* the service name
* the namespace name
* the route domain

all concatenated together. In the future the `expose` command will allow a
hostname to be specified directly.

Now you should be able to access your application with your browser! Go ahead
and do that now. You'll notice that the frontend is happy to run without a
database, but it's not all that exciting. We'll fix that in a moment.

### Add the Database Template
Now we'll demonstrate adding a template to our own project. In
the `eap-latest` folder there is a `mysql-template.json` file. As `alice`, go ahead
and add it to your project:

    [alice]$ oc create -f mysql-template.json

You'll see:

    templates/mysql-ephemeral

Pass the template name to *process* command and make sure to give it the same
values for `MYSQL_*` environment variables as for the frontend template.

    [alice]$ oc process mysql-ephemeral \
        -v=MYSQL_USER=root,MYSQL_PASSWORD=redhat,MYSQL_DATABASE=mydb \
        | oc create -f -

It may take a little while for the MySQL container to download (if you didn't
pre-fetch it). It's a good idea to verify that the database is running before
continuing.  If you don't happen to have a MySQL client installed you can still
verify MySQL is running with curl:

    curl `oc get services | grep database | awk '{print $4}'`:3306

MySQL doesn't speak HTTP so you will see garbled output like this (however,
you'll know your database is running!):

    5.5.41%`:<^H*:��%I!geC`9=c\&mysql_native_password!��#08S01Got packets out of order

### Visit Your Application Again
Visit your application again with your web browser. Why does it still say that
there is no database?

When the frontend was first built and created, there was no service called
"database", so the environment variable `DATABASE_SERVICE_HOST` did not get
populated with any values. Our database does exist now, and there is a service
for it, but Atomic Enterprise could not "inject" those values into the frontend
container.

### Replication Controllers
The easiest way to get this going? Just nuke the existing pod. There is a
replication controller running for both the frontend and backend:

    [alice]$ oc get replicationcontroller

The replication controller is configured to ensure that we always have the
desired number of replicas (instances) running. We can look at how many that
should be:

    [alice]$ oc describe rc frontend-1

So, if we kill the pod, the RC will detect that, and fire it back up. When it
gets fired up this time, it will then have the `DATABASE_SERVICE_HOST` value,
which means it will be able to connect to the DB, which means that we should no
longer see the database error!

As `alice`, go ahead and find your frontend pod, and then kill it:

    [alice]$ oc delete pod `oc get pod | grep -e "frontend-[0-9]" | grep -v build | awk '{print $1}'`

You'll see something like:

    pods/frontend-1-wcxiw

That was the generated name of the pod when the replication controller stood it
up the first time.

After a few moments, we can look at the list of pods again:

    [alice]$ oc get pod | grep frontend

And we should see a different name for the pod this time:

    [alice]$ frontend-1-4ikbl

This shows that, underneath the covers, the RC restarted our pod. Since it was
restarted, it should have a value for the `DATABASE_SERVICE_HOST` environment
variable. Go to the node where the pod is running, and find the Docker container
id as `root`:

    docker inspect `docker ps | grep hello-world | awk \
    '{print $1}'` | grep DATABASE

The output will look something like:

    "MYSQL_DATABASE=mydb",
    "DATABASE_SERVICE_PORT_MYSQL=3306",
    "DATABASE_SERVICE_PORT=3306",
    "DATABASE_PORT=tcp://172.30.249.174:3306",
    "DATABASE_PORT_3306_TCP=tcp://172.30.249.174:3306",
    "DATABASE_PORT_3306_TCP_PROTO=tcp",
    "DATABASE_SERVICE_HOST=172.30.249.174",
    "DATABASE_PORT_3306_TCP_PORT=3306",
    "DATABASE_PORT_3306_TCP_ADDR=172.30.249.174",

### Revisit the Webpage
Go ahead and revisit `http://ruby-hello-world.wiring.cloudapps.example.com` (or
your appropriate FQDN) in your browser, and you should see that the application
is now fully functional!

[//]: # (TODO: Missing template section)
[//]: # (TODO: remake the rollback example)
