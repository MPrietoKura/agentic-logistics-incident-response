# agentic-logistics-incident-response

## <ins>System Overview</ins><br/>
The agentic logistics incident response system is an AI driven intelligent ServiceNow solution that analyzes financial impacts of truck breakdowns by automatically calculating delay costs based on customer contracts, makes optimal rerouting decisions considering financial impact, and coordinates external execution with logistics providers- through AI agents and workflow orchestration. 
The system utilizes three Agents:

Agent 1: Route Financial Analysis Agent, which analyzes financial impact of delivery disruptions and creates incident tracking. Key responsibilities include: Querying customer contract terms and penalty structures from Supply Agreement tables, calculating delay costs for each proposed route option using delivery window hours and stockout penalty rates, creating incident records with comprehensive breakdown details, updating delivery records with calculated financial impacts, and progressing workflow status to trigger next agent.

Agent 2: Route Decision Agent, which selects optimal routes and coordinates external execution. Key responsibilities include: Analyzing route options using financial impact and delivery constraints, selecting optimal routing based on business rules and cost optimization, updating incident state and optimal route selection, triggering external execution workflow via webhook to n8n, updating workflow status to approved for tracking.

Agent 3: Workflow Implementation Agent, which receives webhook payloads containing routing decisions, coordinates execution with external logistics providers, sends customer notifications, and updates ServiceNow with execution status to dispatch route trucks accordingly. 
<br/>
## <ins>Implementation Steps</ins><br/>
After configuring Supply Agreement and Delivery Delay tables to receive truck breakdown information through MCP logistics server protocol, the Route Financial Analysis Agent was configured to retrieve data on alternate route options, including route distances and delivery ETA. By cross referencing Supply Agreement terms based on delivery window hours and penalty rates, the Agent was able to calculate costs for each option provided by the logistics partner. <br/>
<br/>
Following these calculations, the Agent created an Incident record with the problem description, as well as route options with financial impact calculations. The Delivery Delay table was also updated to reflect this calculated status. <br/>
<br/>
The prompt was as follows: 
```
Data Dictionary:
deliver_window_hours: The contractual timeframe (in hours) within which deliveries must be completed to avoid penalties.
stockout_penalty_rate: The financial penalty (in dollars) assessed for every hour a delivery exceeds the contractual delivery window.

You will use only the Delivery Delay table and the Supply Agreement table to calculate delay impacts of the most recent record on the delivery delay table.
You will show the user your accurate calculations for approval, and create an incident record with your analysis results. 
You will update the Delivery Delay record with the sysId that is logged in your memory.

Use the following procedure:
1. Begin by looking up the latest pending record on the delay table, and return the route_id and customer_id to confirm.
2. Read the proposed routes field for the route id, and identify the different routes in bullet point format by {"option_id", "route_number", "distance_miles", "eta_minutes"}.
  - For example: 
•	{"option_id": opt-1, "route_number": 1, "distance_miles": 103, "eta_minutes": 496}.
3. Using the Supply Agreement Reader Tool, look up the Supply Agreement table and refer to the record with matching customer_id. 
4. Read the deliver_window_hours field and the stockout_penalty_rate field.
5. Accurately calculate the financial impact for each option_id:
5a. Convert {eta_minutes} to {eta_hours} by accurately following this schema:
- 0 to 60 = 1 hour
- 61 to 120 = 2 hours
- 121 to 180 = 3 hours
- 181 to 240 = 4 hours
- 241 to 300 = 5 hours
- 301 to 360 = 6 hours
- 361 to 420 = 7 hours
- 421 to 480 = 8 hours
- 481 to 540 = 9 hours
- 541 to 600 = 10 hours
- 601 to 660 = 11 hours
- 661 to 720 = 12 hours
- 721 to 780 = 13 hours
- 781 to 840 = 14 hours
- 841 to 900 = 15 hours
- 901 to 960 = 16 hours
- 961 to 1020 = 17 hours
- 1021 to 1080 = 18 hours
- 1081 to 1140 = 19 hours
- 1141 to 1200 = 20 hours

5b. Subtract {deliver_window_hours} from {eta_hours}. The result is {penalty_hours}.
5c. Multiply {penalty_hours} by {stockout_penalty_rate}. The result is the calculated_impact.
6. Repeat steps 5a,5b,5c for each option_id
7.  **ALWAYS** show the user results for calculated_impact as follows: {option_id, route_number, distance_miles, eta_minutes, eta_hours, penalty_hours, calculated_impact}.
   For example:
{The financial impact for each option has been calculated:
•	{opt-1: route_number: 4, distance_miles: 133, eta_minutes: 229, eta_hours: 4, penalty_hours: 1, calculated impact: 250}
•	{opt-2: route_number: 13, distance_miles: 190, eta_minutes: 329, eta_hours: 6, penalty_hours: 3, calculated impact: 750}
•	{opt-3: route_number: 12, distance_miles: 124, eta_minutes: 441, eta_hours: 8, penalty_hours: 5, calculated impact: 1250}
}


After showing your calculated_impact result to the user, create an incident record with the calculated_impact in the description.
Add route_id, customer_name, and problem_description in the Short Description,
and {option_id, route_number, distance_miles, eta_minutes, eta_hours, penalty_hours, calculated_impact} for each option in the description.
For example, add in incident description:
{The financial impact for each option has been calculated:
•	{opt-1: route_number: 4, distance_miles: 133, eta_minutes: 229, eta_hours: 4, penalty_hours: 1, calculated impact: 250}
•	{opt-2: route_number: 13, distance_miles: 190, eta_minutes: 329, eta_hours: 6, penalty_hours: 3, calculated impact: 750}
•	{opt-3: route_number: 12, distance_miles: 124, eta_minutes: 441, eta_hours: 8, penalty_hours: 5, calculated impact: 1250}
}

After incident creation, log the {sys ID} into memory.
Find the Delivery Delay record that triggered this agent by searching for the matching route_id.
Update the discovered Delivery Delay record with {option_id, route_number, distance_miles, eta_minutes, eta_hours, penalty_hours, calculated_impact} for each option in the Calculated impact field.
Update Incident sys ID field to show the sysId of the incident record created in your memory log.
Update the Status to "calculated".
```
In creating the prompt, the focus was to concisely steer the Agent to find a calculated penalty impact by subtracting the grace period of (in this case) three hours from the proposed ETA minutes, and then multiplying the result by the penalty rate per hour of (in this case) $250. In doing so, the LLM had trouble reliably converting minutes into an hour rate (-even I still have trouble turning 60 minutes into a fixed percentage of an hour!).<br/>
<br/>
Attempts to round up proved fruitless because the Agent would occasionally round down to the nearest hour integer (e.g. 4.25 hours was converted to 4 hours, lopping off 15 minutes from our proposed hourly penalty rate.) The solution instead was to create a tiered hourly schema that the LLM could categorize the ETA minutes into; anything past a given hour would fall into the following category. After all, penalty calculations in the real world are counted by the hour, whether youre one minute over, or 59 minutes over.<br/>
<br/>
After the Route Financial Analysis Agent made its calculations, created an incident record, and updated the appropriate Delivery Delay record to reflect a 'Calculated' status, the second Agent -Route Decision Agent- took over delegation duties. By searching the Delivery Delay table for status:Calculated records, the Agent was able to discover all route alternatives for the corresponding record and select the array of data with the least calculated impact. After showing it to the user, the Route Decision Agent would update the Delivery Delay table to show it had chosen this optimal option (as well as the incident record created by the Route Financial Analysis Agent.) The Agent was instructed exactly to update this chosen option field on the Delivery Delay table in a way that it read as a JSON object:
```
{"route_id": {{route_id}}, "truck_id": {{truck_id}}, "chosen_option": {"option_id": "{{option_id}}", "route_number": {{route_number}}, "distance_miles": {{distance_miles}}, "eta_minutes": {{eta_minutes}}}}
```
This necessary step would be the foundation for the next agent to read the chosen option field properly, before the agent used a script tool to send an API call to an n8n webhook for further external execution.<br/>
<br/>
The full Route Decision Agent prompt read as follows:<br/>

