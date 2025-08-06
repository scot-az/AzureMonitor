This Bicep template provides exactly what you need for monitoring Azure Monitor events with conditional webhook alerting. Here's what it includes:
Key Features:

Two Action Groups: One for P2 webhook and one for P3 webhook
Event ID 1014: Always triggers P2 webhook regardless of time
Event ID 6008:

Triggers P3 webhook during business hours (8 AM - 5 PM, Monday-Friday)
Triggers P2 webhook after hours and weekends


Generic template: For your additional event IDs (currently defaults to P2)

Business Hours Logic:

Uses KQL (Kusto Query Language) functions hourofday() and dayofweek()
Configurable business hours (default: 8 AM - 5 PM)
Considers weekends as after-hours

Customization Options:

Time Zone: Adjust the timeZone parameter
Business Hours: Modify businessHoursStart and businessHoursEnd
Evaluation Frequency: Currently set to 5 minutes, easily adjustable
Event IDs: Add your other event IDs to the eventIds array

To deploy this template:
bash# Create a parameters file (parameters.json)
az deployment group create \
  --resource-group your-rg-name \
  --template-file main.bicep \
  --parameters @parameters.json
Sample parameters.json:
json{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "logAnalyticsWorkspaceName": {
      "value": "your-log-analytics-workspace"
    },
    "p2WebhookUrl": {
      "value": "https://your-p2-webhook-url"
    },
    "p3WebhookUrl": {
      "value": "https://your-p3-webhook-url"
    }
  }
}
When you're ready to add your other event IDs, just let me know the specific webhook routing requirements for each, and I'll update the template accordingly!RetryClaude does not have the ability to run the code it generates yet.Show would i add supression to this?  I want it to only alert once per 30 minutes.  I don't want a second alert right away for all of these monitors.EditI'll show you how to add alert suppression to prevent duplicate alerts within 30 minutes. Azure Monitor has built-in suppression capabilities that we can configure in the alert rules.Azure Monitor Event Alerting with Conditional WebhooksCode ∙ Version 2 // Alert Rule for Event ID 1014 (Always P2 webhook)
resource alertRule1014 'Microsoft.Insights/scheduledQueryRules@2023-03-15-preview' = {
  name: '${alertRuleNamePrefix}-1014'
  location: location
  properties: {
    displayName: 'Event ID 1014 AlerAzure Monitor Event Alerting with Conditional WebhooksCode ∙ Version 3 // Alert Rule for Event ID 6008 during business hours (P3 webhook)
resource alertRule6008BusinessHours 'Microsoft.Insights/scheduledQueryRules@2023-03-15-preview' = {
  name: '${alertRuleNamePrefix}-6008-business-hours'
  location: location
  propertAzure Monitor Event Alerting with Conditional WebhooksCode ∙ Version 4 // Alert Rule for Event ID 6008 after hours (P2 webhook)
resource alertRule6008AfterHours 'Microsoft.Insights/scheduledQueryRules@2023-03-15-preview' = {
  name: '${alertRuleNamePrefix}-6008-after-hours'
  location: location
  properties: {
    displAzure Monitor Event Alerting with Conditional WebhooksCode ∙ Version 5 // Generic alert rule template for other event IDs (customize as needed)
resource alertRuleGeneric 'Microsoft.Insights/scheduledQueryRules@2023-03-15-preview' = [for eventId in eventIds: if(eventId != 1014 && eventId != 6008) {
  name: '${alertRuleNaPerfect! I've added alert suppression to all your monitoring rules. Here's what I added:
Key Suppression Changes:

muteActionsDuration: 'PT30M' - This suppresses alerts for 30 minutes after the first alert fires
customProperties: {} - Required property when using muteActionsDuration

How Alert Suppression Works:

First Alert: When an event ID is detected, the webhook is triggered immediately
Subsequent Alerts: For the next 30 minutes, if the same condition occurs, Azure Monitor will:

Still evaluate the rule every 5 minutes
Still detect the events in your logs
BUT it won't trigger the webhook again until the 30-minute suppression period expires



Suppression Duration Options:
If you want to adjust the suppression duration, you can modify the muteActionsDuration value:

'PT15M' = 15 minutes
'PT30M' = 30 minutes (current setting)
'PT1H' = 1 hour
'PT2H' = 2 hours
'PT24H' = 24 hours

Important Notes:

Per-Rule Suppression: Each alert rule has its own suppression timer
Condition-Based: Suppression applies to the specific condition (event ID + time period combination)
Auto-Reset: Once the suppression period expires, the next occurrence will trigger an alert again
Resolution Alerts: If autoMitigate: true triggers a resolution, it won't be suppressed

This setup ensures you get notified immediately when issues occur, but prevents alert fatigue from repeated notifications for the same ongoing issue within the 30-minute window.RetrySwhat is automitigate: true?  when would that be true?EditGreat question! autoMitigate: true is a crucial setting that controls how Azure Monitor handles alert resolution. Let me explain when and why alerts get resolved automatically.
What autoMitigate: true does:
When set to true, Azure Monitor will automatically resolve (close) an alert when the alerting condition is no longer met. This means:

Alert Fires: Event ID 1014 detected → Alert triggered → Webhook called
Condition Cleared: No Event ID 1014 found in next evaluation → Alert automatically resolved
Resolution Notification: A "resolved" notification can be sent (if configured)

When would alerts resolve with your current queries?
Looking at your current setup, here's when each alert would auto-resolve:
kql// This query looks for events in the last 5 minutes
Event
| where EventID == 1014
| where TimeGenerated > ago(5m)
Alert resolves when: No Event ID 1014 occurs in a 5-minute window.
The Problem with Your Current Configuration:
Your current setup might resolve alerts too quickly because:

Evaluation every 5 minutes looking back 5 minutes
Single event pattern: Most Windows events happen once, not continuously
Quick resolution: Alert might resolve in the next 5-minute cycle even if the underlying issue persists

When you might want autoMitigate: false:
bicepautoMitigate: false  // Manual resolution required
Use this when:

Events indicate persistent problems requiring human investigation
You want alerts to stay open until manually resolved
Single-occurrence events that represent completed error conditions

Better Query Pattern for Event Monitoring:
For most event monitoring scenarios, you might want to adjust the query to be less sensitive to resolution:
kql// Look for events in a longer window for more stable alerting
Event
| where EventID == 1014
| where TimeGenerated > ago(15m)  // Longer window
| summarize EventCount = count(), LastSeen = max(TimeGenerated) by Computer, EventID
Recommendation for Your Use Case:
For Windows Event monitoring, I'd suggest:

Keep autoMitigate: true - You want to know when issues are resolved
Increase the window size - Use 15-30 minutes instead of 5 minutes
Consider alert logic - Maybe alert on "more than X events in Y minutes" rather than "any events"
