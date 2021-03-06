Welcome to the Sensibo-and-Openhab-V1.8 wiki!

These instructions are working on Openhab 1.8, please let me know if they work on other versions, or if you have a more elegant way of doing this.

To get Openhab V1.8 to control Sensibo via the Sensibo API carry out the following steps.

First of all obtain an API Key from Sensibo, this is referred to as MYAPIKEY in the examples. Once you have your key you need to get the ID's of your POD's. Do this from a web browser by entering the following into the address bar

https://home.sensibo.com/api/v2/users/me/pods?apiKey=MYAPIKEY

This should give you a reply similar to this

    {"status": "success", "result": [{"id": "FIRST_POD_ID"}, {"id": "SECOND_POD_ID"}, {"id": "THIRD_POD_ID"}]}

Make a note of these as you will need to add one of them instead of MYPODID in the rules file, you also will need to add your API Key instead of MYAPIKEY

In your items file, default.items in my case, you need to add items similar to this

    Switch Heatpump1
    Number Heatpump1SensiboTemperature

In your sitemap, default.sitemap in my case, you need to add entries similar to this

    Switch item=Heatpump1 label="Heatpump1 Cooling"
    Text item=Heatpump1SensiboTemperature label="Sensibo1 Temperature [%.1f C]" icon="temperature"

At the time of writing the HTTP binding in Openhab doesn't support HTTPS which Sensibo requires to all commands have to be sent and received from the command line using Curl, so everything works via a rule.

Create a rules file named something meaningful, I used Heatpumps.rules, and inside it you need to create 3 rules.

    import org.openhab.core.library.types.*
    import org.openhab.model.script.actions.*
    var Boolean BatteryEmailNotSent = true
    var Boolean Heatpump1Stable = true
    var Number ReplaceBatteryLevel = 2800 // level to send new battery email
    var Timer PauseHeatpump1Updates = null

This first rule sends the commands to the API in response to changes in a virtual heatpump switch set in the items file, it also disables the second rule for 15 seconds so commands can't get overwritten by out of date status updates.

    rule "Send Command to Sensibo from Heatpump Switch"
    when Item Heatpump1 changed then
    if (Heatpump1.state == ON && Heatpump1Stable)
    {   Heatpump1Stable = false
	    logInfo("Heatpumps", "Heatpump1 on Switch Rule Ran")
	    executeCommandLine('curl@@-H@@Content-Type: application/x-www-form-urlencoded@@-X@@POST@@-d@@{"acState":{"on":true,"mode":"cool","fanLevel":"auto","targetTemperature":19}}@@https://home.sensibo.com/api/v2/pods/MYPODID/acStates?apiKey=MYAPIKEY&fields=acState', 5000)
    	PauseHeatpump1Updates = createTimer(now.plusSeconds(15))[|
    	Heatpump1Stable = true
    	]}
    	
    if (Heatpump1.state == OFF && Heatpump1Stable)
    {	Heatpump1Stable = false
    	logInfo("Heatpumps", "Heatpump1 off Switch Rule Ran")
    	executeCommandLine('curl@@-H@@Content-Type: application/x-www-form-urlencoded@@-X@@POST@@-d@@{"acState":{"on":false,"mode":"cool","fanLevel":"auto","targetTemperature":19}}@@https://home.sensibo.com/api/v2/pods/MYPODID/acStates?apiKey=MYAPIKEY&fields=acState', 5000)
    	PauseHeatpump1Updates = createTimer(now.plusSeconds(15))[|
    	Heatpump1Stable = true
    	]}
    end

This second rule queries the API for AC Status and Temperature Measurements and updates the Virtual switch in case the phone app has been used to change the AC Status. It also provides the POD temperature measurement and sends an email when the batteries drop below the level set at the top of the rules file. (This assumes you have the mail binding installed and configured. If you don't need this comment it out by adding // before the sendMail line.)

    rule "Update Heatpump Sensibo Status"
    when Time cron "50 * * * * ?"
    then
    if (Heatpump1Stable)
    try {
    var Heatpump1Status = executeCommandLine('curl -sSH "Accept: application/json"     "https://home.sensibo.com/api/v2/pods/MYPODID/acStates?apiKey=MYAPIKEY&limit=1&fields=acState"', 5000)
    	var String Heatpump1On = (transform("JSONPATH", "$.result..on", Heatpump1Status))
    	if (Heatpump1On == "[true]" && Heatpump1Stable)
    {	postUpdate(Heatpump1, ON)	}	
    	if (Heatpump1On == "[false]" && Heatpump1Stable)
    {	postUpdate(Heatpump1, OFF)	}	
    	var Heatpump1Measurements = executeCommandLine('curl -sSH "Accept: application/json"     "https://home.sensibo.com/api/v2/pods/MYPODID/measurements?apiKey=<MYAPIKEY>&fields=batteryVoltage,temperature,humidity"', 5000)
    	var String Heatpump1TemperatureTransform = (transform("JSONPATH", "$.result..temperature", Heatpump1Measurements))
    	var String Heatpump1TransformString = (Heatpump1TemperatureTransform.replace('[', '').replace(']', ''))
    	val Number Heatpump1TemperatureValue = new Double(Heatpump1TransformString)
    	var String Heatpump1BatteryTransform = (transform("JSONPATH", "$.result..batteryVoltage", Heatpump1Measurements))
    	var String Heatpump1BatteryTransformString = (Heatpump1BatteryTransform.replace('[', '').replace(']', ''))
    	val Number Heatpump1Battery = new Double(Heatpump1BatteryTransformString)
    //	logInfo("Testing", "Heatpump1 Battery Voltage is " +Heatpump1Battery)
    	postUpdate(Heatpump1SensiboTemperature, Heatpump1TemperatureValue)
    if	(Heatpump1Battery < ReplaceBatteryLevel && BatteryEmailNotSent)
        {	sendMail ("MYEMAILADDRESS", "Heatpump1 Heatpump", "Heatpump1 batteries are low! Their voltage is "         +Heatpump1Battery)
    		BatteryEmailNotSent = false	}
    } catch(Throwable t) {
    	logError("Heatpumps Status", "Weird stuff happened: {}", t)}
    end

The third rule resets a variable once a day so only one battery status email is sent per day.

    rule "reset dead battery email"
    when Time cron "0 0 6 * * ?" then
    {	BatteryEmailNotSent = true	}
    end

and finally the "try" "catch(Throwable t) is to replace a 50 line stack dump with something sensible whenever the Sensibo servers don't give the expected response.
