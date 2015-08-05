# Salesforce and IBM Watson Tone Analyzer integration using Mule.

In this competitive world, it is very important to understand the linguistic tone of a customer so that we can decide whether we need to respond to the customer immediately.

These days, customer satisfaction is more important to a sales manager or representative. A sales manager or representative may want to understand the linguistic tone from his customer's sent mail. For example, when a customer sends a mail to a sales manager or representative, the sales manager or representative may want to understand the tone of the mail so that the sales manager or representative can decide on further actions. This powerful, of course most needed, feature can be achieved by leveraging [IBM Watson Tone Analyzer](http://www.ibm.com/smarterplanet/us/en/ibmwatson/developercloud/tone-analyzer.html) service. 

Here we want to show, how a [Salesforce](https://en.wikipedia.org/wiki/Salesforce.com) user(sales manager or representative) can understand the tone of his customer's sent mail by integrating Salesforce and IBM Watson Tone Analyzer service using [Mule Soft ESB](https://www.mulesoft.org/what-mule-esb).

We have written a Mule flow which makes the integration of [Salesforce](https://en.wikipedia.org/wiki/Salesforce.com) and [IBM Watson Tone Analyzer](http://www.ibm.com/smarterplanet/us/en/ibmwatson/developercloud/tone-analyzer.html) service very easy.

Below is the main flow that receives an incoming mail from a customer. First of all, we need to configure the interesed mail settings.

```xml
<flow name="tone-analyzer">
    <imaps:inbound-endpoint host="${gmail.host}" port="${gmail.port}" user="${gmail.user}" password="${gmail.password}" connector-ref="IMAP_for_Gmail" responseTimeout="10000" doc:name="imap"/>
    <logger message="From Address:  #[message.inboundProperties.fromAddress] - To Address: #[message.inboundProperties.toAddresses] - Subject: #[message.inboundProperties.subject] - Payload: #[payload]" level="INFO" doc:name="logger"/>
    <enricher target="#[flowVars.contactId]" doc:name="enrich-with-contact-id">
        <flow-ref name="get-sf-contact-id" doc:name="get-sf-contact-id"/>
    </enricher>
    <logger message="Contact Id: #[flowVars.contactId]" level="INFO" doc:name="logger"/>
    <flow-ref name="get-tone" doc:name="get-tone"/>
    <flow-ref name="update-sf-contact-with-tone" doc:name="update-sf-contact-with-tone"/>
</flow>
```
