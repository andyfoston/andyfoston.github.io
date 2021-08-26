+++
title = "Processing Kenesis Firehose logs for Splunk"
date = "2021-08-26"
author = "Andy Foston"
description = "Golang example lambda for processing Kenesis Firehose logs destined for Splunk"
+++

For a project at work, we needed to set up Kinesis Firehose to send logs from a provider’s logging
infrastructure (New Relic) into our logging infrastructure.

This by itself partly solved our use case, but there was a huge amount of logs that were coming
in which was essentially noise, and the logs that were coming in were for both nonproduction and
production and they were being routed to the same index.

There were many ways we could have solved this:

1. By asking New Relic to set up filtering/routing on their side (not ideal as we are not the owners of
this account and it depends on 3rd parties doing this work for us)
2. Configuring our Splunk Heavy Forwarder to preprocess these before forwarding to Splunk Cloud,
but this wasn’t a redundant service and would have meant that if the Heavy Forwarder had issues,
then we’d lose logging for some time.

In the end, we decided to use a Golang lambda to process these. To be honest, I found the
documentation a little sparse for doing this and figured it could be a good opportunity to help
others who are trying to do something similar to me.

## Skeleton lambda example

Some of this is probably documented elsewhere, but let's start off with a skeleton lambda:

```go
package main

import (
	"fmt"

	"github.com/aws/aws-lambda-go/events"
	"github.com/aws/aws-lambda-go/lambda"
)

func handleRequest(evnt events.KinesisFirehoseEvent) (events.KinesisFirehoseResponse, error) {

	fmt.Printf("InvocationID: %s\n", evnt.InvocationID)
	fmt.Printf("DeliveryStreamArn: %s\n", evnt.DeliveryStreamArn)
	fmt.Printf("Region: %s\n", evnt.Region)

	var response events.KinesisFirehoseResponse

	for _, record := range evnt.Records {
		fmt.Printf("RecordID: %s\n", record.RecordID)
		fmt.Printf("ApproximateArrivalTimestamp: %s\n", record.ApproximateArrivalTimestamp)

		var transformedRecord events.KinesisFirehoseResponseRecord
		transformedRecord.RecordID = record.RecordID
		transformedRecord.Result = events.KinesisFirehoseTransformedStateOk
		transformedRecord.Data = record.Data

		response.Records = append(response.Records, transformedRecord)
	}

	return response, nil
}

func main() {
	lambda.Start(handleRequest)
}
```

This snippet essentially does nothing to the records, but it provides a starting point to begin
processing these records.

### Support routing records to Splunk indexes other than the default

It is possible to inject an index name into the payload for Splunk, however there are a
few things that need to be done first.

Firstly, the Splunk HEC needs to have permission to write to all indexes that you plan to use
here. If you do not, Splunk will likely reject them and they'll likely end up being written
to the S3 bucket for unprocessable events (assuming you've set this up).

The other thing is that the 'Raw endpoint' will ignore the index parameter, and it'll just become part
of the logging payload stored in Splunk.

To fix this, you'll need to change the Splunk endpoint type to use an 'Event endpoint'. However, this
requires some more changes to our lambda in order to work:

```go
package main

import (
  "encoding/json"
	"fmt"
  "strings"

	"github.com/aws/aws-lambda-go/events"
	"github.com/aws/aws-lambda-go/lambda"
)

func formatRecordForSplunk(data map[string]json.RawMessage, index string) ([]byte, error) {
	response := make(map[string]json.RawMessage)
	response["time"] = data["timestamp"]
	response["sourcetype"] = json.RawMessage(`"_json"`)

	response["host"] = data["cluster"]
	response["source"] = data["eventType"]

	event, err := json.Marshal(data)
	if err != nil {
		return []byte(""), fmt.Errorf("Failed to marshal event \"%s\". Error: %s\n", data, err)
	} else {
		response["event"] = event
	}
	if index != "" {
		response["index"] = []byte(index)
	}

	marshalledResponse, err := json.Marshal(response)
	if err != nil {
		return []byte(""), fmt.Errorf("Failed to marshal splunk record \"%s\". Error %s\n", response, err)
	}
	return marshalledResponse, nil
}

func processRecord(record events.KinesisFirehoseRecord) (events.KinesisFirehoseResponseRecord, error) {
  var data map[string]json.RawMessage
	var transformedRecord events.KinesisFirehoseResponseRecord
  var index string
	transformedRecord.RecordID = record.RecordID
	transformedRecord.Result = events.KinesisFirehoseTransformedStateOk
	transformedRecord.Data = record.Data

  err := json.Unmarshal(record.Data, &data)
  if err != nil {
    transformedRecord.Result = events.KinesisFirehoseTransformedStateFailed
    return transformedRecord
  }

  cluster, ok := data["cluster"]
  if ok {
    if strings.Contains(string(cluster), "dev") {
      index = "dev_index"
    }
  }

  splunkRecord, err := formatRecordForSplunk(data, index)
  if err != nil {
    transformedRecord.Result = events.KinesisFirehoseTransformedStateFailed
  } else {
    transformedRecord.Data = splunkRecord
  }

  return transformedRecord
}


func handleRequest(evnt events.KinesisFirehoseEvent) (events.KinesisFirehoseResponse, error) {

	fmt.Printf("InvocationID: %s\n", evnt.InvocationID)
	fmt.Printf("DeliveryStreamArn: %s\n", evnt.DeliveryStreamArn)
	fmt.Printf("Region: %s\n", evnt.Region)

	var response events.KinesisFirehoseResponse

	for _, record := range evnt.Records {
		fmt.Printf("RecordID: %s\n", record.RecordID)
		fmt.Printf("ApproximateArrivalTimestamp: %s\n", record.ApproximateArrivalTimestamp)

		var transformedRecord events.KinesisFirehoseResponseRecord
		transformedRecord.RecordID = record.RecordID
		transformedRecord.Result = events.KinesisFirehoseTransformedStateOk
		transformedRecord.Data = record.Data

		response.Records = append(response.Records, transformedRecord)
	}

	return response, nil
}

func main() {
	lambda.Start(handleRequest)
}
```
This returns an object containing a few key fields that are required by Splunk, like the `source`, `sourcetype`, `host`, `time`
and optionally the `index` field, the actual event payload is then written to the `event` field.

### Dropping records

From the above, you may have noticed lines like `transformedRecord.Result = events.KinesisFirehoseTransformedStateFailed` lines.
The `...StateFailed` triggers Firehose to write the log to the backup destination (probably an S3 bucket).

However, if you want to drop events, you can return the event payload as normal, but set the `Result` to `events.KinesisFirehoseTransformedStateDropped`
this'll trigger the event to not be forwared to Splunk and not to be written to the backup destination either.

### Disclaimer

The code in here is a heavy stripped down and refactored version of what we're using for our lambda, so these versions have not
actually been tested directly, but should hopefully provide a basis to start from.
