# Kubernetes Pod Restart Policies

The Restart Policy in a Kubernetes Pod determines how the Kubelet (the agent running on each node) handles container failures and successful terminations within that Pod. This policy applies only to the containers inside the Pod; it does not cause the Pod to be rescheduled to a different node.

There are three possible values for the `restartPolicy` field:

🔁 Restart Policies for Pods

| Policy | Description | Behavior upon Container Termination |
|---|---|---|
| Always | The container is always restarted by the Kubelet upon termination, regardless of the exit code. | Restarts if the container exits with an error (non-zero code) or if it exits successfully (zero code). |
| OnFailure | The container is only restarted by the Kubelet if it terminates with a non-zero exit code (indicating a failure). | Restarts only if the container fails. If it exits successfully, the Kubelet does nothing. |
| Never | The container is never restarted by the Kubelet. | No restart is attempted, regardless of the exit code. The Pod will eventually move to a Completed or Failed phase. |

⚙️ Practical Usage and Context

1. **Always** (Default for Controllers)
 * Best for: Long-running applications like web servers, APIs, or stateful services (often managed by Controllers like Deployment, DaemonSet, or StatefulSet).
 * Reasoning: You want the application to always be running. If the container crashes due to a bug or resource issue, Kubernetes automatically attempts to bring it back up to maintain service availability.
   * Note: When a Pod is managed by a Controller (like a Deployment), the controller takes over the responsibility of ensuring the desired number of Pods are running. If a container in the Pod repeatedly fails and the Kubelet's restart attempts are limited by an exponential back-off delay, the Controller may choose to terminate the failing Pod and create a brand new Pod to replace it.
2. **OnFailure**
 * Best for: Jobs or processes that might encounter recoverable errors but should not be restarted if they complete their task successfully.
 * Reasoning: This is a middle ground. If the container finishes successfully (exit code 0), the job is done. If it fails (non-zero exit code), the Kubelet tries to restart it, assuming the failure might be transient (e.g., a temporary network issue).
3. **Never** (Default for Jobs)
 * Best for: Batch jobs or one-time tasks (often managed directly by the Job controller).
 * Reasoning: Once the container finishes, whether successfully or with a failure, the work is considered complete (or failed permanently). No further local restarts are desired.
   * Note: The Job controller is typically configured to detect a failed Pod and create a new Pod to retry the entire task, rather than relying on the Kubelet to restart the container within the old Pod.

Important Context: Pod vs. Controller
It's crucial to remember that the `restartPolicy` only governs the Kubelet's actions on the containers within the current Pod.
 * If you set `restartPolicy: Never` and the container fails, the Pod will enter a Failed state and stay there.
 * If the Pod is part of a higher-level object (like a Deployment), it's the Deployment Controller that monitors the Pod's overall status and will create a new replacement Pod if the old one fails or terminates, which is a different mechanism from the Kubelet's restart policy.

# Testing the Restart Policies

The three Pods test each restart policy by using a container that exits with different codes. The BusyBox container runs a command to exit with code 0 (success) or code 1 (failure).

## Always Restart Policy

The Pod with the Always restart policy will restart the container regardless of its exit code. In the [restart-always.yaml](./restart-always.yaml) example, the container runs a command that exits with code 0 (success), but Kubernetes will keep restarting it.

![Kubernetes Pod with Always restart policy showing continuous container restarts despite successful exit code, with restart count incrementing and backoff delays visible in kubectl output](../00-images/restart-always.png)

### Deliberately Failing Container with the OnFailure Restart Policy

```bash
command: ['sh', '-c', 'echo "Running..."; sleep 5; exit 1']
```

This line defines the command that runs inside the container. Breaking it down:

- `sh -c` - Starts a shell to execute the command string
- `echo "Running..."` - Prints "Running..." to stdout
- `sleep 5` - Waits for 5 seconds
- `exit 1` - Exits with a non-zero status code (indicating failure)

The commands are chained with semicolons, so they run sequentially. This Pod is designed to fail deliberately (due to exit 1), which is useful for testing how Kubernetes handles failed containers with the Never restart policy - the container will run once, fail after 5 seconds, and won't restart.

This is what happens when we run this pod:

Container starts - Prints "Running..." and sleeps for 5 seconds
Container fails - Exits with code 1 (failure)
Kubernetes restarts it - Because restartPolicy: OnFailure means "restart only when container fails"
Restart loop with backoff - The container will keep failing and restarting, but with exponentially increasing delays between attempts (10s, 20s, 40s, 80s, up to 5 minutes max)
Status cycles - You'll see the pod status alternate between `Running` → `Error` → `CrashLoopBackOff`
Restart count increases - Each restart increments the restart counter
The pod will remain in this `CrashLoopBackOff` state indefinitely, as Kubernetes keeps trying to restart the failing container with backoff delays.

![Kubernetes Pod with OnFailure restart policy displaying kubectl output showing multiple restart-onfailure pod entries with varying restart counts (0-4), status states including Running, Error, and CrashLoopBackOff, and age timestamps ranging from 2 minutes to 2 minutes 15 seconds, illustrating the cyclical failure and restart behavior with exponential backoff delays](../00-images/restart-onfailure.png)

## Never Restart Policy

The Pod with the Never restart policy will not restart the container regardless of its exit code. In the [restart-never.yaml](./restart-never.yaml) example, the container runs a command that exits with code 1 (failure), and Kubernetes will not attempt to restart it.

# The Exponential Back-off Mechanism

When a container fails and the restart policy is set to Always or OnFailure, the Kubelet employs an exponential back-off strategy to manage restart attempts. This means that after each failure, the time interval before the next restart attempt increases exponentially (10s, 20s, 40s, 80s, up to 5 minutes). This approach helps prevent rapid, repeated restarts that could lead to resource exhaustion or instability in the cluster.

You can see the actual delay times in the pod's status with `kubectl get pod restart-onfailure -o yaml` - look for the containerStatuses section which shows the restart count and backoff state.

> **Production tip**: For jobs, use `restartPolicy: Never` or `OnFailure` to avoid unnecessary restarts. Let the Job controller handle retries by creating new Pods as needed.