````
You will use the Delivery Delay table to look up the latest calculated record. Read the route_id, truck_id, incident_sys_id, and the calculated_impact.

Present calculated_impact to the user in bullet point format.

Next, analyze the calculated_impact field.
You will read the arrays in the calculated_impact field and choose the array with the least value for "calculated impact:". 

SHOW the user this array and all of its contents. Include {option_id, route_number, distance_miles, eta_minutes, eta_hours, penalty_hours, calculated_impact}.
Save this array in your memory log as {chosen_array}.

Once approved, update the record chosen_option field with chosen_array {option_id, route_number, distance_miles, eta_minutes, eta_hours, penalty_hours, calculated_impact}}
and update the status field to "approved".

Next, update the appropriate incident record related to the route_id by using the incident_sys_id provided.
Add the chosen_option to the Description of the incident record, and change the State to In Progress.

Next create a JSON array that includes route_id, truck_id, chosen_option, option_id, route_number, distance_miles, eta_minutes. For example:
{
  "route_id": "751526",
  "truck_id": "1130",
  "chosen_option": {
    "option_id": "opt-2",
    "route_number": 2,
    "distance_miles": 300,
    "eta_minutes": 103
  }
}
SHOW the user the {json_array} and store these values in your memory log. Then,
Use the Call n8n Webhook tool script to send API call to n8n webhook, using the route_id in the array.
````
<br/>
After successfully sending a post API request to n8n, the third Workflow Implementation Agent took over to forward the received optimal routing information (in JSON format) to the logistics MCP server, as well as to the Retail MCP server to politely and courteously notify them of the delay. Finally, the Agent also returned an update request to the ServiceNow MCP server to update the appropriate Delivery Delay record to 'Dispatched."<br/>
<br/>
The Workflow Implementation Agent's prompt was as follows:<br/>
<img width="2487" height="1196" alt="agent node" src="https://github.com/user-attachments/assets/8ad11bd4-5966-4fb5-b1ac-3fa3c77d1a9d" />
<br/>

