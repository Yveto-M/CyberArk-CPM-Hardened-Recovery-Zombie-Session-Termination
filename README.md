# 🛡️ Project: CyberArk CPM Hardened Recovery & "Zombie" Session Termination

### 🚀 Executive Summary
This project documents the diagnosis and resolution of a **"Cascading Failure"** within a CyberArk Privileged Access Manager (PAM) environment. The Component Manager (CPM) entered a restart loop due to a credential desync, compounded by a "Ghost Session" (`CACPM047E`) and an automatic Vault IP Ban (`ITATS433E`).

**Objective:** Restore service availability while adhering to strict "Zero Trust" security protocols (creating a Quadruple-Locked Credential File).

---

## 💥 The Incident (Symptoms)
The **CyberArk Password Manager** service failed to start, entering a "False Negative" loop.
* **Symptom 1:** Service status stuck on "Starting" indefinitely.
* **Symptom 2:** The Vault logs (`italog.log`) revealed the root cause: The CPM was repeatedly trying to authenticate with bad keys, triggering an automatic security suspension.

**Vault Log IP Suspension**
<img width="928" height="380" alt="Screenshot 2025-12-29 224823" src="https://github.com/user-attachments/assets/45d19aa6-e71c-42d2-9831-0b9ae71ccb22" />

*> **Figure 1:** The "Smoking Gun." Vault logs showing `ITATS433E IP Address 10.0.0.2 is suspended`. The Vault was actively blocking the CPM from connecting.*

---

## 🔧 Technical Resolution Phases

### Phase 1: The "Zombie" Process Kill
Standard service restarts failed because a "Ghost Session" was holding the Vault connection open.
* **Diagnosis:** While the "Processes" tab showed nothing, switching to the "Details" tab revealed the hidden culprit: `CACPMScanner.exe` was running as a zombie process, locking the memory files.

![Zombie Process Discovery](<img width="657" height="255" alt="Screenshot 2025-12-29 224038" src="https://github.com/user-attachments/assets/67b76a69-9626-4b51-a745-b3c53ae342a9" />
)
*> **Figure 2:** Diagnosis via Task Manager Details. The `CACPMScanner.exe` (PID 3384) was orphaned and holding the session lock.*

* **Action:** Executed forced termination to clear the memory lock.

### Phase 2: The "Security Gate" Failure
Attempting to generate a new `user.ini` credential file using standard commands failed. The environment returned error **`CASCF001E`** ("Authentication flags are not sufficient"). The system rejected the new keys because they lacked "Machine Binding."

![Security Error CASCF001E](<img width="957" height="429" alt="Screenshot 2025-12-29 233032" src="https://github.com/user-attachments/assets/d6cf657f-26bd-4190-bd57-bb1542fbc02f" />
)
*> **Figure 3:** The "Security Gate." CyberArk rejecting the creation of a standard credential file, demanding higher security flags.*

### Phase 3: The "Quadruple Lock" Re-Key (The Solution)
To satisfy the security requirements, I utilized the `CreateCredFile` utility in **Interactive Mode** to generate a hardened credential file with **four distinct locks**:
1.  **IP Restriction:** Locked to `10.0.0.2`.
2.  **Hostname Restriction:** Locked to the specific server name.
3.  **DPAPI Machine Protection:** Encrypted using the Windows Machine Key.
4.  **Entropy File:** Generated a secondary physical key file (`user.ini.entropy`) for 2-factor file integrity.

![Quadruple Lock Evidence](<img width="767" height="367" alt="Screenshot 2025-12-29 231856" src="https://github.com/user-attachments/assets/cfec6b91-7158-4e63-b46c-480cc104c6b0" />
)
*> **Figure 4:** Success. The presence of `user.ini.entropy` alongside `user` confirms the high-security "Entropy Lock" was successfully applied.*

---

## 📊 Results: Full Recovery
With the Zombie process killed and the "Quadruple Locked" keys in place, the service successfully authenticated.

![Final Healthy Log](<img width="931" height="281" alt="Screenshot 2025-12-29 234623" src="https://github.com/user-attachments/assets/8442b9cc-0e4b-4255-9ae1-6cc48c5e952f" />
)
*> **Figure 5:** Operational Success. The `pm.log` shows `CACPM309I Initiating tasks`, confirming the engine is now reading policies and rotating passwords.*

---

## 🧠 Engineering Takeaways
1.  **Process Visibility:** Windows Service Manager often reports false states; verifying specific PIDs in Task Manager is required during hangs.
2.  **Security Gates:** Modern CyberArk implementations enforce "Machine Binding" (DPAPI/IP) by default. Simple re-keying scripts will fail without these specific flags.
3.  **Log Correlation:** The root cause was only found by correlating the **Component Log** (`pm.log`) with the **Vault Log** (`italog.log`).
