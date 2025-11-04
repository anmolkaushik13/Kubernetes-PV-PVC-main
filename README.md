Kubernetes PV–PVC with Azure File Share (Step-by-Step Guide)
Step 1: Create and Connect the Kubernetes Cluster

First, create your Kubernetes cluster (e.g., in AKS or local with Minikube/Kind).

Connect your terminal or VS Code to the cluster using kubectl.
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Step 2: Prepare Azure Storage

Create an Azure Storage Account.

Inside it, create a File Share (this will be used as the persistent storage backend).
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Step 3: Create the Secret

We need to securely store the Storage Account name and Storage Account key in Kubernetes.
This Secret will later be used by the PersistentVolume (PV) to authenticate with Azure File Share.

File: secret.yaml

apiVersion: v1                  # API version for Kubernetes Secrets
kind: Secret                    # Resource type is Secret
metadata:
  name: my-secret               # Name of the secret
type: Opaque                    # Opaque = generic secret type (key-value pairs)
stringData:                     # Plain text values (Kubernetes will encode them)
  azurestorageaccountname: bambhole   # Storage account name
  azurestorageaccountkey: <your-storage-key> # Storage account key


Commands:

kubectl apply -f secret.yaml
kubectl get secret my-secret


✅ At this stage, the Secret is available for use in PV and Pods.
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Step 4: Create the PersistentVolume (PV)

The PV represents a slice of physical storage (our Azure File Share) in the cluster.

File: pv.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv                        # Name of the PV
spec:
  capacity:
    storage: 2Gi                      # Maximum storage size
  accessModes:
    - ReadWriteMany                   # Multiple pods can read/write at the same time
  persistentVolumeReclaimPolicy: Retain   # Data is retained even after PVC deletion
  storageClassName: my-storage-class      # Must match with PVC
  azureFile:                              
    secretName: my-secret              # The Secret we created earlier
    shareName: pvc                     # Azure File Share name
    readOnly: false                    # Allow write access


Commands:

kubectl apply -f pv.yaml
kubectl get pv


✅ PV is now registered in the cluster.
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Step 5: Create the PersistentVolumeClaim (PVC)

PVC is a request from a Pod asking for storage. It will automatically bind to the PV that matches.

File: pvc.yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc                         # PVC name
spec:
  accessModes:
    - ReadWriteMany                   # Must match PV access mode
  resources:
    requests:
      storage: 1Gi                     # Request 1Gi from PV (less than 2Gi)
  storageClassName: my-storage-class   # Must match PV’s storageClassName


Commands:

kubectl apply -f pvc.yaml
kubectl get pvc


✅ PVC status should become Bound (linked to PV).
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Step 6: Create a Pod that Uses the PVC

Now we mount the PVC inside a Pod to test storage.

File: pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: myapp                        # Pod name
spec:
  containers:
  - name: myapp                      # Container name
    image: nginx                     # Use Nginx image
    ports:
      - containerPort: 80            # Expose port 80
    volumeMounts:
      - name: pendrive               # Attach the PVC volume
        mountPath: /mnt/pendrive     # Mount inside container
  volumes:
  - name: pendrive
    persistentVolumeClaim:
      claimName: mypvc               # Use PVC created earlier


Commands:

kubectl apply -f pod.yaml
kubectl get pod myapp


✅ Pod should be running and the Azure File Share is mounted.
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Step 7: Verify Persistence

Enter the Pod:

kubectl exec -it myapp -- bash


Inside the container:

cd /mnt/pendrive
ls
echo "Hello from Kubernetes" > test.txt
ls -l


Go to Azure Portal → Storage Account → File Share → you will see test.txt.

Delete the Pod:

kubectl delete pod myapp


Recreate the Pod:

kubectl apply -f pod.yaml
kubectl exec -it myapp -- bash
ls /mnt/pendrive


✅ You will still see test.txt. This proves PV + PVC works as persistent storage.

Key Benefits

Data persists even if the Pod is deleted or recreated.

Multiple Pods can share the same storage with ReadWriteMany.

Azure File Share integrates smoothly with Kubernetes via PV–PVC.
