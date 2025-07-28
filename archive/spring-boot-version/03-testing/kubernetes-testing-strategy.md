# Kubernetes Testing Strategy

This document provides comprehensive strategies for testing the First Viscount microservices platform in Kubernetes environments, specifically focusing on k3s for local development.

## Overview

Kubernetes testing validates:
- Pod lifecycle and health checks
- Service discovery and networking
- ConfigMaps and Secrets management
- Persistent volume claims
- Horizontal pod autoscaling
- Resource limits and requests
- Network policies
- Service mesh integration

## k3s Local Testing Environment

### k3s Setup for Testing

```bash
#!/bin/bash
# setup-k3s-test-env.sh

# Install k3s with testing optimizations
curl -sfL https://get.k3s.io | sh -s - \
  --write-kubeconfig-mode 644 \
  --disable traefik \
  --disable metrics-server \
  --kubelet-arg="eviction-hard=memory.available<500Mi" \
  --kubelet-arg="eviction-soft=memory.available<1Gi" \
  --kubelet-arg="eviction-soft-grace-period=memory.available=2m"

# Wait for k3s
kubectl wait --for=condition=ready nodes --all --timeout=60s

# Create test namespace
kubectl create namespace firstviscount-test
kubectl label namespace firstviscount-test environment=test

# Install test infrastructure
kubectl apply -f k8s/test-infrastructure/
```

### Test Infrastructure Configuration

```yaml
# k8s/test-infrastructure/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: firstviscount-test
  labels:
    environment: test
    istio-injection: enabled
---
# Resource quota for test namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: test-quota
  namespace: firstviscount-test
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    persistentvolumeclaims: "10"
    pods: "50"
    services: "20"
```

## Pod Testing

### Pod Lifecycle Tests

```java
@SpringBootTest
@TestPropertySource(properties = {
    "kubernetes.namespace=firstviscount-test",
    "kubernetes.test.enabled=true"
})
public class PodLifecycleTest {
    
    @Autowired
    private KubernetesClient kubernetesClient;
    
    @Test
    void testPodCreationAndReadiness() {
        // Given
        Pod pod = new PodBuilder()
            .withNewMetadata()
                .withName("test-product-service-" + UUID.randomUUID())
                .withNamespace("firstviscount-test")
                .addToLabels("app", "product-service")
                .addToLabels("test", "true")
            .endMetadata()
            .withNewSpec()
                .addNewContainer()
                    .withName("product-service")
                    .withImage("localhost:30500/product-service:test")
                    .withImagePullPolicy("Always")
                    .addNewPort()
                        .withContainerPort(8080)
                        .withName("http")
                    .endPort()
                    .withNewReadinessProbe()
                        .withNewHttpGet()
                            .withPath("/actuator/health/readiness")
                            .withPort(new IntOrString(8080))
                        .endHttpGet()
                        .withInitialDelaySeconds(10)
                        .withPeriodSeconds(5)
                    .endReadinessProbe()
                    .withNewLivenessProbe()
                        .withNewHttpGet()
                            .withPath("/actuator/health/liveness")
                            .withPort(new IntOrString(8080))
                        .endHttpGet()
                        .withInitialDelaySeconds(30)
                        .withPeriodSeconds(10)
                    .endLivenessProbe()
                    .withNewResources()
                        .addToRequests("memory", new Quantity("256Mi"))
                        .addToRequests("cpu", new Quantity("100m"))
                        .addToLimits("memory", new Quantity("512Mi"))
                        .addToLimits("cpu", new Quantity("500m"))
                    .endResources()
                .endContainer()
            .endSpec()
            .build();
        
        // When
        Pod createdPod = kubernetesClient.pods()
            .inNamespace("firstviscount-test")
            .create(pod);
        
        // Then - Wait for pod to be ready
        await().atMost(Duration.ofMinutes(2)).untilAsserted(() -> {
            Pod currentPod = kubernetesClient.pods()
                .inNamespace("firstviscount-test")
                .withName(createdPod.getMetadata().getName())
                .get();
            
            assertThat(currentPod.getStatus().getPhase()).isEqualTo("Running");
            assertThat(currentPod.getStatus().getConditions())
                .filteredOn(condition -> "Ready".equals(condition.getType()))
                .extracting(PodCondition::getStatus)
                .containsExactly("True");
        });
        
        // Cleanup
        kubernetesClient.pods()
            .inNamespace("firstviscount-test")
            .withName(createdPod.getMetadata().getName())
            .delete();
    }
    
    @Test
    void testPodAutoRestart() {
        // Create pod with failing liveness probe
        Pod pod = createPodWithFailingLiveness();
        
        // Wait for restart
        await().atMost(Duration.ofMinutes(3)).untilAsserted(() -> {
            Pod currentPod = kubernetesClient.pods()
                .inNamespace("firstviscount-test")
                .withName(pod.getMetadata().getName())
                .get();
            
            assertThat(currentPod.getStatus().getContainerStatuses())
                .extracting(ContainerStatus::getRestartCount)
                .allMatch(count -> count > 0);
        });
    }
}
```

### Pod Communication Testing

```java
@Test
void testPodToPodCommunication() {
    // Deploy two pods
    Pod clientPod = deployTestPod("client-pod", "curlimages/curl:latest");
    Pod serverPod = deployTestPod("server-pod", "nginx:alpine");
    
    // Create service for server pod
    Service service = new ServiceBuilder()
        .withNewMetadata()
            .withName("test-service")
            .withNamespace("firstviscount-test")
        .endMetadata()
        .withNewSpec()
            .addToSelector("app", "server-pod")
            .addNewPort()
                .withPort(80)
                .withTargetPort(new IntOrString(80))
            .endPort()
        .endSpec()
        .build();
    
    kubernetesClient.services()
        .inNamespace("firstviscount-test")
        .create(service);
    
    // Test communication
    String response = kubernetesClient.pods()
        .inNamespace("firstviscount-test")
        .withName(clientPod.getMetadata().getName())
        .exec("curl", "-s", "http://test-service")
        .getOutput();
    
    assertThat(response).contains("Welcome to nginx");
}
```

## Service Testing

### Service Discovery Tests

```java
@Component
@ConditionalOnProperty(name = "kubernetes.test.enabled", havingValue = "true")
public class ServiceDiscoveryTest {
    
    @Autowired
    private DiscoveryClient discoveryClient;
    
    @Autowired
    private RestTemplate restTemplate;
    
    @Test
    void testKubernetesServiceDiscovery() {
        // When - Discover services
        List<String> services = discoveryClient.getServices();
        
        // Then
        assertThat(services).contains(
            "product-service",
            "order-service",
            "inventory-service"
        );
        
        // Test service instances
        List<ServiceInstance> instances = discoveryClient.getInstances("product-service");
        assertThat(instances).isNotEmpty();
        
        ServiceInstance instance = instances.get(0);
        assertThat(instance.getHost()).isNotBlank();
        assertThat(instance.getPort()).isPositive();
    }
    
    @Test
    void testServiceToServiceCall() {
        // Using Kubernetes service DNS
        String url = "http://product-service.firstviscount-test.svc.cluster.local:8080/api/v1/products";
        
        ResponseEntity<String> response = restTemplate.getForEntity(url, String.class);
        
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody()).isNotBlank();
    }
    
    @Test
    void testHeadlessService() {
        // Create headless service
        Service headlessService = new ServiceBuilder()
            .withNewMetadata()
                .withName("test-headless")
                .withNamespace("firstviscount-test")
            .endMetadata()
            .withNewSpec()
                .withClusterIP("None")
                .addToSelector("app", "test-app")
                .addNewPort()
                    .withPort(8080)
                .endPort()
            .endSpec()
            .build();
        
        kubernetesClient.services().create(headlessService);
        
        // Test DNS resolution
        InetAddress[] addresses = InetAddress.getAllByName(
            "test-headless.firstviscount-test.svc.cluster.local"
        );
        
        assertThat(addresses).hasSizeGreaterThan(0);
    }
}
```

