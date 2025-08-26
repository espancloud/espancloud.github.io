# -*- coding: utf-8 -*-
"""
A Python script to automate Linux tasks, parse output with regex,
and provide logging for execution status and debugging.
"""

# Import necessary standard libraries
import subprocess  # Used to run external commands and manage their input/output
import re          # 're' is the regular expression module, used for pattern matching
import logging     # Used for logging events for debugging and monitoring purposes
import os          # Provides a way of using operating system dependent functionality

# --- Logger Configuration ---
# This section sets up a flexible logging system that can write to both the
# console and a file, with different levels of detail.

# Create a logger object. It's best practice to use the name of the current module.
logger = logging.getLogger(__name__)
logger.setLevel(logging.DEBUG)  # Set the lowest level of severity to capture all messages

# Create a handler to write log messages to a file.
# 'file_handler' will write messages to 'automation.log'.
# 'w' mode means the log file is rewritten each time the script runs.
# Use 'a' for append mode to keep a running history.
file_handler = logging.FileHandler('automation.log', mode='w')
file_handler.setLevel(logging.DEBUG)  # Set the level for the file handler to DEBUG

# Create a handler to write log messages to the console (standard output).
console_handler = logging.StreamHandler()
console_handler.setLevel(logging.INFO)  # Only show INFO and higher messages on the console

# Define the format for the log messages.
# This format includes the timestamp, logger name, severity level, and the message.
formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
file_handler.setFormatter(formatter)
console_handler.setFormatter(formatter)

# Add the configured handlers to the logger object.
logger.addHandler(file_handler)
logger.addHandler(console_handler)


def execute_linux_command(command, success_pattern, failure_pattern=None):
    """
    Executes a Linux command, captures its output, and determines success or
    failure based on regex patterns.

    Args:
        command (str): The full Linux command to be executed.
        success_pattern (str): A regex pattern that indicates the command succeeded.
                               This pattern is searched for in the command's stdout.
        failure_pattern (str, optional): A regex pattern that explicitly indicates
                                         failure. This is useful for commands that
                                         might return a 0 exit code but still fail.
                                         Searched in both stdout and stderr.
                                         Defaults to None.

    Returns:
        bool: True if the command was successful, False otherwise.
    """
    logger.info(f"--- Attempting to execute command: '{command}' ---")

    try:
        # subprocess.run() is the modern and recommended way to run external commands.
        # 'shell=True' allows us to pass the command as a single string, which is
        # convenient but can be a security hazard if the command includes untrusted input.
        # 'capture_output=True' captures stdout and stderr.
        # 'text=True' decodes stdout and stderr as text using the default encoding.
        # 'check=False' prevents subprocess from raising an exception for non-zero exit codes,
        # allowing us to handle them manually.
        result = subprocess.run(
            command,
            shell=True,
            capture_output=True,
            text=True,
            check=False
        )

        # Log the details of the command execution for debugging.
        logger.debug(f"Command: '{command}'")
        logger.debug(f"Return Code: {result.returncode}")
        # .strip() removes leading/trailing whitespace, including newlines.
        logger.debug(f"STDOUT:\n{result.stdout.strip()}")
        logger.debug(f"STDERR:\n{result.stderr.strip()}")

        # --- Output Parsing and Status Determination ---

        # Combine stdout and stderr for comprehensive failure checking.
        full_output = result.stdout + result.stderr

        # 1. Check for an explicit failure pattern first.
        if failure_pattern and re.search(failure_pattern, full_output, re.IGNORECASE):
            logger.error(f"Execution FAILED. Explicit failure pattern '{failure_pattern}' found.")
            return False

        # 2. Check the command's return code. A non-zero code typically indicates an error.
        if result.returncode != 0:
            logger.error(f"Execution FAILED. Command returned a non-zero exit code: {result.returncode}.")
            logger.error(f"Error details (stderr): {result.stderr.strip()}")
            return False

        # 3. If the return code is 0, check for the success pattern in stdout.
        #    re.search() finds the first occurrence of the pattern in the string.
        #    re.IGNORECASE makes the matching case-insensitive (e.g., 'hello' matches 'Hello').
        if re.search(success_pattern, result.stdout, re.IGNORECASE):
            logger.info(f"Execution SUCCEEDED. Success pattern '{success_pattern}' found.")
            return True
        else:
            # This case handles when the command runs (exit code 0) but the output
            # doesn't match what we expect for success.
            logger.warning(f"Execution FAILED. Command ran successfully but the success pattern '{success_pattern}' was not found in the output.")
            return False

    except FileNotFoundError:
        # This error occurs if the command itself is not found (e.g., 'lss' instead of 'ls').
        logger.critical(f"Execution FAILED. Command not found: '{command.split()[0]}'. Ensure it is installed and in your PATH.")
        return False
    except Exception as e:
        # Catch any other unexpected exceptions during the process.
        logger.critical(f"An unexpected error occurred while executing '{command}': {e}")
        return False


