### Path of Execution:
1. Event Fires, calling associated MasterScript
    * imports INCLUDE_ACCELA_FUNCTIONS, INCLUDE_GLOBALS, INCLUDE_CUSTOM, and INCLUDE_CUSTOM_GLOBALS
    * sets variables and does preprocessing
2. Once initialized, the MasterScript checks whether to fire StdChoices and/or Scripts based on EMSE_EXECTUR_OPTIONS
	* This can be one or the other, or both. If an event script (eg. IRSA:Enforcement/\*/\*/\*) exists as both, it will fire as both.
	* The master script loops through all associated std choices 


### Relevant Variables and Their Purposes

Variable Identifier | Purpose | Format
----------- | ------- | -------
vEventName| current running event | 
vScriptName | script to be run | `{prefix}:*/*/*/*"` 



### MasterScript Functions 

```javascript
// Overrides INCLUDE_ACCELA_FUNCTIONS version of this, but they are more or less functionally equivalent
function getScriptText(vScriptName)
{
    var servProvCode = aa.getServiceProviderCode();
    if (arguments.length > 1)
    {
        servProvCode = arguments[1]; // use different serv prov code
    }
    vScriptName = vScriptName.toUpperCase();
    var emseBiz = aa.proxyInvoker.newInstance("com.accela.aa.emse.emse.EMSEBusiness").getOutput();
    try
    {
        var emseScript = emseBiz.getScriptByPK(servProvCode, vScriptName, "ADMIN");
        return emseScript.getScriptText() + "";
    }
    catch (err) { return ""; }
}

//  <===========Main=Loop================>
//  Get the Standard choices entry we'll use for this App type
//  Then, get the action/criteria pairs for this app
//
for (inspCount in inspIdArr)
{
    inspId = inspIdArr[inspCount];
    inspResult = inspResultArr[inspCount];
    inspType = inspTypeArr[inspCount];
    inspObj = aa.inspection.getInspection(capId, inspId).getOutput(); // current inspection object
    inspGroup = inspObj.getInspection().getInspectionGroup();
    inspResultComment = inspObj.getInspection().getResultComment();
    inspComment = inspResultComment; // consistency between events
    inspResultDate = inspObj.getInspectionStatusDate().getMonth() + "/" + inspObj.getInspectionStatusDate().getDayOfMonth() + "/" + inspObj.getInspectionStatusDate().getYear();

    if (inspObj.getScheduledDate())
    {
        inspSchedDate = inspObj.getScheduledDate().getMonth() + "/" + inspObj.getScheduledDate().getDayOfMonth() + "/" + inspObj.getScheduledDate().getYear();
    }
    else
    {
        inspSchedDate = null;
    }

    inspTotalTime = inspObj.getTimeTotal();
    logDebug("Inspection #" + inspCount);
    logDebug("inspId " + inspIdArr[inspCount]);
    logDebug("inspResult = " + inspResultArr[inspCount]);
    logDebug("inspResultComment = " + inspResultComment);
    logDebug("inspComment = " + inspComment);
    logDebug("inspResultDate = " + inspResultDate);
    logDebug("inspGroup = " + inspGroup);
    logDebug("inspType = " + inspType);
    logDebug("inspSchedDate = " + inspSchedDate);
    logDebug("inspTotalTime = " + inspTotalTime);
    
    // Execute standard choices associated with the event (based on std choice prefixes)
    if (doStdChoices)
    {
        doStandardChoiceActions(controlString, true, 0);
    }
    //  Next, execute and scripts that are associated to the record type
    if (doScripts)
    {
        doScriptActions();
    }
}
```


### Standard functions within INCLUDE_ACCELA_FUNCTIONS

