# Hello World - Rust sample

A simple web app written in Rust that you can use for testing.
It reads in an env variable `TARGET` and prints "Hello World: ${TARGET}!". If
TARGET is not specified, it will use "NOT SPECIFIED" as the TARGET.

## Prerequisites

* A Kubernetes cluster with Knative installed. Follow the
  [installation instructions](https://github.com/knative/install/) if you need
  to create one.
* [Docker](https://www.docker.com) installed and running on your local machine,
  and a Docker Hub account configured (we'll use it for a container registry).

## Steps to recreate the sample code

While you can clone all of the code from this directory, hello world
apps are generally more useful if you build them step-by-step. The
following instructions recreate the source files from this folder.

1. Create a new file named `Cargo.toml` and paste the following code:

    ```toml
    [package]
    name = "hellorust"
    version = "0.0.0"
    publish = false

    [dependencies]
    hyper = "0.12.3"
    pretty_env_logger = "0.2.3"
    ```

1. Create a `src` folder, then create a new file named `main.rs` in that folder
   and paste the following code. This code creates a basic web server which
   listens on port 8080:

    ```rust
    #![deny(warnings)]
    extern crate hyper;
    extern crate pretty_env_logger;

    use hyper::{Body, Response, Server};
    use hyper::service::service_fn_ok;
    use hyper::rt::{self, Future};
    use std::env;

    fn main() {
        pretty_env_logger::init();

        let addr = ([0, 0, 0, 0], 8080).into();

        let new_service = || {
            service_fn_ok(|_| {

                let mut hello = "Hello world: ".to_string();
                match env::var("TARGET") {
                    Ok(target) => {hello.push_str(&target);},
                    Err(_e) => {hello.push_str("NOT SPECIFIED")},
                };

                Response::new(Body::from(hello))
            })
        };

        let server = Server::bind(&addr)
            .serve(new_service)
            .map_err(|e| eprintln!("server error: {}", e));

        println!("Listening on http://{}", addr);

        rt::run(server);
    }
    ```

1. In your project directory, create a file named `Dockerfile` and copy the code
   block below into it.

    ```docker
    FROM rust:1.27.0

    WORKDIR /usr/src/app
    COPY . .

    RUN cargo install

    EXPOSE 8080

    CMD ["hellorust"]
    ```

1. Create a new file, `service.yaml` and copy the following service definition
   into the file. Make sure to replace `{username}` with your Docker Hub username.

    ```yaml
    apiVersion: serving.knative.dev/v1alpha1
    kind: Service
    metadata:
    name: helloworld-rust
    namespace: default
    spec:
    runLatest:
        configuration:
        revisionTemplate:
            spec:
            container:
                image: docker.io/{username}/helloworld-rust
                env:
                - name: TARGET
                value: "Rust Sample v1"
    ```

## Build and deploy this sample

Once you have recreated the sample code files (or used the files in the sample
folder) you're ready to build and deploy the sample app.

1. Use Docker to build the sample code into a container. To build and push with
   Docker Hub, run these commands replacing `{username}` with your
   Docker Hub username:

    ```shell
    # Build the container on your local machine
    docker build -t {username}/helloworld-rust .

    # Push the container to docker registry
    docker push {username}/helloworld-rust
    ```

1. After the build has completed and the container is pushed to docker hub, you
   can deploy the app into your cluster. Ensure that the container image value
   in `service.yaml` matches the container you built in
   the previous step. Apply the configuration using `kubectl`:

    ```shell
    kubectl apply -f service.yaml
    ```

1. Now that your service is created, Knative will perform the following steps:
   * Create a new immutable revision for this version of the app.
   * Network programming to create a route, ingress, service, and load balance for your app.
   * Automatically scale your pods up and down (including to zero active pods).

1. To find the IP address for your service, use
   `kubectl get svc knative-ingressgateway -n istio-system` to get the ingress IP for your
   cluster. If your cluster is new, it may take sometime for the service to get asssigned
   an external IP address.

    ```shell
    kubectl get svc knative-ingressgateway -n istio-system

    NAME                     TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                                      AGE
    knative-ingressgateway   LoadBalancer   10.23.247.74   35.203.155.229   80:32380/TCP,443:32390/TCP,32400:32400/TCP   2d

    ```

1. To find the URL for your service, use
    ```
    kubectl get services.serving.knative.dev helloworld-rust  -o=custom-columns=NAME:.metadata.name,DOMAIN:.status.domain
    NAME                DOMAIN
    helloworld-rust     helloworld-rust.default.example.com
    ```

1. Now you can make a request to your app to see the result. Replace
   `{IP_ADDRESS}` with the address you see returned in the previous step.

    ```shell
    curl -H "Host: helloworld-rust.default.example.com" http://{IP_ADDRESS}
    Hello World!
    ```

## Removing the sample app deployment

To remove the sample app from your cluster, delete the service record:

```shell
kubectl delete -f service.yaml
```
