*********************
Python Client Library
*********************

The Python API is available `on Github <https://github.com/WatchBeam/beam-interactive-python>`_ and provides a foundation for interacting with Beam Interactive as a robot. It is built open asyncio and thus requires a Python version of 3.3 or newer. (You were looking for another reason to convince your boss up upgrade off 2.7, right?)

If you've not yet, you should read though :doc:`/talking` and :doc:`/tutorial` to get a general overview of how Beam Interactive works.

Quick Example Code
==================

.. code-block:: python

    import asyncio
    from requests import Session
    from beam_interactive import start
    from beam_interactive import proto
    from random import random


    path = "https://beam.pro/api/v1"
    auth = {
        "username": "USERNAME",
        "password": "PASSWORD"
    }


    def login(session, username, password):
        """Log into the Beam servers via the API."""
        auth = dict(username=username, password=password)
        return session.post(path + "/users/login", auth).json()


    def get_tetris(session, channel):
        """Retrieve interactive connection information."""
        return session.get(path + "/tetris/{id}/robot".format(id=channel)).json()


    def on_error(error, conn):
        """This is called when we get an Error packet. It contains
        a single attribute, 'message'.
        """
        print('Oh no, there was an error!')
        print(error.message)


    def on_report(report, conn):
        """Periodically we'll get Report packets to let us know
        what our viewers are up to. We'll just print out that
        report, then send back a random progress update.

        The progress update, described in more details in the Talking
        to Beam Interactive section, updates the frontend with feedback
        about what the robot is doing. In this case, we'll hint that
        we're a random percentage of the way towards the up arrow
        button (code ) being fired.
        """
        print('We got a report:')
        print(report)

        # See the following link for working with protocol buffers in Python:
        # https://developers.google.com/protocol-buffers/docs/pythontutorial
        report = proto.ProgressUpdate()
        prog = report.progress.add()
        prog.target = prog.TACTILE
        prog.code = 38
        prog.progress = random()

        conn.send(report)


    loop = asyncio.get_event_loop()


    @asyncio.coroutine
    def connect():
        # Initialize session, authenticate to Beam servers, and retrieve Tetris
        # address and key.
        session = Session()
        channel_id = login(session, **auth)['channel']['id']
        
        data = get_tetris(session, channel_id)
        
        # start() takes the remote address of Beam Interactive, the channel
        # ID, and channel the auth key. This information can be obtained
        # via the backend API, which is documented at:
        # https://developer.beam.pro/api/v1/
        conn = yield from start(data['address'], channel_id, data['key'], loop)

        handlers = {
            proto.id.error: on_error,
            proto.id.report: on_report
        }

        # wait_message is a coroutine that will return True when we get
        # a complete message from Beam Interactive, or False if we
        # got disconnected.
        while (yield from conn.wait_message()):
            decoded, packet_bytes = conn.get_packet()
            packet_id = proto.id.get_packet_id(decoded)

            if decoded is None:
                print('We got a bunch of unknown bytes.')
                print(packet_id)
            elif packet_id in handlers:
                handlers[packet_id](decoded, conn)
            else:
                print("We got packet {} but didn't handle it!".format(packet_id))

        conn.close()

    try:
        loop.run_until_complete(connect())
    except KeyboardInterrupt:
        print("Terminated.")
    finally:
        loop.close()

API Documentation
=================

.. py:function:: beam_interactive.start(address, channel, key, loop=None) @coroutine

    :param string|(host, port) address: the address of the Interactive daemon to connect to, in the form ``host:port`` or as a tuple ``(host, port)``
    :param int channel: the channel ID to authenticate as
    :param string key: the auth key of the channel to authenticate as
    :param asyncio.BaseEventLoop loop: the event loop to connect on, defaults to the currently running loop via ``asyncio.get_event_loop()``
    :returns: a Connection instance

.. py:class:: beam_interactive.Connection

    This is used to interface with the Tetris Robot client. It provides methods for reading data as well as pushing protobuf packets on.

    .. py:method:: __init__(reader, writer, loop)

        :param asyncio.Transport reader: connection reader opened in ``loop.create_connection``
        :param asyncio.Protocol writer: connection writer opened in ``loop.create_connection``
        :param asyncio.BaseEventLoop loop: associated event loop

    .. py:method:: wait_message() @coroutine

        Waits until a connection is available on the wire, or until the connection is in a state that it can't accept messages.

        :return: True if a message is available, or False is a message is not and will never again be available (usually as a result of the connection closing).

    .. py:method:: get_packet()

        Returns the last packet from the queue of read packets, as a tuple ``(decoded, bytes)``.

        - ``decoded`` is an instance of a packet class if we recognized the packet, or ``None`` otherwise.
        - ``bytes`` is the raw byte string that generated the packet, **including** the packet headers (the packet size following by its length as variable-length unsigned integers).

        :raises NoPacketException: if there is no packet available
        :return: a tuple ``(decoded, bytes)``

    .. py:method:: send(packet)

        Sends a packet over the wire to Beam Interactive.

        :raises Exception: if what was provided is something other than a valid protobuf packet
        :param packet: a protobuf packet from ``beam_interactive.proto``

    .. py:method:: close()

        Closes the underlying TCP connection to the robot.

    .. py:attribute:: open

        True if the underlying TCP connection is still open.

    .. py:attribute:: closed

        True if the underlying TCP connection has been closed for some reason.


.. py:class:: beam_interactive.proto.id

    Identifier instance used for matching packet IDs to instances and vise versa.

    .. py:attribute:: handshake

        The ID of the Handshake packet.

    .. py:attribute:: handshake_ack

        The ID of the HandshakeACK packet.

    .. py:attribute:: report

        The ID of the Report packet.

    .. py:attribute:: error

        The ID of the Error packet.

    .. py:attribute:: progress_update

        The ID of the Progress Update packet.

    .. py:method:: get_packet_id(packet)

        :param packet: a protobuf packet instance to identify
        :returns: the numeric ID of the packet, or ``None`` if it's not recognized

    .. py:method:: get_packet_from_id(id)

        :param packet: a protobuf packet ID
        :returns: the class of the packet associated with the ID, or ``None`` if it's not recognized


Packet Classes
--------------

Additionally, the following classes are available in ``beam_interactive.proto``:

- Handshake
- HandshakeACK
- Report
- Error
- ProgressUpdate

These are generated classes from Google's protobuf compiler. See their guide on `Python Protocol Buffer Basics <https://developers.google.com/protocol-buffers/docs/pythontutorial>`_, along with the source `tetris.proto <https://github.com/WatchBeam/interactive-reference/blob/master/tetris.proto>`_, for usage.

Instances of these classes are the valid inputs to methods such as ``connection.send`` and ``id.get_packet_id``.
