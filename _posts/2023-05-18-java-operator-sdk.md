---
title: "Java Operator SDK: Scaffolding Kubernetes operators in Java"
date: 2023-05-18
---

## What is a Kubernetes operator?

Operators are software extensions to Kubernetes that make use of custom resources to manage applications and their components. Operators follow Kubernetes principles, notably the control loop.
The operator pattern concept lets you extend the cluster's behaviour without modifying the code of Kubernetes itself by linking controllers to one or more custom resources. Operators are clients of the Kubernetes API that act as controllers for a Custom Resource.

The aim is to code the knowledge of a human operator who is managing a service or set of services. Human operators who look after specific applications and services have a deep knowledge of how the system ought to behave, how to deploy it, and how to react if there are problems.

Additional information can be found here:
- https://kubernetes.io/docs/concepts/architecture/controller/
- https://kubernetes.io/docs/concepts/extend-kubernetes/#extension-patterns
- https://kubernetes.io/docs/concepts/extend-kubernetes/operator/

## Why use Java?

While Golang remains the most widely used language for implementing operators and controllers, not everyone is necessarily familiar with its concepts or its pointers and references style similar to C.

Java is very common in the software world. It uses a Virtual Machine to seperate the programmer from the hardware and its Object-Oriented concepts highly human-readable.
Also, the popular Kubernetes client used in Java - `Fabric8` - has capabilites resembling Golang's clients. We will touch on this in a bit.

## The Java Operator SDK

