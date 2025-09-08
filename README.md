import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.*;

/**
 * Service class
 */
class Service {
    private String serviceId;
    private String name;
    private String repo;
    private String currentVersion;
    private String ownerTeam;

    // Constructor
    public Service(String serviceId, String name, String repo, String currentVersion, String ownerTeam) {
        this.serviceId = serviceId;
        this.name = name;
        this.repo = repo;
        this.currentVersion = currentVersion;
        this.ownerTeam = ownerTeam;
    }

    // Getters & setters (encapsulation)
    public String getServiceId() { return serviceId; }
    public String getName() { return name; }
    public String getRepo() { return repo; }
    public String getCurrentVersion() { return currentVersion; }
    public String getOwnerTeam() { return ownerTeam; }

    public void setName(String name) { this.name = name; }
    public void setRepo(String repo) { this.repo = repo; }
    public void setCurrentVersion(String v) { this.currentVersion = v; }
    public void setOwnerTeam(String ownerTeam) { this.ownerTeam = ownerTeam; }

    public void updateVersion(String newVersion) {
        this.currentVersion = newVersion;
    }

    public String toString() {
        return String.format("Service[id=%s,name=%s,version=%s,team=%s]", serviceId, name, currentVersion, ownerTeam);
    }
}

/**
 * Environment class
 */
class Environment {
    public enum Status { ACTIVE, MAINTENANCE, DOWN }

    private String envName;    // e.g., Dev/QA/Prod
    private String url;
    private int capacity;
    private Status status;
    private String deployedVersion; // currently deployed version (for dashboard)

    public Environment(String envName, String url, int capacity, Status status) {
        this.envName = envName;
        this.url = url;
        this.capacity = capacity;
        this.status = status;
        this.deployedVersion = "none";
    }

    // Getters & setters
    public String getEnvName() { return envName; }
    public String getUrl() { return url; }
    public int getCapacity() { return capacity; }
    public Status getStatus() { return status; }
    public String getDeployedVersion() { return deployedVersion; }

    // Encapsulated state transitions
    public void setStatus(Status s) {
        this.status = s;
        System.out.println("Environment " + envName + " status changed to " + s);
    }

    public void deployVersion(String version) {
        if (status != Status.ACTIVE) {
            throw new IllegalStateException("Cannot deploy to environment " + envName + " because it's " + status);
        }
        this.deployedVersion = version;
    }

    public void clearVersion() {
        this.deployedVersion = "none";
    }

    public String dashboard() {
        return String.format("Env: %s | URL: %s | Capacity: %d | Status: %s | Deployed: %s",
                envName, url, capacity, status, deployedVersion);
    }

    public String toString() {
        return dashboard();
    }
}

/**
 * Base Deployment class
 */
class Deployment {
    public enum State { PENDING, RUNNING, SUCCESS, FAILED, ROLLED_BACK }

    protected static int idCounter = 1000;

    protected final String deployId;
    protected Service service;
    protected Environment environment;
    protected String version;
    protected State state;
    protected LocalDateTime timestamp;

    // Constructor
    public Deployment(Service service, Environment environment, String version) {
        this.deployId = "DEP-" + (++idCounter);
        this.service = service;
        this.environment = environment;
        this.version = version;
        this.state = State.PENDING;
        this.timestamp = LocalDateTime.now();
    }

    // ≥5 methods: getters, setters and behavior
    public String getDeployId() { return deployId; }
    public Service getService() { return service; }
    public Environment getEnvironment() { return environment; }
    public String getVersion() { return version; }
    public State getState() { return state; }
    public LocalDateTime getTimestamp() { return timestamp; }

    public void setVersion(String version) { this.version = version; }
    public void setEnvironment(Environment env) { this.environment = env; }

    // Start a deployment (simple baseline)
    public void start() {
        if (environment.getStatus() != Environment.Status.ACTIVE) {
            throw new IllegalStateException("Environment not ACTIVE for deployment");
        }
        this.state = State.RUNNING;
        this.timestamp = LocalDateTime.now();
        System.out.println("[" + deployId + "] Starting deployment of " + service.getName() + " -> " + environment.getEnvName() + " : " + version);
    }

    // Rollback base logic
    public void rollback() {
        System.out.println("[" + deployId + "] Rolling back deployment " + deployId + " on env " + environment.getEnvName());
        this.state = State.ROLLED_BACK;
        // For base Deployment we'll reset environment to previous (unknown here); just clear version for demo
        environment.clearVersion();
    }