# --- Main Execution Block ---
# This code runs only when the script is executed directly (not when imported).
if __name__ == "__main__":
    logger.info("Starting Linux Task Automation Script")

    # --- Example 1: Successful command (Check if a file exists) ---
    # We'll create a dummy file to ensure the command works.
    dummy_file = "test_file.txt"
    with open(dummy_file, "w") as f:
        f.write("hello world")

    # Command: List the file.
    # Success Pattern: The regex looks for the filename in the output.
    # The '.' in regex means 'any character', so we escape it with '\.' to match a literal dot.
    task1_status = execute_linux_command(
        command=f"ls -l {dummy_file}",
        success_pattern=r"test_file\.txt"
    )
    print(f"Task 1 (File Exists Check) Status: {'PASSED' if task1_status else 'FAILED'}\n")

    # Clean up the dummy file
    os.remove(dummy_file)

    # --- Example 2: Failed command (Check for a non-existent file) ---
    # Command: Try to list a file that doesn't exist.
    # Success Pattern: We still provide a success pattern, but we don't expect to find it.
    # Failure Pattern: We look for the specific error message "No such file or directory".
    task2_status = execute_linux_command(
        command="ls -l non_existent_file.txt",
        success_pattern=r"non_existent_file\.txt",
        failure_pattern=r"No such file or directory"
    )
    # Since we expect this task to fail, a return of 'False' means our script worked correctly.
    print(f"Task 2 (Non-Existent File Check) Status: {'PASSED (as expected failure)' if not task2_status else 'FAILED (unexpected success)'}\n")


    # --- Example 3: Checking service status (e.g., cron) ---
    # Command: Check the status of the cron service using systemctl.
    # Success Pattern: Looks for "active (running)" which indicates the service is up.
    # The '\s+' matches one or more whitespace characters.
    # The '\(' and '\)' escape the parentheses to match them literally.
    task3_status = execute_linux_command(
        command="systemctl status cron",
        success_pattern=r"active\s+\(running\)"
    )
    print(f"Task 3 (Check Cron Service) Status: {'PASSED' if task3_status else 'FAILED'}\n")

    # --- Example 4: Checking disk space ---
    # Command: Display disk usage for the root directory ('/').
    # Success Pattern: A more complex regex to find a percentage usage.
    # '\d{1,3}%' matches a 1 to 3 digit number followed by a '%' sign.
    # This is a good way to confirm the command returned a plausible value.
    task4_status = execute_linux_command(
        command="df -h /",
        success_pattern=r"\d{1,3}%"
    )
    print(f"Task 4 (Check Disk Space) Status: {'PASSED' if task4_status else 'FAILED'}\n")

    logger.info("Automation Script Finished")






------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

CLI/Netconf/Restconf

Netconf Client----------Netconf Server[(Datastore Defined by YANG)]
Datastore contains:  (Configuration Data + State Data)

Configuration datastore (running, candidate, startup)
NE's can support multiple YANG models simultaneously: Native Models, IETF Models, Openconfig Models etc

Reuest - Responce Model 
Clients send <rpc> requests, and servers respond with <rpc-reply>.


Restconf - Restful APIs
HTTP based Rest APIS, (Uses GET, POST, PUT (for update , insert) , DELETE)  Responce can be in XML or JSON

Netconf - RPCs
YANG Model encoded as XML for both REquest and Responce. 



Operations: 

<get>: Retrieves both configuration data and device state information (operational data, statistics, etc.) from the running datastore.


Configuration Operations:
<get-config>: Retrieves all or part of a specified configuration datastore (e.g., running, candidate, startup). This allows clients to fetch the current or intended configuration of the network device.
<edit-config>: Modifies the configuration data in a specified datastore. It supports various edit operations like merge, create, delete, and replace to enable flexible configuration changes.
<copy-config>: Copies configuration data from one datastore to another. This is useful for tasks like backing up, restoring, or cloning configurations.
<delete-config>: Deletes an entire configuration datastore (e.g., startup or candidate). The running configuration datastore cannot be deleted.
<commit>: Applies changes from the candidate datastore to the running datastore, making them active on the device.
<discard-changes>: Discards uncommitted changes in the candidate datastore, reverting it to its previous state.
<validate>: Validates the contents of a specified datastore without applying changes, ensuring the configuration is syntactically and semantically correct.

