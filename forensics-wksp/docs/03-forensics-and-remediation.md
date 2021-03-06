# Module 3: Forensics and remediation

Unfortunately, due to a misconfiguration in your environment, a hacker has been able to gain access to your webserver. Now, with the intruder in your environment you’re getting alerts from the security tools you’ve put in place indicating nefarious activity. These alerts include communication with known malicious IP addresses, account reconnaissance, changes to S3 policies, and disabling security configurations. You must identify exactly what activity the intruder has performed and how they did it so you can block the intruder’s access, remediate the vulnerabilities, and restore the configuration to its proper state.

### Agenda

Perform forensics and remediate – 30 min

## Find out what's happening!

You’ve received the first alerts from GuardDuty. Now what? Since the alert came from GuardDuty, we will check there first.

### Check GuardDuty findings

1.  Go to the [Amazon GuardDuty](https://us-west-2.console.aws.amazon.com/guardduty/home?region=us-west-2) console.
2.  In the navigation pane, click on **Findings**.  You should see all the findings below:
    ![GuardDuty Findings](../images/03-gd-findings.png)

    > Don't panic if all of these findings fail to show up at first. The findings generated will take at least 20 minutes to show up in the GuardDuty console.
3.  The high severity ![High Severity](../images/03-high-severity.png) **UnauthorizedAccess:EC2/SSHBruteForce** finding is a result of the activity going on in the background to simulate the attack and can be ignored. You can archive the finding if you choose with the steps below:
    * Click on the check box next to the "**UnauthorizedAccess:EC2/SSHBruteForce**" **Finding**.
    * Click on **Actions**.
    * Select **Archive**.  

      > If you're interested in seeing all of your findings (current and archived) you can click on the filter icon to the left of *Add filter criteria* to toggle them in the console.
    * After archiving you should have four findings that are associated with this workshop.
4.  Now let's examine the low severity ![Low Severity](../images/03-low-severity.png) **UnauthorizedAccess:EC2/SSHBruteForce** finding since this was one of the first findings to show up. 
    * Click on the **Finding**.
    * Review the **Resource affected** and other sections in the window that popped open on the right.
    * Copy down the **GuardDuty Finding ID** and the **Instance ID**.

      > Was the brute force attack successful?

### Check Inspector assessment and CloudWatch logs

Following security design best practices you already setup your instances to send certain logs to CloudWatch. You’ve also setup a CloudWatch event rule that runs an [AWS Inspector](https://aws.amazon.com/inspector/) scan when GuardDuty detects a particular attack. Let’s look at the Inspector findings to see if the SSH configuration adheres to best practices to determine what the risk is for the brute force attack. We will then examine the CloudWatch logs.

1.  Go to [Amazon Inspector](https://us-west-2.console.aws.amazon.com/inspector/home?region=us-west-2) in the Amazon Console.
2.  Click **Findings** on the left navigation.
    
    > If you do not see any findings after awhile, there may have been an issue with your Inspector agent.  Click on **Assessment Templates**, check the template that starts with **forensics-wksp**, and click **Run**.  Please allow **15 minutes** for the scan to complete.  You can also look in **Assessment runs** and check the **status**. Feel free to continue through this module and check the results later on. 

3.  Filter down the findings by typing in **Password**.
4.  Review the findings.
    ![Inspector Findings](../images/03-inspector-findings.png)

    > Which Inspector [rule packages](https://docs.aws.amazon.com/inspector/latest/userguide/inspector_rule-packages.html) were used for this scan?

Based on the findings you see that password authentication is configured on the instance with no password complexity restrictions which means the instance is more susceptible to a SSH brute force attack. Now that we know that let’s look at the CloudWatch logs and create a metric to see if there are any successful SSH logins (to finally answer the question of whether the SSH brute force attack was successful.)

5.  Go to [CloudWatch logs](https://us-west-2.console.aws.amazon.com/cloudwatch/home?region=us-west-2#logs:).
6.  Click on the log group **/forensics-wksp/var/log/secure**
7.  If you have multiple log streams, filter using the Instance ID you copied earlier and click on the stream.
8.  Within the **Filter Events** text-box put the following Filter Pattern: 

    ```
    [Mon, day, timestamp, ip, id, msg1= Invalid, msg2 = user, ...]
    ```

    > Do you see any failed (invalid user) attempts to log into the instance?
9.  Now replace the Filter with one for successful attempts:

    ```
    [Mon, day, timestamp, ip, id, msg1= Accepted, msg2 = password, ...]
    ```

    > Do you see any successful attempts to log into the instance?

    > Which linux user was compromised?


### Check the remaining GuardDuty findings

Now that you have verified that the brute force attack was successful and your instance was compromised, go back to the [Amazon GuardDuty](https://us-west-2.console.aws.amazon.com/guardduty/home?region=us-west-2) console and view the other findings.

> Does it look like you are early in identifying the attack (just intrusion), or has the intruder started performing malicious actions?

View the following GuardDuty findings and take a note of the resources involved:

* **Recon:IAMUser/MaliciousIPCaller.Custom**
* **UnauthorizedAccess:IAMUser/MaliciousIPCaller.Custom**
* **UnauthorizedAccess:EC2/MaliciousIPCaller.Custom**

You can see by these findings that the compromised instance is communicating with an IP address on your custom threat list and that API calls are being made from a host on your custom threat list. 

> The API calls are being made using AWS IAM temporary security credentials coming from an IAM Role for EC2. How could you determine this fact yourself?

### Query CloudTrail logs with Athena

In order to find out more about what has happened, we'll perform some additional forensics using Amazon Athena, which lets us query the CloudTrail logs contained in S3.

You will first need to set up an Athena database for your CloudTrail logs against which you can run the queries that follow.

Begin by reading the documentation on this page: https://docs.aws.amazon.com/athena/latest/ug/cloudtrail-logs.html

1. Browse to the CloudTrail event history: https://us-west-2.console.aws.amazon.com/cloudtrail/home?region=us-west-2#/events
2. Click the link that says "Run advanced queries in Amazon Athena".
3. For Storage Location, select the CloudTrail S3 bucket that starts with "forensics-wksp" and ends in "-logs".
4. Click "Create table".
5. Once the table is created, click "Go to Athena" to go to the Athena console.
6. Make sure the Database is set to "default".
7. Copy the name of your table, which should be in the format `cloudtrail_logs_forensics_wksp_<account_id>_us_west_2_logs`, by clicking the ellipsis (...) to the right of the table name and selcting "Show properties".

For each query below, replace {TABLE} with the name of your table.

1. Identify some more details about the launch of the suspect EC2 instance. Replace the instance ID in this query with the ID of the suspect EC2 instance.

```sql
SELECT eventtime, useridentity, awsregion,
       sourceipaddress, useragent, requestparameters
FROM "default"."{TABLE}"
WHERE eventsource = 'ec2.amazonaws.com'
      AND eventname = 'RunInstances'
      AND regexp_like(responseelements, 'i-12345678901234567') limit 1;
```

2. Identify if any other EC2 instances were launched with the same EC2 key-pair as the suspect EC2 instance. Replace the string "suspect-keypair" with the name of the EC2 key used by the suspect EC2 instance.

```sql
SELECT *
FROM "default"."{TABLE}"
WHERE eventsource = 'ec2.amazonaws.com'
      AND eventname = 'RunInstances'
      AND regexp_like(requestparameters, '\"keyName\":\"suspect-keypair\"');
```

3. Identify if any other EC2 instances were launched with the same EC2 instance profile as the suspect EC2 instance. replace the string "suspect-instance-profile" with the name of the instance profile, or IAM role, with which the suspect EC2 instance is running.

```sql
SELECT *
FROM "default"."{TABLE}"
WHERE eventsource = 'ec2.amazonaws.com'
      AND eventname = 'RunInstances'
      AND regexp_like(requestparameters, '\"iamInstanceProfile\":{\"name\":\"suspect-instance-profile\"}');
```

4. Identify the last 2 days of activity for the threat actor (e.g. IAM user) whom launched the suspect EC2 instance. Replace the principal ID string with the actual principal ID of the user that launched the suspect EC2 instance.

```sql
SELECT *
FROM "default"."{TABLE}"
WHERE useridentity.principalid = 'AIDAJ45Q7YFFAREXAMPLE'
      AND (to_unixtime(now()) - to_unixtime(from_iso8601_timestamp(eventtime)))/(24*60*60) < 2;
```
 
5. Identify who created an IAM user involved in suspicious activities. Replace the string "suspicious-user" with the name of the IAM user that made the call to CreateUser. (Hint: try looking for that API call in CloudTrail via the console and examine the raw JSON for the event by clicking View Event).

```sql
SELECT eventtime, useridentity, awsregion,
       sourceipaddress, useragent, requestparameters
FROM "default"."{TABLE}"
WHERE eventsource = 'iam.amazonaws.com'
      AND eventname = 'CreateUser'
      AND regexp_like(requestparameters, '\"userName\":\"suspicious-user\"');
```

What additional info were you able to learn about the incident by performing these queries?

## Stop and evaluate

So at this point you have identified a successful intrusion into your network and specific actions taken against your account. Let’s recap what those are:

* A SSH brute force attack against an internet accessible EC2 instance was successful.
* The EC2 instance communicated with a known malicious IP address (possibly indicating that malware was installed.)

Now that you've identified the attacker’s actions you need to stop them from performing any additional activities, restore your systems to their previous configurations, and protect your resources so this can’t happen again.

## Respond and remediate

Before we get ahead of ourselves, we must stop any further actions from taking place. This requires removing the foothold in our environment, revoking any active credentials or removing those credentials' capabilities, and blocking further actions by the attacker. 

### Verify your automated remediation

Based upon your existing work, you’ve implemented the first step by using the CloudWatch Event rule to trigger the Lambda function to update the NACL for the subnet that the instance is located in. Let’s look at what changed.

1.  Go to the [AWS Config](https://us-west-2.console.aws.amazon.com/config/home?region=us-west-2) console.
2.  You may initially see the following message (if you don't see the message just go on to the next step):
    ![Config Message](../images/03-config-message.png)

    Click the **Click Here** button to proceed with using Config without Config Rules

3.  Click **Resources** in the left navigation.
4.  We can search here to find configuration changes to the NACL. Select the radio button for **Tag** and enter **Name** for the **Tag key** and **forensics-wksp-compromised** for the **Tag value** (this is allowing us to search for the NACL with that name):
    ![Config Key Pair](../images/03-config-keypair.png)
6.  Click on the Network ACL ID in the **Config timeline** column.
7.  On the next screen you should see two **Change** events in the timeline. Click on **Change** for each one to see what occurred.
8.  Evaluate the NACL changes.

    > What does the new NACL rule do?

### Modify the security group

Now that the active session from the attacker has been stopped by the update to the NACL, you need to modify the Security Group associated with the instance to prevent the attacker or anyone else from coming from a different IP.

1.  Go to the [Amazon EC2](https://us-west-2.console.aws.amazon.com/ec2/v2/home?region=us-west-2) Console.

2.  Find the running instances with the name **forensics-wksp: Compromised Instance**.

3.  Under the **Description** tab, click on the Security Group for the compromised instance.

4.  View the rules under the **Inbound** tab.

5.  Click **Edit** and delete the inbound SSH rule. You've decided that all administration on EC2 Instances will be done through [AWS Systems Manager](https://aws.amazon.com/systems-manager/) so you no longer need this port open.

    > In your initial setup you already installed the SSM Agent on your EC2 Instance.
6. Click **Save**


### Revoke IAM role active sessions

Now that the attacker can’t SSH into the compromised instance, you need to revoke all active sessions for the IAM role associated with that instance.


1.  Browse to the [AWS IAM](https://console.aws.amazon.com/iam/home?region=us-west-2) console.

2.  Click **Roles** and the **forensics-wksp-compromised-ec2** role (this is the role attached to the compromised instance).

3.  Click on the **Revoke sessions** tab.

4.  Click on **Revoke active sessions**.

5.  Click the acknowledgement **check box** and then click **Revoke active sessions**. 
	> What is the mechanism that is put in place by this step to actually prevent the use of the temp credentials issued by this role? 

### Restart instance to rotate credentials

Now all active sessions for the compromised instance role have been invalidated.  This means the attacker can no longer use those credentials, but it also means that your application that use this role can't as well.  In order to ensure the availability of your application you need to refresh the credentials on the instance.  

To change the IAM credentials on the instance, you must Stop and Start the instance. A simple reboot will not change the keys.  Since you are using AWS Systems Manager for doing administration on your EC2 Instances you can use it to query the metadata to validate that the credentials were rotated.

First verify what the current credentials are.   

1.  Go to [AWS Systems Manager](https://us-west-2.console.aws.amazon.com/systems-manager/managed-instances?region=us-west-2) console and click **Managed Instances** (found under the **Shared Resources** section on the left navigation).
    
    > You should see an instance named **forensics-wksp: Compromised Instance** with a **Ping status** of **Online**.
2.  To see the keys currently active on the instance, click on **Run Command** on the left navigation and then click **Run a Command**.
4.  For **Command document** select **AWS-RunShellScript** 
	> You may need to scroll through the list or just enter the document name in the search bar.
5.  Leave the **Document version** at the default. 
6. Under **Targets** check the checkbox next to **forensics-wksp: Compromised Instance**.
7.  Under **Command Parameters** type the following in **Commands**:

    ```
    curl http://169.254.169.254/latest/meta-data/iam/security-credentials/forensics-wksp-compromised-ec2
    ```

7.  Leave all of the other options at their default, scroll to the end and click **Run**.
8.  Scroll down to the **Targets and outputs** section and click the  **Instance ID** (which should correspond to the instance ID of the compromised instance)
10. Expand **Step 1 - Output** and make note of the **AccessKeyID**, **SecretAccessKey**, and **Token**. (the command will take a minute or so to run)

Next, you need to stop and restart the Instance.

11. In the [EC2 console](https://us-west-2.console.aws.amazon.com/ec2/v2/home?region=us-west-2#Instances:sort=instanceId) **Stop** the Instance named **forensics-wksp: Compromised Instance**.
12. Wait for the Instance State to say **Stopped** and then **Start** the instance.

Lastly, you can need to query the metadata again to validate that the credentials were changed.

13. Repeat the first 10 steps to retrieve the credentials again.

    > Notice the keys are different.

    > If you want, try again after rebooting the instance. The keys will stay the same.

This is a good use case for auto-scaling groups and golden-image AMI’s, but that is out of scope for this workshop. Also out of scope is clearing the suspected malware.

After you have remediated the incident and further hardened your environment, you can proceed to the next module.

> If you are going through this workshop in a classroom setting then the instructor should start the module 4 presentation soon.

### **[Module 4 - Review and Cleanup](../docs/04-review-and-cleanup.md)**