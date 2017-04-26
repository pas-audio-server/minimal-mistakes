---
layout: single
author_profile: false
permalink: /client-api/
---

# Use of Google Proto3's

Communication between the pas server and its front end client (or middleman) is accomplished using Google's protocol buffers version 3. These are used in place of XML and json, etc. 

The protos themselves are located in protos/commands.proto.

## Framing

In order to send structured data using a streaming transport such as TCP, you must manually implement framing in some way. In pas the following method is used...

### From client to pas

1. Send an unsigned int (32 bits) in binary form to pas. This is the number of bytes of proto you are about to send.
2. Check for error.
3. Send the proto string. It should be the same number of bytes as specified by the binary length sent in step 1.
4. Check for error.

### From pas back at the client.

The same steps except instead of send() you are using recv() - expressed in terms of Berkeley sockets.

### In both cases

The proto buffer string does not have to be endian-aware but the binary length does. You must use your equivalent of htonl and ntohl to ensure the binary value intended is sent and received.

### Code example

This is transmitting a serialized protocol buffer. That is, it is assumed that the protocol buffer you wish to send is already found in string s. Remember, the string type is used only as a container. The contents are not strings. 

    static void SendPB(string & s, int server_socket)
    {
        assert(server_socket >= 0);

        size_t length = s.size();
        size_t ll = length;
        LOG2(_log_, to_string(length), VERBOSE);

        length = htonl(length);
        size_t bytes_sent = send(server_socket, (const void *) &length, sizeof(length), 0);
        if (bytes_sent != sizeof(length))
            throw LOG2(_log_, "bad bytes_sent for length", FATAL);
        LOG(_log_, to_string(bytes_sent));

        bytes_sent = send(server_socket, (const void *) s.data(), ll, 0);
        if (bytes_sent != ll)
            throw LOG2(_log_, "bad bytes_sent for message", FATAL);
        LOG2(_log_, to_string(bytes_sent), VERBOSE);
    }

Here is similar code for receiving a protocol buffer. This is the code pas uses to receive everything from clients:

    size_t length = 0;
    bytes_read = recv(socket, (void *) &length, sizeof(length), 0);
    if (bytes_read != sizeof(length))
        throw LOG2(_log_, "bad recv getting length: " + to_string(bytes_read), FATAL);
    LOG2(_log_, "recv of length: " + to_string(length), VERBOSE);
    string s;
    length = ntohl(length);
    s.resize(length);
    bytes_read = recv(socket, (void *) &s[0], length, 0);
    if (bytes_read != length)
        throw LOG2(_log_, "bad recv getting pbuffer: " + to_string(bytes_read), LogLevel::FATAL);

## Using protos in general in pas

All pas protos begin with an integer type chosen from a specific set of values. These values are reflected at the top of the protos file. This type first must be the first item in any pas compatible proto. Also, if the contents of the type field does not match any enum, the buffer will be / should be ignored.

This type field is how you and pas tell commands apart.

Here is the pas protocol buffer specification for the simplest message, the GenericPB discussed below:

    message GenericPB {
        Type type = 1; 
    }

What looks like an assignment to *type* isn't. All members have their ordinal position specified. 'type' is the first member of GenericPB.

*Type* is an enum contained in the commands.proto.

## Receiving an unknown protocol buffer

*This bit applies if you have no idea what kind of proto buffer you're receiving next. Typically clients do not have this problem as pas's response to you should be documented here.*

Once you have received a protocol buffer you must decode its type. Its type must be known before you can deserialize the full message completely. To do so, first sniff it. That is, decode it into a minimal protocol buffer - one containing only its type.

    GenericPB g;
    g.ParseFromString(s);

