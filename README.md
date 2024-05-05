# Coworking Space Service Extension
The Coworking Space Service is a set of APIs that enables users to request one-time tokens and administrators to authorize access to a coworking space. This service follows a microservice pattern and the APIs are split into distinct services that can be deployed and managed independently of one another.

For this project, you are a DevOps engineer who will be collaborating with a team that is building an API for business analysts. The API provides business analysts basic analytics data on user activity in the service. The application they provide you functions as expected locally and you are expected to help build a pipeline to deploy it in Kubernetes.

## Getting Started

### Dependencies
#### Local Environment
1. Python Environment - run Python 3.6+ applications and install Python dependencies via `pip`
2. Docker CLI - build and run Docker images locally
3. `kubectl` - run commands against a Kubernetes cluster

#### Remote Resources
1. AWS CodeBuild - build Docker images remotely
2. AWS ECR - host Docker images
3. Kubernetes Environment with AWS EKS - run applications in k8s
4. AWS CloudWatch - monitor activity and logs in EKS
5. GitHub - pull and clone code

### Setup
#### 1. Configure a Database

1. Set up a Postgres database.
```bash
kubectl apply -f deployment/database
```

2. Test Database Connection
The database is accessible within the cluster. This means that when you will have some issues connecting to it via your local environment. You can either connect to a pod that has access to the cluster _or_ connect remotely via [`Port Forwarding`](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)

* Connecting Via Port Forwarding
```bash
kubectl port-forward --namespace default svc/<SERVICE_NAME>-postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432
```

* Connecting Via a Pod
```bash
kubectl exec -it <POD_NAME> bash
PGPASSWORD="<PASSWORD HERE>" psql postgres://postgres@<SERVICE_NAME>:5432/postgres -c <COMMAND_HERE>
```

4. Run Seed Files
We will need to run the seed files in `db/` in order to create the tables and populate them with data.

```bash
kubectl port-forward --namespace default svc/<SERVICE_NAME>-postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432 < 1_create_tables.sql
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432 < 2_seed_users.sql
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432 < 3_seed_tokens.sql
```

### 2. Running the Analytics Application Locally
In the `analytics/` directory:

1. Install dependencies
```bash
pip install -r requirements.txt
```
2. Run the application (see below regarding environment variables)
```bash
<ENV_VARS> python app.py
```

There are multiple ways to set environment variables in a command. They can be set per session by running `export KEY=VAL` in the command line or they can be prepended into your command.

* `DB_USERNAME`
* `DB_PASSWORD`
* `DB_HOST` (defaults to `127.0.0.1`)
* `DB_PORT` (defaults to `5432`)
* `DB_NAME` (defaults to `postgres`)

If we set the environment variables by prepending them, it would look like the following:
```bash
DB_USERNAME=username_here DB_PASSWORD=password_here python app.py
```
The benefit here is that it's explicitly set. However, note that the `DB_PASSWORD` value is now recorded in the session's history in plaintext. There are several ways to work around this including setting environment variables in a file and sourcing them in a terminal session.

3. Verifying The Application
* Generate report for check-ins grouped by dates
`curl <BASE_URL>/api/reports/daily_usage`

* Generate report for check-ins grouped by users
`curl <BASE_URL>/api/reports/user_visits`

### 3. Deploy the Analytics Application
1. Dockerize the Application

* Build Docker image
```bash
docker build -t test-coworking-analytics .
```

* Verify the Docker Image
```bash
docker run --network="host" test-coworking-analytics
```

2. Set up Continuous Integration with CodeBuild

- S1: Create and add contents with pre-build, build, post-build for file buildspec.yaml in the repository
- S2: Create an Amazon ECR repository on your AWS console
- S3: create an Amazon CodeBuild project that is connected to the GitHub repository
- S4: Click on the "Start Build" button on your CodeBuild console and then check out Amazon ECR to see if the Docker image is created/updated

3. Deploy the Application
* ConfigMap:
    - DB_HOST is the name of the service that you get from running kubectl get svc
    - DB_USERNAME and DB_NAME are the values you set up earlier while configuring the Database service.
    - DB_PORT should be set to 5432 instead of 5433 since we are not working with a forwarded port this time.

* Secret: 
    - DB_PASSWORD is the base64 hash code of the password

* Deployment
    - Update file `deployment/coworking.yaml`
    - Run the command to apply
    ```bash
    kubectl apply -f deployment/configmap.yaml
    kubectl apply -f deployment/coworking.yaml
    ```
    - Verify the Deployment
    ```bash
    kubectl get svc
    ```

4. Setup CloudWatch Logging
- S1: Attach the CloudWatchAgentServerPolicy IAM policy to the worker nodes
```bash
aws iam attach-role-policy \
--role-name <WORKER_NODE_ROLE> \
--policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
```

- S2: Use AWS CLI to install the Amazon CloudWatch Observability EKS add-on
```bash
aws eks create-addon --addon-name amazon-cloudwatch-observability --cluster-name <CLUSTER_NAME>
```

- S3: Trigger logging by accessing the application

- S4: Open up and check CloudWatch Log groups page