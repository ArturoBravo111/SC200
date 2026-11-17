# SC-200 — Kusto Query Language Practice Workbook

## Contents

- [1. Explore the data](#1-explore-the-data)
- [2. Filter and shape results](#2-filter-and-shape-results)
- [3. Count, group and visualise](#3-count-group-and-visualise)
- [4. Investigate Microsoft Entra sign-ins](#4-investigate-microsoft-entra-sign-ins)
- [5. Combine and parse data](#5-combine-and-parse-data)
- [6. Build small investigations](#6-build-small-investigations)
- [Solutions](#solutions)


## 1. Explore the data

### 01. Check that KQL works

**Example**

```kusto
print Total = 3 + 4, Message = "KQL is working"
```

`print` creates a one-row result. No log table is required.

**Exercise**

Create a one-row result with Course = "SC-200", the result of 7 * 6 and the current time.


### 02. Inspect security events

**Example**

```kusto
SecurityEvent_CL
| getschema
```

`getschema` lists the columns and types available in the Windows security-event table.

**Exercise**

Show five complete rows from SecurityEvent_CL.


### 03. Inspect cloud sign-ins

**Example**

```kusto
SigninLogs_CL
| getschema
```

Inspect the Microsoft Entra sign-in schema before reading individual records.

**Exercise**

Show five complete rows from SigninLogs_CL.


### 04. Count the records

**Example**

```kusto
SecurityEvent_CL
| take 3
```

`take` returns a small sample of rows. A count requires an aggregation instead.

**Exercise**

Return one row containing the total number of SecurityEvent_CL records.


### 05. Check the age of the sample data

**Example**

```kusto
SecurityEvent_CL
| summarize Rows = count()
```

`summarize` can return one row describing an entire dataset.

**Exercise**

Find the earliest and latest TimeGenerated values, plus the total number of security-event rows.


## 2. Filter and shape results

### 06. Select useful columns

**Example**

```kusto
SigninLogs_CL
| project TimeGenerated, UserPrincipalName_s
| take 10
```

`project` keeps only the columns needed for a compact view.

**Exercise**

Display TimeGenerated, Computer, Account_s and EventID_s for twenty security events.


### 07. Find successful Windows logons

**Example**

```kusto
SecurityEvent_CL
| where tostring(EventID_s) == "4625"
| take 10
```

`where` selects rows that match a condition. Event ID 4625 is a failed logon.

**Exercise**

Find security events with Windows Event ID 4624 and display the time, computer and account.


### 08. Select two event types

**Example**

```kusto
SecurityEvent_CL
| where tostring(EventID_s) in ("4624", "4634")
| take 10
```

`in` checks whether a value belongs to a specified list.

**Exercise**

Retrieve both Event ID 4624 and Event ID 4625 in a single query.


### 09. Apply two filters

**Example**

```kusto
SecurityEvent_CL
| where AccountType_s =~ "User"
| take 10
```

The `=~` operator performs a case-insensitive equality comparison.

**Exercise**

Find successful logons where AccountType_s is User.


### 10. Find a text fragment

**Example**

```kusto
SecurityEvent_CL
| where Account_s contains "admin"
| project Account_s
| take 10
```

`contains` searches for a substring. It may match within a longer account name.

**Exercise**

Find Windows security events where Account_s contains the text "sql".


### 11. Find the latest events

**Example**

```kusto
SecurityEvent_CL
| sort by TimeGenerated desc
| take 5
```

Sorting before limiting returns the five latest events instead of an arbitrary sample.

**Exercise**

Display the twenty most recent security events, sorted by TimeGenerated.


### 12. Create a calculated label

**Example**

```kusto
SecurityEvent_CL
| extend Category = iff(tostring(EventID_s) == "4625", "Failure", "Other")
| project EventID_s, Category
| take 10
```

`extend` adds a calculated column; `iff` chooses a value according to a condition.

**Exercise**

Add an EventClass column: "Successful logon" for 4624, "Failed logon" for 4625, and "Other event" for everything else.


### 13. Reuse a time window with let

**Example**

```kusto
let Limit = 5;
SecurityEvent_CL
| take Limit
```

`let` names a value or expression, then reuses it in the query.

**Exercise**

Define Window = 7d with let; filter security events to that interval and show the latest twenty.


## 3. Count, group and visualise

### 14. Count events by type

**Example**

```kusto
SecurityEvent_CL
| summarize Events = count()
```

This counts all rows. Add a `by` clause to produce counts for separate groups.

**Exercise**

Count security events for each EventID_s and show the fifteen most frequent types.


### 15. Count events per device

**Example**

```kusto
SecurityEvent_CL
| summarize Events = count() by Account_s
| top 5 by Events desc
```

Grouping by an account produces one row per account; the next task changes the grouping.

**Exercise**

Count security events by Computer and show the fifteen busiest computers.


### 16. Count failed logons per account

**Example**

```kusto
SecurityEvent_CL
| where tostring(EventID_s) == "4624"
| summarize Logons = count() by Account_s
```

Filter before `summarize` to count a particular type of event.

**Exercise**

Count failed logon events (4625) by Account_s and show the top fifteen accounts.


### 17. Count unique accounts

**Example**

```kusto
SecurityEvent_CL
| summarize DistinctDevices = dcount(Computer) by Account_s
| take 10
```

`dcount` estimates the number of distinct values; here it counts devices per account.

**Exercise**

For successful logons (4624), show total successful events and distinct accounts for each Computer.


### 18. Keep a set of account names

**Example**

```kusto
SecurityEvent_CL
| summarize ExampleAccounts = make_set(Account_s, 5) by Computer
| take 10
```

`make_set` collects distinct values into an array, with a maximum output size.

**Exercise**

For successful logons, create a unique account-name list per Computer, limited to twenty names.


### 19. Find the latest record per account

**Example**

```kusto
SecurityEvent_CL
| summarize Latest = max(TimeGenerated) by Account_s
| take 10
```

`max` returns a timestamp. `arg_max` can return columns from the latest complete record.

**Exercise**

Return the latest complete security-event record for each Account_s using arg_max().


### 20. Compare order of filtering

**Example**

```kusto
SecurityEvent_CL
| where tostring(EventID_s) == "4624"
| summarize LatestSuccess = max(TimeGenerated) by Account_s
```

Filtering first asks for the latest successful logon. The challenge asks a different question.

**Exercise**

Find accounts for which the most recent event of any type was 4624. Apply the event filter after arg_max().


### 21. Chart event volume over time

**Example**

```kusto
SecurityEvent_CL
| summarize Events = count() by bin(TimeGenerated, 1d)
| sort by TimeGenerated asc
```

`bin` groups timestamps into intervals. Here we count activity per day.

**Exercise**

Group events into one-hour time intervals and display a timechart.


### 22. Chart the busiest devices

**Example**

```kusto
SecurityEvent_CL
| summarize Events = count() by EventID_s
| top 5 by Events desc
| render columnchart
```

`render columnchart` requests a visualisation of the aggregated results.

**Exercise**

Show a column chart of the ten computers with the most security events.


## 4. Investigate Microsoft Entra sign-ins

### 23. Find the most active identities

**Example**

```kusto
SigninLogs_CL
| project TimeGenerated, UserPrincipalName_s
| take 10
```

The sign-in table contains identities that can be used as grouping keys.

**Exercise**

Count sign-in records by UserPrincipalName_s, and show the fifteen most active identities.


### 24. Find the most used cloud apps

**Example**

```kusto
SigninLogs_CL
| summarize Events = count() by UserPrincipalName_s
| top 5 by Events desc
```

Here sign-ins are counted by user; the challenge groups them by application.

**Exercise**

Count sign-in records and distinct users by AppDisplayName_s.


### 25. Count source IPs for each user

**Example**

```kusto
SigninLogs_CL
| extend SourceIP = coalesce(
    tostring(column_ifexists("IPAddress_s", "")),
    tostring(column_ifexists("IPAddress", "")))
| project UserPrincipalName_s, SourceIP
| take 10
```

`column_ifexists` handles alternate column names. `coalesce` selects an available non-empty value.

**Exercise**

Count the number of distinct source IPs observed for each user. Check IPAddress_s or IPAddress as appropriate.


### 26. Investigate disabled-account errors

**Example**

```kusto
SigninLogs_CL
| extend Description = coalesce(
    tostring(column_ifexists("ResultDescription_s", "")),
    tostring(column_ifexists("ResultDescription", "")))
| project UserPrincipalName_s, Description
| take 10
```

First inspect the result-description text. The challenge adds a search condition.

**Exercise**

Find sign-in records whose result description contains the word "disabled".


### 27. Inspect authentication details JSON

**Example**

```kusto
print Raw = '[{"authenticationMethod":"Password"}]'
| extend Details = parse_json(Raw)
| project FirstMethod = tostring(Details[0].authenticationMethod)
```

`parse_json` turns a JSON string into a dynamic array; `[0]` accesses its first element.

**Exercise**

Parse AuthenticationDetails_s and extract the authenticationMethod from the first array element.


### 28. Expand every authentication step

**Example**

```kusto
print Methods = dynamic(["Password", "Authenticator"])
| mv-expand Methods
```

`mv-expand` transforms array elements into separate rows.

**Exercise**

Expand all authentication steps into separate rows, showing the user and method.


## 5. Combine and parse data

### 29. Combine two sources with union

**Example**

```kusto
union
  (SecurityEvent_CL | project TimeGenerated),
  (SigninLogs_CL | project TimeGenerated)
| take 10
```

`union` stacks rows from multiple sources. Common column names make combined results easier to use.

**Exercise**

Combine Windows and Entra rows into one result containing TimeGenerated, Origin and Identity.


### 30. Compare logons with logoffs using join

**Example**

```kusto
let Employees = datatable(Name:string)["Alice", "Bob"];
let Allowed = datatable(Name:string)["Alice"];
Employees
| join kind=inner (Allowed) on Name
```

An inner join retains rows whose keys appear in both sets.

**Exercise**

Count successful logons and logoffs per account in separate expressions, then join the counts.


### 31. Join events with a derived top-five table

**Example**

```kusto
let CommonAccounts = SecurityEvent_CL
  | summarize Events = count() by Account_s
  | top 2 by Events desc
  | project Account_s;
SecurityEvent_CL
| join kind=inner (CommonAccounts) on Account_s
| take 10
```

`let` can name a table; joining to it selects the matching original events.

**Exercise**

Find the five most active accounts, then join that list to their original security events.


### 32. Use an in-query reference table

**Example**

```kusto
datatable(EventID:string, Meaning:string)
["4624", "Logon", "4625", "Failed logon"]
```

`datatable` creates a small in-query table. No Sentinel Watchlist is required.

**Exercise**

Make an in-query table mapping event IDs 4624, 4625 and 4688 to descriptions; join it to security events and count matches.


### 33. Extract a name using a regular expression

**Example**

```kusto
print Email = "alice@contoso.com"
| extend User = extract(@"^([^@]+)", 1, Email)
```

`extract` uses a regular expression to retrieve the characters before `@`.

**Exercise**

Use extract() to return the part of Account_s before the @ symbol.


### 34. Parse a predictable text format

**Example**

```kusto
print Raw = "user=bob action=allow ip=10.0.0.8"
| parse Raw with "user=" User " action=" Action " ip=" IP
| project User, Action, IP
```

`parse` turns consistently formatted text into named fields.

**Exercise**

Parse the text "user=alice action=deny ip=10.0.0.7" into User, Action and SourceIP.


### 35. Filter JSON array elements with mv-apply

**Example**

```kusto
print Details = dynamic([{"method":"Password"}, {"method":"App"}])
| mv-apply Step = Details on (
    where tostring(Step.method) == "Password"
    | project Method = tostring(Step.method)
)
```

`mv-apply` expands array items and processes them inside a subquery.

**Exercise**

Use mv-apply to return only Password steps from the authentication-details JSON array.


## 6. Build small investigations

### 36. Investigate a single device

**Example**

```kusto
let Device = toscalar(SecurityEvent_CL | take 1 | project Computer);
SecurityEvent_CL
| where Computer == Device
| take 5
```

`toscalar` converts a single-cell query result into a scalar that can be reused.

**Exercise**

Automatically select the busiest computer in the dataset and show its thirty latest events.


### 37. Find repeated failed logons

**Example**

```kusto
SecurityEvent_CL
| where tostring(EventID_s) == "4625"
| summarize Failures = count() by Account_s
| top 10 by Failures desc
```

First identify accounts with failed logons; the challenge adds a threshold and device context.

**Exercise**

Identify accounts with at least five failed logon events, including the number and names of affected computers.


### 38. Find a candidate multi-user sign-in source

**Example**

```kusto
SigninLogs_CL
| summarize Events = count() by UserPrincipalName_s
| top 10 by Events desc
```

Counting by account measures activity. The challenge groups by source IP instead.

**Exercise**

Find source IPs associated with two or more distinct users in the sign-in records.


### 39. Contrast the latest event with latest successful logon

**Example**

```kusto
SecurityEvent_CL
| summarize LatestAny = max(TimeGenerated) by Account_s
| take 10
```

This finds the last recorded event of any type, not the last successful logon.

**Exercise**

Compare each account's latest event type with the timestamp of its latest successful logon.


### 40. Build a compact activity profile

**Example**

```kusto
SecurityEvent_CL
| summarize Events = count(), LastSeen = max(TimeGenerated) by Account_s
| top 10 by Events desc
```

A simple activity profile combines total event volume with the latest activity timestamp.

**Exercise**

Create an activity profile per account: total events, successful logons, failed logons, distinct computers and latest timestamp.


## Solutions

## Explore the data — answers

### 01. Check that KQL works

```kusto
print Course = "SC-200", SevenTimesSix = 7 * 6, StartedAt = now()
```


### 02. Inspect security events

```kusto
SecurityEvent_CL
| take 5
```


### 03. Inspect cloud sign-ins

```kusto
SigninLogs_CL
| take 5
```


### 04. Count the records

```kusto
SecurityEvent_CL
| summarize Events = count()
```


### 05. Check the age of the sample data

```kusto
SecurityEvent_CL
| summarize FirstSeen = min(TimeGenerated),
            LastSeen = max(TimeGenerated),
            Total = count()
```


## Filter and shape results — answers

### 06. Select useful columns

```kusto
SecurityEvent_CL
| project TimeGenerated, Computer, Account_s, EventID_s
| take 20
```


### 07. Find successful Windows logons

```kusto
SecurityEvent_CL
| where tostring(EventID_s) == "4624"
| project TimeGenerated, Computer, Account_s
| take 20
```


### 08. Select two event types

```kusto
SecurityEvent_CL
| where tostring(EventID_s) in ("4624", "4625")
| project TimeGenerated, Computer, Account_s, EventID_s
| take 30
```


### 09. Apply two filters

```kusto
SecurityEvent_CL
| where tostring(EventID_s) == "4624"
| where AccountType_s =~ "User"
| project TimeGenerated, Computer, Account_s
```


### 10. Find a text fragment

```kusto
SecurityEvent_CL
| where Account_s contains "sql"
| project TimeGenerated, Account_s, Computer, EventID_s
| take 25
```


### 11. Find the latest events

```kusto
SecurityEvent_CL
| project TimeGenerated, Computer, Account_s, EventID_s
| top 20 by TimeGenerated desc
```


### 12. Create a calculated label

```kusto
SecurityEvent_CL
| extend EventClass = case(
    tostring(EventID_s) == "4624", "Successful logon",
    tostring(EventID_s) == "4625", "Failed logon",
    "Other event")
| project TimeGenerated, Account_s, EventID_s, EventClass
| take 25
```


### 13. Reuse a time window with let

```kusto
let Window = 7d;
SecurityEvent_CL
| where TimeGenerated >= ago(Window)
| project TimeGenerated, Computer, Account_s, EventID_s
| top 20 by TimeGenerated desc
```


## Count, group and visualise — answers

### 14. Count events by type

```kusto
SecurityEvent_CL
| summarize Events = count() by EventID_s
| top 15 by Events desc
```


### 15. Count events per device

```kusto
SecurityEvent_CL
| summarize Events = count() by Computer
| top 15 by Events desc
```


### 16. Count failed logons per account

```kusto
SecurityEvent_CL
| where tostring(EventID_s) == "4625"
| summarize Failures = count() by Account_s
| top 15 by Failures desc
```


### 17. Count unique accounts

```kusto
SecurityEvent_CL
| where tostring(EventID_s) == "4624"
| summarize SuccessfulLogons = count(),
            DistinctAccounts = dcount(Account_s)
    by Computer
| top 15 by DistinctAccounts desc
```


### 18. Keep a set of account names

```kusto
SecurityEvent_CL
| where tostring(EventID_s) == "4624"
| summarize Accounts = make_set(Account_s, 20)
    by Computer
| take 15
```


### 19. Find the latest record per account

```kusto
SecurityEvent_CL
| summarize arg_max(TimeGenerated, *) by Account_s
| project TimeGenerated, Account_s, Computer, EventID_s
| top 20 by TimeGenerated desc
```


### 20. Compare order of filtering

```kusto
SecurityEvent_CL
| summarize arg_max(TimeGenerated, *) by Account_s
| where tostring(EventID_s) == "4624"
| project Account_s, TimeGenerated, Computer
```


### 21. Chart event volume over time

```kusto
SecurityEvent_CL
| summarize Events = count() by bin(TimeGenerated, 1h)
| render timechart
```


### 22. Chart the busiest devices

```kusto
SecurityEvent_CL
| summarize Events = count() by Computer
| top 10 by Events desc
| render columnchart
```


## Investigate Microsoft Entra sign-ins — answers

### 23. Find the most active identities

```kusto
SigninLogs_CL
| summarize SignIns = count() by UserPrincipalName_s
| top 15 by SignIns desc
```


### 24. Find the most used cloud apps

```kusto
SigninLogs_CL
| summarize SignIns = count(),
            Users = dcount(UserPrincipalName_s)
    by AppDisplayName_s
| top 15 by SignIns desc
```


### 25. Count source IPs for each user

```kusto
SigninLogs_CL
| extend SourceIP = coalesce(
    tostring(column_ifexists("IPAddress_s", "")),
    tostring(column_ifexists("IPAddress", "")))
| where isnotempty(SourceIP)
| summarize Events = count(), DistinctIPs = dcount(SourceIP)
    by UserPrincipalName_s
| top 15 by DistinctIPs desc
```


### 26. Investigate disabled-account errors

```kusto
SigninLogs_CL
| extend Description = coalesce(
    tostring(column_ifexists("ResultDescription_s", "")),
    tostring(column_ifexists("ResultDescription", "")))
| where Description has "disabled"
| project TimeGenerated, UserPrincipalName_s,
          AppDisplayName_s, Description
| top 30 by TimeGenerated desc
```


### 27. Inspect authentication details JSON

```kusto
SigninLogs_CL
| extend Details = parse_json(AuthenticationDetails_s)
| extend FirstMethod = tostring(Details[0].authenticationMethod)
| project TimeGenerated, UserPrincipalName_s,
          FirstMethod, Details
| take 20
```


### 28. Expand every authentication step

```kusto
SigninLogs_CL
| extend Details = parse_json(AuthenticationDetails_s)
| mv-expand Details
| project UserPrincipalName_s,
          Method = tostring(Details.authenticationMethod),
          Detail = tostring(Details.authenticationStepResultDetail)
| take 30
```


## Combine and parse data — answers

### 29. Combine two sources with union

```kusto
let Windows = SecurityEvent_CL
    | project TimeGenerated, Origin = "Windows", Identity = tostring(Account_s);
let Cloud = SigninLogs_CL
    | project TimeGenerated, Origin = "Entra", Identity = tostring(UserPrincipalName_s);
union Windows, Cloud
| top 30 by TimeGenerated desc
```


### 30. Compare logons with logoffs using join

```kusto
let Logons = SecurityEvent_CL
    | where tostring(EventID_s) == "4624"
    | summarize Logons = count() by Account_s;
let Logoffs = SecurityEvent_CL
    | where tostring(EventID_s) == "4634"
    | summarize Logoffs = count() by Account_s;
Logons
| join kind=inner (Logoffs) on Account_s
| top 20 by Logons desc
```


### 31. Join events with a derived top-five table

```kusto
let BusyAccounts = SecurityEvent_CL
    | summarize EventCount = count() by Account_s
    | top 5 by EventCount desc
    | project Account_s;
SecurityEvent_CL
| join kind=inner (BusyAccounts) on Account_s
| project TimeGenerated, Account_s, Computer, EventID_s
| top 30 by TimeGenerated desc
```


### 32. Use an in-query reference table

```kusto
let FocusEventIDs = datatable(EventID:string, Meaning:string)
[
  "4624", "Successful logon",
  "4625", "Failed logon",
  "4688", "Process creation"
];
SecurityEvent_CL
| extend EventID = tostring(EventID_s)
| join kind=inner (FocusEventIDs) on EventID
| summarize Events = count() by EventID, Meaning
| order by Events desc
```


### 33. Extract a name using a regular expression

```kusto
SecurityEvent_CL
| extend ShortName = extract(@"^([^@]+)", 1, tostring(Account_s))
| project Account_s, ShortName
| take 20
```


### 34. Parse a predictable text format

```kusto
print Raw = "user=alice action=deny ip=10.0.0.7"
| parse Raw with "user=" User " action=" Action " ip=" SourceIP
| project User, Action, SourceIP
```


### 35. Filter JSON array elements with mv-apply

```kusto
SigninLogs_CL
| extend Details = parse_json(AuthenticationDetails_s)
| mv-apply Step = Details on (
    where tostring(Step.authenticationMethod) == "Password"
    | project PasswordResult = tostring(Step.authenticationStepResultDetail)
)
| project UserPrincipalName_s, PasswordResult
| take 30
```


## Build small investigations — answers

### 36. Investigate a single device

```kusto
let TargetComputer = toscalar(
    SecurityEvent_CL
    | summarize Events = count() by Computer
    | top 1 by Events desc
    | project Computer
);
SecurityEvent_CL
| where Computer == TargetComputer
| project TimeGenerated, Computer, Account_s, EventID_s
| top 30 by TimeGenerated desc
```


### 37. Find repeated failed logons

```kusto
SecurityEvent_CL
| where tostring(EventID_s) == "4625"
| summarize Failures = count(), Devices = dcount(Computer),
            DeviceNames = make_set(Computer, 10)
    by Account_s
| where Failures >= 5
| order by Failures desc
```


### 38. Find a candidate multi-user sign-in source

```kusto
SigninLogs_CL
| extend SourceIP = coalesce(
    tostring(column_ifexists("IPAddress_s", "")),
    tostring(column_ifexists("IPAddress", "")))
| where isnotempty(SourceIP)
| summarize Events = count(), Users = dcount(UserPrincipalName_s),
            ExampleUsers = make_set(UserPrincipalName_s, 10)
    by SourceIP
| where Users >= 2
| order by Users desc
```


### 39. Contrast the latest event with latest successful logon

```kusto
let LatestAny = SecurityEvent_CL
    | summarize arg_max(TimeGenerated, EventID_s) by Account_s
    | project Account_s, LatestEvent = tostring(EventID_s);
let LatestSuccess = SecurityEvent_CL
    | where tostring(EventID_s) == "4624"
    | summarize LatestSuccessfulLogon = max(TimeGenerated) by Account_s;
LatestAny
| join kind=leftouter (LatestSuccess) on Account_s
| take 25
```


### 40. Build a compact activity profile

```kusto
SecurityEvent_CL
| summarize TotalEvents = count(),
            SuccessfulLogons = countif(tostring(EventID_s) == "4624"),
            FailedLogons = countif(tostring(EventID_s) == "4625"),
            Computers = dcount(Computer),
            LastSeen = max(TimeGenerated)
    by Account_s
| top 20 by TotalEvents desc
```