## ConfigMap and Secret Testing

### ConfigMap Tests

```java
@Test
void testConfigMapMounting() {
    // Create ConfigMap
    ConfigMap configMap = new ConfigMapBuilder()
        .withNewMetadata()
            .withName("test-config")
            .withNamespace("firstviscount-test")
        .endMetadata()
        .addToData("application.properties", """
            server.port=8080
            spring.application.name=test-service
            feature.flag.enabled=true
            """)
        .addToData("database.conf", """
            db.host=postgres
            db.port=5432
            db.name=testdb
            """)
        .build();
    
    kubernetesClient.configMaps().create(configMap);
    
    // Create pod with ConfigMap
    Pod pod = new PodBuilder()
        .withNewMetadata()
            .withName("config-test-pod")
            .withNamespace("firstviscount-test")
        .endMetadata()
        .withNewSpec()
            .addNewContainer()
                .withName("test-container")
                .withImage("busybox")
                .withCommand("sh", "-c", "cat /config/application.properties && sleep 3600")
                .addNewVolumeMount()
                    .withName("config-volume")
                    .withMountPath("/config")
                .endVolumeMount()
            .endContainer()
            .addNewVolume()
                .withName("config-volume")
                .withNewConfigMap()
                    .withName("test-config")
                .endConfigMap()
            .endVolume()
        .endSpec()
        .build();
    
    kubernetesClient.pods().create(pod);
    
    // Verify ConfigMap is mounted
    await().atMost(Duration.ofSeconds(30)).untilAsserted(() -> {
        String logs = kubernetesClient.pods()
            .inNamespace("firstviscount-test")
            .withName("config-test-pod")
            .getLog();
        
        assertThat(logs).contains("spring.application.name=test-service");
        assertThat(logs).contains("feature.flag.enabled=true");
    });
}

@Test
void testConfigMapHotReload() {
    // Deploy pod with config watcher
    deployPodWithConfigWatcher("test-config");
    
    // Update ConfigMap
    ConfigMap configMap = kubernetesClient.configMaps()
        .inNamespace("firstviscount-test")
        .withName("test-config")
        .get();
    
    configMap.getData().put("feature.flag.enabled", "false");
    kubernetesClient.configMaps().replace(configMap);
    
    // Verify hot reload
    await().atMost(Duration.ofSeconds(60)).untilAsserted(() -> {
        String response = callTestEndpoint("/config/feature.flag.enabled");
        assertThat(response).isEqualTo("false");
    });
}
```

### Secret Tests

```java
@Test
void testSecretManagement() {
    // Create Secret
    Secret secret = new SecretBuilder()
        .withNewMetadata()
            .withName("test-secret")
            .withNamespace("firstviscount-test")
        .endMetadata()
        .addToData("username", Base64.getEncoder().encodeToString("admin".getBytes()))
        .addToData("password", Base64.getEncoder().encodeToString("secretpass".getBytes()))
        .addToData("api-key", Base64.getEncoder().encodeToString("sk-1234567890".getBytes()))
        .build();
    
    kubernetesClient.secrets().create(secret);
    
    // Create pod with Secret as environment variables
    Pod pod = new PodBuilder()
        .withNewMetadata()
            .withName("secret-test-pod")
            .withNamespace("firstviscount-test")
        .endMetadata()
        .withNewSpec()
            .addNewContainer()
                .withName("test-container")
                .withImage("busybox")
                .withCommand("sh", "-c", "echo $DB_USERNAME && echo $DB_PASSWORD")
                .addNewEnv()
                    .withName("DB_USERNAME")
                    .withNewValueFrom()
                        .withNewSecretKeyRef()
                            .withName("test-secret")
                            .withKey("username")
                        .endSecretKeyRef()
                    .endValueFrom()
                .endEnv()
                .addNewEnv()
                    .withName("DB_PASSWORD")
                    .withNewValueFrom()
                        .withNewSecretKeyRef()
                            .withName("test-secret")
                            .withKey("password")
                        .endSecretKeyRef()
                    .endValueFrom()
                .endEnv()
            .endContainer()
        .endSpec()
        .build();
    
    kubernetesClient.pods().create(pod);
    
    // Verify secrets are injected
    await().atMost(Duration.ofSeconds(30)).untilAsserted(() -> {
        String logs = kubernetesClient.pods()
            .inNamespace("firstviscount-test")
            .withName("secret-test-pod")
            .getLog();
        
        assertThat(logs).contains("admin");
        assertThat(logs).contains("secretpass");
    });
}
```

