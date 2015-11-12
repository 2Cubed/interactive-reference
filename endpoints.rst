Interactive API Endpoints
=========================

GET /api/v1/tetris/:channel/robot
---------------------------------

Retrieves connection info for an interactive channel.
This endpoint is meant for robots (interactive games) to connect.
It will return an authentication key and an endpoint to connect
to tetrisd.::

@param channel - The channel you want to connect to.

The following object will be returned.

.. code-block:: json

    {
        address: "tetrominos.beam.pro:8021",
        key: "u2psogg2hsphg5oi"
    }

This will return an address for the robot to connect to. If
this is `null` the channel might not be set up correctly, or
there's no server available for connecting.

GET /api/v1/tetris/:channel
---------------------------

Retrieves connection info for an interactive channel.
This endpoint is meant for interactive clients (not robots)
to connect.
It will return an authentication key and an endpoint to connect
to tetrisd.::

@param channel - The channel you want to connect to.

The following object will be returned.

.. code-block:: json

    {
        address: "wss://tetrominos.beam.pro:8019/play/148",
        user: 127,
        key: "qhnyk6u1cewrr3wf",
        influence: 1,
        game: {
            controls: {
                reportInterval: 50,
                joystick: [
                    {
                        axis: 0,
                        analysis: [ 1 ]
                    },
                    {
                        axis: 1,
                        analysis: [ 1, 2 ]
                    }
                ]
            },
            id: 3,
            name: "Tetris",
            description: "",
            version: "1.0.0",
            enabled: true,
            verified: false,
            installation: null,
            download: null
        }
    }

If the `address` is `null`, the streamer hasn't activated his robot yet.
To listen for when this happens the client can register the following
liveloading event: `tetris:<channel>:connect`. (Where `<channel>` is 
replaced with the channelid of that channel you're connecting to.)
Similarily there's the following event that can be listened to to
get notified when the streamer's client disconnects from tetris:
`tetris:<channel>:disconnect`

The user and the key can just be passed to the interactive client.
Finally the `game` object describes the details for the game, and
most importantly, the controls for the game.
