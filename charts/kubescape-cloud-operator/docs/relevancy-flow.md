## On installation
A user is running the operator for the first time.

```mermaid

sequenceDiagram
    Operator ->>+ KubeAPI: Watch all Pods and imagesIDs
    Operator ->>+ Kubevuln: Generate SBOM (imageID, WorkloadMetadata)
    alt SBOM found
        Storage ->>+ Kubevuln: Return SBOM 
        Kubevuln ->>+ Storage: Update workload metadata (if needed) 
    else SBOM not found
        Kubevuln ->>+ Kubevuln: Generate SBOM (Using Grype)
        Kubevuln ->>+ Storage: Store SBOM (key: imageID. metadata: SiftVersion)
    end
    Operator ->>+ Storage: Watch for new SBOMs
    Operator ->>+ Kubevuln: Scan CVEs (imageID, WLID, WorkloadMetadata)
    Kubevuln ->>+ Storage: Store CVE Manifest (key: imageID, metadata: SiftVersion, GrypeDBVersion, WorkloadMetadata)
    Kubevuln ->>+ ARMO Platform: Submit CVE Manifest (key: WLID+ImageID, metadata: TBD)
```


## New Image
A user is creating a new workload with a new image or the user is updating his existing workload with a new image.

```mermaid
sequenceDiagram
    Operator ->>+ KubeAPI: Watch all Pods and imagesIDs
    Operator ->>+ Kubevuln: Generate SBOM (imageID, WorkloadMetadata)
    alt SBOM found
        Storage ->>+ Kubevuln: Return SBOM 
        Kubevuln ->>+ Storage: Update workload metadata (if needed) 
    else SBOM not found
        Kubevuln ->>+ Kubevuln: Generate SBOM (Using Grype)
        Kubevuln ->>+ Storage: Store SBOM (key: imageID. metadata: SiftVersion)
    end
        
    # In the end of the process, the storage will have the SBOM for the new image

    # The Sniffer will start sniffing the node for new containers
    par Generating SBOM'
        Node ->>+ Sniffer: New container is running (ebpf)
        Sniffer ->>+ Sniffer: Start sniffing on this new container
        KubeAPI ->>+ Sniffer: New pod is running on this node
        Storage ->>+ Sniffer: New SBOM is ready for this imageID
        Sniffer ->> Sniffer: Generate SBOM'
        Sniffer ->> Storage: Store SBOM' (key: InstanceID, metadata: SiftVersion)
    end

    # The sniffer will continue sniffing for X minutes (configurable), and will send the updated SBOM' to the storage every X minutes.

```



## Operator - Kubevuln


```mermaid
sequenceDiagram
    Note over Operator: Generate SBOM flow 
    rect rgb(191, 223, 255)
    # Generating SBOM API
    Operator ->>+ KubeAPI: Watch for pods
    Operator ->>+ Kubevuln: Generate SBOM (imageID, WorkloadMetadata)
    Kubevuln ->>+ Storage: Get SBOM (imageID)
    alt SBOM found
        Storage ->>+ Kubevuln: Return SBOM 
        Kubevuln ->>+ Storage: Update workload metadata (if needed) 
    else SBOM not found
        Kubevuln ->>+ Kubevuln: Generate SBOM (Using Grype)
        Kubevuln ->>+ Storage: Store SBOM (key: imageID. metadata: SiftVersion)
    end
    end

    rect rgb(200, 150, 255)
    Note over Sniffer: Generate SBOM' flow 
    Node ->>+ Sniffer: New container is running (ebpf)
    Sniffer ->>+ Sniffer: Start sniffing on this new container
    KubeAPI ->>+ Sniffer: New pod is running on this node
    Storage ->>+ Sniffer: New SBOM is ready for this imageID
    Sniffer ->> Sniffer: Generate SBOM'
    Sniffer ->> Storage: Store SBOM' AFTER duration X OR when container exists with 0 (key: InstanceID, metadata: SiftVersion) OR CPU/Memory usage is high  
    end

    rect rgb(191, 223, 255)
    Note over Operator: Scanning for CVEs flow 
    alt Watch
        Operator ->>+ KubeAPI: Watch for SBOMs in storage
    else
        Operator ->>+ KubeAPI: Watch for SBOM's in storage
    end
    Operator ->>+ Kubevuln: Scan CVEs (instanceID, imageID, metadata)
    Kubevuln ->>+ Storage: Get SBOM (imageID)
    Kubevuln ->>+ Storage: Get SBOM' (instanceID)
    Kubevuln ->>+ Kubevuln: Scan SBOM for CVEs (using Grype)
    Kubevuln ->>+ Kubevuln: Scan SBOM' for CVEs (using Grype)
    Kubevuln ->>+ Storage: Store CVE Manifest (key: imageID, metadata: SiftVersion, GrypeDBVersion)
    Kubevuln ->>+ ARMO Platform: Submit CVE Manifest (key: WLID+ImageID, metadata: TBD)
    end
```

## Recurring scan

The user pre-set a recurring scan for his cluster.

```mermaid
sequenceDiagram
    Job ->>+ Operator: Scan all workloads
    Operator ->>+ KubeAPI: Get all workloads and imagesIDs
    Operator ->>+ Kubevuln: Scan + Relevancy-scan workload (WLID + imagesIDs + InstanceIDs)
    Kubevuln ->>+ Storage: Get SBOM (imageID + SiftVersion)
        Kubevuln ->>+ Kubevuln: Generate SBOM (using Sift)
        Kubevuln ->>+ Storage: Store SBOM (key: imageID. metadata: SiftVersion)
    Kubevuln ->>+ Storage: Get SBOM' (instanceID + SiftVersion)
    Kubevuln ->>+ Kubevuln: Scan SBOM for CVEs (using Grype)
    Kubevuln ->>+ Kubevuln: Scan SBOM' for CVEs (using Grype)
    Kubevuln ->>+ Storage: Store CVE Manifest (key: imageID, metadata: SiftVersion, GrypeDBVersion)
    Kubevuln ->>+ ARMO Platform: Submit CVE Manifest (key: WLID+ImageID, metadata: TBD)
```


## Clear cache

One every X hours, the operator will clear the cache.

```mermaid
sequenceDiagram
    Operator ->>+ KubeAPI: List all workloads and imagesIDs
    Operator ->>+ Storage: List all InstanceIDs and imagesIDs
    Operator ->>+ Operator: List all imagesIDs and instancesIDs that are not in use
    Operator ->>+ Storage: Delete list of imagesIDs and instancesIDs (SBOM, SBOM', CVE Manifest)
```