## Persistent Volume Testing

### PVC Tests

```java
@Test
void testPersistentVolumeClaim() {
    // Create PVC
    PersistentVolumeClaim pvc = new PersistentVolumeClaimBuilder()
        .withNewMetadata()
            .withName("test-pvc")
            .withNamespace("firstviscount-test")
        .endMetadata()
        .withNewSpec()
            .withAccessModes("ReadWriteOnce")
            .withNewResources()
                .addToRequests("storage", new Quantity("1Gi"))
            .endResources()
            .withStorageClassName("local-path")
        .endSpec()
        .build();
    
    kubernetesClient.persistentVolumeClaims().create(pvc);
    
    // Wait for PVC to be bound
    await().atMost(Duration.ofSeconds(30)).untilAsserted(() -> {
        PersistentVolumeClaim currentPvc = kubernetesClient.persistentVolumeClaims()
            .inNamespace("firstviscount-test")
            .withName("test-pvc")
            .get();
        
        assertThat(currentPvc.getStatus().getPhase()).isEqualTo("Bound");
    });
    
    // Create pod with PVC
    Pod pod = new PodBuilder()
        .withNewMetadata()
            .withName("pvc-test-pod")
            .withNamespace("firstviscount-test")
        .endMetadata()
        .withNewSpec()
            .addNewContainer()
                .withName("test-container")
                .withImage("busybox")
                .withCommand("sh", "-c", 
                    "echo 'test data' > /data/test.txt && cat /data/test.txt && sleep 3600")
                .addNewVolumeMount()
                    .withName("data-volume")
                    .withMountPath("/data")
                .endVolumeMount()
            .endContainer()
            .addNewVolume()
                .withName("data-volume")
                .withNewPersistentVolumeClaim()
                    .withClaimName("test-pvc")
                .endPersistentVolumeClaim()
            .endVolume()
        .endSpec()
        .build();
    
    kubernetesClient.pods().create(pod);
    
    // Verify data persistence
    await().atMost(Duration.ofSeconds(30)).untilAsserted(() -> {
        String logs = kubernetesClient.pods()
            .inNamespace("firstviscount-test")
            .withName("pvc-test-pod")
            .getLog();
        
        assertThat(logs).contains("test data");
    });
}

@Test
void testDataPersistenceAcrossRestarts() {
    String pvcName = "persistence-test-pvc";
    String testData = "persistent-data-" + UUID.randomUUID();
    
    // Create PVC
    createPersistentVolumeClaim(pvcName, "1Gi");
    
    // Write data
    Pod writerPod = createPodWithPVC("writer-pod", pvcName, 
        String.format("echo '%s' > /data/persistent.txt", testData));
    waitForPodCompletion(writerPod);
    
    // Delete writer pod
    kubernetesClient.pods().delete(writerPod);
    
    // Read data with new pod
    Pod readerPod = createPodWithPVC("reader-pod", pvcName, 
        "cat /data/persistent.txt");
    
    String output = getPodLogs(readerPod);
    assertThat(output).contains(testData);
}
```

## Horizontal Pod Autoscaler Testing

### HPA Tests

