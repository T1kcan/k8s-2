
# Introduction to docker
## Step 1
### Build Dockerfile and push to registry
A Docker image can be built from an existing Dockerfile using the docker build command. Below is the syntax for this command:
```bash
# build an image
# OPTIONS - optional;  define extra configuration
# PATH - required;  sets the location of the Dockefile and  any referenced files 

docker build [OPTIONS] PATH

# Where OPTIONS can be:
-t, --tag - set the name and tag of the image
-f, --file - set the name of the Dockerfile
--build-arg - set build-time variables

# Find all valid options for this command 
docker build --help
```
For example, to build the image of the Python hello-world application from the Dockerfile, the following command can be used: Build an image using the Dockerfile from the current directory
```bash
docker build -t python-helloworld .
```
Before distributing the Docker image to a wider audience, it is paramount to test it locally and verify if it meets the expected behavior. To create a container using an available Docker image, the docker run command is available. Below is the syntax for this command:
```bash
# execute an image
# OPTIONS - optional;  define extra configuration
# IMAGE -  required; provides the name of the image to be executed
# COMMAND and ARGS - optional; instruct the container to run specific commands when it starts 
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

# Where OPTIONS can be:
-d, --detach - run in the background 
-p, --publish - expose container port to host
-it - start an interactive shell

# Find all valid options for this command 
docker run --help
```
For example, to run the Python hello-world application, using the created image, the following command can be used:

Note: To access the application in a browser, we need to bind the Docker container port to a port on the host or local machine. In this case, 5111 is the host port that we use to access the application e.g. http://127.0.0.1:5111/. The 5000 is the container port that the application is listening to for incoming requests.
```bash
# run the `python-helloworld` <span class='kc-markdown-code-copy'></span> image, in detached mode and expose it on port `5111` <span class='kc-markdown-code-copy'></span>
docker run -d -p 5111:5000 python-helloworld
To retrieve the Docker container logs use the docker logs {{ CONTAINER_ID }} command. For example:

```docker logs 95173091eb5e` ``

docker ps 
docker logs 2e5d26ffaa1c   
```

### Package docker image and distribute it
The last step in packaging an application using Docker is to store and distribute it. So far, we have built and tested an image on the local machine, which does not ensure that other engineers have access to it. As a result, the image needs to be pushed to a public Docker image registry, such as DockerHub, Harbor, Google Container Registry, and many more. However, there might be cases where an image should be private and only available to trusted parties. As a result, a team can host private image registries, which provides full control over who can access and execute the image.

Before pushing an image to a Docker registry, it is highly recommended to tag it first. During the build stage, if a tag is not provided (via the -t or --tag flag), then the image would be allocated an ID, which does not have a human-readable format (e.g. 0e5574283393). On the other side, a defined tag is easily scalable by the human eye, as it is composed of a registry repository, image name, and version. Also, a tag provides version control over application releases, as a new tag would indicate a new release.

To tag an existing image on the local machine, the docker tag command is available. Below is the syntax for this command:
```bash
# tag an image
# SOURCE_IMAGE[:TAG]  - required and the tag is optional; define the name of an image on the current machine 
# TARGET_IMAGE[:TAG] -  required and the tag is optional; define the repository, name, and version of an image
docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]
```
For example, to tag the Python hello-world application, to be pushed to a repository in DockerHub, the following command can be used:

Replace pixelpotato with your docker hub name
```bash
# tag the `python-helloworld`  image, to be pushed 
# in the `pixelpotato` repository, with the `python-helloworld` image name
# and version `v1.0.0`
docker tag python-helloworld pixelpotato/python-helloworld:v1.0.0
```
Once the image is tagged, the final step is to push the image to a registry. For this purpose, the docker push command can be used. Below is the syntax for this command:
```bash
# push an image to a registry 
# NAME[:TAG] - required and the tag is optional; name, set the image name to be pushed to the registry
docker push NAME[:TAG]
```
For example, to push the Python hello-world application to DockerHub, the following command can be used:
```bash
# push the `python-helloworld` application in version v1.0.0 
# to the `pixelpotato` repository in DockerHub
!docker push pixelpotato/python-helloworld:v1.0.0
```

controlplane:~$ helm repo add harbor https://helm.goharbor.io
helm fetch harbor/harbor --untar
helm install my-release harbor/ -n harbor --create-namespace
