# Make JSON data look like an Oracle table #

After the introduction of XML, which Oracle still supports as native type, the focus is currently on JSON. For some programs and functions it is sometimes easier to translate the JSON values into something similar as an Oracle database. This way we can select and join just like a regular table.

One of the most practical functions is the JSON_TABLE function which was introduced in Oracle 12c. This function can transform JSON data and present it like a regular table in Oracle. This way it is very easy to use JSON information in the Oracle database.

## Prerequisites ##

- Oracle autonomous Database instance
- Connection setup with SQL Developer

## Simple translation of JSON using JSON_TABLE ##

When we look at the following data from an Oracle database:

    ID      NAME         DESCRIPTION
    == ========= ===================
    01     First   This is the first 
    02    Second  This is the second 
    03     Third   This is the third

This could be represented by the following JSON data:

    [
     { "id" : "01', "name" : "First", "description" : "This is the first"},
     { "id" : "02', "name" : "Second", "description" : "This is the second"},
     { "id" : "03', "name" : "Third", "description" : "This is the third"}
    ]

To translate the JSON data into a relational way, we use JSON table:

    select id, name, description
    from json_table('[ { "id":"01", "name" : "First", "description" : "This is the first"},'||
                      '{ "id":"02", "name" : "Second", "description" : "This is the second"},'||
                      '{ "id":"03", "name" : "Third", "description" : "This is the third"}'||
                    ']',
                    '$[*]' COLUMNS (id, name, description) )

## Retrieving information using REST API calls in Autonomous ##

As an example, we will query the DomainsDB website for the domain oracle.com. According to the website, there is an API available which will return values in JSON format. Although most APIs require authentication in some form, this API does not. To test the API, you can execute the following URL in your browser: 

    https://api.domainsdb.info/v1/domains/search?domain=oracle&zone=com

The result should look similar to this:

    {"domains": [{"domain": "oracle-advisory-service.com", "create_date": "2019-08-06T02:59:58.802443",
    "update_date": "2019-08-06T02:59:58.802445", "country": null, "isDead": "False", "A": null, "NS": null,
    "CNAME": null, "MX": null, "TXT": null}, {"domain": "oracle-advisory-services.com", "create_date":
    "2019-08-06T02:59:57.975858", "update_date": "2019-08-06T02:59:57.975860", "country": null, "isDead": 
    "False", "A": null, "NS": null, "CNAME": null, "MX": null, "TXT": null}, {"domain": 
    "oracle-messages-amour.com", "create_date": "2019-08-05T03:00:49.008184", "update_date": 
    "2019-08-05T03:00:49.008185", "country": null, "isDead": "False", "A": null, "NS": null, "CNAME": null, 
    "MX": null, "TXT": null}, {"domain": "oracle-abondance.com", (etc etc)..

If the information would be put into a JSON formatting tool, it would look like this:

    {
      "domains": [
                  {
                    "domain": "oracle-advisory-service.com",
                    "create_date": "2019-08-06T02:59:58.802443",
                    "update_date": "2019-08-06T02:59:58.802445",
                    "country": null,
                    "isDead": "False",
                    "A": null,
                    "NS": null,
                    "CNAME": null,
                    "MX": null,
                    "TXT": null
                  },
                  {
                    "domain": "oracle-advisory-services.com",
                    "create_date": "2019-08-06T02:59:57.975858",
                    "update_date": "2019-08-06T02:59:57.975860",
                    "country": null,
                    "isDead": "False",
                    "A": null,
                    "NS": null,
                    "CNAME": null,
                    "MX": null,
                    "TXT": null
                  }, (etc etc...)

Current Oracle APEX installations, even if you are not using the APEX tool itself, come with a list of additional packages. One of those packages is **APEX\_WEB\_SERVICE**. In a regular environment, you need to setup access to the internet and create a wallet with certificates. In Oracle Autonomous Database we have already done this for you.

To retrieve values for the above mentioned URL, we can use the function **MAKE\_REST\_REQUEST**. This function has several parameters but we only have to use the most basic ones in this example; the P_URL and P_HTTP_METHOD.

Since it is a function, we can use it in a select statement. 

> **Hint:** When executed in SQL Developer or SQL Plus, the '&' sign can lead to the system thinking this is a substitution variable. To suppress this we first execute the 'set define off' statement before executing the select statement. 

    set define off

    select apex_web_service.make_rest_request(
           p_url => 'https://api.domainsdb.info/v1/domains/search?domain=oracle&zone=com', 
           p_http_method => 'GET') as result
    from dual

The result should be similar to the output in the browser:

    RESULT
    ==========================================================================================
    {"domains": [{"domain": "oracle-advisory-service.com", "create_date": "2019-08-06T02:59...

   
## Combining REST API calls with with JSON_TABLE ##

We can now combine the JSON table and the REST_REQUEST result to turn the REST API data into something that looks like a query from a regular table:

    select domain, create_date, a
    from json_table ( apex_web_service.make_rest_request(
                          p_url => 'https://api.domainsdb.info/v1/domains/search?domain=oracle&zone=com', 
                          p_http_method => 'GET'),
                     '$.domains[*]' COLUMNS (domain, create_date, a ) )

When executed, the following result should be visible:

    DOMAIN                        CREATE_DATE                A
    ============================= ========================== ==
    oracle-advisory-service.com	  2019-08-06T02:59:58.802443	
    oracle-advisory-services.com  2019-08-06T02:59:57.975858	
    oracle-messages-amour.com     2019-08-05T03:00:49.008184	
    oracle-abondance.com          2019-08-05T03:00:48.413518	
    oracle-amour.com              2019-08-05T03:00:48.271878	
    oracle-host.com               2019-08-04T03:03:23.010232	
    wells-oracle-trust.com        2019-08-02T03:13:04.522278	
    (etc etc...)

## End-of-Lab ##