```java
@Test
void testHorizontalPodAutoscaling() {
    // Deploy application with HPA
    Deployment deployment = new DeploymentBuilder()
        .withNewMetadata()
            .withName("hpa-test-app")
            .withNamespace("firstviscount-test")
        .endMetadata()
        .withNewSpec()
            .withReplicas(1)
            .withNewSelector()
                .addToMatchLabels("app", "hpa-test")
            .endSelector()
            .withNewTemplate()
                .withNewMetadata()
                    .addToLabels("app", "hpa-test")
                .endMetadata()
                .withNewSpec()
                    .addNewContainer()
                        .withName("stress-app")
                        .withImage("localhost:30500/stress-app:test")
                        .addNewPort()
                            .withContainerPort(8080)
                        .endPort()
                        .withNewResources()
                            .addToRequests("cpu", new Quantity("100m"))
                            .addToRequests("memory", new Quantity("128Mi"))
                            .addToLimits("cpu", new Quantity("500m"))
                            .addToLimits("memory", new Quantity("256Mi"))
                        .endResources()
                    .endContainer()
                .endSpec()
            .endTemplate()
        .endSpec()
        .build();
    
    kubernetesClient.apps().deployments().create(deployment);
    
    // Create HPA
    HorizontalPodAutoscaler hpa = new HorizontalPodAutoscalerBuilder()
        .withNewMetadata()
            .withName("hpa-test")
            .withNamespace("firstviscount-test")
        .endMetadata()
        .withNewSpec()
            .withNewScaleTargetRef()
                .withApiVersion("apps/v1")
                .withKind("Deployment")
                .withName("hpa-test-app")
            .endScaleTargetRef()
            .withMinReplicas(1)
            .withMaxReplicas(5)
            .addNewMetric()
                .withType("Resource")
                .withNewResource()
                    .withName("cpu")
                    .withNewTarget()
                        .withType("Utilization")
                        .withAverageUtilization(50)
                    .endTarget()
                .endResource()
            .endMetric()
        .endSpec()
        .build();
    
    kubernetesClient.autoscaling().v2().horizontalPodAutoscalers().create(hpa);
    
    // Generate load
    generateCPULoad("hpa-test-app");
    
    // Wait for scale up
    await().atMost(Duration.ofMinutes(5)).untilAsserted(() -> {
        Deployment current = kubernetesClient.apps().deployments()
            .inNamespace("firstviscount-test")
            .withName("hpa-test-app")
            .get();
        
        assertThat(current.getStatus().getReplicas()).isGreaterThan(1);
    });
    
    // Stop load and wait for scale down
    stopCPULoad();
    
    await().atMost(Duration.ofMinutes(10)).untilAsserted(() -> {
        Deployment current = kubernetesClient.apps().deployments()
            .inNamespace("firstviscount-test")
            .withName("hpa-test-app")
            .get();
        
        assertThat(current.getStatus().getReplicas()).isEqualTo(1);
    });
}

private void generateCPULoad(String deploymentName) {
    // Get pod
    List<Pod> pods = kubernetesClient.pods()
        .inNamespace("firstviscount-test")
        .withLabel("app", "hpa-test")
        .list()
        .getItems();
    
    // Execute CPU intensive task
    pods.forEach(pod -> {
        kubernetesClient.pods()
            .inNamespace("firstviscount-test")
            .withName(pod.getMetadata().getName())
            .exec("sh", "-c", "while true; do echo 'scale' | sha256sum; done &");
    });
}
```

## Network Policy Testing

