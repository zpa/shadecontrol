# Shade control design notes
## Intro
### What
Our house is a semi-smart home. It means that not everything is automated and the things that are automated. The electrician who installed and programmed the automation thought that it was a mistake to install radio controlled shades, because they were hopeless to be controlled, as opposed to wired ones. Challenge accepted! Let's build a shade control system (SCS).

Our house has six shades on th ground floor, and seven on the first floor. There is a remote for each floor. Each shade has a number assigned to it on the respective remote. Before controlling a shade, you need to select its number on the remote. The remote can issue three different commands to each shade: up, down and stop. If the selected shade number is zero, then all shades will be controlled on the floor simultaneously. The communication is one-way: there is no way for the remote to know the status of a shade.

### Why
Here are the main reasons why I started working on this:
 - Two shades in the same room (e.g., one for a window and another for a door to the balcony) can't be controlled together comfortably without controlling the other shades on the floor.
 - There is no way to commit some favorite shade setups (e.g., all shades looking South on the ground floor are lowered to 1/4) to any memory to easily reproduce them later.
 - Nothing happens to the shades when we are not at home. It would be good to have them open during the day because of the flowers in the rooms, and have them lowered completely for the night for security.
 - Hacking is fun.

### How
By capturing and replaying shade control messages sent by the remotes using a custom-flashed Sonoff RF Bridge and building a web app that provides a REST API for shade control, which is used by a mobile app, to be built to outsmart remotes.

## Specification
### Transferring the control problem into software domain
### Data
#### Shade characteristics
A shade works as follows. Let's say it's fully open. When the remote issues the down command, the electric motor start rolling the drum onto which the shade is wound. The shade then starts to slide down along the rails on the two sides of the opening it is supposed to cover.

When the bottom of the shade touches the bottom of the opening, the shade covers the entire opening, but it is not fully closed yet. The reason is that the shade is built from smaller rigid sections to allow it to be wound up on the drum. The joints between these sections have little holes in them. If you stop the shade at the moment when its bottom touches the bottom of the opening, light can still get through these little holes, which is useful for dimming the room in bright sunlight. If you then continue to lower the shade, the joints will slide into the side of the rigid sections, which will then form a solid surface without any holes.

The speed at which the bottom of the shade approaches the bottom of the opening is not constant: the lower it is, the slower the approach. The reason is that the electrical engine rolling the drum operates at constant speed, and as the shade unwinds, the "effective circumference" of the drum decreases.

Thinking about the physics in the reverse direction in a somewhat simplified way: there is an initial circumference, $2r\pi$, of the drum. After one full revolution, the circumference increases by $2d\pi$, where $d$ is the thickness of the shade. So each revolution will wind up more and more of the shade, and the speed will look like a step function of time, with equal amount of time between the steps. To simplify that even further, we will assume acceleration is constant.

In order to have a decent model of a shade, we need to know the initial speed and the acceleration. To determine two unknowns, you need two equations. To construct them, we have to make two measurements. What we can measure easily is time needed for the shade to travel from one point to the other. So let $t_{\text{half}}$ be the time needed by the shade to travel from the highest position to halfway down, and let $t_{\text{bottom}}$ be the time needed by the shade to travel from the highest position to where it just touches the bottom of the opening. Then we can write the following two equations:

$\frac{s}{2} = \frac{a}{2} t_{\text{half}}^2 + v_0 t_{\text{half}}$,

$s = \frac{a}{2} t_{\text{bottom}}^2 + v_0 t_{\text{bottom}}$.

We don't really care about what $s$ is, since we only want to set shade position relative to $s$, e.g., lower (from the top) by $\frac{s}{3}$. So we will just conveniently assume that $s$ equals one unit. This then gives us

$v_0 = \frac{2 - (\frac{t_{\text{bottom}}}{t_{\text{half}}})^2}{2 t_{\text{bottom}} - 2 \frac{t_{\text{bottom}}^2}{t_{\text{half}}}}$,

$a = \frac{1}{t_{\text{half}}^2} - 2 \frac{v_0}{t_{\text{half}}}$.

Knowing these two quantities will allow us to work out the time needed to lower the shade from the top by $0 \lt x \leq 1$:

$x = \frac{a}{2} t_x^2 + v_0 t_x$, and

$t_x = \frac{-v_0 + \sqrt{v_0^2 + 4 \frac{a}{2} x}}{a}$,

because we need the root with the lower absolute value.

In addition to all of this, we will need to record the time the shade takes while moving from fully open to fully closed position, when the holes are no longer visible between its sections ($t_{\text{full}}$). More discussion about that follows below.

Each shade, therefore, should have the following properties:
 - id
 - description
 - up code
 - stop code
 - down code
 - $v_0$
 - $a$
 - $t_{\text{full}}$