```
You will receive webhook payloads from previous nodes containing routing decisions: 
{
  "route_id": "{{ $json.body.route_id }}",
  "truck_id": "{{ $json.body.truck_id }}",
  "chosen_option": {
    "option_id": "{{ $json.body.chosen_option.chosen_option.option_id }}",
    "route_number": {{ $json.body.chosen_option.chosen_option.route_number }},
    "distance_miles": {{ $json.body.chosen_option.chosen_option.distance_miles }},
    "eta_minutes": {{ $json.body.chosen_option.chosen_option.eta_minutes }}
  }
}

Coordinate execution with external logistics providers, send retail customer notifications, and update ServiceNow with execution status. You should construct appropriate payloads for each external system while maintaining consistent data flow.

Connect to the Logistics MCP Client by using tool execute_route and provide (route_id, truck_id, chosen_option). 
For example:
{"route_id": "{{ $json.body.chosen_option.route_id }}", "truck_id": "{{ $json.body.chosen_option.truck_id }}", "chosen_option": "{{ $json.body.chosen_option.chosen_option }}"}

Required payload format for execute_route:
{
  "route_id": "751526",
  "truck_id": "1130",
  "chosen_option": {
    "option_id": "opt-2",
    "route_number": 2,
    "distance_miles": 300,
    "eta_minutes": 103
  }
}

Connect to the Retail MCP Client by using tool notify_delivery_delay and provide (route_id, truck_id, chosen_option). 
For example:
{"route_id": "{{ $json.body.chosen_option.route_id }}", "truck_id": "{{ $json.body.chosen_option.truck_id }}", "chosen_option": "{{ $json.body.chosen_option.chosen_option }}"}

Required payload format for notify_delivery_delay:
{
  "route_id": "751526",
  "truck_id": "1130",
  "chosen_option": {
    "option_id": "opt-2",
    "route_number": 2,
    "distance_miles": 300,
    "eta_minutes": 103
  }
}

Connect to the ServiceNow MCP Client by using tool update_execution_status and provide (route_id, status: dispatched). 
For example:
{"route_id": "{{ $json.body.chosen_option.route_id }}", "status": "dispatched"}

Required payload format for update_execution_status:
{
  "route_id": "751526",
  "status": "dispatched"
}

ALWAYS use tools provided.
DO NOT deviate from examples and DO NOT make assumptions.
ALWAYS use JSON output.
```
<br/>

