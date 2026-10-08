# Nutanix AHV, CVM, CLI, SDK, and Active Directory Reference Notes

## Purpose

This document consolidates operational notes for future reference while working with Nutanix AHV, Controller Virtual Machines (CVMs), cluster CLI commands, Service Delivery Kit (SDK) resources, host maintenance, failure scenarios, and directory-based access control.

> Operational note: Command syntax and UI labels can vary by AOS, Prism, and SDK version. Validate commands with the approved runbook and change process before using them in production.


## 1. Unlocking a Locked User

### Command

```bash
allssh sudo faillock --user admin --reset
```

### Procedure

1. Confirm that the account is locked because of failed authentication attempts.
2. Log in with an authorized administrative account.
3. Run the command from the appropriate Nutanix node or CVM shell.
4. Confirm that the command completes successfully on the intended CVMs.
5. Retry the user login using the approved credentials.
6. If the user remains locked, check whether an external system or automation is repeatedly submitting an old password.

### Important considerations

* `allssh` can execute the command across multiple Nutanix Controller Virtual Machines. Confirm the scope before running it.
* The example targets the `admin` user. Replace the username only when the approved operational procedure requires it.
* Investigate the cause of repeated lockouts instead of repeatedly clearing the lock.

## 2. Downloading the Service Delivery Kit / SDK

