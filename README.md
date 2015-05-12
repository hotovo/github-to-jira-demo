# JIRA Rest Connector Github example

+ [Use Case](#usecase)
+ [Run it!](#runit)
	* [Running on Studio](#runonstudio)
	* [Properties to be configured](#propertiestobeconfigured)
+ [Customize It!](#customizeit)
+ [Report an Issue](#reportissue)


### Author
Hotovo s.r.o.

### Supported Mule runtime versions
Mule 3.5.x or higher

### JIRA supported versions
JIRA 5.2 or higher

### Service or application supported modules
JIRA Rest API 5.2 or higher


# Use Case <a name="usecase"/>
This example application shows simplified use of the JIRA Rest Connector in conjunction with Github connector. It serves as a simple example about how to automatically 
create Issue's worklog entries from commits of a linked Github repository. It searches for Issue key pattern in commit messages and automatically creates worklog entries for given Issue. 
The section immediately below describes the functions of each of the flows in the application. 

This example assumes that you are familiar with Mule, the Anypoint Studio interface, Global Elements, DataSense, and theDataMapper transformer. Further, it is assumed that you are 
familiar with Github and JIRA and have a Github and JIRA account. Learn more about flows and subflows to gain more insight into the design of this example.

Please note that this example serves only as an example and it is not suitable to run in production systems as the performance of this approach would not be sufficient for production environment and 
larger repositories.

# Run it! <a name="runit"/>
Follow these simple steps to get this example running

### Where to Download Mule Studio and Mule ESB
First thing to know if you are a newcomer to Mule is where to get the tools.

+ You can download Mule Studio from this [Location](http://www.mulesoft.com/platform/mule-studio)
+ You can download Mule ESB from this [Location](http://www.mulesoft.com/platform/soa/mule-esb-open-source-esb)


### Importing example into Studio
Mule Studio offers several ways to import a project into the workspace, for instance: 

+ Anypoint Studio generated Deployable Archive (.zip)
+ Anypoint Studio Project from External Location
+ Maven-based Mule Project from pom.xml
+ Mule ESB Configuration XML from External Location

You can find a detailed description on how to do so in this [Documentation Page](http://www.mulesoft.org/documentation/display/current/Importing+and+Exporting+in+Studio).

### Running on Studio <a name="runonstudio"/>
Once you have imported this example into Anypoint Studio you need to follow these steps to run it:

+ install Jira Rest Connector into Anypoint Studio [Installing connector](#installingConnector)
+ install Github connector into Anypoint Studio
+ Locate the properties file `common.properties`, in src/main/resources
+ Complete all the properties required as per the examples in the section [Properties to be configured](#propertiestobeconfigured)
+ Once that is done, right click on project folder 
+ Hover you mouse over `"Run as"`
+ Click on  `"Mule Application"`

## Installing connector <a name="installingConnector"/>
The following sections details how to use and install this module under different scenarios / environments.
To make the module available to a Mavenized Mule application, first add the following repositories to your POM:

```
<repositories>
    <repository>
        <id>mulesoft-releases</id>
        <name>MuleSoft Repository</name>
        <url>http://repository-master.mulesoft.org/releases/</url>
        <layout>default</layout>
    </repository>
    <repository>
        <id>mulesoft-snapshots</id>
        <name>MuleSoft Snapshot Repository</name>
        <url>http://repository-master.mulesoft.org/snapshots/</url>
        <layout>default</layout>
    </repository>
</repositories>
```

Then add the module as a dependency to your project. This can be done by adding the following under the dependencies element your POM:

```
<dependency>
    <groupId>org.mule.modules</groupId>
    <artifactId>jirarest-connector</artifactId>
    <version>RELEASE</version>
</dependency>
```

or if you want to be on the bleeding edge

```
<dependency>
    <groupId>org.mule.modules</groupId>
    <artifactId>jirarest-connector</artifactId>
    <version>LATEST</version>
</dependency>
```

If you plan to use this module inside a Mule application, you need add it to the packaging process. That way the final ZIP file which will contain your flows and Java code will also contain this module and its dependencies. Add an special inclusion to the configuration of the Mule Maven plugin for this module as follows:

```
<plugin>
    <groupId>org.mule.tools</groupId>
    <artifactId>maven-mule-plugin</artifactId>
    <extensions>true</extensions>
    <configuration>
        <excludeMuleDependencies>false</excludeMuleDependencies>
        <inclusions>
            <inclusion>
                <groupId>org.mule.modules</groupId>
                <artifactId>jirarest-connector</artifactId>
            </inclusion>
        </inclusions>
    </configuration>
</plugin>
```


## Properties to be configured (With examples) <a name="propertiestobeconfigured"/>
In order make this example running you need to configure properties (Credentials, configurations, etc.) in common.properties file

```
poll.frequencyMillis=20000
poll.startDelayMillis=0
poll.offset=3000

### Github
github.username=test@user.com
github.password=passwd
github.repository=repository-name
github.repository.owner=repositor-owner

### JIRA
jira.username=jirauser
jira.password=passwd
jira.host=https://example.atlassian.net
```

# Customize It!<a name="customizeit"/>
This section contains a brief description about how this example works. Flow configuration file is separated in three disctinct flows to make it more readable. 

## mainflow
Inbound Github connector is Invoked here by the Poll. It periodically fetches all commits for configured Github repository.Main processing is done in the expression component. Each commit message is parsed here to find issue with duration pattern. Issue with duration pattern follows this structure
 #ISSUE-KEY [number]w [number]d [number]h [number]m
Example:

```
#TEST-123 2w 1d 2h will result in worklog for issue TEST-123 with duration 2 weeks 1 day 2 hours
#TEST-124 2h 30m will result in worklog for issue TEST-124 with duration 2 hours 30 minutes
```

At the end of the expression component parsed issue key and duration pattern is appended to the current payload (that is to the current commit key-value Map)

```
   <flow name="mainFlow" doc:name="mainFlow">
        <poll doc:name="Poll">
            <fixed-frequency-scheduler frequency="${poll.frequencyMillis}" startDelay="${poll.startDelayMillis}"/>
            <github:get-commits config-ref="GitHub" owner="${github.repository.owner}" repository="${github.repository}" doc:name="get all Commits"/>
        </poll>
        <data-mapper:transform config-ref="List_RepositoryCommit__To_Map" doc:name="Pojo to Map"/>
        <foreach doc:name="For Each Commit">
            <expression-component doc:name="Extract Issue key"><![CDATA[		
		import java.util.regex.Matcher;
		import java.util.regex.Pattern;
		import java.text.ParseException;
		import java.text.SimpleDateFormat;
		import java.util.Date;
		String input = payload.comment;
        String key = null;
        String duration = null;
        // search for issue key pattern #ISSUE-KEY
        Pattern pattern = Pattern.compile("(?:#)([A-Z]+-\\d+)");
        Matcher matcher = pattern.matcher(input);
        if (matcher.find())
            key = matcher.group(1);
            
        // search for duration pattern 1w 1d 1h 1m
        if (key != null)
        {
            String substring = input.substring(input.indexOf(key) + key.length());
            String durationPattern = "";
            pattern = Pattern.compile("\\d+w");
            matcher = pattern.matcher(substring);
            if (matcher.find())
                durationPattern += matcher.group() + " ";
            pattern = Pattern.compile("\\d+d");
            matcher = pattern.matcher(substring);
            if (matcher.find())
                durationPattern += matcher.group() + " ";
            pattern = Pattern.compile("\\d+h");
            matcher = pattern.matcher(substring);
            if (matcher.find())
                durationPattern += matcher.group() + " ";
            pattern = Pattern.compile("\\d+m");
            matcher = pattern.matcher(substring);
            if (matcher.find())
                durationPattern += matcher.group();
            duration = durationPattern;
        }
        
		payload.duration = duration;
		payload.issue = key;
		SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSSZ");
		payload.started = sdf.format(payload.date);
		
		// format custom description for worklog
		String url = "https://github.com/${github.repository.owner}/${github.repository}/commit/"+payload.sha;
		payload.worklogdescription = "[View commit on Github|"+url+"] \n"+payload.comment;
]]></expression-component>
            <expression-filter expression="#[payload.issue != null]" doc:name="filter out commits without issue pattern"/>
            <flow-ref name="getIssueWorklogsFlow" doc:name="run getIssueWorklogs"/>
        </foreach>
    </flow>
```

## getIssueWorklogsFlow
This subflow searches for existing Issue matched by issue pattern from the mainFlow. If Issue with matched key exists, subflow continues execution by selecting existing worklog entries of the Issue with datamapper component.  Subsequently it compares start date of each worklog entry with each commit message create date to decide whether commit with same start date already exists for this issue. So only commits with different created date will proceed to the next subflow. Commits with matching created date are skipped. 

```
     <sub-flow name="getIssueWorklogsFlow" doc:name="getIssueWorklogsFlow">
        <expression-component doc:name="set createWorklog"><![CDATA[flowVars.put("commit",payload);
flowVars.put("createWorklog",true);
flowVars.put("issue", payload.issue);
]]></expression-component>
        <jirarest:issues-issue-get config-ref="JiraRest" entityType="Issue" expand="worklog" issueKeyOrId="#[payload.issue]" doc:name="query for Issue by Issue key"/>
        <data-mapper:transform config-ref="Issue_To_Worklog" doc:name="Issue To Worklog"/>
        <foreach doc:name="For Each existing worklog">
            <expression-component doc:name="convert string to date"><![CDATA[import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSSZ");
Date date = sdf.parse(payload.get('started'));
payload.put('started', date);]]></expression-component>
            <expression-filter expression="#[payload.started.getTime() == flowVars.commit.date.getTime()]" doc:name="filter out existing worklogs"/>
            <expression-component doc:name="store 'false' in createWorklog"><![CDATA[flowVars.put("createWorklog",false);
]]></expression-component>
        </foreach>
        <flow-ref name="createWorklog" doc:name="run createWorklog"/>
    </sub-flow>
```

## createWorklogFlow
In this sublow only non-existing commits as worklogs are inserted inseted to JIRA Issue using WORKLOGS - create worklog operation.

```
    <sub-flow name="createWorklog" doc:name="createWorklog">
        <expression-filter expression="#[flowVars.createWorklog == true &amp;&amp; payload.size() &gt;0]" nullReturnsTrue="true" doc:name="filter out commits with createWorklog set to 'false'"/>
        <jirarest:issues-worklog-create config-ref="JiraRest" entityType="Worklog" issueKeyOrId="#[flowVars.issue]" doc:name="insert new Worklog">
            <jirarest:payload>
                <jirarest:payload key="started">#[flowVars.commit.started]</jirarest:payload>
                <jirarest:payload key="timeSpent">#[flowVars.commit.duration]</jirarest:payload>
                <jirarest:payload key="comment">#[flowVars.commit.worklogdescription]</jirarest:payload>
            </jirarest:payload>
        </jirarest:issues-worklog-create>
        <logger message="worklog created #[payload]" level="INFO" doc:name="Logger"/>
    </sub-flow>
```

# Report an Issue <a name="reportissue"/>
We use JIRA for tracking issues with this connector. You can report new issues at this location https://hotovo.jira.com
