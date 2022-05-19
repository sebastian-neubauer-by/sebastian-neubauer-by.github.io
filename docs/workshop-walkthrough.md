password: DevCon22

# The developer Portal

## Login
First we login into the developer portal:

[https://bylumuiportalplpna.azureedge.net/home/](https://bylumuiportalplpna.azureedge.net/home/){:target="_blank"}

You need to enter the realm, which is `by-developer` for the developer portal.

Then login with your account credentials.

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

By exectuing this GET request we will get a list of all prediction overrides stored in the LDE instance.

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

We can create m2m clients in a self service in the Identity & Access application available in the app gallery.

<figure markdown>
  ![Image title](images/devportal_identityaccess.png){: align=left }
  <figcaption>Access the Identity & Access application</figcaption>
</figure>

In the Identiy & Access application change to the "Machines" tab and with the "+" icon we can create a new m2m client 
for our workflow.

<figure markdown>
  ![Image title](images/create_m2mclient.png){: align=left }
  <figcaption>Create a new m2m client using the "+" icon in the "Machines" tab</figcaption>
</figure>

In the pop-up give the m2m client a good name (that's hard, I know). As this is a shared environment, probably it is a 
good idea to append your name, so why not `devcon-workshop-<yourname>`.

<figure markdown>
  ![Image title](images/m2mclient_addsecret_marked.png){: align=left }
  <figcaption>Note the Client ID and add a secret</figcaption>
</figure>

!!! note
    Don't forget to note the client secret and store it somewhere safe!
    (A password manager would be a good option)

<figure markdown>
  ![Image title](images/m2mclient_copysecret_marked.png){: align=left }
  <figcaption>Copy the secret and store it somehwere safe</figcaption>
</figure>

Now we need to add the role to be able to access the LDE oeverrides API.
This role is called `lde-full-access`, just start to type and select in the picker.


<figure markdown>
  ![Image title](images/m2mclient_addrole_marked.png){: align=left }
  <figcaption>Add the `lde-full-access` role</figcaption>
</figure>

That's it, we created a m2m client. Easy, right?

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
    The Application Lifecycle Management service (ALM) let's you create and maintain the lifecycle of applications you 
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
  <figcaption>As every participant created their own module, let's include our name in the module name, why not YOURNAME-workshop</figcaption>
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
  <figcaption>As your module is brandnew it should be quite empty, so let's click on "Add resource" and select "Workflow" as the resource type</figcaption>
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

show low-code-pro code navigation.

## The Save, Publish, Run Cycle

show how to save publisch run with pics from get liam.


!!! info "How to use results from previous actions"
    If you want to use results from previous actions, you can access them with `body('NAME OF THE ACTION')`, as everything
    in Workflow is json, you can easily access values from the json body using `body('NAME OF THE ACTION')?['KEY']` and 
    finally, if we want to actually return the value (as a string), we use the `@` operator.  
    So, this should return the key 'KEY' of the result of the 'NAME OF THE ACTION' action as as tring which can be directly embedded in another string:

    `"Hello @{body('NAME OF THE ACTION')?['KEY']}"`

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
Now, if we go back to the LDE Predictions Overrides API documentation in the 
API Catalog we saw that the request to get a list of predictions looks like this.

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

If jou came that far, just "Save, Publish, Run" and check the latest run if it succeded. 

It succeeded? Great! But there are a lot of prediction overrides right, and they are all outdated which is quite 
inconvenient because they clutter the results.

But wasn't there a parameter in the api documentation to restrict the result to only prediction overrides created after 
some date?

Can you figure that out and restrict the results to only overrides from some days back?

"Save, Publish, Run" and check the result! How many overrides do you have now in the response?

??? warning "Look behind you, a Three-Headed Monkey!"
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