```java
@Test
void testNetworkPolicyIsolation() {
    // Create network policy
    NetworkPolicy policy = new NetworkPolicyBuilder()
        .withNewMetadata()
            .withName("test-network-policy")
            .withNamespace("firstviscount-test")
        .endMetadata()
        .withNewSpec()
            .withNewPodSelector()
                .addToMatchLabels("app", "protected-app")
            .endPodSelector()
            .addNewIngress()
                .addNewFrom()
                    .withNewPodSelector()
                        .addToMatchLabels("app", "allowed-client")
                    .endPodSelector()
                .endFrom()
                .addNewPort()
                    .withProtocol("TCP")
                    .withPort(new IntOrString(8080))
                .endPort()
            .endIngress()
        .endSpec()
        .build();
    
    kubernetesClient.network().v1().networkPolicies().create(policy);
    
    // Deploy protected pod
    Pod protectedPod = deployPod("protected-app", Map.of("app", "protected-app"));
    
    // Deploy allowed client
    Pod allowedClient = deployPod("allowed-client", Map.of("app", "allowed-client"));
    
    // Deploy denied client
    Pod deniedClient = deployPod("denied-client", Map.of("app", "denied-client"));
    
    // Test allowed connection
    String allowedResponse = execInPod(allowedClient, 
        "curl", "-s", "-m", "5", "http://protected-app:8080");
    assertThat(allowedResponse).contains("200 OK");
    
    // Test denied connection
    assertThatThrownBy(() -> 
        execInPod(deniedClient, "curl", "-s", "-m", "5", "http://protected-app:8080")
    ).hasMessageContaining("timeout");
}
```

## Service Mesh Testing (Istio)

```java
@Test
@ConditionalOnProperty(name = "istio.enabled", havingValue = "true")
void testIstioTrafficManagement() {
    // Deploy v1 and v2 of service
    deployServiceVersion("product-service", "v1", 8080);
    deployServiceVersion("product-service", "v2", 8080);
    
    // Create VirtualService for canary deployment
    String virtualService = """
        apiVersion: networking.istio.io/v1beta1
        kind: VirtualService
        metadata:
          name: product-service
          namespace: firstviscount-test
        spec:
          hosts:
          - product-service
          http:
          - match:
            - headers:
                test-version:
                  exact: v2
            route:
            - destination:
                host: product-service
                subset: v2
          - route:
            - destination:
                host: product-service
                subset: v1
              weight: 90
            - destination:
                host: product-service
                subset: v2
              weight: 10
        """;
    
    kubectl("apply", "-f", "-", virtualService);
    
    // Test traffic distribution
    Map<String, Integer> versionCounts = new HashMap<>();
    
    for (int i = 0; i < 100; i++) {
        String response = callService("product-service", "/version");
        versionCounts.merge(response, 1, Integer::sum);
    }
    
    // Verify traffic split (allowing for some variance)
    assertThat(versionCounts.get("v1")).isBetween(80, 95);
    assertThat(versionCounts.get("v2")).isBetween(5, 20);
    
    // Test header-based routing
    String v2Response = callServiceWithHeaders("product-service", "/version", 
        Map.of("test-version", "v2"));
    assertThat(v2Response).isEqualTo("v2");
}

@Test
void testIstioCircuitBreaker() {
    // Create DestinationRule with circuit breaker
    String destinationRule = """
        apiVersion: networking.istio.io/v1beta1
        kind: DestinationRule
        metadata:
          name: product-service
          namespace: firstviscount-test
        spec:
          host: product-service
          trafficPolicy:
            connectionPool:
              tcp:
                maxConnections: 1
              http:
                http1MaxPendingRequests: 1
                http2MaxRequests: 1
            outlierDetection:
              consecutive5xxErrors: 1
              interval: 5s
              baseEjectionTime: 30s
              maxEjectionPercent: 100
        """;
    
    kubectl("apply", "-f", "-", destinationRule);
    
    // Deploy faulty service
    deployFaultyService("product-service");
    
    // Test circuit breaker activation
    List<Integer> statusCodes = new ArrayList<>();
    
    for (int i = 0; i < 10; i++) {
        try {
            callService("product-service", "/api/products");
            statusCodes.add(200);
        } catch (Exception e) {
            if (e.getMessage().contains("503")) {
                statusCodes.add(503);
            }
        }
        sleep(1000);
    }
    
    // Verify circuit breaker opened
    assertThat(statusCodes).contains(503);
}
```

## Chaos Engineering Tests

