# Deployment_to_Serverless_Cheatshee
This is a cheatsheet for use case you want to convert your OpenShift Deployment to OpenShift Serverless Deployment(aka knative service,ksvc)
## Check list for you deployment:
1. **SIGTERM Handling**
    SIGTERM Handling Knative sends a SIGTERM signal to your pod when autoscaling to zero. It's crucial that your pod handles SIGTERM gracefully to exit accordingly; otherwise, it lingers without exiting and won't receive traffic.

    **Quick check for SIGTERM handling:**
    ```bash
    oc delete pod your-pod-name --grace-period=30
    ```
    If your pod does not exit within 30 seconds, it doesn't respond to SIGTERM properly and needs improvement. Modern frameworks like Quarkus and Spring Boot handle SIGTERM. For Python Flask, use Gunicorn:

    ```bash
    FROM your-base-image-for-python
    ...
    CMD ["gunicorn", "--workers", "3", "--bind", "0.0.0.0:8080", "app:app"]
    ```

    For Node.js using Express.js, test SIGTERM handling and use the following example if needed:
    ```javascript
    // Start Express Server
    const server = app.listen(port, () => {
        logger.info(`Server running on http://localhost:${port}`);
    });

    process.on('SIGTERM', () => {
        logger.info({ message: 'SIGTERM signal received: closing HTTP server, graceful shutdown'});
        // Close any resources ocuppied, graceful shutdown
        server.close(() => {
        logger.info({ message: 'HTTP server closed'});
        });
        process.exit(0);
    });
    ```
1. **Long Process Request Handling**

    Knative defaults to terminating your pod after 30 seconds (param: scale-to-zero-grace-period). To handle longer requests, configure per revision by tuning scale-to-zero-pod-retention-period:

    If it's hard to decide how long to keep the retention-period, it's possible to send a probe to your ksvc from your pod, send HTTP GET to http://your-knative-internal-ksvc-endpoint, eg: 
    ```bash
    curl http://knative-hello-world.my-namespace.svc.cluster.local/health
    ```
1. Startup Time
    
    Knative autoscaling requires attention to startup time. Tips for faster startup:
    * Use IfNotPresent instead of Always for pull Policy
    ```yaml
    imagePullPolicy: IfNotPresent 
    ```
    * Use native builds for Java if possible to reduce startup time (e.g., Quarkus).

    * Use smaller base images (e.g., ubi, ubi-minimal).
        Tips: Search and download ubi, ubi-minimal for your base image choice from https://access.redhat.com/sites/default/files/spas/container-catalog-spa/

1. HealthCheck

    Frequent up and down scaling necessitates health and readiness checks to maintain service SLA. Ensure your workload has proper health checks before moving to Knative.

1. Internal service endpoint

    Knative services expose port 80 by default. (Not configurable as classic k8s service) Ensure the correct port is used for Knative private service references.

1. Parallel running for your deployment and Serverless deployment

    When transitioning from deployment to serverless, consider:

    * Use different names for deployment and Knative services to avoid conflicts.

    * Avoid duplicate request/event handling in event-driven architectures.

    * Ensure both deployments handle state persistence if applicable.

1. Application scalability
    
    This is to ensure your application architecture to allow scalability because by default knative service would autoscale to handle large traffic or system load.  

1. Application Route

    Knative services expose routes by default. Disable it if you don't want the route exposed.

1. Event Driven

    Knative services are a great fit for event-driven architecture. However, Knative services need to be configured to receive events even when they are not active. To achieve this, configure an event source that connects to your existing event store, such as Kafka. When the event source receives a message from Kafka, it will send an HTTP POST request to your application, which should be configured as a Sink in the Knative eventing context.

    Configure your application with a POST method at the "/" path so that the Knative event source can automatically send events to your service. This approach requires minimal changes to your existing event-driven application, as the POST method serves as a way to receive messages without handling them. Ensure that the consumer-group-id for your consumer application and the event source are different to avoid conflicts.

    An alternative approach is to use Knative’s POST method to both receive and handle message consumption. This eliminates the need for your existing consumer logic, allowing the application to receive messages via POST without integrating directly with the event store like Kafka. This is ideal for new applications but may require refactoring for existing ones.


1. API Gateway integration

    If you need to integrate your knative svc to api gateway, there is one thing you need to check: health check from your gateway to your backend service. The default grace-period is 30s, so your healthcheck configuration (if applied) needs to be longer than that. otherwise your knative svc will not scale to zero before next health check is probing.

1. Namespace level network isolation

If your need to configure networkpolicy for namespace level isolation for your deployment, you need to configure the networkpolicy in knative-serving-ingress, this is because all knative svc are route through the interal proxy in knative-serving-ingress, so apply the networkpolicy in knative-serving-ingress namespace instead of the application project.

# Cheatsheet for configure knative service
1. Configure ksvc retention time
    Configure scale-to-zero-pod-retention-period for linger longer to use before terminating. system will decide to terminate by using scale-to-zero-grace-period(default to 30s)
    ```yaml
    apiVersion: serving.knative.dev/v1
    kind: Service
    metadata:
    ...
    spec:
    template:
        metadata:
        annotations:
            autoscaling.knative.dev/scale-to-zero-pod-retention-period: "200s" # stay 200s more after system decide to termiate

    ```

1. Configure Concurrent target per revision
   ```yaml
    apiVersion: serving.knative.dev/v1
    kind: Service
    metadata:
    ...
    spec:
    template:
        metadata:
        annotations:
            autoscaling.knative.dev/target: "70" # for 70 concurrency request per pod


    ```

1. Disable scale to zero per revision
    ```yaml
    apiVersion: serving.knative.dev/v1
    kind: Service
    metadata:
    ...
    spec:
    template:
        metadata:
        annotations:
            autoscaling.knative.dev/min-scale: "3"
    ```
    Use *autoscaling.knative.dev/max-scale* to controle the max replica it can scale

1. Configure private knative svc
    ```yaml
    apiVersion: serving.knative.dev/v1
    kind: Service
    metadata:
      Labels:
        networking.knative.dev/visibility: cluster-local
    ...
    spec:
    ...
    ```

1. Configure networkpolicy for knative-serving-ingress

    This is a sample, please modify and test according to your requirement:

    ```yaml
        kind: NetworkPolicy
        apiVersion: networking.k8s.io/v1
        metadata:
        name: deny-by-default
        namespace: knative-serving-ingress
        spec:
        podSelector:
        ingress: []
        ---
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        metadata:
        name: allow-knative-system-namespace
        namespace: knative-serving-ingress
        spec:
        ingress:
        - from:
            - namespaceSelector:
                matchLabels:
                knative.openshift.io/system-namespace: "true"
        podSelector: {}
        policyTypes:
        - Ingress
        ---
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        metadata:
        name: allow-apigateway-namespace
        namespace: knative-serving-ingress
        spec:
        ingress:
        - from:
            - namespaceSelector:
                matchLabels:
                knative.openshift.io/apigateway-namespace: "true"
        podSelector: {}
        policyTypes:
        - Ingress
        ---
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        metadata:
        name: allow-managed-namespace
        namespace: knative-serving-ingress
        spec:
        ingress:
        - from:
            - namespaceSelector:
                matchLabels:
                knative.openshift.io/knative-project-namespace: "true"
        podSelector: {}
        policyTypes:
        - Ingress
        ---
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        metadata:
        name: allow-openshift-ingress-namespace
        namespace: knative-serving-ingress
        spec:
        ingress:
        - from:
            - namespaceSelector:
                matchLabels:
                kubernetes.io/metadata.name: "openshift-ingress"
        podSelector: {}
        policyTypes:
        - Ingress
    ```


