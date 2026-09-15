# Kubernetes + Java on Your Local Machine

This gets you from zero to a Java app running in a local Kubernetes cluster, while teaching you the core concepts along the way.

---

## Part 1: Kubernetes Concepts

Kubernetes (K8s) is a system for running containers reliably at scale. You describe the *desired state* ("I want 3 copies of my app running"), and Kubernetes constantly works to make reality match that state.

| Concept | What it is | Analogy |
|---|---|---|
| **Cluster** | A set of machines running Kubernetes | The whole factory |
| **Node** | A single machine (VM or physical) in the cluster | One factory floor |
| **Pod** | The smallest deployable unit — one or more containers that share networking/storage | A worker (or worker + assistant) |
| **Deployment** | Manages a set of identical Pods, handles rollouts & self-healing | The shift manager — keeps N workers always on duty |
| **Service** | A stable network endpoint that routes traffic to Pods (Pods die/restart with new IPs, Services don't) | The factory's front desk/phone number |
| **ReplicaSet** | Ensures a specific number of Pod replicas exist (managed by Deployment, rarely touched directly) | Headcount tracker |
| **kubectl** | The CLI you use to talk to the cluster | Your control panel |
| **YAML manifest** | A file describing desired state (a Deployment, Service, etc.) | The blueprint |
| **Namespace** | A way to partition a cluster into virtual sub-clusters | Departments in the factory |

**The core loop:** You write a YAML file -> `kubectl apply -f file.yaml` -> Kubernetes' control plane compares desired vs actual state → it creates/kills Pods to match.

---

## Part 2: Install Everything (Local Setup)

You need three things: a container tool (Docker), a local cluster (Minikube), and the CLI (kubectl).

### macOS
```bash
brew install --cask docker
brew install kubectl
brew install minikube
```

### Windows (PowerShell, with Chocolatey)
```powershell
choco install docker-desktop
choco install kubernetes-cli
choco install minikube
```

### Linux (Debian/Ubuntu)
```bash
# Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER   # log out/in after this

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

### Start your cluster
```bash
minikube start
kubectl get nodes    # should show one node, status "Ready"
```

You now have a real (single-node) Kubernetes cluster running locally inside a VM/container managed by Minikube.

---

## Part 3: A Minimal Java App

We'll use a plain Spring Boot app with one REST endpoint. No need to install Java/Maven locally — we'll build it inside Docker.

### Project structure
```
k8s-java-demo/
├── pom.xml
├── Dockerfile
├── deployment.yaml
├── service.yaml
└── src/main/java/com/example/demo/DemoApplication.java
```

Create the folder and files:

```bash
mkdir -p k8s-java-demo/src/main/java/com/example/demo
cd k8s-java-demo
```

### `pom.xml`
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.4</version>
  </parent>
  <groupId>com.example</groupId>
  <artifactId>demo</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <properties>
    <java.version>17</java.version>
  </properties>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
  </dependencies>
  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
</project>
```

### `src/main/java/com/example/demo/DemoApplication.java`
```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@RestController
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

    @GetMapping("/")
    public String hello() {
        return "Hello from Java running inside Kubernetes!";
    }

    @GetMapping("/health")
    public String health() {
        return "OK";
    }
}
```

### `Dockerfile` (multi-stage build — no local Maven/JDK needed)
```dockerfile
# Stage 1: build
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: run
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## Part 4: Build the Image with Local Docker, Load into Minikube

Build the image with your normal, regular Docker — no special environment switching needed:

```bash
docker build -t java-demo:1.0 .
```

Then load it directly into the cluster:

```bash
minikube image load java-demo:1.0
```

Confirm it landed:
```bash
minikube image ls | grep java-demo
```

**Every time you change the code and rebuild, repeat both steps** (`docker build` then `minikube image load`) and then restart the deployment so it picks up the new image:

```bash
docker build -t java-demo:1.0 .
minikube image load java-demo:1.0
kubectl rollout restart deployment java-demo
```

---

## Part 5: Kubernetes Manifests

### `deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-demo
  labels:
    app: java-demo
spec:
  replicas: 2                  # run 2 copies — try killing one and watch it self-heal
  selector:
    matchLabels:
      app: java-demo
  template:
    metadata:
      labels:
        app: java-demo
    spec:
      containers:
        - name: java-demo
          image: java-demo:1.0
          imagePullPolicy: Never   # use the local image we just built, don't fetch from a registry
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 5
```

### `service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: java-demo-service
spec:
  type: NodePort
  selector:
    app: java-demo
  ports:
    - port: 80
      targetPort: 8080
```

**What these do:**
- The **Deployment** tells Kubernetes: "always keep 2 Pods running this image." If one crashes, it's replaced automatically.
- The **readinessProbe** stops traffic from reaching a Pod until `/health` responds — so a slow-starting JVM doesn't get requests too early.
- The **Service** gives you one stable address that load-balances across whichever Pods are currently healthy.

---

## Part 6: Deploy and Verify

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# Watch pods come up
kubectl get pods -w
```

Once pods show `Running` and `1/1 Ready`, open the service:

```bash
minikube service java-demo-service
```

This opens your browser to the app — you should see `Hello from Java running inside Kubernetes!`.

### Useful commands to explore what's happening
```bash
kubectl get deployments              # see Deployment status
kubectl get pods                     # see individual Pods
kubectl describe pod <pod-name>      # detailed info, events, errors
kubectl logs <pod-name>              # app logs (great for debugging Spring Boot startup)
kubectl get svc                      # see Services and their ports
```

### See self-healing in action
```bash
kubectl get pods                     # note a pod name
kubectl delete pod <pod-name>        # kill it
kubectl get pods -w                  # watch Kubernetes immediately spin up a replacement
```

### Scale it live
```bash
kubectl scale deployment java-demo --replicas=4
kubectl get pods
```

### Clean up
```bash
kubectl delete -f service.yaml
kubectl delete -f deployment.yaml
minikube stop        # or `minikube delete` to remove the cluster entirely
```

---

## Part 7: What to Learn Next

Once this is working, these are the natural next concepts, roughly in order:

1. **ConfigMaps & Secrets** — externalize config (DB URLs, API keys) instead of hardcoding them in the image.
2. **Environment-specific config** — using `env:` in the Deployment to feed Spring profiles (`SPRING_PROFILES_ACTIVE`).
3. **Persistent Volumes** — for apps that need to store data beyond a Pod's lifetime.
4. **Ingress** — nicer HTTP routing than NodePort, with hostnames and paths (`minikube addons enable ingress`).
5. **Helm** — a package manager for Kubernetes manifests; stops you copy-pasting YAML.
6. **Liveness probes** — distinct from readiness; restarts a Pod if it's stuck, not just unready.
7. **Horizontal Pod Autoscaler** — auto-scale replicas based on CPU/memory load.
8. **`kubectl exec -it <pod> -- sh`** — shell into a running container to debug directly.

---

## Quick Reference Cheat Sheet

```bash
minikube start / stop / delete           # cluster lifecycle
docker build -t java-demo:1.0 .          # build image locally
minikube image load java-demo:1.0        # load image into the cluster
kubectl apply -f <file>.yaml             # create/update resources
kubectl get pods|svc|deployments         # list resources
kubectl describe <resource> <name>       # deep detail + events
kubectl logs <pod-name>                  # container logs
kubectl exec -it <pod-name> -- sh        # shell into a pod
kubectl delete -f <file>.yaml            # remove resources
kubectl scale deployment <name> --replicas=N
```
