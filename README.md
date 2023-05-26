# F5 ClearPass Guest Login Simulation

The `f5-sim-cp-login` script simulates a guest login to ClearPass Guest with HTTPS, using the F5 external monitor shell script and curl. This simulation offers a more efficient alternative to the standard HTTPS monitor proposed in the outdated "Deploying CPPM with F5 BIG-IP Local Traffic Manager (LTM)" tech note from 2014.

The script tests the guest application's default login page at `/guest/auth_login.php` by performing a guest operator login and logout. This provides a more robust monitor than the standard one as it can detect potential issues like corrupted Web Login pages, which might still pass a stock HTTPS monitor and even return an HTTP status code 200.

## Installation and Update

Follow these steps to install or update the script on an F5 device:

1. Login to the F5 User Interface (UI).
2. Navigate to `System > File Management > External Monitor Program File List`.
3. Click `Import`.

To create a monitor object:

- On an LTM device, go to Local Traffic > Monitors > Create. Fill in the name and description fields, and select 'External' as the type.
- On a GTM (BigIP DNS) device, go to DNS > GSLB > Monitors > Create. Fill in the name and description fields, and select 'External' as the type.

Consider including a link to this GitHub repository in the description field of the monitor to facilitate future references and updates, especially since the operational details of this script aren't documented elsewhere.

Then, add USERNAME and PASSWORD variables. In the Variables section of the monitor configuration, set USERNAME to your test username and click 'Add'. Then set PASSWORD to your test password and click 'Add' again. For a quick setup, you can initially use the plaintext PASSWORD variable. However, it's recommended to switch to an encrypted password once the monitor is operational (refer to the Password Encryption section).

Finally, click 'Finished' to complete the configuration of your monitor object.

## Script Variables

The script uses the following variables, which are set in the monitor object itself:

- `USERNAME`: your monitor username
- `PASSWORD`: plaintext password (should only be used for testing)
- `ENCRYPTED_PASSWORD`: an aes-256-cbc encrypted password (overrides `PASSWORD` if set)
- `DECRYPTION_KEY_FILE_NAME`: filename of the decryption key iFile for `ENCRYPTED_PASSWORD`
- `LOG_LEVEL`: (optional) logger level (debug, info, notice, warning, err, crit, alert, emerg)

There is no need to encode special characters with URL encoding (e.g., '%21' for '!') on the tested versions. However, this could be version-dependent. 

The script must be installed on all LTM pairs AND each GTM for accurate functioning. For GTM devices, you must manually install or update the script as these changes don't sync like other GTM configurations.

## Behavior and Error Detection

The script frequently uses "> /dev/null 2>&1". This is to ensure that the monitor only returns "UP" when the script completes fully. This is due to the behavior of the F5 external monitor, which considers the monitor successful if ANYTHING is output to STDOUT or STDERR.

The script checks for HTTP status code 302 intentionally, as an invalid username/password would return a status code 200 along with an error message, while a correct login would return a 302.

On the ClearPass side, configure your guest operator login policy to map the test username to the default "Null Profile". This allows login but assigns no privileges. The test username/password can be configured in the local user database, or an external resource like Active Directory if desired.

## Password Encryption

1. Generate a secure, random 32-character (or longer) decryption key. Save this key in a text file temporarily.
2. Import this file into your LTMs and GTMs as an iFile via `System > File Management > iFile List`.
3. Run this command to encrypt your password: `echo 'your-password' | openssl enc -aes-256-cbc -base64 -k 'your-decryption-key'`
4. Save the encrypted password as a variable in the monitor with the name `ENCRYPTED_PASSWORD`.

### Password Decryption Key File

Only the first line of this file is read as the decryption key; all subsequent lines are disregarded. 

It's essential to upload this iFile to EVERY device - remember, GTMs don't automatically sync this information. If you have LTMs that are part of a sync group, they will synchronize this iFile among themselves.


## Limitations

