[comment]: <> (password: DevCon22)


# The developer Portal

## Login
First we login into the developer portal:

[https://bylumuiportalplpna.azureedge.net/home/](https://bylumuiportalplpna.azureedge.net/home/){:target="_blank"}

You need to enter the realm, which is `by-developer` for the developer portal.

Then login with your account credentials.

!!! note "Note"
    Please try to tidy up behind you, so if not used anymore delete the resources you create.
    Nonetheless, we will clean up all resources once per day, so don't be surprised if everything is gone, that's intentionally.

## Introduction

A very brief overview of what we are going to build in this workshop.

We are using the Luminate Platform service "Workflow" to call the LDE Prediction Overrides API. 
We will configure that workflow to be executed periodically every minute.
With the returned prediction overrides, we are going to implement a filter on prediction overrides which are created 
not longer than a minute ago. 
As the workflow runs every minute, we therefore get exclusively the newly created prediction overrides. 
For each new prediction override we then create an exception in the Luminate Platform Exception Service.


That's it, easy, right?

## Navigate the portal

<figure markdown>
  ![Image title](images/devportal_dashboard.png){: align=left }
  <figcaption>After login you see the dashboard</figcaption>
</figure>

For now we are only interested in the "App Gallery" whoch shows all services and application that you have access to.

<figure markdown>
  ![Image title](images/devportal_app-gallery_marked.png){: align=left }
  <figcaption>The App Gallery view</figcaption>
</figure>

## The API Catalog

We now want to find out if there are APIs we can use to build our notification feature.
For this we launch the API Catalog in the API Gallery

<figure markdown>
  ![Image title](images/devportal_apicatalog.png){: align=left }
  <figcaption>Launch the API Catalog</figcaption>
</figure>

Inside the API Catalog, search for LDE. You then will find all the API exposed by LDE for you.
You should find the "Blue Yonder LDE Prediction Overrides v1" API and click on it.

<figure markdown>
  ![Image title](images/devportal_apicatalog_overrides.png){: align=left }
  <figcaption>Finding the LDE Prediction Overrides API in the API Catalog</figcaption>
</figure>

Once you clicked on it, you will be navigated to the API documentation.

<figure markdown>
  ![Image title](images/LDE_api_doc_tryout.png){: align=left }
  <figcaption>Documentation of the LDE Prediction Overrides API</figcaption>
</figure>

Here you can try our an API call by first clicking on the "Try it out" button
and subsequently on the "Execute" button.



<figure markdown>
  ![Image title](images/LDE_api_doc_execute.png){: align=left }
  <figcaption>You can try the API by clicking on "Execute"</figcaption>
</figure>

By executing this GET request we will get a list of all prediction overrides stored in the LDE instance.

<figure markdown>
  ![Image title](images/PredcitionOverrides_docs_response.png){: align=left }
  <figcaption>Hey, we have a list of all prediction overrides stored in the LDE instance!</figcaption>
</figure>

The response should look like this

``` json hl_lines="10"
[
  {
    "processing_status": "FINISHED",
    "override_id": "6caccdf4-eeef-4fb7-83b6-6e96b334719b",
    "override_type": "create",
    "mode": "workbench",
    "product_column_name": null,
    "location_column_name": null,
    "value_column_name": null,
    "creation_time": "2022-04-25T22:17:22.563620+00:00",
    "is_active": true,
    "desired_is_active": true,
    "last_update": "2022-04-25T22:17:30.766706+00:00",
    "creation_user": "someone@youdelivers.com",
    "last_updated_user": "someone@youdelivers.com",
    "reason": "other",
    "description": "",
    "min_affected_date": "2022-04-28",
    "max_affected_date": "2022-05-17",
    "number_clp_combination": 273,
    "number_rows": null,
    "source": "lde-ui",
    "prediction_accuracy": null,
    "override_accuracy": null
  },
...
]
```

For our purpose, we are only interested in the `creation_time` value. But just 
look what's there, endless possibilities for cool new features, right?


## Create LIAM m2m client

Luminate Identy Access Management (LIAM) is the authentication and authorization 
service in the Luminate Platform. 

You are already logged in into the portal, so your identity is known to all services.
But when we want to build a service accessing API's, it is not you accessing the APIs, but this service.

This is what machine-to-machine clients (m2m client) are for in LIAM. So we can create new identities that we
assign to services and give it any subset of roles and permissions that our own identy has.

This is increasing security dramatically, as we can shape those permissions such that it is "just enough" to do the job.
Through the "scope" we can limit which services can be accessed, so that even if this m2m client is leaked, it
cannot be used for anything else that the anticipated use-case.

We pre-created one such m2m client. In a production setting we would share the client_secret via Azure Key Vaults or 
some other secure secrets sharing mechanism. For the sake of simplicity, we use a poor-man's out-of-band second-factor secret sharing. 
Replace the first 4 `_` with the 4 characters on the slides

```
____Q~pR26qXQ8qFgAa552NbVzBwwV1F4FJ1oaz8
```


Alternatively, in the "DevCon22 Workshop" application in ALM, there is a module called "shared m2m client" and in there 
is a workflow called "shared m2m client secret". If you enter the workflow editor for this workflow, cou can copy&paste 
the m2m client credentials from there. You learn how to access and do all that in the next section.

## Building the Workflow

Now we now there is an API there is an API we have a m2m client to be able to access this API.
Now we need something to actually call the API and trigger actions based on the results.
That is exactly what the "Workflow" feature is for. 

!!! info "Workflow in a nutshell"
    Workflow is based on the Azure Logic Apps service. Using a domain specific language (DSL) based on json one can easily 
    define workflows that connect all kinds of APIs, services and events. The documentation of the Azure Logic Apps DSL can 
    be found [here](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-workflow-definition-language).
    
    We will make use of the DSL, but also use the no-code visual workflow editor in the Luminate Portal.

<figure markdown>
  ![Image title](images/devportal_startALM.png){: align=left }
  <figcaption>Start the ALM application in the app gallery</figcaption>
</figure>

!!! info "ALM in a nutshell"
    The Application Lifecycle Management service (ALM) lets you create and maintain the lifecycle of applications you 
    create on the Luminate Platform. An application consists of multiple modules, and each module consists of 
    several resources. In our case, the only available resource type is Workflow. We have one application for our 
    workshop and every participant will create their own module in which they can build one or more workflows.


<figure markdown>
  ![Image title](images/ALMStudio_application_overview_marked.png){: align=left }
  <figcaption>Click on the DevCon22 Workshop application</figcaption>
</figure>


<figure markdown>
  ![Image title](images/ALMStudio_add_module_marked.png){: align=left }
  <figcaption>In the module list overview of the DevCon 22 Workshop click on "Add Module"</figcaption>
</figure>


<figure markdown>
  ![Image title](images/ALMStudio_create_module_name.png){: align=left }
  <figcaption>As every participant creates their own module, let's include our name in the module name, why not YOURNAME-workshop</figcaption>
</figure>


<figure markdown>
  ![Image title](images/ALMStudio_create_module_create.png){: align=left }
  <figcaption>Here you can leave everything like it is and just click on "Create Module"</figcaption>
</figure>


<figure markdown>
  ![Image title](images/ALMStudio_create_module_created_marked.png){: align=left }
  <figcaption>Now click on your newly created module</figcaption>
</figure>


<figure markdown>
  ![Image title](images/ALMStudio_add_resource_workflow_marked.png){: align=left }
  <figcaption>As your module is brand-new it should be quite empty, so let's click on "Add resource" and select "Workflow" as the resource type</figcaption>
</figure>


<figure markdown>
  ![Image title](images/ALMStudio_add_resource_workflow_template.png){: align=left }
  <figcaption>Here you have to select a template which your workflow will be based on. It happens that "LIAM M2M Authentication" is a quite good starting point for our workshop...we are really lucky!</figcaption>
</figure>


<figure markdown>
  ![Image title](images/ALMStudio_add_resource_workflow_name.png){: align=left }
  <figcaption>Oh gosh, we have to choose another name, I thought maybe "prediction_overrides_workflow" is a pretty good one.</figcaption>
</figure>


<figure markdown>
  ![Image title](images/ALMStudio_add_resource_workflow_created_marked.png){: align=left }
  <figcaption>Hey, our workflow should be created, let's jump right into the implementation and hit the edit icon!</figcaption>
</figure>

## From Low-Code to Pro-Code

!!! info "Low-Code <> Pro-Code"
    There is a low-code representation of your workflow, but there you have only limited customizability. 
    So you will often have to switch between the low-code editor, and the pro-code editor. You can either enter the 
    pro-code of the full workflow, or just the pro-code representation of a single node. In this section you will 
    learn how to jump between those.

<figure markdown>
  ![Image title](images/WorkflowEditor_template.png){: align=left }
  <figcaption>Once you are in the low-code editor you should see the template which is just one HTTP action.</figcaption>
</figure>


<figure markdown>
  ![Image title](images/WorkflowEditor_global_switchprocode_marked.png){: align=left }
  <figcaption>When no node is selected (click on the white background), "Switch to Pro-Code" brings you to the pro-code representation of the global workflow.</figcaption>
</figure>

<figure markdown>
  ![Image title](images/WorkflowEditor_global_procode_marked.png){: align=left }
  <figcaption>"Switch to Workflow editor" brings you back to the low-code representation.</figcaption>
</figure>


<figure markdown>
  ![Image title](images/WorkflowEditor_node_switchprocode.png){: align=left }
  <figcaption>If you select a node (notice the "Action Properties" window in the top right) "Switch to Pro-Code" brings you to the pro-code representation of only the selected node.</figcaption>
</figure>


<figure markdown>
  ![Image title](images/WorkflowEditor_node_procode.png){: align=left }
  <figcaption>And "Switch to Workflow Editor" brings you back.</figcaption>
</figure>

Now you know everything about low-code, pro-code. If you just stay in the pro-code representation, I will not hate you ;-)