Operational and Session Management Operations:
<get>: Retrieves both configuration data and device state information (operational data, statistics, etc.) from the running datastore.
<lock>: Locks a specified configuration datastore to prevent other clients from making changes, ensuring exclusive access for critical operations.
<unlock>: Releases a configuration datastore lock previously obtained with the <lock> operation.

<close-session>: Requests graceful termination of a NETCONF session.
<kill-session>: Forcefully terminates a NETCONF session. This operation is typically restricted to administrators.




Physical Topology
--------------------

NE ------------ Linux VM


S/W Design
------------

Netconf Client --------------------Netconf Server<-->(Datastore (Defined by YANG))


NE
----
username adminuser privilidge 15 secret mypassword 
netconf-yang

interface tenGig0/1
ip address 192.168.50.1
no shutdown
exit



linux VM
-----------
ssh adminuser@192.168.50.1 -p 830 -s netconf


*recieve capabilities in XML








----------------------------------------------------- Restconf -------------------------------------
username adminuser privilidge 15 secret mypassword 
ip http secure-server

restconf

interface tenGig0/1
ip address 192.168.50.1
no shutdown
exit



linux VM
-----------
curl -k https://192.168.50.1/restconf/ -u "adminuser:mypassword"
curl -k https://192.168.50.1/restconf/data/Cisco-IOS-XE-native:native/ -u "adminuser:mypassword"
curl -k https://192.168.50.1/restconf/data/Cisco-IOS-XE-native:native/interface/GigabitEthernet=1 -u "adminuser:mypassword"

* Look through the URL path of resources to make sense of the returned data





----------------------
Authorization using NACM Rules

Messages Layer: Defines the format for Remote Procedure Call (RPC) messages and notifications. It uses XML encoding for all protocol messages and data.

    <rpc>: The client encapsulates a request in an <rpc> element and sends it to the server. Each RPC has a message-id attribute for correlation with replies.
    <rpc-reply>: The server responds to each <rpc> with an <rpc-reply>. If the operation is successful, it typically contains an <ok/> element. If there's an error, it contains an <rpc-error> element.
    <notification>: (Defined in RFC 5277, but referenced by 6241) Allows the server to asynchronously send event notifications to subscribed clients.
	
	
	

NETCONF is designed to be extensible. Clients and servers negotiate capabilities during session establishment (via the <hello> message) to indicate which optional features they support.

Base Capabilities: Every NETCONF implementation must support the base NETCONF capabilities (identified by urn:ietf:params:netconf:base:1.0 or 1.1).
Optional Capabilities (examples defined in RFC 6241):

    :candidate: Support for the <candidate> datastore, <commit>, and <discard-changes> operations.
    :confirmed-commit: Allows for a "two-phase commit" where changes are reverted if not explicitly confirmed within a timeout.
    :rollback-on-error: If an error occurs during an <edit-config>, the entire operation is rolled back.
    :startup: Support for the <startup> datastore.
    :validate: Support for the <validate> operation to check configuration validity without applying it.
    :xpath: Enables the use of XPath expressions for filtering in <get> and <get-config>.
    :url: Allows specifying configuration data from a URL.
	
	
Base Operations (RPCs):

    <get-config>: Retrieves all or a filtered portion of a specified configuration datastore (running, candidate, or startup).
    <edit-config>: Modifies configuration data in a specified datastore. It supports merge, replace, create, delete, and remove operations to manipulate data elements within the XML tree.
    <copy-config>: Copies an entire configuration datastore to another.
    <delete-config>: Deletes a configuration datastore (e.g., candidate or startup, but not running).
    <lock>: Acquires a lock on a specified datastore to ensure exclusive access, preventing other NETCONF sessions from modifying it.
    <unlock>: Releases a previously acquired lock on a datastore.
    <get>: Retrieves both configuration data and operational state information from the running datastore.
    <close-session>: Gracefully terminates the NETCONF session.
    <kill-session>: Forcefully terminates another NETCONF session (typically requires administrator privileges).




	Clients:
	
	NETCONF client (like netconf-console, ncclient in Python, or Postman with a NETCONF plugin) to send these XML payloads to the device.
	
	--------------------------------------------------------------------------------------------------------------------------------------
	**NACM stands for NETCONF Access Control Model.**

It's a standardized mechanism that defines how access to NETCONF operations, data nodes (configuration and operational data), and notifications is controlled for different users or groups. In essence, it's the "authorization" component for NETCONF.

**Key Aspects of NACM (defined in RFC 8341, which obsoleted RFC 6536):**

