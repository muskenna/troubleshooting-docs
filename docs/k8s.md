

**Procedure to Determine if a Subnet Ran Out of IP Allocation**

**Symptom**:
Error indicating failure to create a pod sandbox due to an inability to assign an IP address to a container.

**Steps**:

1. **Identify Pods in Pending State**:
   Check for pods that are in the `Pending` state in the desired namespace.
   ```bash
   kubectl get pods -o=custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,STATUS:.status.phase --field-selector=status.phase=Pending -n [NAMESPACE]
   ```

2. **Retrieve EC2 Instance ID of the Node**:
   For a node that has pods in the `Pending` state, retrieve its associated EC2 `instance-id`.
   ```bash
   kubectl get node [NODE_NAME] -o=jsonpath='{.spec.providerID}' | awk -F'/' '{print $NF}'
   ```

3. **Identify the Subnet of the EC2 Instance**:
   Use the `instance-id` to find out the subnet in which the EC2 instance resides.
   ```bash
   aws ec2 describe-network-interfaces --filters Name=attachment.instance-id,Values=[INSTANCE_ID] --profile [PROFILE_NAME] --region [REGION] | jq '.NetworkInterfaces[0].SubnetId'
   ```

4. **Check Available IP Addresses in the Subnet**:
   Determine the number of available IP addresses in the identified subnet.
   ```bash
   aws ec2 describe-subnets --subnet-ids [SUBNET_ID] --profile [PROFILE_NAME] --region [REGION] | jq '.Subnets[0].AvailableIpAddressCount'
   ```

5. **Identify All Nodes in the Problematic Subnet**:
   If the available IP address count is zero or close to zero, you might want to identify all nodes in that subnet.
   ```bash
   for node in $(kubectl get nodes -o=jsonpath='{.items[*].metadata.name}'); do
       instance_id=$(kubectl get node $node -o=jsonpath='{.spec.providerID}' | awk -F'/' '{print $NF}')
       subnet_id=$(aws ec2 describe-network-interfaces --filters Name=attachment.instance-id,Values=$instance_id --profile [PROFILE_NAME] --region [REGION] | jq -r '.NetworkInterfaces[0].SubnetId')
       if [[ "$subnet_id" == "[TARGET_SUBNET_ID]" ]]; then
           echo $node
       fi
   done
   ```

**Notes**:
- Replace placeholders like `[NAMESPACE]`, `[NODE_NAME]`, `[INSTANCE_ID]`, `[PROFILE_NAME]`, `[REGION]`, and `[TARGET_SUBNET_ID]` with appropriate values as per your environment.
- This procedure assumes you have `kubectl`, `aws-cli`, and `jq` installed and properly configured.
- Always ensure you're working in the right AWS profile and region to avoid discrepancies in results.
