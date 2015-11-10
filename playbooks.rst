*********
Playbooks
*********

Playbooks are used in the Interaction Lab to provide pre-recorded sets of inputs, which can be played back and used for robots to test against.

The .playbook File Format
=========================

The playbook file format is used to store created playbooks, and is also used internally as the generator for test data. Playbooks are a JSON string, and are permitted to be stored either in plain text or gzipped. This file format is purposely made to be modular and unopinionated, with the goal that future changes should be made with as few backwards-incompatible changes as possible.

The following are top-level properties of the playbook:

- ``version`` defines the version the playbook file is in. Currently, the only valid version is 1, but this may be incremented if backwards incompatible changes are made.
- ``stage`` defines several common properties of the workspace, including duration and background information.
- ``history`` is an optional property that contains undo/redo information for the playbook.
- ``layers`` layers are the meat of the playbook, defining actions that our robotic users take.

Stage
-----

The stage records the duration of the playbook (given in milliseconds), aspect ratio, as well as the last playback time, and information about the background. The background is a pluggable module. Currently, the following modules are available:

- ``image`` displays a static image as the background of the stage, stretched to cover the stage. It contains the URL of the image, as well as the X, Y, Width, and Height, all given as percentages (where 0.0 is 0% and 1.0 is 1%; there are no bounds).

.. code-block:: js

    {
        duration: 30000,
        lastTime: 15200,
        ratio: [16, 9],
        background: {
            type: 'image',
            data: {
                url: 'http://i.imgur.com/PDiajRI.jpg',
                width: 1.0,
                height: 1.0,
                x: 0,
                y: 0
            }
        }
    }

History
-------

The ``history`` is an optional property that contains the point in the history that the user was last on, as well as JSON patch objects that record mutations to the playbook. (One of the wonderful things about storing playbooks as JSON, is that history is made simple!)

The list of actions is an array of JSON patches. Each patch is the *complement* to the action that the user took. That is, to undo an action, that individual patch should be applied. To redo an action, the complement to the patch should be applied. The first element of the array corresponds to the action most recently taken by the user; the last element is the action taken furthest in the past.

.. code-block:: js

    {
        point: 1,
        actions: [
            [
                { "op": "replace", "path": "/stage/duration", "value": 2.3 },
                { "op": "add", "path": "/hello", "value": ["world"] }
            ],
            [
                { "op": "remove", "path": "/foo" }
            ]
        ]
    }

Layers
------

Layers are the meat of the playbook, defining actions that our robotic users take. You can visualize a layer as being similar to what you might find in a video editing programme, such as Adobe After Effects. Layers an ordered array -- their order, while not (currently) in use in Tetris, is the way that they are displayed in the Interactive Lab editor.

Layers provide their data through modules. Additionally, they define a "count" of users, which are how many instances of the layer are run when the robot connects. The available modules are:

- ``inputlist`` module is a simple module that simply contains an array of :ref:`Report <report-packet>` structures, and a specific time offset (in milliseconds) these should be sent at, relative to the layer start time.

.. code-block:: js

    [{
        start: 3200,
        name: 'Layer 0',
        color: '#f00',
        active: true,
        count: 1,
        data: {
            type: 'inputlist',
            repots: [
                { offset: 0, report: { /* ... */ }},
                { offset: 50, report: { /* ... */ }},
                { offset: 100, report: { /* ... */ }},
                // ...
            ]
        }
    }]

A Typed Playbook Structure
--------------------------

Below is the complete playbook structure, displayed as it might be represented in the Go programming language. Go's type system is a bit finicky, so we represent modules as a generic ``Module`` type, with an associated structure for their ``Data`` shown below.

.. code-block:: go

    type Module struct {
        type string
        data json.RawMessage // structure of the module varies
    }

    type Playbook struct {
        Version uint
        Stage struct {
            Duration   uint
            LastTime   uint
            Ratio      []float64
            Background Module
        }
        History struct {
            Position uint
            History  [][]interface{}
            // for structure of history items, see:
            // https://tools.ietf.org/html/rfc6902
        }
        Layers []struct {
            Start  uint
            Name   string
            Color  string
            Active bool
            Count  uint
            Data   Module
        }
    }

    // module type "background", used in Playbook.Stage.Background
    type BackgroundModule {
        Url    string
        X      float64
        Y      float64
        Width  float64
        Height float64
    }

    // module type "inputlist", used in Playbook.Layers.Module
    type RecordedInputModule {
        // see the "Talking to Tetris" section of this documentation
        // for the structure of reports.
        Reports []struct {
            Offset uint
            Report Report
        }
    }