    // Base health check (overridden in strategies)
    public boolean healthCheck() {
        // Basic random health check (50/50) — strategies will override with different logic
        boolean healthy = new Random().nextBoolean();
        System.out.println("[" + deployId + "] Base healthCheck -> " + (healthy ? "PASS" : "FAIL"));
        if (!healthy) this.state = State.FAILED;
        return healthy;
    }

    // Release notes stub
    public String releaseNotes() {
        return "Release notes for " + service.getName() + " version " + version + " (deployed at " + formatTimestamp() + ")";
    }

    // Promotion hook (for strategies)
    public boolean promote() {
        // Base just writes version to environment if HEALTHY
        if (this.state == State.SUCCESS) {
            environment.deployVersion(version);
            service.updateVersion(version);
            return true;
        }
        return false;
    }

    protected String formatTimestamp() {
        return timestamp.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
    }

    public String toString() {
        return String.format("%s | %s -> %s | ver=%s | state=%s | at=%s",
                deployId, service.getName(), environment.getEnvName(), version, state, formatTimestamp());
    }
}

/**
 * BlueGreenDeployment extends Deployment
 */
class BlueGreenDeployment extends Deployment {
    private String blueVersion;
    private String greenVersion;
    private boolean usingBlue; // indicates which color currently active in this env (for simulation)
    private int validationScore; // 0..100 to simulate health

    public BlueGreenDeployment(Service service, Environment environment, String version) {
        super(service, environment, version);
        this.blueVersion = environment.getDeployedVersion().equals("none") ? "none" : environment.getDeployedVersion();
        this.greenVersion = version; // green is new
        this.usingBlue = true; // assume blue active initially
        this.validationScore = 0;
    }

    // Overriding healthCheck to simulate deeper checks (e.g., integration)
    @Override
    public boolean healthCheck() {
        // Simulate more thorough checks: compute score; pass if >= 70
        validationScore = 60 + new Random().nextInt(41); // 60..100
        boolean healthy = validationScore >= 70;
        System.out.println("[" + deployId + "] BlueGreen healthCheck score=" + validationScore + " -> " + (healthy ? "PASS" : "FAIL"));
        if (healthy) this.state = State.SUCCESS;
        else this.state = State.FAILED;
        return healthy;
    }

    // Promotion for blue-green: switch traffic from blue -> green
    @Override
    public boolean promote() {
        if (this.state != State.SUCCESS) {
            System.out.println("[" + deployId + "] Cannot promote because state is " + state);
            return false;
        }
        // switch active version to green
        String toActivate = greenVersion;
        environment.deployVersion(toActivate);
        service.updateVersion(toActivate);
        System.out.println("[" + deployId + "] Blue-Green promotion: activated " + toActivate + " on " + environment.getEnvName());
        return true;
    }

    // Additional helper
    public String getBlueVersion() { return blueVersion; }
    public String getGreenVersion() { return greenVersion; }
}

/**
 * CanaryDeployment extends Deployment
 */
class CanaryDeployment extends Deployment {
    private int trafficPercent; // how much traffic to send to canary
    private int successRate; // simulation metric 0..100

    // Constructor for standard canary (default 10%)
    public CanaryDeployment(Service service, Environment environment, String version, int trafficPercent) {
        super(service, environment, version);
        this.trafficPercent = trafficPercent;
        this.successRate = 0;
    }

    // Overloaded constructor (default percent)
    public CanaryDeployment(Service service, Environment environment, String version) {
        this(service, environment, version, 10);
    }

    @Override
    public void start() {
        if (environment.getStatus() != Environment.Status.ACTIVE) {
            throw new IllegalStateException("Environment not ACTIVE for deployment");
        }
        this.state = State.RUNNING;
        this.timestamp = LocalDateTime.now();
        System.out.println("[" + deployId + "] Starting CANARY deployment of " + service.getName() + " -> " + environment.getEnvName()
                + " : " + version + " (" + trafficPercent + "% traffic)");
    }

    // Overriding healthCheck to simulate canary-specific metrics
    @Override
    public boolean healthCheck() {
        // successRate increases randomly; require >=75 to promote
        successRate = 50 + new Random().nextInt(51); // 50..100
        boolean healthy = successRate >= 75;
        System.out.println("[" + deployId + "] Canary healthCheck successRate=" + successRate + "% -> " + (healthy ? "PASS" : "FAIL"));
        if (healthy) this.state = State.SUCCESS;
        else this.state = State.FAILED;
        return healthy;
    }

