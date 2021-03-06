---
id: ch08-Forms
title: Forms
---

Flowable provides a convenient and flexible way to add forms for the manual steps of your business processes.
We support two strategies to work with forms: Built-in form rendering with a form definition (created with the form designer) and external form rendering.
For the external form rendering strategy form properties can be used (that was supported in the Explorer web application in version 5), or a form key definition that points to an external form reference that can be resolved with custom coding.

## Form definition

Full information about the form definitions and Flowable form engine can be found in the [Form Engine user guide](http://www.flowable.org/docs/userguide-form/).
Form definitions can be created with the Flowable Form Designer that’s part of the Flowable Modeler web application, or created by hand with a JSON editor.
The Form Engine user guide describes the structure of the form definition JSON in full length. The following form field types are supported:

-   Text: rendered as a text field

-   Multiline text: rendered as a text area field

-   Number: rendered as a text field, but only allows numeric values

-   Checkbox: rendered as a checkbox field

-   Date: rendered as a date field

-   Dropdown: rendered as a select field with the option values configured in the field definition

-   Radio buttons: rendered as a radio field with the option values configured in the field definition

-   People: rendered as a select field where a person from the Identity user table can be selected

-   Group of people: rendered as a select field where a group from the Identity group table can be selected

-   Upload: rendered as an upload field

-   Expression: rendered as a label and allows you to use JUEL expressions to use variables and/or other dynamic values in the label text

The Flowable task application is able to render an html form from the form definition JSON.
You can also use the Flowable API to get the form definition JSON yourself.

    FormModel RuntimeService.getStartFormModel(String processDefinitionId, String processInstanceId)

or

    FormModel TaskService.getTaskFormModel(String taskId)

The FormModel object is a Java object representation of the form definition JSON.

To start a process instance with a start form definition you can use the following API call:

    ProcessInstance RuntimeService.startProcessInstanceWithForm(String processDefinitionId, String outcome,
        Map<String, Object> variables, String processInstanceName)

When a form definition is defined on (one of) the start event(s) of a process definition, this method can be used to start a process instance with the values filled-in in the start form.
The Flowable task application uses this method to start a process instance with a form as well.
All form values need to be passed in the variables map and an optional form outcome string and process instance name can be provided.

In a similar way, a user task can be completed with a form using the following API call:

    void TaskService.completeTaskWithForm(String taskId, String formDefinitionId,
        String outcome, Map<String, Object> variables)

Again, for more information about form definitions have a look at the [Form Engine user guide](http://www.flowable.org/docs/userguide-form).

## Form properties

All information relevant to a business process is either included in the process variables themselves or referenced through the process variables. Flowable supports complex Java objects to be stored as process variables like Serializable objects, JPA entities or whole XML documents as Strings.

Starting a process and completing user tasks is where people are involved into a process. Communicating with people requires forms to be rendered in some UI technology. In order to facilitate multiple UI technologies easy, the process definition can include the logic of transforming of the complex Java typed objects in the process variables to a Map&lt;String,String&gt; of **'properties'**.

Any UI technology can then build a form on top of those properties, using the Flowable API methods that expose the property information. The properties can provide a dedicated (and more limited) view on the process variables. The properties needed to display a form are available in the **FormData** return values of for example

    StartFormData FormService.getStartFormData(String processDefinitionId)

or

    TaskFormdata FormService.getTaskFormData(String taskId)

By default, the built-in form engine 'sees' the properties as well as the process variables. So there is no need to declare task form properties if they match 1-1 with the process variables. For example, with the following declaration:

    <startEvent id="start" />

All process variables are available when execution arrives in the startEvent, but

    formService.getStartFormData(String processDefinitionId).getFormProperties()

will be empty since no specific mapping was defined.

In the above case, all the submitted properties will be stored as process variables. This means that by simply adding a new input field in the form, a new variable can be stored.

Properties are derived from process variables, but they don’t have to be stored as process variables. For example, a process variable could be a JPA entity of class Address. And a form property StreetName used by the UI technology could be linked with an expression \#{address.street}

Analogue, the properties that a user is supposed to submit in a form can be stored as a process variable or as a nested property in one of the process variables with a UEL value expression like e.g. \#{address.street} .

Analogue the default behavior of properties that are submitted is that they will be stored as process variables unless a formProperty declaration specifies otherwise.

Also type conversions can be applied as part of the processing between form properties and process variables.

For example:

    <userTask id="task">
      <extensionElements>
        <flowable:formProperty id="room" />
        <flowable:formProperty id="duration" type="long"/>
        <flowable:formProperty id="speaker" variable="SpeakerName" writable="false" />
        <flowable:formProperty id="street" expression="#{address.street}" required="true" />
      </extensionElements>
    </userTask>

-   Form property room will be mapped to process variable room as a String

-   Form property duration will be mapped to process variable duration as a java.lang.Long

-   Form property speaker will be mapped to process variable SpeakerName. It will only be available in the TaskFormData object. If property speaker is submitted, an FlowableException will be thrown. Analogue, with attribute readable="false", a property can be excluded from the FormData, but still be processed in the submit.

-   Form property street will be mapped to Java bean property street in process variable address as a String. And required="true" will throw an exception during the submit if the property is not provided.

It’s also possible to provide type metadata as part of the FormData that is returned from methods StartFormData FormService.getStartFormData(String processDefinitionId) and TaskFormdata FormService.getTaskFormData(String taskId)

We support the following form property types:

-   string (org.flowable.engine.impl.form.StringFormType

-   long (org.flowable.engine.impl.form.LongFormType)

-   double (org.flowable.engine.impl.form.DoubleFormType)

-   enum (org.flowable.engine.impl.form.EnumFormType)

-   date (org.flowable.engine.impl.form.DateFormType)

-   boolean (org.flowable.engine.impl.form.BooleanFormType)

For each form property declared, the following FormProperty information will be made available through List&lt;FormProperty&gt; formService.getStartFormData(String processDefinitionId).getFormProperties() and List&lt;FormProperty&gt; formService.getTaskFormData(String taskId).getFormProperties()

    public interface FormProperty {
      /** the key used to submit the property in {@link FormService#submitStartFormData(String, java.util.Map)}
       * or {@link FormService#submitTaskFormData(String, java.util.Map)} */
      String getId();
      /** the display label */
      String getName();
      /** one of the types defined in this interface like e.g. {@link #TYPE_STRING} */
      FormType getType();
      /** optional value that should be used to display in this property */
      String getValue();
      /** is this property read to be displayed in the form and made accessible with the methods
       * {@link FormService#getStartFormData(String)} and {@link FormService#getTaskFormData(String)}. */
      boolean isReadable();
      /** is this property expected when a user submits the form? */
      boolean isWritable();
      /** is this property a required input field */
      boolean isRequired();
    }

For example:

    <startEvent id="start">
      <extensionElements>
        <flowable:formProperty id="speaker"
          name="Speaker"
          variable="SpeakerName"
          type="string" />

        <flowable:formProperty id="start"
          type="date"
          datePattern="dd-MMM-yyyy" />

        <flowable:formProperty id="direction" type="enum">
          <flowable:value id="left" name="Go Left" />
          <flowable:value id="right" name="Go Right" />
          <flowable:value id="up" name="Go Up" />
          <flowable:value id="down" name="Go Down" />
        </flowable:formProperty>

      </extensionElements>
    </startEvent>

All that information is accessible through the API. The type names can be obtained with formProperty.getType().getName(). And even the date pattern is available with formProperty.getType().getInformation("datePattern") and the enumeration values are accessible with formProperty.getType().getInformation("values")

The following XML snippet

    <startEvent>
      <extensionElements>
        <flowable:formProperty id="numberOfDays" name="Number of days" value="${numberOfDays}" type="long" required="true"/>
        <flowable:formProperty id="startDate" name="First day of holiday (dd-MM-yyy)" value="${startDate}" datePattern="dd-MM-yyyy hh:mm" type="date" required="true" />
        <flowable:formProperty id="vacationMotivation" name="Motivation" value="${vacationMotivation}" type="string" />
      </extensionElements>
    </userTask>

could be used to render to a process start form in a custom app.

## External form rendering

The API also allows for you to perform your own task form rendering outside of the Flowable Engine. These steps explain the hooks that you can use to render your task forms yourself.

Essentially, all the data that’s needed to render a form is assembled in one of these two service methods: StartFormData FormService.getStartFormData(String processDefinitionId) and TaskFormdata FormService.getTaskFormData(String taskId).

Submitting form properties can be done with ProcessInstance FormService.submitStartFormData(String processDefinitionId, Map&lt;String,String&gt; properties) and void FormService.submitTaskFormData(String taskId, Map&lt;String,String&gt; properties)

To learn about how form properties map to process variables, see [Form properties](bpmn/ch08-Forms.md#form-properties)

You can place any form template resource inside the business archives that you deploy (in case you want to store them versioned with the process). It will be available as a resource in the deployment, which you can retrieve using: String ProcessDefinition.getDeploymentId() and InputStream RepositoryService.getResourceAsStream(String deploymentId, String resourceName); This could be your template definition file, which you can use to render/show the form in your own application.

You can use this capability of accessing the deployment resources beyond task forms for any other purposes as well.

The attribute &lt;userTask flowable:formKey="..." is exposed by the API through String FormService.getStartFormData(String processDefinitionId).getFormKey() and String FormService.getTaskFormData(String taskId).getFormKey(). You could use this to store the full name of the template within your deployment (e.g. org/flowable/example/form/my-custom-form.xml), but this is not required at all. For instance, you could also store a generic key in the form attribute and apply an algorithm or transformation to get to the actual template that needs to be used. This might be handy when you want to render different forms for different UI technologies like e.g. one form for usage in a web app of normal screen size, one form for mobile phone’s small screens and maybe even a template for an IM form or an email form.
