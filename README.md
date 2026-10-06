# Nebula — Cloud-Native Distributed Compute Platform

Nebula is a scalable, cloud-native distributed computing platform. More specifically, it is a Kubernetes-based architecture for running custom asynchronous dockerized tasks in a Docker-In-Docker environment.

Author: [sumansingh20](https://github.com/sumansingh20) (Suman Singh).

Nebula was created as a personal project by Suman to learn more about distributed systems, stateless APIs, containerization, event-based architectures, Docker, and scalable Kubernetes deployments. Using Nebula, clients can upload a Dockerfile and any files needed by that Dockerfile to a REST API. Nebula will build and run that Docker image, and once the image exits, the resulting files will be available for retrieval through a REST API.

## Architecture
### Overview
![Nebula Architecture](./Nebula%20Architecture.jpg)


Task Manager pods, also known as the Master API, contain a RESTful API for uploading & monitoring tasks, as well as retrieving results. The Master API is responsible for uploading task files and information to a PostgreSQL DB, and queueing up the task by ID to a Kafka Topic. Task Manager pods are easily horizontally scalable.

Task are queued up using an Apache Kafka cluster. Each broker is a separate Kubernetes pod managed by Kafka KRaft, and for fault tolerance all pods are responsible for managing one synchronized topic. The Kafka cluster can be horizontally scaled with more brokers.

Each worker pod acts as a Kafka consumer through a Springboot application. Once a task is consumed from the queue, the consuming worker is responsible for retrieving the relevant task files from PostgreSQL, updating task status in PostgreSQL, mounting and executing the dockerized task using a Docker-In-Docker architecture, and returning logs and output files to PostgreSQL. These results can then be monitored by clients through the master API. Worker pods are also horizontally scalable.

### Worker Pods
![Nebula Architecture - Worker](./Nebula%20Architecture%20-%20Worker.jpg)


To support custom dockerized tasks, each worker pod runs its own containerized Docker Daemon and Springboot application. By having a daemon local to each pod, all systems remain containerized. This also reduces security vulnerabilities stemming from running custom docker files on a system-wide docker daemon. The Spring application is responsible for orchestrating the docker-in-docker setup, consuming task ids from a kafka queue, cleaning up files and images in between running tasks, as well as fetching and updating task information from PostgreSQL. Since the task container is run using the daemon, a shared output volume is mounted pod-wide as both the Spring app and the task require access to the volume, and the daemon is responsible for mounting the volume to the task container.

Note: Nebula automatically mounts an output folder to every task at `/output/`. Only files in this volume will be saved and retrievable after task completion. Standard output and error streams are always saved as log files.

## Components
| Component | Module | Description |
| --- | --- | --- |
| Nebula Master API | `master/` | REST API for submitting, monitoring and retrieving tasks |
| Nebula Worker Service | `worker/` | Kafka consumer that builds and runs dockerized tasks (Docker-In-Docker) |
| Nebula Kafka (KRaft) | `kafka/` | Event queue used to distribute job ids to workers |
| Nebula Kubernetes manifests | `kube/` | Deployment, Service and storage manifests for the whole platform |

Project coordinates: `groupId` `com.nebula`, artifacts `master` (Nebula Master API) and `worker` (Nebula Worker Service).

## Running and Testing
Running Nebula can be done on any Kubernetes cluster. An example deployment can be created locally using [minikube](https://minikube.sigs.k8s.io/docs/start/):

### Starting the cluster
```
minikube start --nodes 3
kubectl label nodes minikube-m03 node-role.kubernetes.io/master-node=master-node
kubectl label nodes minikube-m03 role=master-node
kubectl label node minikube-m02 node-role.kubernetes.io/worker=worker
kubectl label node minikube-m02 role=worker
```
Note that pods will only deploy once a correctly labeled node is available.

### Building the Nebula images
The manifests reference locally built images (`nebula-master:1.0.3`, `nebula-worker:1.0.8`, `nebula-kafka-kraft:1.0.1`), so build them inside the minikube cluster first:
```
mvn -f master/pom.xml package -DskipTests
mvn -f worker/pom.xml package -DskipTests
eval $(minikube docker-env)
docker build -t nebula-master:1.0.3 master
docker build -t nebula-worker:1.0.8 worker
docker build -t nebula-kafka-kraft:1.0.1 kafka
```

### Initial Launch
```
cd kube
kubectl apply -f kafka.yaml
kubectl apply -f postgres.yaml
```
Once Kafka and Postgres are running, the Kafka topic needs to be set up before connecting any master or worker pods:
```
kubectl exec -it kafka-0 -- /bin/sh
kafka-topics.sh --create --topic jobIdTopic--partitions 3 --replication-factor 3 --bootstrap-server localhost:9092
kafka-topics.sh --describe --topic jobIdTopic--bootstrap-server localhost:9092
```
Once this is complete, run `exit` to get out of the Kafka pod, and apply the rest of the services:
```
kubectl apply -f master.yaml
kubectl apply -f worker.yaml
kubectl apply -f postgres.yaml
```
To expose the master REST API outside of the cluster, run:
```
minikube tunnel
```
Nebula should now be accessible at localhost port 80. Note some more useful commands for the kubernetes deployment are available in `kube/commands.txt`.

### Running a test job
A test dockerfile and job is available in `master/DemoResources`. If you would like to run the test job quickly to see how it works, you can send a POST request to `/submitDemo`, which returns the job ID. The job ID can then be used to monitor status with a GET request to the `/getJobStatus` endpoint, and results can be received with a GET request to the `/getResultingFiles` endpoint.

Alternatively, `master/DemoResources/demo-request.py` contains a Python script to submit the same custom job to Nebula using the fully-fledged `/submit` API endpoint, and prints the job id of the submitted job.

## Limitations and Improvements
It is currently difficult to monitor job completion for clients calling the REST API. A webhook should be implemented in the master API that can notify users of any changes to job status.
Moreover both the master and worker springboot APIs currently lack any unit testing. Althogh a complete systems test can be ran using the provided sample job.

## Notes
Diagram sources for the architecture above are in `Nebula Architecture.drawio` and `Nebula Architecture - Worker.drawio`.
