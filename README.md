# openmrs-contrib-slackform

A simple form for joining OpenMRS on Slack.

Thanks to [this blog post](https://intoli.com/blog/make-a-public-slack-community/) for tips on how to set this up. The blog post has better descriptions and includes images walking through each step. Key parts are added to this README for posterity sake (in case the blog post becomes unavailable in the future).

---

* We created an "OpenMRS Slack invitation request" Google form that just takes a user email and puts it into a "Slack Signups" spreadsheet (i.e., timestamp and email address on each row). Both the form and spreadsheet are stored in the OpenMRS Infrastructure Google Shared Drive folder. 
* We scraped the form's submit action by inspecting the POST produced by submitting the Google Form and put that into our [own signup HTML form](index.html).
* We created an "OpenMRS Inviter" user in Slack with admin privs and generated a slack API token for that user to use for generating invitations.
* Using the Script Editor for the spreadsheet, we added the following script:

```javascript
/**
 * Configure this `slack` object with the name of your workspace and the legacy API token.
 * These will be used to automatically send invitations using Slack's API whenever the
 * invitation form is submitted.
 */
var slackConfig = {
  workspace: 'openmrs',
  token: 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx',
};


/**
 * Handles form submission events by sending invitation emails using Slack's API.
 */
function sendInvitation(event) {
  // Extract the form data from the event.
  const timestamp = event.values[0];
  const emailAddress = event.values[1];

  // Log a message about sending the invitation.
  // Logger.log(JSON.stringify(event)); // log raw event for debugging
  Logger.log('Invitation request received at ' + timestamp + ' for ' + emailAddress + '.');

  // Use Slack's API to send the invitation.
  const url = (
    'https://' + slackConfig.workspace + '.slack.com/api/users.admin.invite' +
    '?email=' + encodeURIComponent(emailAddress) +
    '&token=' + encodeURIComponent(slackConfig.token)
  );
  const response = UrlFetchApp.fetch(url);
  const status = response.getResponseCode();
  const responseText = response.getContentText();

  // Log the outcome.
  if (status >= 200 && status <= 299) {
    Logger.log('Invitation successfully sent to ' + emailAddress + '.');
  } else {
    Logger.log('Invitation sending failed for ' + emailAddress + ': ' + responseText);
  }
}


/**
 * Attaches the `sendInvitation()` method to a trigger that fires whenever
 * the invitation form is submitted.
 */
function attachSendInvitationHandler() {
  // This is the name of the function that will handle form submissions.
  const handlerFunction = 'sendInvitation';

  // Bail out if the trigger already exists.
  const alreadyAttached = ScriptApp
    .getProjectTriggers()
    .some(function (trigger) {
      return trigger.getHandlerFunction() === handlerFunction
    });
  if (alreadyAttached) {
    return;
  }

  // Create a new trigger that will call our handler when the form is submitted.
  ScriptApp
    .newTrigger(handlerFunction)
    .forSpreadsheet(
      SpreadsheetApp.getActiveSpreadsheet()
    )
    .onFormSubmit()
    .create();
}
```

Finally, running the `attachSendInvitationHandler()` method once registers the script to run on form submissions and, the first time we run it, prompts for us to grant permissions for the script. After that is done, any submissions for the Google Form (or, in our case, our custom form that mimics the Google Form submission) will add a row to the spreadsheet and then invoke `sendInvitation(event)`.
