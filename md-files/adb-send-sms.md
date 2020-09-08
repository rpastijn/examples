# Send an SMS from Autonomous Database #
In Oracle Autonomous Database, we can leverage the installed APEX environment. Besides a low code development environment, Oracle APEX also supplies a list of PL/SQL packages that can be used outside of the APEX environment.

One of the packages used is the APEX\_WEB\_SERVICE package which is much easier to use than separate calls to UTL\_TCP or other packages. In this lab we will demonstrate how your can use the package to send an SMS to your phone. 

Sending of SMS messages (or other messages like Slack, Whatsapp, MatterMost, Discord etc) requires a gateway. You can either set one up yourself or use a commercial provider that offers the functionality. For the SMS example we will use an external party called Messagebird. This company is one of many who offers API gateways to various services for both sending but also receiving messages.

> **Note 1:** Oracle does not own the Messagebird service nor (specifically) promotes or endorses it.

> **Note 2:** You need to create an account with the example message provider for this lab to work. Please check the [Messagebird website](https://www.messagebird.com/). Or you can use any other message provider (but you may have to change the PL/SQL code in order to do this.

## Prerequisites ##

- Running Autonomous Database instance
- SQL Developer connection setup to connect to the instance
- Worksheet active and connected in SQL Developer
	- So you can execute statements and commands
- Authentication key for the service (available after account creation)

## Create a procedure that sends an SMS ##

In our example, the SMS provider offers an REST-API gateway for sending messages. We need to send our number and message as parameters in our call. 

> **Hint:** More information on the API gateway and other options for messages is available on the website [Messagebird website](https://developers.messagebird.com/api/ "Messagebird API documentation").

Copy-and-paste the following information in your worksheet. Press the green 'Run Statement' to execute the create procedure.

    create or replace procedure SEND_SMS (p_sender varchar2, p_phonenumber number, p_message varchar2, p_accesskey varchar2) is
    
      l_clob       clob;
      l_accesskey  varchar2(100);

    begin

      l_accesskey := 'AccessKey ' || p_accesskey;

      apex_web_service.g_request_headers(1).name := 'Content-Type';
      apex_web_service.g_request_headers(1).value := 'application/x-www-form-urlencoded'; 
      apex_web_service.g_request_headers(2).name := 'Authorization';
      apex_web_service.g_request_headers(2).Value := l_accesskey;

      l_clob := apex_web_service.make_rest_request( p_url => 'https://rest.messagebird.com/messages',
                                                    p_http_method => 'POST',
                                                    p_parm_name => apex_util.string_to_table('recipients:originator:body'),
                                                    p_parm_value => apex_util.string_to_table(p_phonenumber||':'||p_sender||':'||p_message)
                                                  );
      
      dbms_output.put_line(l_clob);


    end;

## Test the procedure before sending an SMS ##
If the compilation was successful, you should now be able to send your first SMS through a procedure. But first, we are going to test if all works well.

Enter the following code in your worksheet.

> **Hint 1:** before executing the code, replace the `<xx>` with appropriate values.<br>
> **Hint 2:** the phonenumber must be given in international notation like `316xxxxxxxx`

    set serveroutput on

    begin
      send_sms ( p_sender      => 'OGH',
                 p_phonenumber => <your mobile number>,
                 p_message     => '<your message>',
                 p_accesskey   => 'Your key like OKhvussIB7gNNkBMUZ8XBPxxY' );
    end;

 Execute the code as a script in your worksheet. Output similar to the below should be displayed:

>**Note:** the supplied access key is a 'test' access key. No actual messages will be sent but the response is the same.

    {"id":"7618fd9258e64477940066bfd50f7b7a","href":"https://rest.messagebird.com/messages/7618fd9258e64477940066bfd50f7b7a","direction":"mt","type":"sms","originator":"OGH","body":"Small test","reference":null,"validity":null,"gateway":10,"typeDetails":{},"datacoding":"plain","mclass":1,"scheduledDatetime":null,"createdDatetime":"2019-12-10T13:42:06+00:00","recipients":{"totalCount":1,"totalSentCount":1,"totalDeliveredCount":0,"totalDeliveryFailedCount":0,"items":[{"recipient":31615078203,"status":"sent","statusDatetime":"2019-12-10T13:42:06+00:00","messagePartCount":1}]}}
    
    PL/SQL procedure successfully completed.

## Send your first SMS ##
If the test was successful, you should now be able to send your first SMS through the procedure. 

Change the code in your worksheet replacing the test 'Access Key' for a live one.

    set serveroutput on

    begin
      send_sms ( p_sender      => 'OGH',
                 p_phonenumber => <your mobile number>,
                 p_message     => '<your message>',
                 p_accesskey   => '<live_access_key' );
    end;

 Execute the code (again) as a script in your worksheet. Output similar to the below should be displayed but this time an SMS should be send

    {"id":"7f7b8d6aa6c045618232385ab7a67e3e","href":"https://rest.messagebird.com/messages/7f7b8d6aa6c045618232385ab7a67e3e","direction":"mt","type":"sms","originator":"OGH","body":"Small test","reference":null,"validity":null,"gateway":10,"typeDetails":{},"datacoding":"plain","mclass":1,"scheduledDatetime":null,"createdDatetime":"2019-12-10T13:48:52+00:00","recipients":{"totalCount":1,"totalSentCount":1,"totalDeliveredCount":0,"totalDeliveryFailedCount":0,"items":[{"recipient":31615078203,"status":"sent","statusDatetime":"2019-12-10T13:48:51+00:00","messagePartCount":1}]}}

    PL/SQL procedure successfully completed.

Check your phone !

## End-of-Labs



