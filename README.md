# Mintlify Starter Kit

Click on `Use this template` to copy the Mintlify starter kit. The starter kit contains examples including

- Guide pages
- Navigation
- Customizations
- API Reference pages
- Use of popular components

### Development

Install the [Mintlify CLI](https://www.npmjs.com/package/mintlify) to preview the documentation changes locally. To install, use the following command

```
npm i -g mintlify
```

Run the following command at the root of your documentation (where mint.json is)

```
mintlify dev
```

### Publishing Changes

Install our Github App to autopropagate changes from youre repo to your deployment. Changes will be deployed to production automatically after pushing to the default branch. Find the link to install on your dashboard. 

#### Troubleshooting

- Mintlify dev isn't running - Run `mintlify install` it'll re-install dependencies.
- Page loads as a 404 - Make sure you are running in a folder with `mint.json`

# TODO

# Which API should I choose?

## Real-Time

- rate limit
- higher unknown % (greylisting, connections taking 40+ seconds)

## Batch

- batches are processed sequentially by default

```javascript
while(true) {

  // check status of active batches
  var batchesActive = getActiveBatches();
  batchesActive.forEach(batch => {
    var batchStatus = bouncer.batchStatus(batch.batchId);
    if (batchStatus.status === 'completed') {
      // download results
      var batchResults = bouncer.batchResults(batch.batchId);
      storeBatchResults(batch.batchId, batchResults);
      storeBatchInformation(batch.batchId, 'done')
    }
  });


  if (batchesActive.size() < 10) {
    // create batch if needed
    var emailsToVerify = prepareBatchOfEmailsToVerify();
    if (emailsToVerify.size() > 0) {
      var batchStatus = bouncer.batchCreate(emailToVerify);
      storeBatchInformation(batchStatus.batchId, 'active');
    }
  }

  sleepSeconds(10);
}
```

## Batch/Sync

- integration should be single threaded
- we keep internal queue and remember results for 24h

Best way to implement integration using batch/sync endpoint, would be to create a queue of emails to be verified.
In some kind of pseudocode it would look like this.

```javascript
while(true) {

  var emailsToVerify = sql('select email from queue where result == null order by queuedAt limit 10000');
  if (emailsToVerify.size() == 0) {
    sleepSeconds(10);
  }
  var results = bouncer.batchSync(emailToVerify);
  updateQueueWithResults(results);
  sleepSeconds(3);
}
```

This way Bouncer's processing throughput won't be limited by the number of emails sent in each request.

# Integration Implementation Steps

- use sandbox emails
- write client (TODO prepare clients for most popular languages)

## Authentication

## Encoding

when using real-time
- make sure that '+' is encoded properly 
otherwise bouncer will receive 'test+123@usebouncer.com' as 'test 123@usebouncer.com' and return undeliverable invalid_email

when using batch
- '\' is a special character in json and "deliverable+6@sandbox.usebouncer.com\" is not a valid json value

## Error Handling

- 400 there was something wrong with the payload
- 402 purchase is needed
- 429 delay and retry
- 503 and 504 delay and retry

## How to interpret results?

Suppress emails if:
- Toxicity level is 4 or 5: These emails are considered highly risky and should be suppressed to avoid potential spam complaints.
- Email is disposable: Disposable email addresses are often used for one-time purposes and are not reliable for ongoing communication. Depending on the time of email content sent out, these emails can be suppressed from the beginning or further down the subscriber lifecycle.
- Score is between 0-50: This indicates a low-quality email address that is likely to cause deliverability issues. We suggest investigating these with internal data, not only related to email metrics as this can be quite a harsh segmentation.
- Abuse flag is set to yes (toxic in api response): These addresses indicate that it has a history of abuse or was involved in spamming or fraud.
- "Did you mean" column is not empty: This suggests that the email address may be incorrect or mistyped.
- Freebox is yes + Character Count low: Freebox addresses (e.g., @hotmail.com, @gmail.com) are often used for spam and should be monitored closely. (+) Character count before "@" is 5 or fewer: Short local parts of email addresses are often indicative of spam traps.
- Freebox is yes + Role is yes: Role account emails using freemail addresses such as test@gmail.com or info@hotmail.com most likely have no actual human using the inbox.
- Mailbox is full: Addresses with full mailboxes are not receiving emails, indicating they may be inactive. Please note, these may already be unsubscribed/suppressed by your ESP.

To Check Rules
Mark emails as "To Check" if:

Status is unknown AND:
- Accept all is yes: These domains accept all emails, making it hard to verify if the address is valid.
- Reason is timeout: These addresses may not have responded during validation.
- Provider is Yahoo or Microsoft: These providers have specific filtering rules, and addresses should be checked manually. (Yahoo for example is also an accept-all email address)
- Abuse is unknown: It's unclear if these addresses have a history of abuse.
- Toxicity is 1: These addresses are slightly risky and require further checking.

Status is risky AND:
- Toxicity is 2 or 3: These addresses are moderately risky.
- Role account: Addresses like info@, admin@, etc., should be manually verified.
- Provider is Yahoo or Microsoft: These providers' addresses should be checked manually.
- Accept all is yes: These domains accept all emails, making it hard to verify if the address is valid.