* **Granular Control:** NACM allows administrators to define fine-grained access rules. Instead of just "read-only" or "read-write" for an entire device, NACM can specify:
    * **Protocol Operation Authorization:** Which specific NETCONF operations (e.g., `<get>`, `<edit-config>`, `<copy-config>`, `<action>`) a user or group can execute.
    * **Module Authorization:** Access to all data defined within a particular YANG module.
    * **Data Node Authorization:** Access to specific branches or leaves (individual configuration parameters or operational states) within the YANG data model. This is very powerful, allowing for role-based access to specific configurations.
    * **Notification Authorization:** Whether a user can receive specific types of notifications.
    * **Action Authorization:** Control over execution of YANG `action` statements.

* **Rule-based:** NACM operates on a set of rules. When a NETCONF request comes in, the server evaluates these rules based on the authenticated user and their associated groups. The first matching rule (based on predefined order) determines the permission (permit or deny).

* **Groups:** Users are typically assigned to groups, and permissions are then defined for these groups. This simplifies management, as you can assign a user to a group and they inherit all the group's permissions.

* **Default Behaviors:** NACM defines default behaviors for access (e.g., typically "deny" for write operations by default if no explicit permit rule is found, and "permit" for read and exec). Administrators can override these defaults.

* **Emergency Recovery Session:** NACM supports a concept of an "emergency recovery session" that can bypass all access control rules. This is critical for recovering from misconfigurations that might lock out all regular users.

* **YANG Integration:** NACM itself is defined using a YANG data model (`ietf-netconf-acm.yang`). This means NACM rules are configured and managed via NETCONF, just like any other configuration on the device.

**Does Cisco support NACM?**

**Yes, Cisco devices that support NETCONF (especially modern platforms running IOS XE, IOS XR, and Nexus OS) generally support NACM.**

While Cisco might have its own proprietary access control mechanisms for CLI or SNMP, for NETCONF, they adhere to the IETF's NACM standard. This is a crucial aspect of their strategy to support model-driven programmability and automation.

You'll find that on Cisco devices, when you enable NETCONF, the authorization of users accessing it will be governed by NACM rules, often linked to local user accounts or external AAA (Authentication, Authorization, and Accounting) servers. Administrators would configure NACM rules to grant specific users or groups the necessary permissions to interact with different parts of the device's configuration and operational data via NETCONF.

So, if you're working with Cisco devices using NETCONF, understanding and configuring NACM is essential for secure and effective management.


	--------------------------------------------------------------------------------------------------------------------------------------
	
	
	
	Please note that I cannot execute commands directly on a Cisco device. The "commands" I'm providing are the **YANG/XML structures you would send to a Cisco device via a NETCONF client** using an `<edit-config>` operation to configure NACM.

Cisco devices (especially IOS XE, IOS XR) support NACM through their NETCONF interface. You would typically use a NETCONF client (like `netconf-console`, `ncclient` in Python, or Postman with a NETCONF plugin) to send these XML payloads to the device.

Here are sample NACM configuration examples, focusing on common use cases. These assume you are sending them as part of an `<edit-config>` RPC operation, likely targeting the `candidate` datastore and then committing.

**Prerequisites:**

  * NETCONF is enabled and configured on your Cisco device.
  * You have a NETCONF client.
  * You have an authenticated NETCONF session with appropriate permissions to modify NACM (e.g., using the "emergency recovery user" or a user with full `nacm` access).

-----

### **Example 1: Deny all NETCONF write access to a specific user/group for a module**

Let's say you want to create a group `read-only-ops` and a user `op_user` belonging to this group. This group should *not* be able to modify anything related to the `Cisco-IOS-XE-interface` module.

**1. Create a NACM Group (if not already existing):**

```xml
<rpc message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <edit-config>
    <target>
      <candidate/>
    </target>
    <default-operation>merge</default-operation>
    <config>
      <nacm xmlns="urn:ietf:params:xml:ns:yang:ietf-netconf-acm">
        <groups>
          <group>
            <name>read-only-ops</name>
          </group>
        </groups>
      </nacm>
    </config>
  </edit-config>
</rpc>
```

**2. Assign a User to the Group (assuming `op_user` is already created locally):**

```xml
<rpc message-id="102" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <edit-config>
    <target>
      <candidate/>
    </target>
    <default-operation>merge</default-operation>
    <config>
      <nacm xmlns="urn:ietf:params:xml:ns:yang:ietf-netconf-acm">
        <groups>
          <group>
            <name>read-only-ops</name>
            <user-name>op_user</user-name>
          </group>
        </groups>
      </nacm>
    </config>
  </edit-config>
</rpc>
```

**3. Create a NACM Rule to Deny Write Access to `Cisco-IOS-XE-interface` for `read-only-ops`:**

