# kubernetes-environment
## `> bookservice-kong-keycloak`

The goal of this project is to run inside [`Kubernetes`](https://kubernetes.io) ([`Minikube`](https://github.com/kubernetes/minikube)): [`book-service`](https://github.com/ivangfr/springboot-testing-mongodb-keycloak) ([`Spring Boot`](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/) application), [`Keycloak`](https://www.keycloak.org) (authentication and authorization service) and [`Kong`](https://konghq.com) (gateway service).

## Clone example repository

Clone [`springboot-testing-mongodb-keycloak`](https://github.com/ivangfr/springboot-testing-mongodb-keycloak) repository. For it, open a terminal and run
```
git clone https://github.com/ivangfr/springboot-testing-mongodb-keycloak.git
```

### book-service

`book-service` is a `Spring Boot` Web Java application that exposes some endpoints to manage books. Once deployed in `Kubernetes` cluster, it won't be exposed to outside, i.e, it won't be possible to be accessed directly from the host machine. In order to bypass it, we are going to use `Kong` as a gateway service. So, to access `book-service`, you will have to call `Kong` REST API and then, `Kong` will redirect the request to `book-service`. Furthermore, the plugin `Rate Limiting` will be installed in `Kong`. It will be configured to just allow 5 requests a minute to any `book-service` endpoints.

Besides, `book-service` implements `Keycloak` security configuration. The endpoints related to _"managing books"_, like create book (`POST /api/books`), update book (`PATCH /api/books/{id}`) and delete book (`DELETE /api/books/{id}`) will require a `Bearer JWT Access Token`.

## Start Minikube

First of all, start `Minikube` as explained in [Start Minikube](https://github.com/ivangfr/kubernetes-environment#start-minikube)

## Build Docker Image

Instead of pushing the docker image to Docker Registry, we will simply build the image using the `Minikube` Docker daemon. Below are the steps

- In a terminal, navigate to `springboot-testing-mongodb-keycloak` root folder

- Set `Minikube` host
  ```
  eval $(minikube docker-env)
  ```

- Build `book-service` docker image
  ```
  ./gradlew book-service:clean book-service:build docker -x test -x integrationTest
  ```
   
- \[Optional\] To check whether `book-service` docker image was created
  ```
  docker images
  ```

- Get back to Host machine Docker Daemon   
  ```
  eval $(minikube docker-env -u)
  ```

## Create a namespace

- In a terminal, run the following command to create a new namespace called `dev`
  ```
  kubectl create namespace dev
  ```
  > To delete run
  > ```
  > kubectl delete namespace dev
  > ```

- \[Optional\] To list all namespaces run
  ```
  kubectl get namespaces
  ```

## Install services

- In a terminal, navigate to `kubernetes-environment/bookservice-kong-keycloak` folder

- To install the services, run the script below
  ```
  ./install-services.sh
  ```
  > To uninstall run
  > ```
  > ./uninstall-services.sh
  > ```

  It will install `MySQL`, `Postgres`, `MongoDB` `Kong` and `Keycloak`. It can take some time (pulling docker images, starting services, etc).
  
- Check the status/progress of the service installation
  ```
  kubectl get pods --namespace dev
  ```

## Configure Keycloak

- Before start, check if `Keycloak` is ready
  ```
  kubectl get pods --namespace dev
  ```
  The column `READY` must show `1/1`. If it is showing `0/1`, wait a little bit

- There are two ways to configure `Keycloak`

  - **Running a script**
   
    - In a terminal, make sure you are in `springboot-testing-mongodb-keycloak` root folder
    
    - Get `KEYCLOAK_URL` environment variable
      ```
      KEYCLOAK_URL="$(minikube ip):$(kubectl get services --namespace dev my-keycloak-http -o go-template='{{(index .spec.ports 0).nodePort}}')"
      ```
    
    - Run the following script to configure `Keycloak` for `book-service` application
      ```
      ./init-keycloak.sh $KEYCLOAK_URL
      ```
      
    - Copy the `BOOK_SERVICE_CLIENT_SECRET` value printed at the end. It will be used on the next steps
  
  - **Using Keycloak website**
  
    - Run the command below to open `Keyloak` website in a browser
      ```
      minikube service my-keycloak-http --namespace dev 
      ```
      
    - Add `realm`, `client`, `client-roles` and `user` as explained in [`ivangfr/springboot-testing-mongodb-keycloak`](https://github.com/ivangfr/springboot-testing-mongodb-keycloak#using-keycloak-website)

## Create application database secrets

- In a terminal, run the command below to create a secret used by `book-service` to connect to `MongoDB`
  ```
  kubectl create secret --namespace dev generic book-service-db \
   --from-literal=username=bookuser --from-literal=password=bookpass
  ```
  > To delete run
  > ```
  > kubectl delete secrets --namespace dev book-service-db
  > ```

- \[Optional\] To list the secrets present in `dev` namespace run
  ```
  kubectl get secrets --namespace dev
  ```

- \[Optional\] To get more information about `book-service-db` secret run
  ```
  kubectl get secrets --namespace dev book-service-db -o yaml
  ```

## Install book-service

- In a terminal, make sure you are in `kubernetes-environment/bookservice-kong-keycloak` folder

- Install `book-service`
  ```
  kubectl apply --namespace dev -f deployment-files/bookservice-deployment.yaml
  ```
  > To delete run
  > ```
  > kubectl delete --namespace dev -f deployment-files/bookservice-deployment.yaml
  > ```

## Configuring Kong

- Get `KONG_ADMIN_URL` and `KONG_PROXY_URL` environment variables
  ```
  KONG_ADMIN_URL="$(minikube ip):$(kubectl get services --namespace dev my-kong-kong-admin -o go-template='{{(index .spec.ports 0).nodePort}}')"
  KONG_PROXY_URL="$(minikube ip):$(kubectl get services --namespace dev my-kong-kong-proxy -o go-template='{{(index .spec.ports 0).nodePort}}')"
  ```

- Add service `book-service`
  ```
  curl -i -X POST http://$KONG_ADMIN_URL/services/ \
    -d 'name=book-service' \
    -d 'protocol=http' \
    -d 'host=bookservice-service' \
    -d 'port=8080'
  ```

- Add `book-service` route
  ```
  curl -i -X POST http://$KONG_ADMIN_URL/services/book-service/routes/ \
    -d "protocols[]=http" \
    -d "hosts[]=book-service" \
    -d "strip_path=false"
  ```

- In order to test the added route, we will use `GET /actuator/health`
  ```
  curl -i http://$KONG_PROXY_URL/actuator/health -H 'Host: book-service'
  ```
  
  It should return
  ```
  HTTP/1.1 200
  {
    "status": "UP",
    "details": {
      "diskSpace": {
        "status": "UP",
        "details": {
        	...
        }
      },
      "mongo": {
        "status": "UP",
        "details": {
          ...
        }
      }
    }
  }
  ```

- Add Rate Limiting plugin to `book-service` service
  ```
  curl -X POST http://$KONG_ADMIN_URL/services/book-service/plugins \
    -d "name=rate-limiting"  \
    -d "config.minute=10"
  ```
   
- Make some calls to
  ```
  curl -i http://$KONG_PROXY_URL/actuator/health -H 'Host: book-service'
  ```
  
  After exceeding 10 calls in one minute, you should see
  ```
  HTTP/1.1 429 Too Many Requests
  {"message":"API rate limit exceeded"}
  ```

## Final test

- In a terminal, make sure you are in `kubernetes-environment/bookservice-kong-keycloak` folder

- Get `KONG_PROXY_URL` environment variable
  ```
  KONG_PROXY_URL="$(minikube ip):$(kubectl get services --namespace dev my-kong-kong-proxy -o go-template='{{(index .spec.ports 0).nodePort}}')"
  ```
  
- Create `BOOK_SERVICE_CLIENT_SECRET` environment variable that contains the `Client Secret` generated by `Keycloak` to `book-service` at [Configure Keycloak](#configure-keycloak)
  ```
  BOOK_SERVICE_CLIENT_SECRET=...
  ```

- Call `GET /api/books` endpoint
  ```
  curl -i http://$KONG_PROXY_URL/api/books -H 'Host: book-service'
  ```
  
  It should return
  ```
  HTTP/1.1 200
  []
  ```

- Call `POST /api/books` endpoint without access token
  ```
  curl -i -X POST http://$KONG_PROXY_URL/api/books -H 'Host: book-service' \
    -H "Content-Type: application/json" \
    -d '{ "authorName": "ivan", "title": "java 8", "price": 10.5 }'
  ```
  It should return
  ```
  HTTP/1.1 302
  ```

- Get access token from Keycloak
  ```
  BEARER_MY_ACCESS_TOKEN=$(./get-access-token.sh $BOOK_SERVICE_CLIENT_SECRET)
  ```
  > To see the access token value run
  > ```
  > echo $BEARER_MY_ACCESS_TOKEN
  > ```

- Call `POST /api/books` endpoint informing the access token
  ```
  curl -i -X POST http://$KONG_PROXY_URL/api/books -H 'Host: book-service' \
    -H "Authorization: $BEARER_MY_ACCESS_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{ "authorName": "ivan", "title": "java 8", "price": 10.5 }'
  ```
  
  It should return
  ```
  HTTP/1.1 201 
  {
    "id":"6d1270d5-716f-46b1-9a9d-e152f62464aa",
    "title":"java 8",
    "authorName":"ivan",
    "price":10.5
  }
  ```

## Cleanup

- In a terminal, make sure you are in `kubernetes-environment/bookservice-kong-keycloak` folder

- Run the script below to uninstall all services, `book-service` application and `dev` namespace.
  ```
  ./cleanup.sh
  ```
