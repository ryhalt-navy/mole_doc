# **Configuration**

## **Event Generator Pulsar Function**

There is a Pulsar function that aims to implement a Python based event generator, contained in `functions/_simple_event_gen.py`. This function is aided by a python application (`message_cacher.py`) that caches relevant incoming Pulsar messages into a Redis cache with timestamped storage and easy retrieval for the purposes of trigger evaluation. This python application will reside in its own separate Docker container. 

Note: The file that contains the Pulsar function is prepended by an underscore to stop the pulsar function auto-discovery in `create_functions.py` from creating the function. `message_cacher.py` will create this function once it can retrieve the input topics from the Django server. 


### **Trigger Setup**

To create a `Trigger`, you need to have some pre-requisites models first, starting with the `ConditionVariable`. The `ConditionVariable` is used during the `Trigger` condition evaluation, where it acts as a placeholder. 
#### **ConditionVariable**
=== "Django ORM"

    ``` python
    import data_collection.models as dcm
    my_condition_var = dcm.ConditionVariable.objects.create(
        name="node_state_var",
        description="node state (e.g. unconfigured / active)",
        variable="node_state_var : /node/status.state",
    )
    ```
=== "Factory Boy"

    ``` python
    from data_collection.factories import factories
    my_condition_var = factories.ConditionVariableFactory(
        name="node_state_var",
        description="node state (e.g. unconfigured / active)",
        variable="node_state_var : /node/status.state",
    )
    ```
The `variable` field should be in a similar format to the example: `condition_variable_name : /topic/name.field_name`. 

`condition_variable_name` is the variable name used in the `Trigger` evaluation. 

The topic name will be converted to underscores (i.e. `/topic/name` -> `_topic_name`) and used as part of a Pulsar topic name. The Pulsar tenant and namespace cannot be changed and are set to `public` and `default` respectively. The resulting full Pulsar topic in this example would be `persistent://public/default/_node_status`.

`field_name` signifies which key to use when replacing the `condition_variable_name`.

When a message is received, the message is expected to have key-value pairs, one of which should have a key indicated by the `field_name` part of the `variable` field. `condition_variable_name` is replaced with the value of that key-value pair and the condition on the `Trigger` is evaluated. 

The example in this case will use the value from a message on Pulsar topic `persistent://public/default/_node_status` with a key of `state` and replace it wherever `node_state_var` appears in the `Trigger`'s condition.


#### **RequestedData**
`RequestedData` is any other data that you wish to post to some API endpoint. 
=== "Django ORM"

    ``` python
    import data_collection.models as dcm
    my_requested_data = dcm.RequestedData.objects.create(
        name="Node Subject",
        description="Get subject entity when node registers",
        destination_url="$EVENT$",
        payload={
            "subject_entity": "/node/status.entity_name",
            "pose_source": "[http://localhost:8000/api/pose_sources/1/]",
            "cached_timestamp": "$TIME$",
        },
        },
    )
    ```
=== "Factory Boy"

    ``` python
    from data_collection.factories import factories
    my_requested_data = factories.RequestedDataFactory(
        name="Node Subject",
        description="Get subject entity when node registers",
        destination_url="$EVENT$",
        payload={
            "subject_entity": "/node/status.entity_name",
            "pose_source": "[http://localhost:8000/api/pose_sources/1/]",
            "cached_timestamp": "$TIME$",
        },
    )
    ```
`destination_url` indicates where the data will be posted to. `$EVENT$` is a special string that indicates that the newly created event should be the destination. Since the event url won't be known prior to creating it, the string acts as a workaround to attach relevant data to the event. Another possible destination is the poses API endpoint which will create a pose.

`payload` is a Python dictionary where the values are taken from message fields, similar to the `ConditionVariable`'s `variable` field. If the value is enclosed in square brackets, that static data will be posted. If the value is $EVENT$ or $TIME$, it will be replaced with the associated event or a timezone-aware datetime of the current time respectively. In this example, the `subject_entity`'s value is the `entity_name` field on a message from the Pulsar topic `persistent://public/default/_node_status`. The value of `pose_source` would be the literal string `"http://localhost:8000/api/pose_sources/1/"` regardless of the message contents. The value of `cached_timestamp` would be the time that the message is cached into Redis by `message_cacher.py`.


#### **Trigger**
With these pieces in place, you can create a `Trigger` now.
=== "Django ORM"

    ``` python
    import data_collection.models as dcm
    dcm.Trigger.objects.create(
        name="Node Online Trigger",
        key="node_online",
        description="Create event when a node comes online.",
        is_active=True,
        creates_event=True,
        condition='node_state_var == "online"',
        condition_variables=[my_condition_var],
        requested_dataset=[my_requested_data],
        event_type=node_online_event_type,
    )
    ```