```javascript
function getScriptText(vScriptName, servProvCode, useProductScripts) 
{
	if (!servProvCode)  
    {
        servProvCode = aa.getServiceProviderCode();
    }
	
    vScriptName = vScriptName.toUpperCase();
    
	var emseBiz = aa.proxyInvoker.newInstance("com.accela.aa.emse.emse.EMSEBusiness").getOutput();
	try {
        if (useProductScripts) 
        {
			var emseScript = emseBiz.getMasterScript(aa.getServiceProviderCode(), vScriptName);
		} 
        else 
        {
			var emseScript = emseBiz.getScriptByPK(aa.getServiceProviderCode(), vScriptName, "ADMIN");
		}
        return emseScript.getScriptText() + "";
	} catch (err) { return ""; }
}

function include(s) 
{
	try 
    {
	    var thisDate = new Date();
		var thisTime = thisDate.getTime();
		var st = getScriptText(s);
		if (st.length) 
        {
			logDebug("Executing script : " + s + ", Elapsed Time: " + ((thisTime - startTime) / 1000) + " Seconds");
			eval(st);
        }
    } catch (err) { handleError(err, s); }
}

function doScriptActions() 
{
	if (typeof(appTypeArray) == "object")
    {
        include(prefix + ":" + appTypeArray[0] + "/*/*/*");
        include(prefix + ":" + appTypeArray[0] + "/" + appTypeArray[1] + "/*/*");
        include(prefix + ":" + appTypeArray[0] + "/" + appTypeArray[1] + "/" + appTypeArray[2] + "/*");
        include(prefix + ":" + appTypeArray[0] + "/*/" + appTypeArray[2] + "/*");
        include(prefix + ":" + appTypeArray[0] + "/*/" + appTypeArray[2] + "/" + appTypeArray[3]);
        include(prefix + ":" + appTypeArray[0] + "/*/*/" + appTypeArray[3]);
        include(prefix + ":" + appTypeArray[0] + "/" + appTypeArray[1] + "/*/" + appTypeArray[3]);
        include(prefix + ":" + appTypeArray[0] + "/" + appTypeArray[1] + "/" + appTypeArray[2] + "/" + appTypeArray[3]);
    }
}
 
function doStandardChoiceActions(stdChoiceEntry, doExecution, docIndent) 
{
    var thisDate = new Date();
    var thisTime = thisDate.getTime();
    var lastEvalTrue = false;
    stopBranch = false;  // must be global scope

    logDebug("Executing : " + stdChoiceEntry + ", Elapsed Time: " + ((thisTime - startTime) / 1000) + " Seconds");

    var pairObjArray = getScriptAction(stdChoiceEntry);
    if (!doExecution) docWrite(stdChoiceEntry, true, docIndent);
    for (xx in pairObjArray) 
    {
        doObj = pairObjArray[xx];
        if (doExecution) 
        {
            if (doObj.enabled) 
            {

                if (stopBranch)
                {
                    stopBranch = false;
                    break;
                }

                logDebug(aa.env.getValue("CurrentUserID") + " : " + stdChoiceEntry + " : #" + doObj.ID + " : Criteria : " + doObj.cri, 2)

                try
                {

                    if (eval(token(doObj.cri)) || (lastEvalTrue && doObj.continuation)) 
                    {
                        logDebug(aa.env.getValue("CurrentUserID") + " : " + stdChoiceEntry + " : #" + doObj.ID + " : Action : " + doObj.act, 2);

                        eval(token(doObj.act));
                        lastEvalTrue = true;
                    }  
                    else 
                    {
                        if (doObj.elseact) 
                        {
                            logDebug(aa.env.getValue("CurrentUserID") + " : " + stdChoiceEntry + " : #" + doObj.ID + " : Else : " + doObj.elseact, 2);
                            eval(token(doObj.elseact));
                        }
                        lastEvalTrue = false;
                    }
                }
                catch(err)
                {
                    showDebug = 1;
                    logDebug("**ERROR An error occured in the following standard choice " + stdChoiceEntry + "#" + doObj.ID + "  Error:  " + err.message);
                }
            }
        }
        else // just document
        {
            docWrite("|  ", false, docIndent);
            var disableString = "";
            if (!doObj.enabled) 
            {
                disableString = "<DISABLED>";
            }

            if (doObj.elseact)
            {
                docWrite("|  " + doObj.ID + " " + disableString + " " + doObj.cri + " ^ " + doObj.act + " ^ " + doObj.elseact, false, docIndent);
            }
            else
            {
                docWrite("|  " + doObj.ID + " " + disableString + " " + doObj.cri + " ^ " + doObj.act, false, docIndent);
            }

            for (yy in doObj.branch) 
            {
                doStandardChoiceActions(doObj.branch[yy], false, docIndent + 1);
            }
        }
    } // next sAction
    
    if (!doExecution) 
    {
        docWrite(null, true, docIndent);
    }
    var thisDate = new Date();
    var thisTime = thisDate.getTime();
    logDebug("(std choice) Finished: " + stdChoiceEntry + ", Elapsed Time: " + ((thisTime - startTime) / 1000) + " Seconds");
}
```
