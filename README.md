# Welcome to the Detection-as-Code (DAC) & MITRE ATT&CK Workshop
This guide will provide you with a step-by-step of all the commands that will be needed during the hands-on portion of the workshop. If you have questions, feel free to ask your group moderator.

**Helpful Links**
- [All Available Rule Functions](https://github.com/panther-labs/panther-analysis/blob/master/templates/example_rule.py)
- [What is Deep_Get?](https://docs.panther.com/writing-detections/globals#deep_get)
- [What are Packs?](https://docs.panther.com/writing-detections/detection-packs)
- [Panther Analysis Tool](https://docs.panther.com/panther-developer-workflows/panther-analysis-tool#overview)
- [Lookup Tables](https://docs.panther.com/enrichment/lookup-tables)
- [Unit Tests](https://docs.panther.com/writing-detections/testing#mocks)
- [MITRE ATT&CK Mapping in Panther](https://docs.panther.com/detections/report-mapping)




## Lesson 1 - Writing a Detection for CloudTrail IAM Logs

**Part 1 - Prepare Detection Writing**
1. In the Panther Console - Navigate to Build > Detections > Create New
2. Give it a unique name "[YOUR NAME] Failed Login IAM Detection" 
3. Set Severity to "Medium" and Log Types "AWS.CloudTrail"

**Part 2 - Create Unit Test**
1. Select "Functions and Tests" in the tab below
2. Scroll down and select the "Create Test" button
3. Delete the brackets populated. Copy and paste the sample event below into your console

**CloudTrail IAM Sample Log**
``` json
{
	"additionalEventData": {
		"LoginTo": "https://console.aws.amazon.com/console/",
		"MFAUsed": "No",
		"MobileVersion": "No"
	},
	"awsRegion": "us-east-1",
	"eventID": "1",
	"eventName": "ConsoleLogin",
	"eventSource": "signin.amazonaws.com",
	"eventTime": "2019-01-01T00:00:00Z",
	"eventType": "AwsConsoleSignIn",
	"eventVersion": "1.05",
	"p_event_time": "2021-06-04 09:59:53.650807",
	"p_log_type": "AWS.CloudTrail",
	"p_parse_time": "2021-06-04 10:02:33.650807",
	"recipientAccountId": "123456789012",
	"requestParameters": null,
	"responseElements": {
		"ConsoleLogin": "Failure"
	},
	"sourceIPAddress": "111.111.111.111",
	"userAgent": "Mozilla",
	"userIdentity": {
		"accountId": "123456789012",
		"arn": "arn:aws:iam::123456789012:user/tester",
		"principalId": "1111",
		"type": "IAMUser",
		"userName": "tester"
	}
}
```

<details>
	<summary>Click To View Answer </summary>
	
```
from panther_base_helpers import deep_get

def rule(event):
    return event.get("eventName") == "ConsoleLogin" and deep_get(event,"responseElements","ConsoleLogin") == "Failure"
```
</details>

**Part 3 - Writing your detection code**

1. Import deep_get function from the panther_base_helpers library ```from panther_base_helpers import deep_get```
2. All rules require a "rule" function that is a boolean to trigger an alert - True fires and alert 
3. Create a rule function with ```def rule(event)```
4. To write the rule, identify the event attributes that associate with a failed login. This should be "eventName" and "ConsoleLogin"
5. Using event.get and deep_get to grab attributes from the event log, write a return statement that is TRUE when a console login attempt fails







## Lesson 2 - Tune an Existing GuardDuty Detection created by Panther


**Part 1 - Clone a Managed Detection**
1. In the Panther Console - Navigate to Build > Packs > Panther Core AWS Pack
2. Select the AWS GuardDuty High Severity Finding
3. Select Clone & Edit on the Top Right | IF you're using a shared dev instance, please copy & paste detection to a new one. Do NOT clone & edit to avoid merge conflicts

**Part 2 - Prepare Unit Test**

1. Name the detection a unique name with your initials - Sample "AWS GuardDuty High Severity Finding - [YOUR NAME]"
2. Select Functions & Tests
3. Scroll down and populate test with log if not already done

**CloudTrail GuardDuty Log**
```
{
"accountId": "123456789012",
"arn": "arn:aws:guardduty:us-west-2:123456789012:detector/111111bbbbbbbbbb5555555551111111/finding/90b82273685661b9318f078d0851fe9a",
"createdAt": "2020-02-14T18:12:22.316Z",
"description": "Principal AssumedRole:IAMRole attempted to add a highly permissive policy to themselves.",
"id": "eeb88ab56556eb7771b266670dddee5a",
"partition": "aws",
"region": "us-east-1",
"schemaVersion": "2.0",
"service": {
	"action": {
		"actionType": "AWS_API_CALL",
		"awsApiCallAction": {
			"affectedResources": {
				"AWS::IAM::Role": "arn:aws:iam::123456789012:role/IAMRole"
			},
			"api": "PutRolePolicy",
			"callerType": "Domain",
			"domainDetails": {
				"domain": "cloudformation.amazonaws.com"
			},
			"serviceName": "iam.amazonaws.com"
		}
	},
	"additionalInfo": {},
	"archived": false,
	"count": 1,
	"detectorId": "111111bbbbbbbbbb5555555551111111",
	"eventFirstSeen": "2020-02-14T17:59:17Z",
	"eventLastSeen": "2020-02-14T17:59:17Z",
	"evidence": null,
	"resourceRole": "TARGET",
	"serviceName": "guardduty"
},
"severity": 8,
"title": "Principal AssumedRole:IAMRole attempted to add a policy to themselves that is highly permissive.",
"type": "PrivilegeEscalation:IAMUser/AdministrativePermissions",
"updatedAt": "2020-02-14T18:12:22.316Z"
}
```



**Part 3 - Tune Detection with Severity Function**
1. Capture all guardduty detections as alerts in Panther, but tune out the lower end ones. 
2. Modify the rule function to alert on events from severity 1 to 10
3. To reduce noise of this detection, use the severity function to create dynamic categorization of alerts
4. Use an IF statement to send severity 5 and below alerts to "INFO" level and 8 and above to "HIGH". For any other severity, return "MEDIUM"

<details>
	<summary>Click To View Answer </summary>
	
```
def severity(event):
    if float(event.get("severity",0)) <= 5.0:
        return "INFO"
    if float(event.get("severity",0)) >= 8.0:
        return "HIGH"
    else:
        return "MEDIUM"
```
</details>

## Lesson 3 - Adding MITRE ATT&CK Tags

1. Panther Managed Detections as well as Crowdstrike are automatically tagged with tactics and techniques
2. We will add tags for MITRE ATT&CK technique to the first detection that we wrote
3. Navigate to BUILD -> MITRE ATT&CK 
4. Find the "Brute Force" technique under "Credential Access" and add your detection to it
