```mermaid
flowchart TD
    %% --- [Swimlane: CI System (Daily Build)] ---
    subgraph CISystem["CI System (Daily Build)"]
        direction TB
        Step0[<b>[Step 0] Create Baseline Snapshot</b>] --> BuildFinish[Build Job Finished]
        BuildFinish --> SyncCmd[Execute `sync` command\n(Data Consistency)]
        SyncCmd --> RequestSnap[Request Create VolumeSnapshot]
        RequestSnap --> RegisterBase[Register New Baseline (v2)\n(ReadyToUse: true)]
    end

    %% --- [Swimlane: User / CLI / Web] ---
    subgraph UserCLI["User / CLI / Web"]
        direction TB
        Step1[<b>[Step 1] Request Baseline Update</b>\n(Target: v2)]
        
        CheckOption{"Option: <code>--force</code> / <code>--yes / -y</code> set?"}
        Step1 --> CheckOption
        
        Alert[<b>[Alert]</b> Display Warning:\n'Existing workspace/data will be lost.']
        NoteDocs["1-1-2. Docs & Release Note\nCheck required"]
        
        Confirm{"User Confirms?"}
        
        CheckOption -- No --> Alert
        Alert --- NoteDocs
        Alert --> Confirm
        
        Cancel[Cancel Update]
        Confirm -- No --> Cancel
    end

    %% --- [Swimlane: Bee Controller System] ---
    subgraph BeeController["Bee Controller System"]
        direction TB
        CheckSnapStatus[Check Target Snapshot Status]
        
        IsReady{"Is Snapshot 'ReadyToUse'?"}
        CheckSnapStatus --> IsReady
        
        ErrorStop[<b>[Error]</b> Return & Stop\n(Snapshot not ready)]
        IsReady -- False --> ErrorStop
        
        %% --- Partition: Execution Phase (Critical) ---
        subgraph ExecPhase["Execution Phase (Critical)"]
            direction TB
            DeletePVC[<b>1. Delete</b> All PVCs\n(Force Clean Up for Zero/Incremental)]
            NoteDelete["Zero Type: Log Warning Only\nIncremental: Force Delete"]
            DeletePVC --- NoteDelete
            
            ApplySpec[<b>2. Apply New Spec (Sleep & Update)</b>\n- Scale Replicas -> 0\n- Set 'baseline=v2' Label]
            NoteAtomic["Atomic Operation\n(Single API Call)"]
            ApplySpec --- NoteAtomic
            
            DeletePVC --> ApplySpec
        end
        
        %% --- Partition: Recovery Phase (Auto Wake Up) ---
        subgraph RecoveryPhase["Recovery Phase (Auto Wake Up)"]
            direction TB
            RunWorker[<b>3. Run</b> Worker\n(Scale Replicas -> 1)]
            
            CheckLabel{"Check 'baseline' Label?"}
            RunWorker --> CheckLabel
            
            CreateFromSnap[Create PVC from <b>Snapshot (v2)</b>]
            CreateEmpty[Create <b>Empty</b> PVC]
            
            CheckLabel -- "Exists (v2)" --> CreateFromSnap
            CheckLabel -- "Empty / Missing" --> CreateEmpty
            
            PodRunning[Pod Running & Online]
            CreateFromSnap --> PodRunning
            CreateEmpty --> PodRunning
        end
    end

    %% --- Cross-Swimlane Connections ---
    RegisterBase --> Step1
    CheckOption -- Yes --> CheckSnapStatus
    Confirm -- Yes --> CheckSnapStatus
    IsReady -- True --> DeletePVC
    ApplySpec --> RunWorker
    PodRunning --> Stop((Stop))
    Cancel --> Stop
    ErrorStop --> Stop

    %% --- Styling ---
    style ExecPhase fill:#ffebee,stroke:#b71c1c,stroke-width:2px
    style NoteDelete fill:#fff,stroke:#333,stroke-dasharray: 5 5
    style NoteAtomic fill:#fff,stroke:#333,stroke-dasharray: 5 5
    style NoteDocs fill:#fff,stroke:#333,stroke-dasharray: 5 5
    style Alert fill:#fff9c4,stroke:#fbc02d
    style ErrorStop fill:#ffcdd2,stroke:#c62828
    style Cancel fill:#cfd8dc,stroke:#546e7a

```