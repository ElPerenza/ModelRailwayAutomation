# Arduino model railway automation


## Introduction
This document describes a system to automate a model railway layout using Arduino microcontrollers. Main objectives:
- Layout configuration editable at build-time.
- User-defined automated train routes editable at run-time.
- Fully automatic, only need to define routes, assign trains and start.
- Real-time updates on the status of the layout to an external computer.
- Fail-safe, to prevent collisions.


## System Components
- [**Automation controller**](#automation-controller)
- [**DCC command station**](#dcc-command-station)
- [**Switch controller**](#switch-controller)
- [**Computer user interface**](#computer-user-interface)


## Automation controller
The automation controller is the core component, responsible for dispatching automation sequences defined at run-time and ensuring safe and smooth operation of trains. To achieve this, the controller has full knowledge of
the infrastructure (section, switches) of the railway layout, position and speed of trains running, and the itineriaries they need to follow.

### Data structures
Note: code is purely illustrative and is not to be taken as implementation details.

A `TrackSection` is a physical section of track. It has occupancy sensors to determine its status, brake sensors (optional) for correct stopping distance of trains and (optional) current sensors to detect presence of a train at system startup.

```c
struct TrackSection {
    int id;
    SectionState state; // state of section: FREE (no train inside), OCCUPIED (train with ID <trainId> inside), RESERVED (free, but reserved for <trainId>)
    DCCAddress trainId; // ID of occupying/reserving train, empty if FREE
    SectionRoute routings[]; // Routes that can be taken through a section
    OpticalSensor forwardsBrake; // Brake sensor for forwards direction through section (null means "entry sensor of routing")
    OpticalSensor reverseBrake; // Brake sensor for backwards direction through section (null means "use exit sensor of routing")
    CurrentSensor currentSensor; // Only for sections that have them
};
```

Since a section can be traversed in many different ways based on entry/exit points and if the train will reverse (dedicated to only a few sections), each section has defined set of `SectionRoute`s that can be used. These have a defined direction only for semantic reasons: they can be traversed in reverse.  
`SectionRoute`s also represent the graph structure of the layout, defining which sections can be reached via the routing.  
Other required information is the position of switches needed for this route and if any sections must be free (needed for overlapping sections, see section 1 on the layout route from sensor 12 to sensor 9).

```c
struct SectionRoute {
    int id;
    TrackSection previous[]; // Sections found adjacent to entrance sensor (can be empty)
    TrackSection next[]; // Sections found adjacent to exit sensor (can be empty)
    OpticalSensor entrance;
    OpticalSensor exit;
    bool deadEnd; // If true, train needs to reverse to exit (entrance = exit). These routings are the only ones where trains are allowed to reverse.
    SwitchPosition switchPositions[];
    TrackSection requiresFree[]; // Sections that need to be free for this routing to be available (see section 1 on the layout as for why it's needed)
};

struct SwitchPosition {
    int switchId;
    bool position; // 0 = normal, 1 = reverse
};
```

An `Itinerary` is a (linked) list of `SectionRoute`s with attached actions that a train must perform.  
The `Itinerary` is built "a priori" by the computer, which takes care of generating a cohesive automation sequence for all defined trains. The Arduino just dispatches the trains based on the pre-built itineraries.  
While `TrackSection` defines the physical infrastructure and `SectionRoute` defines the logical routing, `ItinerarySegment` defines the behavior of the train while traversing the section. This can include things like stopping, setting a certain speed step, activating DCC functions, etc...

```c
struct ItinerarySegment {
    SectionRoute routing;
    int stopFor; // In seconds (0 = don't stop). Required >0 for deadEnd routings. Stops can happen anywhere, not just in designated reversing sections. If the train will reverse is handled by the deadEnd flag of the routing.
    // here other actions to take can be added, like activating certain DCC functions
};

struct Itinerary {
    ItinerarySegment segments[];
    DCCAddress train;
    // here can also be defined other "base" properties of the train, like base speed, always on functions, etc...
};
```


## DCC command station
The command station is responsible for implementing the [DCC](https://www.nmra.org/digital-command-control-dcc) protocol for controlling trains on the railway layout. 
In this case, it's an Arduino running the open-source [DCC-EX](https://dcc-ex.com/) EX-CommandStation software.
In automatic operation, the command station works the same as it would in a conventional setup where trains are operated manually by people with throttle controllers, 
except that the "trottle" in this case is the automation controller, connected via a serial port and communicating with the [DCC-EX Native](https://dcc-ex.com/reference/software/command-summary-consolidated.html) protocol.


## Switch controller
The switch controller's job is actuating the model railway's switches to route trains on their correct paths. The controller itself isn't aware of what routes trains need to take, it just moves the switches based on the automation controller's requests.  
The switches are moved by servos connected to the Arduino's I/O. The software is configured via a file at build time to define all switch ID -> servo pin associations.  

Note: the rationale for keeping switch management separate from automation is to keep the layout working with automation turned off. The switch controller software allows switches to also be controlled manually via levers wired to the Arduino.

### Communication with automation controller
The switch and automation controllers exchange data via a wired UART connection.
Each message is 2 bytes long, where the first byte indicates the ID of the switch to move (0-255), and the second byte the position to move it to (`0` = normal, `1` = reverse).  
The communication protocol is as follows:
1. To change a switch position, the automation controller sends a message to the switch controller with switch ID and desired position.
2. If the received ID corresponds to a valid switch ID, the switch controller moves the requested switch and replies with a message containing the moved switch ID and the new position.
   If the received ID is not a valid switch ID or an error occurs, the switch controller replies with the invalid switch ID and the value -1, indicating an error/invalid value.
3. The automation controller checks if the response corresponds with the expected outcome (essentially, the response should be identical to the request). If not, trains are stopped to prevent incidents.


## Computer user interface
This is the user-facing part of the system, where automation sequences can be created and sent to the automation controller for dispatching. The layout status is also updated in real-time.
