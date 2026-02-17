graph TB
    subgraph "PARTENAIRES (Réseau Privé)"
        Partner1[Application Partenaire 1<br/>Vérifie signatures]
        Partner2[Application Partenaire 2<br/>Chiffre données]
    end
    
    subgraph "INFRASTRUCTURE ON-PREMISE"
        F5[F5 BigIP<br/>Load Balancer<br/>TLS Offload]
        FW[Firewall<br/>ACLs/IDS/IPS]
        LB[Load Balancer HA<br/>Pair]
    end
    
    TGW[IBM Cloud<br/>Transit Gateway<br/>IPSec Encrypted]
    
    subgraph "IBM CLOUD VPC - RÉSEAU PRIVÉ"
        subgraph "OpenShift - Zone Application"
            JWKS[JWKS Service<br/>Lecture fichiers JSON<br/>GET /jwks, /jwk/kid]
            Apps[Applications Internes<br/>Banking Core, Payment GW<br/>SEULES à utiliser KMS]
            PVC[(PersistentVolume<br/>/data/jwks.json<br/>/data/jwk-*.json)]
        end
        
        subgraph "Zone Management"
            CT[CipherTrust Manager<br/>HA Cluster<br/>Key Lifecycle]
        end
        
        subgraph "Zone HSM Isolée"
            HSM1[Luna Cloud HSM<br/>Primary AZ1<br/>FIPS 140-2 L3]
            HSM2[Luna Cloud HSM<br/>Standby AZ2<br/>FIPS 140-2 L3]
        end
        
        Cron[CronJob<br/>Refresh Keys<br/>Toutes les heures]
    end
    
    subgraph "Monitoring & Audit"
        QRadar[IBM QRadar SIEM<br/>Audit Logs]
        Prometheus[Prometheus<br/>Métriques]
    end
    
    Partner1 -->|Réseau Privé Dédié<br/>HTTPS| F5
    Partner2 -->|VPN/Direct Connect<br/>HTTPS| F5
    F5 --> FW
    FW --> LB
    LB -->|Via Transit GW| TGW
    TGW -->|Réseau VPC Privé| JWKS
    
    JWKS -->|Lit fichiers| PVC
    JWKS -.->|PAS d'accès| CT
    
    Apps -->|PKCS#11 mTLS<br/>Sign/Decrypt UNIQUEMENT| CT
    
    Cron -->|REST API<br/>Fetch public keys| CT
    Cron -->|Write JSON files| PVC
    
    CT -->|PKCS#11/NTLS| HSM1
    CT -.->|Failover| HSM2
    HSM1 <-->|Replication| HSM2
    
    CT -->|Audit Logs| QRadar
    JWKS -->|Metrics| Prometheus
    Apps -->|Metrics| Prometheus
    
    style Partner1 fill:#e3f2fd
    style Partner2 fill:#e3f2fd
    style F5 fill:#fff3cd
    style FW fill:#fff3cd
    style JWKS fill:#c8e6c9
    style Apps fill:#f8bbd0
    style CT fill:#b3e5fc
    style HSM1 fill:#ff6b6b
    style HSM2 fill:#ff6b6b
    style PVC fill:#ffe0b2
    
    classDef privateNetwork stroke:#4caf50,stroke-width:3px
    class F5,FW,LB,TGW,JWKS privateNetwork