From here, the call back to the ServiceNow MCP Client proved successful, indicating the appropriate alternate route information was passed to logistics partners as well as retail partners.
<br/>

<img width="1391" height="550" alt="dd dispatched" src="https://github.com/user-attachments/assets/eacd223b-4a6c-4fe4-a64b-62b085fb36c5" />

## <ins>Architecture Diagram</ins><br/>
<img width="1986" height="1381" alt="Diagram" src="https://github.com/user-attachments/assets/99cef248-10f7-45d4-95bf-1c54a5c79c02" /><br/>

## <ins>Testing Results</ins><br/>
### Route Financial Analysis Agent
<img width="2552" height="1308" alt="agent test" src="https://github.com/user-attachments/assets/9f861edf-705e-49a8-8374-80b8b256e559" />
<br/>
<br/>

Delivery Delay record updated with routing calculations
<img width="1420" height="559" alt="ddrecord calculated" src="https://github.com/user-attachments/assets/2abd5690-04c0-456f-9f37-42f41149fe17" />
<br/>
<br/>

Created Incident record
<img width="1313" height="444" alt="inc with calculated" src="https://github.com/user-attachments/assets/ffb5c9f1-9772-4b38-8eed-c1d823e649f2" />
<br/>
<br/>

### Route Decision Agent<br/>
<img width="766" height="1209" alt="result1" src="https://github.com/user-attachments/assets/6bce6ed4-39bf-43db-928a-81cb3a1684b5" />
<img width="758" height="1207" alt="result2" src="https://github.com/user-attachments/assets/9ddef836-c903-49e0-89c2-dea8d1d84ed1" />
<br/>
<br/>

Delivery Delay record updated with approved option
<img width="1376" height="551" alt="dd approved" src="https://github.com/user-attachments/assets/5b006a4c-271e-4126-afed-ec74768324f9" />
<br/>
<br/>

Incident record updated with optimal route choice
<img width="1254" height="436" alt="inc updated" src="https://github.com/user-attachments/assets/d7235d9a-b6c1-4f2f-b05a-3a3f19882eac" />
<br/>
<br/>

Logistics MCP Client Server communication
<img width="2529" height="536" alt="execute route" src="https://github.com/user-attachments/assets/1ea9a042-0606-4328-8cae-95627f11f478" />
<br/>
<br/>

Retail MCP Client Server communication
<img width="2549" height="549" alt="notify delivery delay" src="https://github.com/user-attachments/assets/325024cf-ed76-4bae-9c2d-b6920a2956f6" />
<br/>
<br/>

ServiceNow MCP Client Server communication
<img width="2531" height="573" alt="update execution status" src="https://github.com/user-attachments/assets/1d0d7e4e-c244-4ca7-9ce9-bc989f4ee704" />
<br/>
<br/>

Delivery Delay record updated with dispatched status
<img width="1391" height="550" alt="dd dispatched" src="https://github.com/user-attachments/assets/f73ec25a-cc53-4718-8416-67954d3564c9" />
<br/>
<br/>


## <ins>Business Value</ins><br/>
In our use case example involving PepsiCo, the system improves PepsiCo's supply chain operations by reducing manual intervention and response times to as little as three to four minutes. It optimizes delivery cost management by accurately calculating alternate route options by ETA instead of depending solely on distance. At larger scale, the system can identify independent Supply Agreement terms for different retail partners and calculate financial impacts as necessary. Additionally, retail partners are automatically and instantly notified of changes to their expected deliveries. 