However, the latter two will appear as $t_{\text{half}}$ and $t_{\text{bottom}}$, when presented to the user. Sanity checks to be done when handling user inputs: $t_{\text{bottom}} \gt 2 t_{\text{half}}$, $v_0 \gt 0$, and $a \lt 0$.

#### Space

Shades are located in rooms, which are part of a floor. Well-behaved shades don't move to other rooms, although more than one shade may be located in a room. Well-behaved rooms don't move across floors, and they are located on a single floor at all times.

#### Shade state

It's an easy task from the storage point of view: we have no idea about the position of any of the shades, so there is nothing to store. The associated complexity appears in the control logic. Whenever one needs to set a shade in a particular position, let's say lowered to 50%, without having any idea about its current position, there are the following choices:
 - ask the user about the shade's current position, and determine the time needed to reach the target based on that input, or
 - fully lower or fully raise the shade first (hence the need for $t_{\text{full}}$), and then move it to the target position.

Once we learned about the shade's position, we could, of course, store that information. But given that the existing remotes will remain in use, state information can become stale easily, which would lead to all sorts of weird behavior. For now, let's not do that. In the remotely possible case that all members of my family find SCS the best option to use, I may reconsider this.

Whether to reach the target position through the fully lowered or through the fully raised position would depend on where the shade is currently. If we need to make an assumption about this, we will assume that in the morning, the shades tend to be closer to the fully lowered position, whereas in the afternoon they tend to be closer to the fully raised position. Of course, when the target position is one of the extremes, then there is no need to go through the other extreme first.

#### Shade operations

It's good practice to keep a log of who did what to which shade via SCS.

#### Shade configurations

I can imagine the following use cases for SCS
 - I want to move one particular shade in one particular direction for some time (what the current remote control device can do)
 - I want to set one or more shades in a particular position (which may not be the same for the different shades), which I will call a shade configuration
 - I want to store (and recall on demand) shade configurations, which may include any number of shades and their respective positions
 - I want to replay a full day's operations as they happened over time, possibly with some random noise added to the actual time of day when each action was executed (TBD later)

So storing shade configurations is a must. In fact, this is one of the shortcomings of using remote control devices that I want to overcome, as written earlier. Luckily, a shade configuration is no more than just a set of shade - position pairs.

#### User profiles

Multiple people will use SCS (I wish...), and each of them may want to store different shade configurations for their own use. In addition to that some shade configurations may be of general interest to all users, e.g., lowering all shades for the night. So there will be two kinds of shade configurations: common and user-specific. Which requires adding a representation of user profiles to the system. Hah, we're getting more and more complex. Is this really a good idea?

### Architecture
The Sonoff RF Bridge has built-in WiFi. This is used to join the same WiFi network as the RPi, which will run an MQTT server. The Sonoff RF Bridge can connect to the MQTT server, and accepts commands sent as MQTT messages.

The RPi will also run a python FastAPI app, which will contain all control logic, take care of all the data, and provide a REST API to be used by the mobile app for executing various commands.

### Control

#### REST API
This should probably go directly into Swagger... Anyway, here are some quick notes on the high level structure.

##### Basic commands

    /cmnd/{shadeId}/{up|down|stop}

response body:

    {
        "result": "success" or "failure"
    }

##### Advanced commands

    /cmnd/{shadeId}/set/{target integer percentage|fullyClosed}
    /cmnd/{shadeId}/setThrough/{open|close}/{target integer percentage|fullyClosed}
    /cmnd/{shadeId}/setFrom/{current integer percentage|fullyClosed}/{target integer percentage|fullyClosed}

response body:

    {
        "result": "success" or "failure"
    }



TBC later


##### Saving shade configurations

    /save
    
request body:

    {
        "shades": [
            {
                "id": shadeId
                "position": 0..100 or full
            },
            {
                "id": shadeId
                "position": 0..100 or full
            }
        ]
    }

response body:

    {
        "id": configId
    }

##### Restoring shade configurations

    /load?id={configId}
    
response body:

    {
        "shades": [
            {
                "id": shadeId
                "position": 0..100 or full
            },
            {
                "id": shadeId
                "position": 0..100 or full
            }
        ]
    }

##### Deleting shade configurations

    /drop?id={configId}

##### Implementing shade configurations

To load and implement a saved configuration, one needs to use

    /setup?id={configId}

To implement an ad-hoc configuration, one calls

    /bulk

with request body:

    {
        "shades": [
            {
                "id": shadeId
                "position": 0..100 or full
            },
            {
                "id": shadeId
                "position": 0..100 or full
            }
        ]
    }

In either case, response body is:

    {
        "result": "success" or "failure"
    }