The [Operator SDK](https://sdk.operatorframework.io/docs/overview/) is capable of automatically generating a lot of the boiler-plate code needed for operator implementation. This allows the user to focus on modeling and coding the knowledge, without worrying about network interaction with Kubernetes.
Plugins are supported to extend the SDK's options. We will focus specifically on the `quarkus` plugin.

### Wait... Quarkus?

The [Quarkus Java Framework](https://quarkus.io/) stands for fast application start-up times, low memory consumption, and lower space requirements for native images. This should get our operator up and handling our custom resources efficiently.
Quarkus also uses [GraalVM](https://www.graalvm.org/) which supports changing code while the application is running.

## Let's get started

### Prerequisites
- Operator SDK and Maven should be installed on your system. If you are using MacOS these can be installed with `brew`. For Linux use the package managers available for your distribution such as `dnf` or `apt`, or download them from their websites: [Operator SDK][OSDK], [Maven][MVN].
- Connection to a Kubernetes or OpenShift cluster via Kube Config.
- An IDE you are comfortable with should be available. In this example we will use VSCode with common Java extensions.
- It is recommended to familiarize yourself with the [Group Version Kind](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.27/#-strong-api-groups-strong-) concept of Kubernetes resources.

[OSDK]: https://sdk.operatorframework.io/docs/installation/#install-from-github-release
[MVN]: https://maven.apache.org/download.cgi

### Step 0: Defining the use case

The company we work with `Example.com` has a product named `Echo` which repeats a user's input.
We would like to code the knowledge needed to repeat the input in a new Kubernetes Custom Resource. 

### Step 1: Scaffolding our first Java Operator

Create an empty directory named `echo-operator`, and `cd` into it, then run
```
operator-sdk init --plugins quarkus --domain example.com --project-name echo-operator
```
Next, we'll follow up with scaffolding our API and open the IDE
```
operator-sdk create api --group example --version v1 --kind EchoResource
```
![image](https://github.com/itroyano/blogs/assets/83590748/db6b2537-951c-4cd7-99c0-c52537561480)

### Step 2: Custom Resources in Java code

Focusing on the files created under `src/main/java`, let's look at the structure bottom-to-top -

1. `EchoResourceSpec` is the spec inside our Custom Resource. this is where the users will be providing their input.
   Add the following field with a Getter and Setter:
   ```
   private String inputMessage;

    public String getInputMessage() {
        return inputMessage;
    }

    public void setInputMessage(String inputMessage) {
        this.inputMessage = inputMessage;
    }
   ```
2. `EchoResourceStatus` is the output status our operator returns back to the user.
   Add the following output field:
   ```
    private String echoMessage;

    public String getEchoMessage() {
        return echoMessage;
    }

    public void setEchoMessage(String echoMessage) {
        this.echoMessage = echoMessage;
    }
   ``` 
3. `EchoResource` is the Java class representing our Custom Resource. it extends the `Fabric8` class `CustomResource` for our spec and status.
   No changes required here.

Finally, we'll simulate an Echo Resource input given by the user.

Create a new file named `cr-test-echo-resource.yaml` under `src/test/resources` and paste the content:
```
apiVersion: example.example.com/v1
kind: EchoResource
metadata:
  name: test-echo-resource
spec:
  inputMessage: "Hello from test-echo-resource"
```

### Step 3: The Reconciler and how Operator SDK helps us implement it

Let's open the file named `EchoResourceReconciler`.

This class implements Operator SDK's `Reconciler<T>` method for our Echo Resource. It is required to implement the method `reconcile()`.

We will implement the control loop here with our knowledge about the `Echo` product, which repeats the user's input.

![image](https://github.com/itroyano/blogs/assets/83590748/9c76a290-014d-42a3-a071-92f40dfd358f)

Add the following code for convenience:

```
private static final Logger log = LoggerFactory.getLogger(EchoResourceReconciler.class);
```

Then the implementation of `reconcile`:

```
log.info("This is the control loop of the echo-operator. resource message is {}", resource.getSpec().getInputMessage());
if (reconcileStatus(resource,context)){
      return UpdateControl.updateStatus(resource);
}
return UpdateControl.noUpdate();
```

And finally the implementation of handling status:
```
private boolean reconcileStatus(EchoResource resource, Context<EchoResource> context) {
    String desiredMsg = resource.getSpec().getInputMessage();
    if (resource.getStatus() == null){
      // initialize if needed
      resource.setStatus(new EchoResourceStatus());
      resource.getStatus().setEchoMessage("");
    }
    if (!resource.getStatus().getEchoMessage().equalsIgnoreCase(desiredMsg)){
       // the status needs to be updated with a new echo message
       resource.getStatus().setEchoMessage(desiredMsg);
       log.info("Setting echo resource status message to {}", desiredMsg);
       // return true to signal the need to update status in Kubernetes
       return true;
    }
    return false;
  }
```

### Step 4: Testing out a custom resouce (with live coding)

For convenience, we will instruct Quarkus to create the Custom Resource Definition on our cluster, in case it doesn't exist.
Open `src/main/resources/application.properties` and change `quarkus.operator-sdk.crd.apply` to `true`.

Now let's run Quarkus via Maven: `mvn clean compile && mvn quarkus:dev`.

The controller is now running and ready to accept user input. We can follow up with our test resource: `kubectl apply -f src/test/resources/cr-test-echo-resource.yaml`.

Observe the output and check the `status` inside the `EchoResource` on the cluster.
You can also change the input spec message again and see it updated in status.

And we can change the code too! going back to `EchoResourceReconciler` modify the log message to `This is the reconciler of the echo-operator` and press `r` in the Quarkus terminal.

Feel free to change and experiment. when done, exit out of Quarkus using `q` and clean up the test resource with `kubectl delete -f src/test/resources/cr-test-echo-resource.yaml`.

## Wrap Up!

We have implemented a very basic operator using the Java Operator SDK, Quarkus and the Fabric8 Kubernetes Client.

Please re-run, modify code, experiment and look at the files generated.

Fabric8 is capable of also creating other resources in Kubernetes, via either a Builder Pattern or by reading an input template yaml.
Take a look at these code snippets for additional experimentation:

- Building a Service with Fabric8's builders:
  ```
  private boolean reconcileService(EchoResource resource, Context<EchoResource> context) {
    String desiredName = resource.getMetadata().getName();

    Service echoService = client.services().withName(desiredName).get();
    if (echoService == null){
      log.info("Creating a service {}", desiredName);
      Map<String,String> labels = createLabels(desiredName);

      echoService = new ServiceBuilder()
        .withMetadata(createMetadata(resource, labels))
        .withNewSpec()
            .addNewPort()
                .withName("http")
                .withPort(8080)
            .endPort()
            .withSelector(labels)
            .withType("ClusterIP")
        .endSpec()
        .build();

    client.services().resource(echoService).createOrReplace();
    return true;
    }
    return false;
  }
  
  private Map<String, String> createLabels(String labelValue) {
    Map<String,String> labelsMap = new HashMap<>();
    labelsMap.put("owner", labelValue);
    return labelsMap;
  }
  
  private ObjectMeta createMetadata(EchoResource resource, Map<String, String> labels){
    final var metadata=resource.getMetadata();
    return new ObjectMetaBuilder()
        .withName(metadata.getName())
        .addNewOwnerReference()
            .withUid(metadata.getUid())
            .withApiVersion(resource.getApiVersion())
            .withName(metadata.getName())
            .withKind(resource.getKind())
        .endOwnerReference()
        .withLabels(labels)
    .build();
  }
  ```
 - Parsing and applying a yaml with a Kubernetes resource: 
  ```
  private void createFromYaml(String pathToYaml) throws FileNotFoundException {
  // Parse a yaml into a list of Kubernetes resources
  List<HasMetadata> result = client.load(new FileInputStream(pathToYaml)).get();
  // Apply Kubernetes Resources
  client.resourceList(result).createOrReplace();
  }
  ```