## The Save, Publish, Run, Check Cycle

In this section we learn how we do the usual development cycle: change something, run it, check if the results are as 
expected (which barely ever is the case, right?)

<figure markdown>
  ![Image title](images/WorkflowEditor_template_cycle.png){: align=left }
  <figcaption>The development cyle is 1) Save 2) Publish 3) Run 4) Open Run History.</figcaption>
</figure>

If we do the cycle with the template we should see a failed first run.


<figure markdown>
  ![Image title](images/WorkflowEditor_failedrun_marked.png){: align=left }
  <figcaption>Surprise, the result is not quite what we expected, the run failed. Maybe you spot what might be wrong. Click on "Close" to go back to the workflow editor.</figcaption>
</figure>

<figure markdown>
  ![Image title](images/WorkflowEditor_addm2m_cred_marked.png){: align=left }
  <figcaption>Sure, we haven't entered the credentials from our m2m client yet. Let's do that and do the cycle. (Don't forget to hit "Save Changes" before.)</figcaption>
</figure>


<figure markdown>
  ![Image title](images/WorkflowEditor_getLIAMToken_suceeded.png){: align=left }
  <figcaption>Nice!</figcaption>
</figure>

We just debugged our workflow template! We are now ready for new challenges!

## Developing our first own action

Time to develop our first own action. Now that we have the LIAM token, we can authenticate to the any Blue yonder API. 
So let's call the LDE Overrides API!

Hit the plus sign after the "Get LIAM Token" and select "Http".

<figure markdown>
  ![Image title](images/WorkflowEditor_rename_httpaction_marked.png){: align=left }
  <figcaption>Now we have a new Http action and rename it to Get Prediction Overrides.</figcaption>
</figure>

<figure markdown>
  ![Image title](images/WorkflowEditor_empty_httpaction.png){: align=left }
  <figcaption>In the pro-code view, we see it is empty, let's fill it with the right content!</figcaption>
</figure>

!!! info "How to use results from previous actions"
    If you want to use results from previous actions, you can access them with `body('NAME OF THE ACTION')`, as everything
    in Workflow is json, you can easily access values from the json body using `body('NAME OF THE ACTION')?['KEY']` and 
    finally, if we want to actually return the value (as a string), we use the `@` operator.  
    So, this should return the key 'KEY' of the result of the 'NAME OF THE ACTION' action as as tring which can be directly embedded in another string:

    `"Hello @{body('NAME OF THE ACTION')?['KEY']}"`
    
    For more details, please have a look at the Azure Logic Apps Workflow Definition Language [documentation](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-workflow-definition-language#expressions)

This is how the empty HTTP action looks like:

``` json
{
  "type": "Http",
  "inputs": {
    "uri": "http://",
    "method": "",
    "headers": {}
  },
  "runAfter": {
    "Get LIAM Token": [
      "Succeeded"
    ]
  }
}
```
Now, if we remember LDE Predictions Overrides API documentation in the 
API Catalog we saw that the request to get a list of predictions looks like this.
If you don't have a photographic memory, here is what we saw there: 

``` bash
curl -X 'GET' \
  'https://api.jdadelivers.com/lde/prediction-overrides/v1/overrides' \
  -H 'accept: application/json' \
  -H 'Authorization: Bearer eyJ0eXAiOiJKV1QiLCJh********'
```

So, the "method" seems to be "GET" and we can just enter it in the pro-code json.
``` json hl_lines="5"
{
  "type": "Http",
  "inputs": {
    "uri": "http://",
    "method": "GET",
    "headers": {}
  },
  "runAfter": {
    "Get LIAM Token": [
      "Succeeded"
    ]
  }
}
```
Can you figure out the rest? 

Sure! I help you a bit, the url we can again just copy&paste. For the "headers" we need to add some double quotes and 
commas to get it proper json, can you do that?

??? warning "this is how the "headers" should look like"
    ``` json
    "headers": {
        "accept": "application/json",
        "Authorization": "Bearer eyJ0eXAiOiJKV1QiLCJh********"
    }
    ```

Now, The "Bearer" token seems to be a bit tricky, isn't it? The actual token should not be that static string, but the 
LIAM token we created in the previous action 'Get LIAM Token'. 
It seems we need to embed a string from the result of a previous action, if only we would know how this works...

??? warning "If you want to cheat, solution is in here"
    ``` json
    {
      "type": "Http",
      "inputs": {
        "uri": "https://api.jdadelivers.com/lde/prediction-overrides/v1/overrides",
        "method": "GET",
        "headers": {
          "accept": "application/json",
          "Authorization": "Bearer @{body('Get LIAM Token')?['access_token']}"
        }
      },
      "runAfter": {
        "Get LIAM Token": [
          "Succeeded"
        ]
      }
    }
    ```

If you came that far, just "Save, Publish, Run" and check the latest run if it succeded. 

It succeeded? Great! But there are a lot of prediction overrides right, and they are all outdated which is quite 
inconvenient because they clutter the results.

But wasn't there a parameter in the api documentation to restrict the result to only prediction overrides created after 
some date?

Can you figure that out and restrict the results to only overrides from yesterday?

??? hint
    Just check the LDE Overrides Prediction API doc and figure our how the url in the curl command changes when you add the "minAffectedDate"

You think you got it? "Save, Publish, Run" and check the result! How many overrides do you have now in the response?

??? warning "Look behind you, a Three-Headed Monkey! (full solution in here)"
    If you fail to get the json right, just copy&paste. This should work: 
    ``` json
    {
      "type": "Http",
      "inputs": {
        "uri": "https://api.jdadelivers.com/lde/prediction-overrides/v1/overrides?minAffectedDate=2022-05-15",
        "method": "GET",
        "headers": {
          "accept": "application/json",
          "Authorization": "Bearer @{body('Get LIAM Token')?['access_token']}"
        }
      },
      "runAfter": {
        "Get LIAM Token": [
          "Succeeded"
        ]
      }
    }
    ```

## The Foreach Loop

Now we have a list of all recent overrides predictions, but we are only interested in the newly created overrides 
predictions since the last run of the workflow.

For this, the idea is to loop over all entries in the result of the previous HTTP action and filter for those entries, 
where the creation timestamp is greater than the last run of the workflow. 

We will later on trigger the workflow periodically every minute. So the timestamp should be greater then the current 
time 1 minute ago.

Lets start with the loop over all entries.

Just hit the plus sign after the "Get Prediction Overrides" action and select "Foreach" (see [documentation](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-workflow-actions-triggers#foreach-action)).

Let's rename it to "Process Prediction Overrides".


<figure markdown>
  ![Image title](images/WorkflowEditor_foreach_empty.png){: align=left }
  <figcaption>Again pretty empty inside. Let's figure it out!</figcaption>
</figure>


So, this is how the structure looks like. 
``` json hl_lines="4 5 6 7"
{
  "type": "Foreach",
  "actions": {
    "action": {
      "type": "",
      "runAfter": {}
    }
  },
  "foreach": "",
  "runAfter": {
    "Get Prediction Overrides": [
      "Succeeded"
    ]
  }
}
```

First we need to delete the empty action, otherwise the parser will complain.

``` json
{
  "type": "Foreach",
  "actions": {
  },
  "foreach": "",
  "runAfter": {
    "Get Prediction Overrides": [
      "Succeeded"
    ]
  }
}
```

Do you notice the foreach key? 

``` json hl_lines="9"
{
  "type": "Foreach",
  "actions": {
  },
  "foreach": "",
  "runAfter": {
    "Get Prediction Overrides": [
      "Succeeded"
    ]
  }
}
```
The "foreach" key takes a list as a value...as the result of the previous action was a list of prediction overrides...
maybe this works as before? (@ .. body... do you remember?). Try it out.

!!! hint
    This is a hard one to figure out: Curly braces `{}` convert the data to a string. 
    If you do not want to have a string, but the original data structure, you need to leave out the curly braces `{}`.
    Hey, don't worry, took me several days to figure that out ;-) (see "String interpolation" in the [documentation](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-workflow-definition-language#expressions))

??? warning "No, I don't get it, show me"
    Pretty simple if you know how to do it.
    ``` json
    {
      "type": "Foreach",
      "actions": {},
      "foreach": "@body('Get Prediction Overrides')",
      "runAfter": {
        "Get Prediction Overrides": [
          "Succeeded"
        ]
      }
    }
    ``` 

<figure markdown>
  ![Image title](images/WorkflowEditor_foreach_runstats_dummy.png){: align=left }
  <figcaption>If all goes well, we should see a dummy iteration doing nothing for every prediction override.</figcaption>
</figure>


Now, let's add some actions that are executed in every iteration over all prediction overrides.

As a reminder: Based on a "if" condition (timestamp > current time - 1 minute) we want to create an exception in the 
exception service.

Lets start with the "if" condition. In order to see the empty structure of a condition, we do a trick. As nested structures 
("If" inside "Foreach" loop) is not supported by the low-code representation currently,  
we just add a dummy condition, copy the pro-code json of it and delete it again.

Now we can remove the "else" entry and empty the "runAfter" field (as it will run inside a loop).

You can of course also refer to the [documentation](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-workflow-actions-triggers#if-action)

Or, you just copy it from here, there is no shame, as admittedly this is quite cumbersome:

``` json hl_lines="4 5 6 7 8 9"
{
  "type": "Foreach",
  "actions": {
    "Our_filter": {
      "actions": {},
      "expression": {},
      "runAfter": {},
      "type": "If"
    }
  },
  "foreach": "@body('Get Prediction Overrides')",
  "runAfter": {
    "Get Prediction Overrides": [
      "Succeeded"
    ]
  }
}
```
So, we added a new action in the empty actions section and called it "Our_Filter". The parser will complain, as the 
expression secton must not be empty, so we have to fill it...remember (timestamp > current time - 1 minute)?

!!! hint
    Inside a foreach loop, you have access to the data of the current entity with the "item" keyword. As nested loops 
    are possible, you need to pass it the name of the foreach loop. 

But this time I have to help you, because of our limited time and without reading the documentation of the Azure 
Logic Apps documentation and experimenting a lot, this is impossible to guess. (see [documentation](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-workflow-actions-triggers#how-conditions-use-expressions))

So this is the expression:

``` json
        "and": [
          {
            "greater": [
              "@{ticks(items('Loop')['creation_time'])}",
              "@{ticks(getPastTime(1, 'Minute'))}"
            ]
          }
        ]
```
Not exactly intuitive, right? And maybe I did a tiny bug in this expression...
So insert it in our condition inside the loop and debug!

??? warning "You tried very hard, I know..."
    ``` json
    {
      "type": "Foreach",
      "actions": {
        "Our_filter": {
          "actions": {},
          "expression": {
            "and": [
              {
                "greater": [
                  "@{ticks(items('Process Prediction Overrides')['creation_time'])}",
                  "@{ticks(getPastTime(1, 'Minute'))}"
                ]
              }
            ]
          },
          "runAfter": {},
          "type": "If"
        }
      },
      "foreach": "@body('Get Prediction Overrides')",
      "runAfter": {
        "Get Prediction Overrides": [
          "Succeeded"
        ]
      }
    }
    ``` 


<figure markdown>
  ![Image title](images/WorkflowEditor_foreach_runstats_if.png){: align=left }
  <figcaption>In the runstats you should now see the expression result.</figcaption>
</figure>

!!! info "Can you tweak the getPastTime in a way that the expression turns to true?"
    Might help for debugging...

So we are nearly there!

All we need is a Http if the expression is true to create an exception in the Exception Service.

!!! hint
    The API spec for the exception service can be found in the API Catalog as "LUI Services - Exception Service APIs v1"
    But for your convenience I post it here:
    ```
    curl -X 'POST' \
      'https://api.jdadelivers.com/lui/exception/v1/exceptions' \
      -H 'accept: application/json' \
      -H 'Content-Type: application/json' \
      -H 'Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImtpZCI6IlJHYmJjdzhTS05QUEtHRTI0QjZBUjRKZEJvOXJhdTREMHZOeG5OWUxSSzgifQ.eyJpc3MiOiJodHRwczovL2JsdWV5b25kZXJ1cy5iMmNsb2dpbi5jb20vZmEzZTBiNmUtYjJjYy00Njc4LWI4MmItNzIwZWIxOWU3ZDI4L3YyLjAvIiwiZXhwIjoxNjUyOTg1MzQ0LCJuYmYiOjE2NTI5ODE3NDQsImF1ZCI6IjY5YWRiMDRkLTY1OGYtNGI4Ni1hNjU5LTY3ZmUwZjIzYmQxZiIsImVtYWlsIjoiU2ViYXN0aWFuLk5ldWJhdWVyQGJsdWV5b25kZXIuY29tIiwiZ2l2ZW5fbmFtZSI6IlNlYmFzdGlhbiIsImZhbWlseV9uYW1lIjoiTmV1YmF1ZXIiLCJuYW1lIjoiU2ViYXN0aWFuIE5ldWJhdWVyIiwiaWRwX3ByZWZlcnJlZF91c2VybmFtZSI6IjEwMjQwNDdAamRhZGVsaXZlcnMuY29tIiwiaWRwIjoiYzgyNTRhMmEtOTEzYi00Y2Q2LTliNjYtMzM5ZGIzM2RhOGExIiwiYnlfcmVhbG0iOiJieS1kZXZlbG9wZXIiLCJieV9yZWFsbV9pZCI6IjdkYmVhNWJiLTI4YjUtNDA1OS1iZjJhLTViNWJmNmRhZTEwNyIsInRpZCI6ImM5NTM2Njg1LTEyNjItNDIwYi1hOTRiLTRmNTQyMDI4NDY4MCIsInN1YiI6ImMxMGZmYWZmLTcwNjgtNGIzNi1hZWEwLTY4ZWExNzY3NmIwMSIsIm9pZCI6ImMxMGZmYWZmLTcwNjgtNGIzNi1hZWEwLTY4ZWExNzY3NmIwMSIsImJ5X3Njb3BlcyI6WyJieS5pYW0uYWxsIiwiYnkucG9ydGFsLmFsbCIsImJ5LmRwLmFsbCIsImJ5LmxkZS5kZW1hbmRwcmVkaWN0aW9ucyIsImJ5LmxkZS51aSIsImJ5LmxkZS5hbGwiLCJieS5sZXh0LmFsbSJdLCJub25jZSI6IjczMjQzNzQ0LTE4MWUtNDMzOS1iYzJiLWNlZGZjNDY0ZWFmYSIsInNjcCI6ImFjY2VzcyIsImF6cCI6IjFlYzM2YjVlLWY1MzYtNDkzZi1hMDgxLWMyZTk2MjEzZTQ4NyIsInZlciI6IjEuMCIsImlhdCI6MTY1Mjk4MTc0NH0.c-mKcVoqlapCUoEd3nRom5TOLo_vsv0m9VS7W0Ifon7TJrWMw-HcWWngaPxB1rw9Ay36gay_z-YvaDCe9hfsqn2_hbcR5Zu5lvjWj7lD3fXX5Twl-Fo8XqtQbDpomi5QzFjJ_UJmBr1GN6xkEth_eqjdU4hY8BFqX-IYpBHMEBok5spkKBppE7Zf7OD3js2HH7qoEaKoULisD4wRCkNP2lCfZeohpRE1mDUNyo5annp095YRBftN98eWHWjmrYsCXQyDq_oF2hse8_y3tbBmY_Pa0vpa2cBE9heoVyCiH76eMRsGXoeGtZuLaUCNIBptt2lxSUmT9gOI19xFksSw6w' \
      -d '[
      {
        "referenceId": "string",
        "referenceType": "string",
        "source": "string",
        "title": "string",
        "subTitle": "string",
        "description": "string",
        "type": "string",
        "resources": [
          "string"
        ],
        "associatedData": {},
        "extFields": "{}",
        "status": "OPEN",
        "severity": 0,
        "navLink": "string"
      }
    ]'
    ```

Can you do that? 
As there are many people create the same exception, maybe it would be a good idea to include your name in the exception title.

Also, we want to have some information about the newly created prediction override. These sound interestesting: 

* `'min_affected_date'`
* `'creation_time'`
* `'creation_user'`

Do you remember how to embed data from items in strings? Would be handy...

Did you manage to add a working HTTP post request?


??? warning "Don't worry, the solution here..."
    Oh, that got quite long...
    ``` json
    {
      "type": "Foreach",
      "actions": {
        "Our_filter": {
          "actions": {
            "HTTP": {
              "inputs": {
                "body": [
                  {
                    "associatedData": {},
                    "description": "Active from: @{items('Process Prediction Overrides')['min_affected_date']}, created: @{items('Process Prediction Overrides')['creation_time']} by @{items('Process Prediction Overrides')['creation_user']}",
                    "extFields": "{}",
                    "financialImpact": 0,
                    "navLink": "navLink",
                    "referenceId": "NA",
                    "referenceType": "LDE Prediction Overrides",
                    "resources": [
                      "resources"
                    ],
                    "severity": 0,
                    "source": "DEVCON",
                    "subTitle": "NA",
                    "title": "<Your name> Unprocessed Prediction Override",
                    "type": "Unprocessed Prediction Override",
                    "unitImpact": 0
                  }
                ],
                "headers": {
                  "Authorization": "Bearer @{body('Get LIAM Token')?['access_token']}",
                  "Content-Type": "application/json",
                  "accept": "application/json"
                },
                "method": "POST",
                "uri": "https://api.jdadelivers.com/lui/exception/v1/exceptions"
              },
              "runAfter": {},
              "type": "Http"
            }
          },
          "expression": {
            "and": [
              {
                "greater": [
                  "@{ticks(items('Process Prediction Overrides')['creation_time'])}",
                  "@{ticks(getPastTime(1, 'Minute'))}"
                ]
              }
            ]
          },
          "runAfter": {},
          "type": "If"
        }
      },
      "foreach": "@body('Get Prediction Overrides')",
      "runAfter": {
        "Get Prediction Overrides": [
          "Succeeded"
        ]
      }
    }
    ```

Now, a very last thing is missing, we still trigger the workflow manually...but we wanted to run the workflow
based on a trigger every minute.

If you have a look at the global pro-code spec, you will find this section called "triggers"

``` json 
    "triggers": {
      "request": {
        "inputs": {
          "schema": {}
        },
        "kind": "Http",
        "type": "Request"
      }
    }
```

all we need to do is replace this with a so called recurrent trigger:

``` json 
    "triggers": {
      "Recurrence": {
        "evaluatedRecurrence": {
          "frequency": "Minute",
          "interval": 1
        },
        "recurrence": {
          "frequency": "Minute",
          "interval": 1
        },
        "type": "Recurrence"
      }
    }
```

Woohoo, that's it...let's test it and I will create some new prediction overrides...

??? warning "Screwed something up on the way...hey, that's software engineering...just copy ;)"
    ``` json
    
      "definition": {
        "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
        "actions": {
          "Get LIAM Token": {
            "type": "Http",
            "inputs": {
              "uri": "https://login.microsoftonline.com/blueyonderus.onmicrosoft.com/oauth2/v2.0/token",
              "method": "POST",
              "headers": {
                "Content-Type": "application/x-www-form-urlencoded"
              },
              "body": "client_id=@{parameters('client_id')}&client_secret=@{parameters('client_secret')}&grant_type=client_credentials&scope=@{parameters('scope')}"
            },
            "runAfter": {}
          },
          "Get Prediction Overrides": {
            "type": "Http",
            "inputs": {
              "uri": "https://api.jdadelivers.com/lde/prediction-overrides/v1/overrides?minAffectedDate=2022-05-15",
              "method": "GET",
              "headers": {
                "accept": "application/json",
                "Authorization": "Bearer @{body('Get LIAM Token')?['access_token']}"
              }
            },
            "runAfter": {
              "Get LIAM Token": [
                "Succeeded"
              ]
            }
          },
          "Process Prediction Overrides": {
            "type": "Foreach",
            "actions": {
              "Our_filter": {
                "actions": {
                  "HTTP": {
                    "inputs": {
                      "body": [
                        {
                          "associatedData": {},
                          "description": "Active from: @{items('Process Prediction Overrides')['min_affected_date']}, created: @{items('Process Prediction Overrides')['creation_time']} by @{items('Process Prediction Overrides')['creation_user']}",
                          "extFields": "{}",
                          "financialImpact": 0,
                          "navLink": "navLink",
                          "referenceId": "NA",
                          "referenceType": "LDE Prediction Overrides",
                          "resources": [
                            "resources"
                          ],
                          "severity": 0,
                          "source": "DEVCON",
                          "subTitle": "NA",
                          "title": "<Your name> Unprocessed Prediction Override",
                          "type": "Unprocessed Prediction Override",
                          "unitImpact": 0
                        }
                      ],
                      "headers": {
                        "Authorization": "Bearer @{body('Get LIAM Token')?['access_token']}",
                        "Content-Type": "application/json",
                        "accept": "application/json"
                      },
                      "method": "POST",
                      "uri": "https://api.jdadelivers.com/lui/exception/v1/exceptions"
                    },
                    "runAfter": {},
                    "type": "Http"
                  }
                },
                "expression": {
                  "and": [
                    {
                      "greater": [
                        "@{ticks(items('Process Prediction Overrides')['creation_time'])}",
                        "@{ticks(getPastTime(1, 'Minute'))}"
                      ]
                    }
                  ]
                },
                "runAfter": {},
                "type": "If"
              }
            },
            "foreach": "@body('Get Prediction Overrides')",
            "runAfter": {
              "Get Prediction Overrides": [
                "Succeeded"
              ]
            }
          }
        },
        "contentVersion": "2021.1.0.0",
        "outputs": {},
        "parameters": {
          "client_id": {
            "defaultValue": "",
            "type": "String"
          },
          "client_secret": {
            "defaultValue": "",
            "type": "String"
          },
          "scope": {
            "defaultValue": "",
            "type": "String"
          }
        },
        "triggers": {
          "Recurrence": {
            "evaluatedRecurrence": {
              "frequency": "Minute",
              "interval": 1
            },
            "recurrence": {
              "frequency": "Minute",
              "interval": 1
            },
            "type": "Recurrence"
          }
        }
      },
      "parameters": {
        "client_id": {
          "value": "----myid-----"
        },
        "client_secret": {
          "value": "-----my secret------"
        },
        "scope": {
          "value": "https://blueyonderus.onmicrosoft.com/69adb04d-658f-4b86-a659-67fe0f23bd1f/.default"
        }
      }
    }
    ```