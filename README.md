# OpenHAB & Robotan
OpenHAB &amp; Robotan 

How to integrate the Robotan Module as described here https://github.com/fredlcore/robotan/ to monitor/control lawn mower via OpenHAB (www.openhab.org)

In my case I use "Acceleration_Sensor" with no GPS to detect if the roboter is stuck or not. If roboter is stuck but not in Garage, I consider the lawn mower roboter is stuck and need my help. If the roboter is in the garage or not is detected via an external sensor.

External Switch:
- NEO CoolCam DOOR/WINDOW Sensor (Z-Wave) (removed the reed switch and connected below Limit Switch)
- Limit Switch (Endschalter ME-8108)

Note: I still have issues with my NEO CoolCam and reliability so still in testing phase.


### Pre-requisits:
-	At least OpenHAB Version 2.4.* or later
-	JSONPath Transformation (OpenHAB Transformation Addon)
-	HTTP Binding (OpenHAB Binding/Addon)
-	Access to Robotan WebUI from OpenHAB Server
    See: https://github.com/fredlcore/robotan/

Optional but used in Examples
-	Expire Binding (OpenHAB Binding/Addon)


## OpenHAB Configuration

### HTTP Binding Configuration
Replace ThePassword with your Roboter specific password and 192.168.178.180 with the IP of the Robotan module. Make sure the IP is fix and is NOT changing with every reboot (e.g. via FritzBox DHCP fixed IP option or use DNS name instead of IP)

# http.cfg (/etc/openhab2/services/http.cfg)

```bash
roboURL.url=http://Robotan:ThePassword@192.168.178.180/json
roboURL.updateInterval=30000

```


### OpenHAB Item Configuration

```bash
// Reset Alert after 24h (but rule robotanRuleAlert also sends NA if all okay again)
String roboAlert "Roboter Alarm: [%s]" (gAlerts) {expire="24h,NA" }

// If Robo hasn’t left garage for 24h and so no update received, we may have an error
String roboStatus "Roboter Status [%s]" {expire="24h,FEHLER-KeineUpdates" }


//roboURL as defined in services/http.cfg (HTTP Binding)
//30.000 ms = 30 Sekunden Refresh

String roboUptime 			{ http="<[roboURL:30000:JSONPATH($.800..value)]", expire="60m,NoConnectionRobo" }
String roboAcceleration 	{ http="<[roboURL:30000:JSONPATH($.803..value)]" }
String roboStuckStatus 		{ http="<[roboURL:30000:JSONPATH($.804..value)]" }
String roboLastResetReason 	{ http="<[roboURL:30000:JSONPATH($.906..value)]" }


DateTime RoboPresence_LST  "Roboter Garage - Letzte Änderung  [%1$td.%1$tm %1$tH:%1$tM]"
```

The Items roboUptime, roboAcceleration, roboStuckStatus, roboLastResetReason are read via the OpenHAB HTTP Binding and parsed according the defined parsing rule.
```
String roboUptime 			{ http="<[roboURL:30000:JSONPATH($.800..value)]", expire="60m,NoConnectionRobo" }
```

Above example quries the Webpage roboURL (defined in http.cfg) every 30.000 MS (= 30Sec), search for the paramter 800 and put the value of parameter 800 in the OH item named roboUptime. The Optional specification ' expire="60m,NoConnectionRobo ' defines that if we have no update of the value uptime at least every 60m (60Minutes), we set the value NoConnectionRobo. 


As reference, the output of http://<ip-of-robotan>/json  looks like belwo (only partial output):
  
```
},
  "73": {
    "name": "Secondary Area 3 Activity Day 4",
    "value": "0"
  },
  "74": {
    "name": "Gyro",
    "value": "0"
  },
  "800": {
    "name": "Uptime",
    "value": "0d 2:58:05"
  },
  "801": {
    "name": "Firmware_Version",
    "value": "16220"
  },
  "802": {
    "name": "Hall_Sensor",
    "value": "1"
  },
  "803": {
    "name": "Acceleration_Sensor",
    "value": "506"
  },
  "804": {
    "name": "Stuck_Status",
    "value": "1"
  },
  "805": {
    "name": "Error_Status",
    "value": "0"
  },
```




## OpenHAB Rules
Below 4x rules are in use to monitor and send alerts about Robotan

The Item NCCSwitch01_Switch is an external "NEO CoolCam DOOR/WINDOW Sensor" which checks via Switch (Endschalter ME-8108) if the roboter is in the garage or not.


- robotanRule1
- robotanRuleNoConnection
- robotanRuleLastAWAY
- robotanRuleAlert


