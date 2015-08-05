# Salesforce and IBM Watson Tone Analyzer integration using Mule.

In this competitive world, it is very important to understand the linguistic tone of a customer so that we can decide whether we need to respond to the customer immediately.

These days, customer satisfaction is more important to a sales manager or representative. A sales manager or representative may want to understand the linguistic tone from his customer's sent mail. For example, when a customer sends a mail to a sales manager or representative, the sales manager or representative may want to understand the tone of the mail so that the sales manager or representative can decide on further actions. This powerful, of course most needed, feature can be achieved by leveraging [IBM Watson Tone Analyzer](http://www.ibm.com/smarterplanet/us/en/ibmwatson/developercloud/tone-analyzer.html) service. 

Here we want to show, how a [Salesforce](https://en.wikipedia.org/wiki/Salesforce.com) user(sales manager or representative) can understand the tone of his customer's sent mail by integrating Salesforce and IBM Watson Tone Analyzer service using [Mule Soft ESB](https://www.mulesoft.org/what-mule-esb).

We have written a Mule flow which makes the integration of [Salesforce](https://en.wikipedia.org/wiki/Salesforce.com) and [IBM Watson Tone Analyzer](http://www.ibm.com/smarterplanet/us/en/ibmwatson/developercloud/tone-analyzer.html) service very easy.

### Prerequisites
However, below are the prerequisites to test the flow shown below:

* An account at [IBM Blumix](https://console.ng.bluemix.net/)
* A deployed [Tone Analyzer](https://console.ng.bluemix.net/catalog/tone-analyzer/) service to [IBM Blumix](https://console.ng.bluemix.net/)
* An account at [Salesforce](https://login.salesforce.com/)
* A custom contact field `CutomerTone__c` on [Salesforce](https://login.salesforce.com/). See [here](https://help.salesforce.com/HTViewHelpDoc?id=adding_fields.htm) to create a custom field on any Salesforce object.

### <a name="tone-analyzer"></a>tone-analyzer flow:
Below is the main flow that receives an incoming mail from a customer.
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
The corresponding IMAPS connector configuration is below
```xml
<imaps:connector name="IMAP_for_Gmail" validateConnections="true" deleteReadMessages="false" doc:name="IMAP" checkFrequency="1000">
	<imaps:tls-client path="*" storePassword="*" />
	<imaps:tls-trust-store path="*" storePassword="*" />
</imaps:connector>
```
The [tone analyzer flow](#tone-analyzer) has three sub-flows, they are [get-sf-contact-id](#get-sf-contact-id), [get-tone](#get-tone) and [update-sf-contact-with-tone](#update-sf-contact-with-tone). Each flow is detailed in its own section. 

### <a name="get-sf-contact-id"></a>get-sf-contact-id flow:
This sub-flow retrieves the Salesforce contact id corresponding to the customer mail address. 
```xml
<sub-flow name="get-sf-contact-id">
  <sfdc:query config-ref="Salesforce__Basic_authentication" query="dsql:SELECT Id FROM Contact WHERE email = #["'" + org.apache.commons.lang3.StringUtils.substringBetween(message.inboundProperties.fromAddress, "<", ">") + "'"]" doc:name="get-sf-contact-id"/>
  <component class="com.capiot.sf.ContactIdRetriever" doc:name="retrieve-contact-id"/>
</sub-flow>
```
To retrieve the contact id, we use Mule Salesforce connector and below is the connector configuration
```xml
<sfdc:config name="Salesforce__Basic_authentication" username="${sf.username}" password="${sf.password}" securityToken="${sf.securityToken}" doc:name="Salesforce: Basic authentication"/>
```
The **sfdc:query** returns an instance of `java.util.Iterator`. So we have written a custom java component to retrieve the id.
```java
import java.util.Iterator;
import java.util.Map;
import org.mule.api.MuleEventContext;
import org.mule.api.lifecycle.Callable;

public class ContactIdRetriever implements Callable {
	public Object onCall(MuleEventContext eventContext) throws Exception {
		Map<?, ?> contactIdMap = null;
		if (eventContext.getMessage().getPayload() instanceof Iterator) {
			Iterator<?> iterator = (Iterator<?>) eventContext.getMessage().getPayload();
			if (iterator.hasNext()) {
				contactIdMap = (Map<?, ?>) iterator.next();
			}
			return contactIdMap.get("Id");
		} 
		return null;
	}
}
```
So, [this](#get-sf-contact-id) sub-flow finally returns a contact id corresponding to the customer mail address. In the [tone analyzer flow](#tone-analyzer), we use an enricher to enrich the message with the contact id retrieved from Salesforce and this contact id is stored in a flow variable `#[flowVars.contactId]`

### <a name="get-tone"></a>get-tone flow:
This sub-flow gets tone of the mail from the cusomer. It posts the mail body, received from the customer, to [IBM Watson Tone Analyzer](http://www.ibm.com/smarterplanet/us/en/ibmwatson/developercloud/tone-analyzer.html) and retrieves linguistic tone of the mail.
```xml
<sub-flow name="get-tone">
  <dw:transform-message doc:name="Transform Message">
    <dw:set-payload resource="classpath:EmailBodyToTonePayload.dwl"></dw:set-payload>
  </dw:transform-message>
  <http:request config-ref="HTTP_Request_Configuration" path="/v1/tone" method="POST" port="443" doc:name="get-tone">
    <http:request-builder>
      <http:header headerName="Content-Type" value="application/json"/>
      <http:header headerName="Authorization" value="${authorization}"/>
    </http:request-builder>
  </http:request>
  <object-to-string-transformer doc:name="object-to-string"/>
  <logger message="#[payload]" level="INFO" doc:name="logger"/>
</sub-flow>
```
The corresponding HTTP Request Connector is shown below
```xml
<http:request-config name="HTTP_Request_Configuration" protocol="HTTPS" host="gateway.watsonplatform.net" port="443" basePath="tone-analyzer-experimental/api" doc:name="Tone_Analyzer_HTTP_Request_Configuration"/>
```
### <a name="update-sf-contact-with-tone"></a>update-sf-contact-with-tone flow:
