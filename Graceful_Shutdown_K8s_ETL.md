# Graceful Shutdown in Kubernetes: Preventing Data Loss During Pod Scale-Down on AWS

## Overview
When running a large-scale ETL (Extract, Transform, Load) system on AWS EKS with hundreds or thousands of pods, autoscaling (HPA/KEDA and Cluster Autoscaler) is essential for cost optimization. However, when the system scales down due to dropping CPU or memory utilization, Kubernetes will terminate existing pods to reclaim resources.

If a pod is processing a critical ETL task when it receives a termination signal, abrupt termination can lead to data loss, corrupted states, or inconsistent databases. This document outlines the strategy to implement a **graceful shutdown** mechanism to ensure no in-flight work is lost during scale-down events.

## The Kubernetes Termination Lifecycle
When an HPA decides to scale down, or the Cluster Autoscaler removes a node, Kubernetes follows a specific sequence to terminate a pod:
1. **Pod Status is set to `Terminating`**: The pod is removed from the Endpoints list of all Services (so it stops receiving new network traffic).
2. **`SIGTERM` Signal is Sent**: The container's main process (PID 1) receives a `SIGTERM` signal.
3. **Grace Period**: Kubernetes waits for the process to exit voluntarily. This duration is defined by `terminationGracePeriodSeconds` (default is 30 seconds).
4. **`SIGKILL` Signal is Sent**: If the process is still running after the grace period expires, Kubernetes forcefully kills it using `SIGKILL`, resulting in immediate data loss for in-flight tasks.

## Strategy for ETL Systems
For an ETL worker pulling tasks from a queue or database, the graceful shutdown strategy involves:
1. **Catching the `SIGTERM` signal.**
2. **Stopping the intake of new tasks** (e.g., stop pulling from Kafka, AWS SQS, or RabbitMQ).
3. **Finishing the currently active tasks** or safely checkpointing their state so another worker can resume.
4. **Closing connections** (databases, message brokers) cleanly.
5. **Exiting with code 0.**

---

## Example 1: Graceful Shutdown in Go

In Go, we use a signal channel and `context` to notify workers to stop accepting new jobs and finish their current ones. `sync.WaitGroup` ensures the main program doesn't exit until all workers are done.

```go
package main

import (
	"context"
	"fmt"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

// Simulate an ETL worker process
func etlWorker(ctx context.Context, workerID int, wg *sync.WaitGroup) {
	defer wg.Done() // Notify WaitGroup when this worker fully exits
	
	for {
		select {
		case <-ctx.Done():
			// 2. Stop accepting new tasks
			fmt.Printf("Worker %d: Stop signal received. Finishing current task...\n", workerID)
			
			// 3. Finish currently active task safely
			time.Sleep(2 * time.Second) 
			fmt.Printf("Worker %d: State saved. Cleanly exited.\n", workerID)
			return
		default:
			// Simulate fetching and processing a normal ETL task
			fmt.Printf("Worker %d: Processing task...\n", workerID)
			time.Sleep(3 * time.Second) // Task takes 3 seconds
		}
	}
}

func main() {
	fmt.Println("Starting ETL System...")
	
	// 1. Create a context that listens for SIGTERM (K8s) and SIGINT (Ctrl+C)
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	var wg sync.WaitGroup
	numWorkers := 5 // Simulating multiple concurrent workers per pod

	// Start workers
	for i := 1; i <= numWorkers; i++ {
		wg.Add(1)
		go etlWorker(ctx, i, &wg)
	}

	// Block main thread until a termination signal is received
	<-ctx.Done()
	fmt.Println("\nShutdown initiated. Waiting for workers to finish in-flight tasks...")

	// 4. Wait for all workers to finish their current loop
	wg.Wait()
	
	// 5. Cleanly close database connections, etc. here
	fmt.Println("All workers finished. Database connections closed. Exiting safely.")
}
```

---