```
// ---------------------------------------------------------------------
// Logic about Stuck and inGarage?
// ---------------------------------------------------------------------
// NCCSwitch01_Switch is a Contact (OPEN & CLOSED)
// CLOSED = InGarage
// OPEN   = Not in Garage (mowing)


rule "robotanRule1"
when
    Item roboStuckStatus changed or
	  Item NCCSwitch01_Switch changed 
then
	
	Thread::sleep(2000)
	
	var String vRoboPresence = NCCSwitch01_Switch.state.toString
	var String vroboStuckStatus = roboStuckStatus.state.toString
	
	//Open = mowing
	if ( vRoboPresence == "OPEN")
	{
		// Check if Robo just left Garage and wait for 35 seconds to avoid false alarms
		if (RoboPresence_LST.state === NULL ) 
		{
			//Last change not set yet, set with current datetime 
			logInfo("robotanRule1","Update RoboPresence_LST as NULL")
			RoboPresence_LST.postUpdate(new DateTimeType())
			Thread::sleep(1000)
		}
		var vRoboPresence_LST = (RoboPresence_LST.state as DateTimeType).zonedDateTime.toInstant.toEpochMilli
		var epoch_now = now.millis
		var Number time_dif = (epoch_now - vRoboPresence_LST ) / 1000
		if ( time_dif < 35)
			{
				logInfo("robotanRule1", "Robo just left Garage SecondsAgo: " + time_dif + "Secs - Wait at least 35Sec")
				roboStatus.sendCommand("OKAY-Faehrt")
				return
			}
    }
	
	// Closed -> inGarage
	if ( vRoboPresence == "CLOSED")
	{
	 logInfo("robotanRule1", "Robo in Garage")
	 roboStatus.sendCommand("OKAY-inGarage")
	}
	else if ( vroboStuckStatus == "0" )
	{
	 logInfo("robotanRule1", "Robo is moving...")
	 roboStatus.sendCommand("OKAY-Faehrt")
	}
	else if ( (vRoboPresence == "OPEN" ) && (vroboStuckStatus == "1" ) )
	{
	 logInfo("robotanRule1", "Robo not in Garage and not moving! Presence: " + vRoboPresence + " StuckStatus:" + vroboStuckStatus)
	 roboStatus.sendCommand("FEHLER-RoboHaengt")
	}
	else
	{
	 logError("robotanRule1", "Error in Robotan Rule robotanRule1 - Presence:" + vRoboPresence + " StuckStatus:" + vroboStuckStatus)
	}
	
	

end

// ---------------------------------------------------------------------
//Report connectivity issues with Robotan
// ---------------------------------------------------------------------
rule "robotanRuleNoConnection"
when
    Item roboUptime changed
then

 var String vroboUptime = roboUptime.state.toString
 var String vroboStatus = roboStatus.state.toString
 
 if ( vroboUptime == "NoConnectionRobo" )
 {
  logInfo("robotanRuleNoConnection", "No Updates from Roboter!")
  roboStatus.sendCommand("FEHLER-KeineVerbindung")
 }
 else
 {
  logDebug("robotanRuleNoConnection", "Uptime uptime... UptimeValue:" + vroboUptime)
  
  if ( vroboStatus == "FEHLER-KeineVerbindung" )
  {
   //Onls set OKAY-WifiReconnected if present state is "FEHLER-KeineVerbindung" but we have again updates from robotan
   roboStatus.sendCommand("OKAY-WifiReconnected")
  }
  
 }

end

// ---------------------------------------------------------------------
//Track last change of NCCSwitch01_Switch
// ---------------------------------------------------------------------
rule "robotanRuleLastAWAY"
when
	Item NCCSwitch01_Switch changed 
then

 //Save timestamp when roboter Garage YES/NO was last changed
 logInfo("robotanRuleLastAWAY", "Update RoboPresence_LST..")
 postUpdate(RoboPresence_LST, now.toString)
end



// ---------------------------------------------------------------------
//Report error stati to item roboAlert
// ---------------------------------------------------------------------
rule "robotanRuleAlert"
when
    Item roboStatus changed
then

 var String vroboStatus = roboStatus.state.toString
 
 if ( vroboStatus.contains("FEHLER")  )
 {
  logInfo("robotanRuleAlert", "Robot Alert detected - robotStatus:" + vroboStatus)
  roboAlert.sendCommand(vroboStatus)
  //sendTelegram("OHBot", "roboAlert changed %s", roboAlert.state.toString)
 }
 else
 {
  logDebug("robotanRuleAlert", "roboStatus changes but no alert")
  roboAlert.sendCommand("NA")
 }

end

```


## More documentation:
https://github.com/fredlcore/robotan/
https://www.openhab.org/addons/bindings/http1/
https://www.openhab.org/addons/bindings/expire1/
https://www.openhab.org/addons/transformations/jsonpath/