=== "Factory Boy"

    ``` python
    from data_collection.factories import factories
    factories.TriggerFactory(
        name="Node Online Trigger",
        key="node_online",
        description="Create event when a node comes online.",
        is_active=True,
        creates_event=True,
        condition='node_state_var == "online"',
        condition_variables=[my_condition_var],
        requested_dataset=[my_requested_data],
        event_type=node_online_event_type,
    )
    ```
`key` is a unique string that identifies the `Trigger`. 

`is_active` is a flag that will tell the event generator whether or not to monitor the `Trigger` condition. The flag is currently only read on start-up. It won't update while the event generator is running.

`creates_event` is a flag that indicates whether or not the `Trigger` evaluation creates an event.

`condition` is a string that represents the Python-like expression that is evaluated as the condition for the `Trigger`.

`condition_variables` is a list of `ConditionVariables` that is used in the `condition`. It is an error if the `condition` contains any variables that doesn't have a corresponding `condition_variable`.

`requested_dataset` is a list of `RequestedData` that will be attached to their respective destinations if this `Trigger` evaluates True.

`event_type` is the default event type for events created by the `Trigger`. The event type can be overridden if it is present in the message for the `Trigger`. This is not used if  `creates_event` is False.




### **E2E Testing**  

You can run the following command to run end-to-end tests for the event generator.

    docker-compose -f docker-compose-e2e.yml up --abort-on-container-exit --build -V

This will spin up the containers for the event generator, run e2e tests, then stop the containers.

## **Map / Timeline Marker Styles**

This section describes marker style definition and associating these styles with event_types and entity_types.  These marker styles are used to represent event types and entity types on the map and in the event timeline.  The icons are typically taken from the FontAwesome set or from locally-served .svg icons.

### **Marker Styles**

Markers are styled using the PointStyle model in Mole.  It has the following fields:

* **name:** Name of this point style
* **description:** Description of this point style
* **icon:** String representing the icon to be used.  If a FontAwesome icon is to be used, this string should be something like `fa-car`.  If a locally served .svg is to be used, the path to the icon (e.g., `/static/app/mapview/markers/gas-01.svg`)
* **color:** CSS color code to be applied to the icon (e.g., `#FFFFFF` for white)
* **use_marker_pin:** Boolean field indicating wheather this icon should be shown in a marker pin on the map.  Note this has no effect on the event timeline.
* **marker_color:** CSS color code to be applied to the map marker pin icon (e.g., `#FFFFFF` for white)
* **scale_factor:** An optional parameter to adjust sizing of the icon.
* **animation:** An optional animation name to draw attention to the point when plotted (e.g., `invert-toggle-3` to give the point a flashing effect for 3 seconds.

### **Marker Compositing**

Point style is an element (foreign key) of the EventType and EntityType models.  In order to determine how a particular event marker is to be styled, the following method is used:

* The point style associated with the event's event_type is used as a starting point.
* The metadata_style_fields (on event_type) is used to override this style.  It represents a list of fields (keys) within the event's metadata whose values represent the name of an entity.  
* For each entity in this list, the entity's entity_type point_style replaces (for any non-null values) the associated event's event_type point_style in order.  
* In this way, fields occuring at the end of the metadata_style_fields have precedence over earlier occuring fields.  
* Note: if only a single point_style element is desired to be replaced by the entity_type style (e.g. the icon string), all other fields within its point_style may be null.
* The resulting point_style is returned in the event's point_style field.

## **Bulk POSTs**

Mole also has the ability to create multiple instances for a single HTTP POST request. Currently this ability only exists for the `api/events/` and the `api/poses/` endpoints. To use this ability, simply pass a list of objects rather than a single object

Single instance

    POST /api/events
    {
        "start_datetime": <ISO_datetime>,
        "start_pose": <pose_url>,
        "event_type": <event_type_url>,
        "metadata": <metadata>,
    }

Multiple instances

    POST /api/events
    [
        {
            "start_datetime": <ISO_datetime>,
            "start_pose": <pose_url>,
            "event_type": <event_type_url>,
            "metadata": <metadata>,
        },
        {
            "start_datetime": <ISO_datetime>,
            "start_pose": <pose_url>,
            "event_type": <event_type_url>,
            "metadata": <metadata>,
        },
        {
            "start_datetime": <ISO_datetime>,
            "start_pose": <pose_url>,
            "event_type": <event_type_url>,
            "metadata": <metadata>,
        }
    ]


## **Access Logs**

An access log for django can be found at `mole/_logs/access.log`. This will contain the remote address, username, date of request, where the request is for, HTTP status, response length, and referer.