Now you can check g's type to know what else to do. For example:

    if (g.type() == SELECT_QUERY) {
        SelectQuery c;
        if (!c.ParseFromString(s))
            throw LOG2(_log_, "select failed to parse", FATAL);

You can now access all the fields in *SelectQuery c* using their proper getters and setters.

## Receiving an expected protocol buffer

If you know what type to expect, feel free to deserialize it directly into the specific message type you expect. It would be wise, however, to check the type field anyway.

## Worked examples

Please see client/client.cpp and curses/client.cpp to see actual sample clients. 

# Message architecture

Originally, every kind of object (proto) is defined separately. For example:

    // A OneInteger is expected back.
    message TrackCountQuery {
        Type type = 1; 
    }

    // A OneInteger is expected back.
    message ArtistCountQuery {
        Type type = 1; 
    }

In gaining more experience with proto3 we noticed this is gratuitous. See how the structure of both messages are the same? In fact, they are identical to the GenericPB discussed above. We observe that the only thing really necessary to distinguish messages from one another is the value in the *type* enum.

Going forward we will use generically named messages depending upon the type field to distinguish them. You have to set the type regardless, so there is no extra work. For example, both the *TrackCountQuery* and the *ArtistCountQuery* can be replaced with the *GenericPB*. Also, notice that the return expected by both of these is the same generically named *OneInteger* whose definition follows:

    message OneInteger {
        Type type = 1; 
        uint64 value = 2;
    }

## Variadic messages

Sophisticated data types are handled using proto3 *repeated* and *map* type. These are analogous to arrays and, well, maps. Learning to use these is not straightforward. Fortunately, this is documented here to short circuit the joys of discovery. If you enjoy Internet searching *a lot* then feel free to skip this section.

Let us examine two messages:

    message Row {
        Type type = 1;
        map <string, string> results = 2;
    }

    message DacInfo {
        Type type = 1;
        repeated Row row = 2;
    }

A *Row* is well suited for returning the rows of a database in which every column is identified by a *key* in the *map*.

A *DacInfo* contains an array of *Row*.

Here is complete c++ code illustrating building these things:

    case Type::DAC_INFO_COMMAND:
    {
       SelectResult sr;
       sr.set_type(SELECT_RESULT);
       LOG2(_log_, "SELECT_RESULT", CONVERSATIONAL);
       for (int i = 0; i < ndacs; i++)
       {
            if (acs[i] == nullptr)
                continue;

            Row * r = sr.add_row();
            LOG2(_log_, nullptr, VERBOSE);
            r->set_type(ROW);
            google::protobuf::Map<string, string> * result = r->mutable_results();
            LOG2(_log_, nullptr, VERBOSE);
            (*result)[string("index")] = to_string(i);
            (*result)[string("name")] = acs[i]->HumanName();
            (*result)[string("who")] = acs[i]->Who();
            (*result)[string("what")] = acs[i]->What();
            (*result)[string("when")] = acs[i]->TimeCode();
            LOG2(_log_, nullptr, VERBOSE);
        }
        if (!sr.SerializeToString(&s))
            throw LOG2(_log_, "t_c could not serialize", FATAL);
        LOG2(_log_, nullptr, VERBOSE);
        SendPB(s, socket);
    }
    break;

Here is complete code receiving this message (from the *curses* client):

    SelectResult sr;
    if (!sr.ParseFromString(s)) {
        throw string("dacinfocommand parsefromstring failed");
    }
    for (int i = 0; i < sr.row_size(); i++) {
        Row r = sr.row(i);
        google::protobuf::Map<string, string> results = r.results();

        dac_name = results[string("name")];
        // not germane code removed here
        string g = results[string("who")];
        // not germane code remove here
        g = results[string("what")];
        // not germane code removed here
        waddstr(bottom_window, results[string("when")].c_str());
    }

# Message errors

If you send garbage to pas, or request something awful (for example, you send a message to a device that is disabled) pas will respond with a OneInteger with type set to ERROR_MESSAGE. The value field will contain a more specific error message. 

These are the "specific error codes" currently defined:

    UNKNOWN_MESSAGE = 0;
    INVALID_DEVICE = 1;
    INTERNAL_ERROR = 2;

UNKNOWN_MESSAGE indicates you sent a message whose 'type' didn't make sense.

INVALID_DEVICE indicates you asked to poke a DAC that isn't present.

INTERNAL_ERROR hides a wealth of sins. Not so specific eh? Bottom line: if you get a lot of these, please tell us and send us a passlog.txt from /tmp.

# DAC indexing

pas is tolerant of DACs that fail to initialize. This happens in Linux audio more than we'd like. If a DAC fails to initialize it simply goes missing, leaving a hole in the array of valid DAC numbers. This is how you can tell if a DAC failed.

DACs are numbered from 0. The 'DAC_INFO_COMMAND' used in an example above returns an array of only *initialized* DACs.

Sending commands to an uninitialized DAC will not be answered - you can hang because of this. Maybe we should change this so you don't hang. In the mean time, ensure any DAC you send commands to appears in a previous 'DAC_INFO_COMMAND'.

# Namespaces

We observed that in our first version of pas, we could only support *one* root folder or directory. In order to provide for any number of root folders, we added the notion of namespaces akin to many programming languages. This permits non-unique tracks ids. Other design choices could have been to make all track id's unique. This would work, but would not permit easy answers to queries such as 'search only NAS 7 or search only Mary's machine.' This was one of the motivations behind the namespace idea. A *future* idea is that the namespace field can allow for de-duplication of tracks that are in common to more than one track repository. In this regard, namespaces can act like links across file systems in Linux / Unix.

Commands such as PLAY_TRACK_DEVICE that predate namespaces are backwards compatible in that they imply the 'default' namespace.

# Player commands

These are intended for the player thread. Some really are injected directly into the real-time player thread. None of these have return values.

## Queuing a track for play

There is no *play* command per se. All tracks to be played go through an intermediate step of entering a queue of tracks to be played. Only if the queue was empty will the real-time thread be poked.

**Note: This command predates the addition of namespaces. It therefore has a default namespace of "default".**

    message PlayTrackCommand {
        Type type = 1; 
        uint64 device_id = 2;
        uint64 track_id = 3;
    }

The type must be set to 'PLAY_TRACK_DEVICE'.

'device_id' is the DAC number. DACs are numbered from zero. See above for DAC indexes.

track_id is the integer identifying the track you want queued. You have previously asked for, and received, a list of tracks with their id's. Note that with the addition of namespaces, the full id of a track queued using this command is 'default.id'.

You can continue using this command if you're content with the 'default' namespace. Otherwise use the 'WE_HAVENT_DEFINED_THIS_YET_COMMAND_SO_DONT_USE_IT' command.

## Pausing a device

The real-time thread receives this command directly. If a track is playing, it pauses. The player thread goes to sleep. Only a resume, stop or next will wake it up.

    message PauseDeviceCommand {
        Type type = 1; 
        uint64 device_id = 2;
    }

The type must be set to 'PAUSE_DEVICE'.

'device_id' is the DAC number. DACs are numbered from zero. See above for DAC indexes.

This command can be replaced by the OneInteger message.

If no track is playing or a track is already playing, no harm is done.

## Resuming a device

The real-time thread receives this command directly. If a track is paused, it resumes. The player thread is awakened from its slumber if it had been asleep.

    message ResumeDeviceCommand {
        Type type = 1; 
        uint64 device_id = 2;
    }

The type must be set to 'RESUME_DEVICE'.

'device_id' is the DAC number. DACs are numbered from zero. See above for DAC indexes.

This command can be replaced by the OneInteger message.

If no track is playing or a track is already playing, no harm is done.

## Stopping a device

The real-time thread receives this command directly. A playing track is stopped. Note it has already been removed from the play queue so there isn't (yet) a provision for something like 'restart'. Tracks on the DAC's play queue are not disturbed.

    message StopDeviceCommand {
        Type type = 1; 
        uint64 device_id = 2;
    }

The type must be set to 'STOP_DEVICE'.

'device_id' is the DAC number. DACs are numbered from zero. See above for DAC indexes.

This command can be replaced by the OneInteger message.

If no track is playing or a track is already playing, no harm is done.

## Clear a device's queue

The real-time thread does not receive this command. A playing track is not disturbed. However, all tracks queued for the DAC are removed from the queue. A subsequent 'NEXT_DEVICE' will not play another track unless new tracks have been queued.

    message ClearDeviceCommand {
        Type type = 1; 
        uint64 device_id = 2;
    }

The type must be set to 'CLEAR_DEVICE'.

'device_id' is the DAC number. DACs are numbered from zero. See above for DAC indexes.

This command can be replaced by the OneInteger message.

If the queue is empty when this is issued, no harm is done.

## Skipping to the next track on a DAC's queue

The real-time thread receives this command directly. A playing track is stopped immediately. The next track on the DAC's queue is started. If the queue is empty, silence will result.

There isn't a dedicated message for this command as this is where we stopped using dedicated commands the simply reused others, setting the type field correctly. 

The type must be set to 'NEXT_DEVICE'.

'device_id' is the DAC number. DACs are numbered from zero. See above for DAC indexes.

This command will be replaced by the OneInteger message

If the queue is empty when this is issued, no harm is done.

# Current track information

These commands ask the basic questions of who, what and when. They each expect back a OneString message.

First, here is the OneString:

    message OneString {
        Type type = 1; 
        string value = 2;
    }

The type must be ONE_STRING.

## Who is playing

Specifying a DAC index, this command will return the artist playing on that DAC. Or, if the DAC isn't playing, it will return an empty string. The WhoDeviceCommand is given next, but can be replaced by a OneInteger message provided that type is set to WHO_DEVICE.

    message WhoDeviceCommand {
        Type type = 1; 
        uint64 device_id = 2;
    }

## What is playing

Specifying a DAC index, this command will return the title of the track playing on that DAC. Or, if the DAC isn't playing, it will return an empty string. The WhoDeviceCommand is given next, but can be replaced by a OneInteger message provided that type is set to WHAT_DEVICE.

    message WhatDeviceCommand {
        Type type = 1; 
        uint64 device_id = 2;
    }


## When is playing

Specifying a DAC index, this command will return the timecode of the track playing on that DAC. Or, if the DAC isn't playing, it will return an empty string. The WhenDeviceCommand is given next, but can be replaced by a OneInteger message provided that type is set to WHEN_DEVICE.

If a track is playing, its current timecode will be returned in the form of hh:mm:ss.

    message WhenDeviceCommand {
        Type type = 1; 
        uint64 device_id = 2;
    }

# Miscellaneous data requests

Each of the next two messages can be replaced with GenericPB as they only thing relevant is the message type.

TrackCountQuery returns the total number of tracks that pas knows about. Type must be set to TRACK_COUNT. A OneInteger message is returned.

    message TrackCountQuery {
        Type type = 1; 
    }

ArtistCountQuery returns the total number of "artists" that pas knows about. This value comes from the audio file metadata so its quality may not be high. Type must be set to ARTIST_COUNT. A OneInteger message is returned.

    message ArtistCountQuery {
        Type type = 1; 
    }

# DAC Info

To find out the name, type and index of all initialized DACs, issue this command. It returns a SelectResult (described above, in fact the source code to the pas handler is given above). The type must be DAC_INFO_COMMAND.

There is no dedicated message specified for this command. Simply use a OneInteger (setting the device correctly).

The returned rows are maps which will have the following tags:

    "index"   This is the DAC number. These may not be consecutive.
    "name"    A string describing the DAC (such as a product name).
    "who"     Identical to the Who command.
    "what"    Identical to the What command.
    "when"    Identical to the When command.

The size of the array tells you how many working DACs you have. There will be that many rows.

# A nearly generic universal MySQL select

Still to be documented