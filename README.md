# This is not an official Google project.

This script is for **educational purposes only**, is **not certified** and is **not recommended** for production environments.

## Copyright 2022 Google LLC

 Licensed under the Apache License, Version 2.0 (the "License");
 you may not use this file except in compliance with the License.
 You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.


## Deploy a HTML site to Cloud Run

The first step installs the html5-boilerplate package globally using npm, the default package manager for the JavaScript runtime environment Node.js.
The next step creates a new project using the create-html5-boilerplate command, and then changes into the directory of the new project. The following two commands, npm install and npm start, install the project's dependencies and start the development server, respectively.

```sh
npm install html5-boilerplate -g


npx create-html5-boilerplate new-site

cd new-site
npm install
npm start
```


### Create and configure a Static Server

Create a file named server.js with the next code.

The server.js file that is created next defines a StaticServer, which is a module for serving static files over HTTP. The file sets up the server with a number of options, such as the port to use, the root path of the server, and the index and 404 templates to use. The file also contains event listeners for various events that the server can emit, such as when a request is made or when a symbolic link is followed.

```js
var StaticServer = require('static-server');
var server = new StaticServer({
  rootPath: '.',
  port: process.env.PORT || 8080,
  name: 'help',
  host: '0.0.0.0',
  cors: '*',
  followSymlink: true,
  templates: {
    index: 'index.html',
    notFound: '404.html'
  }
});

server.start(function () {
  console.log('Server listening to', server.port);
});

server.on('request', function (req, res) {
});

server.on('symbolicLink', function (link, file) {
  console.log('File', link, 'is a link to', file);
});

server.on('response', function (req, res, err, file, stat) {
});
```

The next command installs the static-server package as a production dependency of the project. This package is used in the server.js file to create the StaticServer instance.

```sh
npm i -S static-server
```

Edit the package.json file to specify that the node server.js command should be used to start the server when running npm start. This replaces the default command that was specified in the package.json file when the project was created.

```json
"start": "node server.js",
```

### Define a Dockerfile

Create a file named dockerfile with the next code

The Dockerfile defines a Docker image for the project. It starts from the node:alpine base image, which is a lightweight version of the Node.js runtime. It then sets the working directory, copies the project files and dependencies, and sets the default command to start the server.

```dockerfile 
FROM node:alpine
WORKDIR /usr/src/app
COPY package.json package*.json ./
RUN npm install --only=production
COPY . .
CMD [ "npm", "start" ]
```

### Create a Deployment specification

Create a deployment.yaml file with the next code

The deployment.yaml file defines a Kubernetes service using the Knative serving extension. The service is named new-site, and it specifies that the container image to use is gcr.io/PROJECT_ID/new-site:0.0.1 on port 8080. It also sets an environment variable SOME_VAR with the value some value, and specifies resource limits for the container.
Please replace PROJECT_ID with the ID of your google cloud project
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: new-site
  annotations:
    run.googleapis.com/ingress: all
    run.googleapis.com/ingress-status: all
spec:
  template:
    spec:
      containers:
      - image: gcr.io/PROJECT_ID/new-site:0.0.1
        ports:
        - containerPort: 8080
        env:
        - name: SOME_VAR
          value: 'some value'
        resources:
          limits:
            cpu: 1000m
            memory: 256Mi
```

### Deploy your service to Google Cloud

create a deploy.sh file with the next code
The deploy.sh file is a shell script that can be used to build and deploy the container image specified in the deployment.yaml file. The script first builds the image and then replaces the service with the new image using the gcloud command-line tool. The script can be made executable by running the chmod +x deploy.sh command.

```sh
#!/bin/bash
echo "Desploying Microservice"
export PROJECT_ID=$(gcloud config list --format 'value(core.project)')
gcloud builds submit --tag gcr.io/$PROJECT_ID/new-site:0.0.1
gcloud beta run services replace deployment.yaml --platform managed --region southamerica-west1
gcloud run services add-iam-policy-binding new-site \
    --member="allUsers" \
    --role="roles/run.invoker"
```

```sh
chmod +x deploy.sh
```

```sh
./deploy.sh
```


### Give it to the next level with CI/CD

Gitlab example pipeline file

The gitlab-ci.yml file is an example GitLab CI/CD pipeline configuration file. The pipeline has two stages: release and deploy. The release stage builds the container image and pushes it to the Google Container Registry (GCR), and the deploy stage deploys the image to Google Cloud Run. The pipeline only runs on the master branch.

gitlab-ci.yml
```yaml
image: scratch

stages:
  - release
  - deploy

image_build:
  stage: release
  image: google/cloud-sdk:alpine
  script:
    - cp . $(pwd)/
    - echo $GCLOUD_SERVICE_KEY | base64 -d > ${HOME}/gcloud-service-key.json
    - gcloud auth activate-service-account --key-file ${HOME}/gcloud-service-key.json
    - gcloud config set project $GCP_PROJECT_ID
    - gcloud builds submit --tag gcr.io/${GCP_PROJECT_ID}/${PROJECT_NAME}:latest
  only:
    - master

deploy_live:
  image: google/cloud-sdk:alpine
  stage: deploy
  script:
    - echo "Deploy to Cloud Run"
    - echo $GCLOUD_SERVICE_KEY | base64 -d > ${HOME}/gcloud-service-key.json
    - gcloud auth activate-service-account --key-file ${HOME}/gcloud-service-key.json
    - gcloud config set project $GCP_PROJECT_ID
    - gcloud beta run deploy --image gcr.io/${GCP_PROJECT_ID}/${PROJECT_NAME}:latest --platform managed
  dependencies:
    - image_build
  only:
    - master
```