    // Promotion gradually increases traffic and eventually updates environment version
    @Override
    public boolean promote() {
        if (this.state != State.SUCCESS) {
            System.out.println("[" + deployId + "] Cannot promote canary because state is " + state);
            return false;
        }
        System.out.println("[" + deployId + "] Promoting canary: increasing traffic to 100% and routing fully to version " + version);
        environment.deployVersion(version);
        service.updateVersion(version);
        return true;
    }

    public int getTrafficPercent() { return trafficPercent; }
    public void setTrafficPercent(int tp) { this.trafficPercent = tp; }
}

/**
 * DevOpsManager: build, deploy, rollback, health checks, releases
 */
class DevOpsManager {
    private List<Service> services;
    private List<Environment> environments;
    private List<Deployment> deployments; // polymorphism: can store specific strategies
    private Map<String, List<Deployment>> releaseHistoryByService;

    public DevOpsManager() {
        this.services = new ArrayList<>();
        this.environments = new ArrayList<>();
        this.deployments = new ArrayList<>();
        this.releaseHistoryByService = new HashMap<>();
    }

    // Manage services & envs
    public void addService(Service s) { services.add(s); }
    public void addEnvironment(Environment e) { environments.add(e); }

    public Service findServiceById(String id) {
        for (Service s : services) if (s.getServiceId().equals(id)) return s;
        return null;
    }
    public Environment findEnvByName(String name) {
        for (Environment e : environments) if (e.getEnvName().equalsIgnoreCase(name)) return e;
        return null;
    }

    // Build simulation
    public boolean build(Service s, String version) {
        System.out.println("Building " + s.getName() + " version " + version + " from repo " + s.getRepo() + " ...");
        try {
            Thread.sleep(300); // simulate
        } catch (InterruptedException ignored) {}
        boolean success = new Random().nextInt(100) < 95; // 95% build success
        System.out.println("Build " + (success ? "SUCCEEDED" : "FAILED"));
        return success;
    }

    // Method Overloading: deploy by version only -> create a standard Deployment (non-strategy)
    public Deployment deploy(Service service, Environment env, String version) {
        Deployment d = new Deployment(service, env, version);
        return runDeployment(d);
    }

    // Overloaded deploy: by version + trafficPercent -> create CanaryDeployment
    public Deployment deploy(Service service, Environment env, String version, int trafficPercent) {
        CanaryDeployment d = new CanaryDeployment(service, env, version, trafficPercent);
        return runDeployment(d);
    }

    // Deploy a blue-green explicitly
    public Deployment deployBlueGreen(Service service, Environment env, String version) {
        BlueGreenDeployment d = new BlueGreenDeployment(service, env, version);
        return runDeployment(d);
    }

    // Run deployment common steps (polymorphism)
    private Deployment runDeployment(Deployment d) {
        // ensure environment is ACTIVE
        if (d.getEnvironment().getStatus() != Environment.Status.ACTIVE) {
            System.out.println("Cannot deploy: environment " + d.getEnvironment().getEnvName() + " status=" + d.getEnvironment().getStatus());
            d.state = Deployment.State.FAILED;
            deployments.add(d);
            addToHistory(d);
            return d;
        }

        d.start(); // strategy-specific start may be invoked (overridden in Canary)
        // Simulate some runtime
        try { Thread.sleep(200); } catch (InterruptedException ignored) {}
        boolean healthy = d.healthCheck(); // polymorphic call
        if (!healthy) {
            System.out.println("[" + d.getDeployId() + "] Health check failed. Initiating rollback...");
            d.rollback();
            deployments.add(d);
            addToHistory(d);
            return d;
        }
        // If health passed, set state SUCCESS and perform promotion
        d.state = Deployment.State.SUCCESS;
        boolean promoted = d.promote(); // polymorphic promote
        if (!promoted) {
            System.out.println("[" + d.getDeployId() + "] Promotion failed.");
        } else {
            System.out.println("[" + d.getDeployId() + "] Deployment SUCCESS and promoted.");
        }
        deployments.add(d);
        addToHistory(d);
        return d;
    }

    // Rollback a specific deployment by id
    public boolean rollback(String deployId) {
        for (Deployment d : deployments) {
            if (d.getDeployId().equals(deployId)) {
                d.rollback();
                System.out.println("Rolled back " + deployId);
                addToHistory(d);
                return true;
            }
        }
        System.out.println("Deployment " + deployId + " not found for rollback.");
        return false;
    }