1. When you upload a file as an iFile, the BIG-IP will append some numeric identifier, which isn't predictable. To avoid accidentally matching the wrong file, ensure you're using a unique file name for the decryption key file.
2. If you want to monitor the same resource in multiple locations (e.g., at the node and in the pool), create a separate monitor (with a different name) for each. Using the same monitor could unintentionally kill the process because the same pid file is used.

Please refer to the ClearPass F5 Tech Note here: https://www.arubanetworks.com/techdocs/ArubaDocPortal/content/cp-resources/cp-tech-notes.htm (Click the link for F5 Load Balancers)

The project has been tested on ClearPass 6.9.13 and BigIP 14.1.4.6.

Your updated troubleshooting section provides some more helpful details. Here's a refined version:

## Troubleshooting

In case you encounter issues while running the script, refer to the following troubleshooting steps:

1. **Check Login Credentials:** Ensure your username and password can successfully log into `/guest/auth_login.php` from a web browser. Use the Access Tracker in ClearPass to confirm a successful login status.
2. **Open Log:** Log into the F5 bash shell and execute the command `tail -f /var/log/ltm`. Keep this window open for further troubleshooting.
3. **Set Debug Level:** In the monitor configuration, set the `LOG_LEVEL` variable to `debug`.
4. **Configure Password:** Remove the `ENCRYPTED_PASSWORD` variable and set the `PASSWORD` variable to your plaintext password.
5. **Check Log for Errors:** With the log still open, verify if the script functions as expected with the plaintext password. Any errors or issues will be logged here.
6. **Verify ClearPass Access Tracker:** Check the ClearPass Access Tracker to confirm if it's receiving the request from the test username and verify its response.
7. **Test Encrypted Password:** Once the script functions correctly with the plaintext password, retry with the encrypted password. Ensure `DECRYPTION_KEY_FILE_NAME` is correctly set to the appropriate file. If necessary, uncomment the following lines in your script for additional debugging:
        `#LOG_MESSAGE "Decryption Key: $DECRYPTION_KEY" "debug"`
        `#LOG_MESSAGE "Decrypted Password: $PASSWORD" "debug"`
   Remember that this could potentially expose sensitive information in your logs, especially if they're transmitted in cleartext to a syslog server. To avoid this risk, consider redirecting these messages to a separate output file by uncommenting these lines and replacing `/path/to/outputfile` with a valid path:
        `#echo "Decryption Key: $DECRYPTION_KEY" >> /path/to/outputfile`
        `#echo "Decrypted Password: $PASSWORD" >> /path/to/outputfile`
   Ensure to securely delete this output file once you're done with debugging.
8. **Unset Debug Level:** Once everything functions as expected, delete the `LOG_LEVEL` variable to stop the verbose logging and return it to the default level. Also, remember to re-comment any lines that were uncommented during the debugging process in step 7 to maintain the security of your environment and prevent potentially sensitive information from being logged.

## About

### Version History

- **1.3**: Simplified and improved logging, enhanced error handling, added the ability to specify any decryption key file, and fixed minor bugs.
- **1.2.1**: Moved MIN_LOG_LEVEL and LOGGING variables to be configured by the monitor. Enabled logging by setting LOGGING to true or specifying a valid MIN_LOG_LEVEL.
- **1.2**: Added password encryption and improved PID management, error detection, and cleanup.
- **1.1**: Initial published version with cleartext passwords and limited logging.

### Disclaimer

This script is provided for the purpose of testing and troubleshooting, and is intended to be used in a responsible manner. It is not designed for, and should not be used for, unauthorized access to any systems. While efforts have been made to ensure its accuracy and reliability, the author assumes no responsibility for any issues or complications that may arise from the use of this script within your environment. Users are advised to carefully evaluate the script's applicability to their specific needs and to take appropriate precautions in its usage.

### License

This project is licensed under the MIT License. See the LICENSE file for more details.

### Author

Tim Haynie, CWNE #254, ACMX #508 [LinkedIn Profile](https://www.linkedin.com/in/timhaynie/)