```xml
<rpc message-id="103" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <edit-config>
    <target>
      <candidate/>
    </target>
    <default-operation>merge</default-operation>
    <config>
      <nacm xmlns="urn:ietf:params:xml:ns:yang:ietf-netconf-acm">
        <rules>
          <rule-list>
            <name>read-only-ops-rules</name>
            <group>read-only-ops</group>
            <rule>
              <name>deny-interface-writes</name>
              <module-name>Cisco-IOS-XE-interface</module-name>
              <access-operations>create update delete</access-operations>
              <action>deny</action>
              <rule-type>
                <protocol-operation/>
              </rule-type>
            </rule>
            <rule>
              <name>permit-all-reads</name>
              <access-operations>read</access-operations>
              <action>permit</action>
            </rule>
          </rule-list>
        </rules>
      </nacm>
    </config>
  </edit-config>
</rpc>
```

-----

### **Example 2: Permit specific read access to an operational data node for a user**

Let's say a monitoring user (`monitor_user`) should only be able to read the interface statistics (e.g., `statistics` container) but nothing else from the `Cisco-IOS-XE-interfaces-oper` module.

**1. Create a NACM Group for monitoring (e.g., `monitor-group`) and assign `monitor_user`.** (Similar to steps 1 & 2 in Example 1)

**2. Create a Rule List for `monitor-group` and define specific access:**

```xml
<rpc message-id="104" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <edit-config>
    <target>
      <candidate/>
    </target>
    <default-operation>merge</default-operation>
    <config>
      <nacm xmlns="urn:ietf:params:xml:ns:yang:ietf-netconf-acm">
        <rules>
          <rule-list>
            <name>monitor-group-rules</name>
            <group>monitor-group</group>
            <rule>
              <name>permit-interface-stats</name>
              <module-name>Cisco-IOS-XE-interfaces-oper</module-name>
              <path>/interfaces/interface/statistics</path> <access-operations>read</access-operations>
              <action>permit</action>
              <rule-type>
                <data-node/>
              </rule-type>
            </rule>
            <rule>
              <name>deny-all-others</name>
              <action>deny</action>
            </rule>
          </rule-list>
        </rules>
      </nacm>
    </config>
  </edit-config>
</rpc>
```

-----

### **Example 3: Enable NACM and set default access**

By default, NACM might be disabled or have a permissive default. You often want to explicitly enable it and set secure defaults.

```xml
<rpc message-id="105" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <edit-config>
    <target>
      <candidate/>
    </target>
    <default-operation>merge</default-operation>
    <config>
      <nacm xmlns="urn:ietf:params:xml:ns:yang:ietf-netconf-acm">
        <enable-nacm>true</enable-nacm>
        <read-default>permit</read-default>     <write-default>deny</write-default>      <exec-default>permit</exec-default>     </nacm>
    </config>
  </edit-config>
</rpc>
```

-----

### **Example 4: Delete a NACM rule**

If you want to remove a previously configured rule.

```xml
<rpc message-id="106" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <edit-config>
    <target>
      <candidate/>
    </target>
    <config>
      <nacm xmlns="urn:ietf:params:xml:ns:yang:ietf-netconf-acm">
        <rules>
          <rule-list>
            <name>read-only-ops-rules</name>
            <rule operation="delete">
              <name>deny-interface-writes</name>
            </rule>
          </rule-list>
        </rules>
      </nacm>
    </config>
  </edit-config>
</rpc>
```

-----

### **Final Step: Commit Changes**

After sending any of the `<edit-config>` operations, you must commit the changes to the `running` datastore for them to take effect.

```xml
<rpc message-id="107" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <commit/>
</rpc>
```

**Important Considerations:**

  * **Order of Rules:** NACM processes rules in a defined order. Rules with a more specific `module-name` and `path` are generally evaluated before more general rules. The order of rules within a `<rule-list>` also matters on some implementations.
  * **Default Behavior:** Be extremely careful when setting `read-default`, `write-default`, and `exec-default`. A misconfiguration here can lock you out of the device.
  * **Emergency User:** Always ensure you have an "emergency" user or method to access the device if NACM rules accidentally lock out all users.
  * **YANG Models:** You need access to the relevant YANG models (e.g., `ietf-netconf-acm.yang`, `Cisco-IOS-XE-interface.yang`, `Cisco-IOS-XE-interfaces-oper.yang`) to correctly formulate the `module-name` and `path` elements in your NACM rules. These are usually found on Cisco's developer documentation portals.

These examples provide a starting point. Real-world NACM deployments can become quite complex depending on the granularity of control required.