    // Run health checks for all recent deployments (or a single one)
    public void runHealthChecks() {
        System.out.println("Running health checks for all RUNNING/RECENT deployments...");
        for (Deployment d : deployments) {
            if (d.getState() == Deployment.State.RUNNING || d.getState() == Deployment.State.PENDING) {
                boolean ok = d.healthCheck();
                if (!ok) {
                    d.rollback();
                } else {
                    d.state = Deployment.State.SUCCESS;
                    d.promote();
                }
            }
        }
    }

    // Release notes collector
    public void printReleaseNotes(String serviceId) {
        Service s = findServiceById(serviceId);
        if (s == null) {
            System.out.println("Service not found");
            return;
        }
        System.out.println("Release notes for service " + s.getName() + ":");
        List<Deployment> history = releaseHistoryByService.getOrDefault(serviceId, Collections.emptyList());
        for (Deployment d : history) {
            System.out.println("- " + d.getDeployId() + " | ver=" + d.getVersion() + " | state=" + d.getState() + " | " + d.releaseNotes());
        }
    }

    // Print environment dashboards
    public void printEnvDashboards() {
        System.out.println("=== Environment Dashboards ===");
        for (Environment e : environments) {
            System.out.println(e.dashboard());
        }
    }

    // Print release history (all)
    public void printReleaseHistory() {
        System.out.println("=== Release History (All Deployments) ===");
        for (Deployment d : deployments) {
            System.out.println(d);
        }
    }

    private void addToHistory(Deployment d) {
        String sid = d.getService().getServiceId();
        releaseHistoryByService.computeIfAbsent(sid, k -> new ArrayList<>()).add(d);
    }
}

/**
 * DevOpsAppMain to test flows
 */
public class DevOpsAppMain {
    public static void main(String[] args) {
        DevOpsManager manager = new DevOpsManager();

        // Define services
        Service svc1 = new Service("SVC-001", "UserService", "git://repo/usersvc", "1.0.0", "Team-A");
        Service svc2 = new Service("SVC-002", "OrderService", "git://repo/ordersvc", "2.1.3", "Team-B");
        manager.addService(svc1);
        manager.addService(svc2);

        // Define environments
        Environment dev = new Environment("Dev", "http://dev.example.com", 10, Environment.Status.ACTIVE);
        Environment qa = new Environment("QA", "http://qa.example.com", 50, Environment.Status.ACTIVE);
        Environment prod = new Environment("Prod", "https://prod.example.com", 200, Environment.Status.ACTIVE);
        manager.addEnvironment(dev);
        manager.addEnvironment(qa);
        manager.addEnvironment(prod);

        // Print initial dashboards
        manager.printEnvDashboards();
        System.out.println();

        // Simulate build + simple deploy (non-strategy) to Dev
        String newVersion1 = "1.1.0";
        if (manager.build(svc1, newVersion1)) {
            Deployment d1 = manager.deploy(svc1, dev, newVersion1); // uses base Deployment
            System.out.println("Deployment result: " + d1);
        }
        System.out.println();

        // Simulate Canary deployment to QA: overloaded method with trafficPercent
        String canaryVersion = "1.2.0-canary";
        if (manager.build(svc1, canaryVersion)) {
            Deployment canary = manager.deploy(svc1, qa, canaryVersion, 20); // 20% traffic
            System.out.println("Canary deployment recorded: " + canary);
            if (canary.getState() == Deployment.State.FAILED) {
                System.out.println("Canary failed -> rollback was executed.");
            } else {
                System.out.println("Canary succeeded -> promoted to full traffic.");
            }
        }
        System.out.println();

        // Simulate Blue-Green deployment to Prod
        String bgVersion = "1.2.0";
        if (manager.build(svc1, bgVersion)) {
            Deployment bg = manager.deployBlueGreen(svc1, prod, bgVersion);
            System.out.println("Blue-Green deployment recorded: " + bg);
            if (bg.getState() == Deployment.State.FAILED) {
                System.out.println("Blue-Green failed -> rollback performed.");
            } else {
                System.out.println("Blue-Green succeeded -> green promoted.");
            }
        }

        System.out.println();
        // Another service canary flow and forced rollback demo
        String orderNew = "2.2.0";
        if (manager.build(svc2, orderNew)) {
            Deployment orderCanary = manager.deploy(svc2, qa, orderNew, 15);
            // force simulate a bad health by invoking rollback manually if failed
            if (orderCanary.getState() == Deployment.State.FAILED) {
                manager.rollback(orderCanary.getDeployId());
            }
        }

        System.out.println();
        // Show release history & dashboards after operations
        manager.printReleaseHistory();
        System.out.println();
        manager.printEnvDashboards();

        // Print release notes for a service
        System.out.println();
        manager.printReleaseNotes("SVC-001");
    }
}
