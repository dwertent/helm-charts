## On installation
A user is running the operator for the first time.

```mermaid

sequenceDiagram
    Operator ->>+ KubeAPI: Get all workloads and imagesIDs
    Operator ->>+ Kubevuln: Scan workload (WLID + imagesIDs)
    Kubevuln ->>+ Storage: Get SBOM (imageID + SiftVersion)
    alt SBOM found
        Storage ->>+ Kubevuln: Return SBOM 
    else SBOM not found
        Kubevuln ->>+ Kubevuln: Generate SBOM (using Sift)
        Kubevuln ->>+ Storage: Store SBOM (key: imageID. metadata: SiftVersion)
    end
    Kubevuln ->>+ Kubevuln: Scan SBOM for CVEs (using Grype)
    Kubevuln ->>+ Storage: Store CVE Manifest (key: imageID, metadata: SiftVersion, GrypeDBVersion)
    Kubevuln ->>+ ARMO Platform: Submit CVE Manifest (key: WLID+ImageID, metadata: TBD)
```


## New Image
A user is creating a new workload with a new image or the user is updating his existing workload with a new image.

```mermaid
sequenceDiagram
    User ->>+ KubeAPI: Create new workload/update image
    par Generating SBOM
        KubeAPI ->>+ Operator: New workload (the operator is watching for new workloads using the pod watch)
        Operator ->>+ Kubevuln: Generate SBOM (WLID + imagesIDs)
        Kubevuln ->>+ Storage: Get SBOM (imageID + SiftVersion)
        alt SBOM found
            Storage ->>+ Kubevuln: Return SBOM 
        else SBOM not found
            Kubevuln ->>+ Kubevuln: Generate SBOM (using Sift)
            Kubevuln ->>+ Storage: Store SBOM (key: imageID. metadata: SiftVersion)
        end
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