```java
@Test
@Tag("chaos")
void testPodChaos() {
    // Deploy Litmus ChaosEngine
    String chaosEngine = """
        apiVersion: litmuschaos.io/v1alpha1
        kind: ChaosEngine
        metadata:
          name: pod-delete-chaos
          namespace: firstviscount-test
        spec:
          appinfo:
            appns: firstviscount-test
            applabel: 'app=product-service'
            appkind: deployment
          chaosServiceAccount: litmus-admin
          experiments:
          - name: pod-delete
            spec:
              components:
                env:
                - name: TOTAL_CHAOS_DURATION
                  value: '60'
                - name: CHAOS_INTERVAL
                  value: '10'
                - name: FORCE
                  value: 'false'
        """;
    
    kubectl("apply", "-f", "-", chaosEngine);
    
    // Monitor application resilience
    AtomicInteger successCount = new AtomicInteger(0);
    AtomicInteger failureCount = new AtomicInteger(0);
    
    ExecutorService executor = Executors.newFixedThreadPool(10);
    
    for (int i = 0; i < 100; i++) {
        executor.submit(() -> {
            try {
                String response = callService("product-service", "/api/v1/products");
                if (response != null) {
                    successCount.incrementAndGet();
                }
            } catch (Exception e) {
                failureCount.incrementAndGet();
            }
        });
        
        sleep(1000);
    }
    
    executor.shutdown();
    executor.awaitTermination(2, TimeUnit.MINUTES);
    
    // Verify resilience
    assertThat(successCount.get()).isGreaterThan(80); // At least 80% success
    assertThat(failureCount.get()).isLessThan(20);
    
    // Cleanup
    kubectl("delete", "chaosengine", "pod-delete-chaos");
}

@Test
void testNetworkChaos() {
    // Inject network latency
    String networkChaos = """
        apiVersion: chaos-mesh.org/v1alpha1
        kind: NetworkChaos
        metadata:
          name: network-delay
          namespace: firstviscount-test
        spec:
          action: delay
          mode: all
          selector:
            namespaces:
            - firstviscount-test
            labelSelectors:
              app: order-service
          delay:
            latency: "100ms"
            jitter: "10ms"
          duration: "5m"
        """;
    
    kubectl("apply", "-f", "-", networkChaos);
    
    // Test application behavior under network delays
    long startTime = System.currentTimeMillis();
    String response = callService("order-service", "/api/v1/orders");
    long responseTime = System.currentTimeMillis() - startTime;
    
    // Verify increased latency
    assertThat(responseTime).isGreaterThan(100);
    
    // Verify application still functions
    assertThat(response).isNotNull();
}
```

## Resource Testing

```java
@Test
void testResourceLimitsEnforcement() {
    // Deploy pod with low memory limit
    Pod pod = new PodBuilder()
        .withNewMetadata()
            .withName("memory-test-pod")
            .withNamespace("firstviscount-test")
        .endMetadata()
        .withNewSpec()
            .addNewContainer()
                .withName("memory-hog")
                .withImage("progrium/stress")
                .withArgs("-m", "1", "--vm-bytes", "150M", "--vm-hang", "1")
                .withNewResources()
                    .addToLimits("memory", new Quantity("100Mi"))
                    .addToRequests("memory", new Quantity("50Mi"))
                .endResources()
            .endContainer()
        .endSpec()
        .build();
    
    kubernetesClient.pods().create(pod);
    
    // Wait for OOMKilled
    await().atMost(Duration.ofMinutes(2)).untilAsserted(() -> {
        Pod currentPod = kubernetesClient.pods()
            .inNamespace("firstviscount-test")
            .withName("memory-test-pod")
            .get();
        
        ContainerStatus containerStatus = currentPod.getStatus()
            .getContainerStatuses().get(0);
        
        assertThat(containerStatus.getLastState().getTerminated())
            .isNotNull();
        assertThat(containerStatus.getLastState().getTerminated().getReason())
            .isEqualTo("OOMKilled");
    });
}

@Test
void testResourceQuotaEnforcement() {
    // Create ResourceQuota
    ResourceQuota quota = new ResourceQuotaBuilder()
        .withNewMetadata()
            .withName("test-quota")
            .withNamespace("firstviscount-test")
        .endMetadata()
        .withNewSpec()
            .addToHard("requests.cpu", new Quantity("1"))
            .addToHard("requests.memory", new Quantity("1Gi"))
            .addToHard("pods", new Quantity("3"))
        .endSpec()
        .build();
    
    kubernetesClient.resourceQuotas().create(quota);
    
    // Deploy pods until quota exceeded
    List<Pod> pods = new ArrayList<>();
    
    for (int i = 0; i < 5; i++) {
        try {
            Pod pod = createPodWithResources(
                "quota-test-" + i,
                "500m", "500Mi"
            );
            pods.add(pod);
        } catch (KubernetesClientException e) {
            // Expected when quota exceeded
            assertThat(e.getMessage()).contains("exceeded quota");
            break;
        }
    }
    
    // Verify quota enforcement
    assertThat(pods).hasSize(2); // Only 2 pods should fit
}
```

