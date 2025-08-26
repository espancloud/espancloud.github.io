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