1. Open [partner.nutanix.com](https://partner.nutanix.com).
2. Sign in with an account that has access to partner resources.
3. Open the **Library** section.
4. Search for the required SDK or **Service Delivery Kit**.
5. Review the version and compatibility information.
6. Download the package and store it in the approved internal repository.
7. Record the downloaded version for future troubleshooting and reproducibility.

## 3. AHV and CVM Architecture Overview

### AHV

AHV is the Nutanix hypervisor layer that runs virtual machines. Each AHV host provides compute resources and hosts VM workloads.

### CVM

The Controller Virtual Machine provides Nutanix cluster services, including storage and data-management functions for the VMs running on the AHV hosts.

### Failure behavior

* **If an AHV host is down:** VMs may restart on another healthy AHV host. Do not describe this as guaranteed live migration: a hard host failure and planned maintenance have different behaviors, and VM settings, capacity, HA configuration, and migration eligibility affect the result.
* **If a CVM is down:** The VM can remain running on its AHV host while the redundant Nutanix data path uses another healthy CVM as needed. This does not mean every I/O operation is necessarily remote; expect possible temporary interruption or degraded performance and check the CVM and cluster status promptly.

## 4. Example Internal Network Mapping

The following values are examples captured during training and should be validated against the target cluster before use.

| Component | Example address |
|---|---|
| AHV host | `192.168.5.1` |
| CVM examples | `192.168.5.254`, `192.168.5.2` |
| CVM-side address example | `10.58.x.x` |
| AHV-side address example | `10.x.x.x` |

### Connectivity notes

* An AHV host and its CVM use different IP addresses; do not interpret the phrase "same IP" literally.
* From an AHV host, reach the associated CVM using the CVM IP recorded for that node.
* An AHV host may communicate with another healthy CVM when the local CVM is unavailable.
* Confirm the actual AHV-to-CVM mapping from Prism, the cluster configuration, or the approved environment diagram. Do not assume that the example addresses apply to every cluster.

### Example SSH connection

```bash
ssh nutanix@192.168.5.2
```

Use the correct username, IP address, and authentication method for the target environment. Treat the IP above as an example CVM address.

## 5. Basic Cluster Validation Commands

### Check cluster status

```bash
cluster status
```

Use this command before and after maintenance or failure testing to compare cluster health and service state.

### List AHV hosts

```bash
acli host.list
```

This displays the AHV hosts available to the cluster and helps identify the host that will be placed into or removed from maintenance mode.

## 6. CVM Shutdown Procedure

### Command

```bash
cvm_shutdown -P now
```

### Recommended sequence

1. Confirm the target CVM and the reason for the shutdown.
2. Remember that `cvm_shutdown -P now` is a graceful CVM shutdown command, not an AHV host shutdown command.
3. Check the cluster status:

   ```bash
   cluster status
   ```

4. Confirm that another healthy CVM is available to provide cluster services.
4. Check for active incidents, replication activity, maintenance operations, or other changes.
6. Obtain the required approval and execute the shutdown command from the target CVM:

   ```bash
   cvm_shutdown -P now
   ```

7. Monitor the cluster after the CVM shuts down.
7. Confirm that VMs remain operational and that storage services are available through healthy CVMs.
8. After the maintenance activity, verify that the CVM returns to service and that cluster status is healthy.

> Although `-P now` requests an immediate graceful shutdown, do not use it as a routine troubleshooting step without understanding the impact and following the approved Nutanix procedure.

## 7. AHV Host Maintenance Mode

### Entering maintenance mode

1. Identify the target host:

   ```bash
   acli host.list
   ```

2. Check the cluster before making the change:

   ```bash
   cluster status
   ```

3. Run the maintenance-mode pre-check:

   ```bash
   acli host.enter_maintenance_mode_check <host IP>
   ```

4. Review the VMs running on the host, including any VM-host affinity or anti-affinity rules.
5. Confirm that the target host has sufficient capacity to evacuate or restart workloads as required.
6. Open ACLI:

   ```bash
   acli
   ```

7. At the ACLI prompt, enter the AHV host into maintenance mode. This is the host operation that evacuates eligible VMs; CVM maintenance mode alone does not migrate VMs:

   ```text
   host.enter_maintenance_mode <host IP>
   ```

8. Verify the host state:

   ```bash
   acli host.get <host IP>
   ```

   Confirm `node_state` is `EnteredMaintenanceMode` and `schedulable` is `False`.

9. Monitor VM movement and confirm that the expected workloads have migrated or been handled according to policy.
10. Recheck cluster status and document the result.

### Exiting maintenance mode

1. Confirm that maintenance work is complete.
2. Verify that the host is powered on, reachable, and ready to rejoin the cluster.
3. Open ACLI if required:

   ```bash
   acli
   ```

4. At the ACLI prompt, run:

   ```text
   host.exit_maintenance_mode <host IP>
   ```

5. Confirm that the host returns to normal status.
6. Recheck the cluster and confirm that workloads and services are healthy.

### Affinity VM validation

Before placing a host into maintenance mode, specifically check VMs with affinity rules:

* Record which VMs are constrained to the host or to a host group.
* Confirm whether the rule allows migration to another host.
* Check for pinned VMs, PCI-passthrough devices, RF1 data, or other non-migratable configurations.
* Verify that sufficient resources are available on eligible destination hosts.
* If a VM cannot evacuate, follow the approved process to modify the rule temporarily, power off the VM if permitted, or schedule separate handling.
* After maintenance, confirm that the intended affinity policy is restored and that the VM placement is correct.

## 8. CVM Failure and Data Locality Scenario

### Scenario

A CVM becomes unavailable while the AHV host and its VMs remain online.

### Expected behavior

1. The VM continues to run on the AHV host.
2. Storage requests that cannot be served locally are handled through the Nutanix storage network by another healthy CVM.
3. The cluster may continue operating in a degraded state until the failed CVM is restored.
4. Performance and resiliency should be monitored while the CVM is unavailable.

### Validation steps

1. Check the cluster:

   ```bash
   cluster status
   ```

2. Identify the affected CVM and AHV host.
3. Confirm that another CVM is healthy and reachable.
4. Confirm that the affected VMs remain powered on and responsive.
5. Check for storage, network, or alert conditions in Prism.
6. Restore or reboot the affected CVM according to the approved procedure.
7. Recheck cluster health and confirm data services have returned to the expected state.

## 9. Active Directory and SAML-Based Directory Integration

### Why use directory integration?

Centralized directory integration avoids creating and maintaining a separate local account for every user. Instead, users are managed in the organization's identity system and are granted Nutanix access through roles and group mappings.

### Directory examples

* Microsoft Active Directory can provide centralized authentication and group membership.
* SAML-based identity providers, such as Google Workspace, can provide federated authentication where supported by the Nutanix platform and organization.

### Recommended implementation sequence

1. **Define access requirements**
   * Identify administrator, operator, viewer, and other required access levels.
   * Follow least privilege; do not grant every user administrative access.

2. **Create authorization roles**
   * Define roles based on job responsibilities.
   * Grant only the permissions required for each role.
   * Document the purpose and owner of each role.

3. **Create directory groups**
   * Create groups in Microsoft Active Directory or the selected SAML identity provider.
   * Use role-based group names that are easy to understand and audit.

4. **Configure the identity provider**
   * Configure the supported directory or SAML connection in the appropriate Nutanix management interface.
   * Confirm DNS, time synchronization, certificates, network reachability, and required identity-provider settings.

5. **Map groups to Nutanix roles**
   * Map each directory group to the appropriate Nutanix authorization role.
   * Avoid assigning permissions directly to individual users when a group-based assignment is practical.

6. **Test with representative users**
   * Test at least one user from each role.
   * Confirm successful login.
   * Confirm that the user can perform permitted actions.
   * Confirm that restricted actions are denied.

7. **Validate administration and recovery**
   * Confirm that a designated break-glass or emergency administrative method is available and secured according to policy.
   * Confirm that access can be revoked by removing the user from the directory group.
   * Record the configuration, group mappings, owners, and review date.

### Key principle

For large environments, manage access through directory groups and authorization roles rather than creating thousands of local Nutanix users individually. Local accounts should be limited to approved emergency or service-account use cases and governed by security policy.

## 10. Suggested Operational Checklist

### Before a change

* Confirm the target cluster, host, CVM, and IP address.
* Check `cluster status`.
* Review alerts and active maintenance activities.
* Identify affected VMs and affinity rules.
* Confirm capacity on healthy AHV hosts and CVMs.
* Obtain change approval and define a rollback or recovery plan.

### During a change

* Use the exact command and target documented in the change plan.
* Monitor VM availability, host state, CVM state, and cluster services.
* Avoid unrelated configuration changes.
* Record timestamps, command results, and observed impact.

### After a change

* Run `cluster status` again.
* Confirm AHV hosts are in the intended state.
* Confirm CVMs and storage services are healthy.
* Confirm affected VMs are online and correctly placed.
* Confirm affinity policies and directory access mappings remain correct.
* Update the change record and operational documentation.

## 11. Command Summary

```bash
# Clear failed-login lock for the admin user
allssh sudo faillock --user admin --reset

# Connect to an example CVM
ssh nutanix@192.168.5.2

# Gracefully shut down a CVM
cvm_shutdown -P now

# Check cluster status
cluster status

# List AHV hosts
acli host.list

# Open ACLI
acli

# At the ACLI prompt: enter host maintenance mode
host.enter_maintenance_mode <host IP>

# At the ACLI prompt: exit host maintenance mode
host.exit_maintenance_mode <host IP>
```