## Test Utilities

### Kubernetes Test Helper

```java
@Component
@ConditionalOnProperty(name = "kubernetes.test.enabled", havingValue = "true")
public class KubernetesTestHelper {
    
    @Autowired
    private KubernetesClient client;
    
    private final String namespace = "firstviscount-test";
    
    public Pod deployPod(String name, String image, Map<String, String> labels) {
        Pod pod = new PodBuilder()
            .withNewMetadata()
                .withName(name)
                .withNamespace(namespace)
                .withLabels(labels)
            .endMetadata()
            .withNewSpec()
                .addNewContainer()
                    .withName(name)
                    .withImage(image)
                    .withImagePullPolicy("Always")
                .endContainer()
            .endSpec()
            .build();
            
        return client.pods().inNamespace(namespace).create(pod);
    }
    
    public void waitForPodReady(String podName) {
        await().atMost(Duration.ofMinutes(2)).untilAsserted(() -> {
            Pod pod = client.pods()
                .inNamespace(namespace)
                .withName(podName)
                .get();
                
            assertThat(pod.getStatus().getConditions())
                .filteredOn(c -> "Ready".equals(c.getType()))
                .extracting(PodCondition::getStatus)
                .containsExactly("True");
        });
    }
    
    public String execInPod(String podName, String... command) {
        ByteArrayOutputStream out = new ByteArrayOutputStream();
        
        client.pods()
            .inNamespace(namespace)
            .withName(podName)
            .writingOutput(out)
            .exec(command);
            
        return out.toString();
    }
    
    public void cleanupTestResources() {
        // Delete all test resources
        client.pods()
            .inNamespace(namespace)
            .withLabel("test", "true")
            .delete();
            
        client.services()
            .inNamespace(namespace)
            .withLabel("test", "true")
            .delete();
            
        client.configMaps()
            .inNamespace(namespace)
            .withLabel("test", "true")
            .delete();
    }
}
```

### Test Configuration

```yaml
# application-k8s-test.yml
kubernetes:
  test:
    enabled: true
    namespace: firstviscount-test
    cleanup-after-test: true
  
  client:
    master-url: https://localhost:6443
    ca-cert-file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    oauth-token-file: /var/run/secrets/kubernetes.io/serviceaccount/token
    
chaos:
  enabled: true
  experiments:
    - pod-delete
    - network-delay
    - cpu-stress
    
istio:
  enabled: true
  test:
    traffic-management: true
    circuit-breaker: true
    mtls: true
```

## Best Practices

1. **Namespace Isolation**: Use dedicated test namespaces
2. **Resource Cleanup**: Always clean up test resources
3. **Realistic Scenarios**: Test actual Kubernetes behaviors
4. **Chaos Testing**: Include failure scenarios
5. **Performance**: Monitor resource usage during tests
6. **Security**: Test RBAC and network policies
7. **Observability**: Verify logging and monitoring
8. **Documentation**: Document k8s-specific test requirements

---
*Last Updated: January 2025*