## Example 2: Graceful Shutdown in Python

In Python, we can use the `signal` module to catch `SIGTERM` and use a boolean flag to instruct the worker loop to terminate safely.

```python
import time
import signal
import sys
import threading

class ETLWorker:
    def __init__(self):
        self.is_shutting_down = False
        self.threads = []

    def handle_sigterm(self, signum, frame):
        print(f"\n[Main] Received SIGTERM ({signum}). Initiating graceful shutdown...")
        # 1. Flag the system to stop accepting new tasks
        self.is_shutting_down = True

    def process_data(self, worker_id):
        # Continue working as long as shutdown hasn't been triggered
        while not self.is_shutting_down:
            print(f"[Worker {worker_id}] Processing ETL batch...")
            
            # Simulate processing time (e.g., 5 seconds)
            time.sleep(5) 
            print(f"[Worker {worker_id}] Batch complete.")
            
        # 2. Loop breaks when shutting down. Finish up the state safely.
        print(f"[Worker {worker_id}] Shutting down gracefully. State checkpointed.")

    def run(self):
        # Register the signal handlers for SIGTERM (K8s) and SIGINT (Ctrl+C)
        signal.signal(signal.SIGTERM, self.handle_sigterm)
        signal.signal(signal.SIGINT, self.handle_sigterm)

        # Start worker threads
        for i in range(3):
            t = threading.Thread(target=self.process_data, args=(i+1,))
            t.start()
            self.threads.append(t)

        # Keep main thread alive and responsive to signals
        while not self.is_shutting_down:
            time.sleep(1)

        # 3. Wait for all active threads to finish their current batch
        print("[Main] Waiting for workers to finish current tasks...")
        for t in self.threads:
            t.join()
            
        # 4. Cleanup resources (e.g., DB connections)
        print("[Main] All workers stopped. Cleanup complete. Exiting.")
        sys.exit(0)

if __name__ == "__main__":
    app = ETLWorker()
    app.run()
```

---

## Kubernetes Configuration (Crucial Step)

Code-level handling is only half the battle. You **must** configure your Kubernetes Deployment to give your application enough time to finish its work before the forceful `SIGKILL` is sent.

If your longest ETL task takes 2 minutes to process, you must set `terminationGracePeriodSeconds` to at least `150` (2.5 minutes).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: etl-worker
spec:
  replicas: 1000
  selector:
    matchLabels:
      app: etl-worker
  template:
    metadata:
      labels:
        app: etl-worker
    spec:
      containers:
      - name: etl-worker
        image: your-account.dkr.ecr.us-east-1.amazonaws.com/etl-worker:latest
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
      # IMPORTANT: Give your application enough time to finish in-flight tasks.
      # Default is 30 seconds. Increase this based on your max task duration.
      terminationGracePeriodSeconds: 180 
```

## Additional AWS Best Practices

1. **Pod Disruption Budgets (PDB)**: If you are using the AWS Cluster Autoscaler or Karpenter, ensure you configure a `PodDisruptionBudget`. This prevents the node autoscaler from evicting too many pods simultaneously during node scale-down.
2. **Message Queues (SQS/Kafka/RabbitMQ)**: Instead of holding strict state in memory, design your ETL workers to acknowledge (ACK) a message only *after* the processing is fully complete. If a pod is killed unexpectedly (e.g., Node crash, AWS Spot Instance interruption, or OOMKilled), the unacknowledged message will automatically return to the queue and be processed by another pod.
3. **Node Autoscaler Drain Timeouts**: Note that node autoscalers have maximum drain timeouts (e.g., AWS Cluster Autoscaler defaults to 10 minutes; Karpenter is configurable). If your `terminationGracePeriodSeconds` is longer than the node drain timeout, the underlying EC2 instance might be terminated forcefully by AWS anyway. Keep your ETL tasks broken down into chunks smaller than 5-10 minutes if possible.
