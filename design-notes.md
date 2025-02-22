# Shade control design notes
## Intro
### What
Our house is a semi-smart home. It means that not everything is automated and the things that are automated. The electrician who installed and programmed the automation thought that it was a mistake to install radio controlled shades, because they were hopeless to be controlled, as opposed to wired ones. Challenge accepted!

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
#### Shade data
A shade works as follows. Let's say it's fully open. When the remote issues the down command, the electric motor start rolling the drum onto which the shade is wound. The shade then starts to slide down along the rails on the two sides of the opening it is supposed to cover.

When the bottom of the shade touches the bottom of the opening, the shade covers the entire opening, but it is not fully lowered yet. The reason is that the shade is built from smaller rigid sections, which is necessary to allow it to be wound up. The joints between these sections have little holes in them. If you stop the shade at the moment when its bottom touches the bottom of the opening, light can still get through these little holes, which is useful for dimming the room in bright sunlight. If you then continue to lower the shade, the joints will slide into the side of the rigid sections, which will then form a solid surface without any holes.

The speed at which the bottom of the shade approaches the bottom of the opening is not constant: the lower it is, the slower the approach. The reason is that the electrical engine rolling the drum operates at constant speed, and as the shade unwinds, the "effective circumference" of the drum decreases.

Thinking about the physics in the reverse direction in a somewhat simplified way: there is an initial circumference, $2r\pi$, of the drum. After one full revolution, the circumference increases by $2d\pi$, where $d$ is the thickness of the shade. So each revolution will wind up more and more of the shade, and the speed will look like a step function of time, with equal amount of time between the steps. To simplify that even further, we will assume acceleration is constant.

In order to have a decent model of a shade, we need to know the initial speed and the acceleration. To determine two unknowns, you need two equations. To construct them, we have to make two measurements. What we can measure easily is time needed for the shade to travel from one point to the other. So let $t_{\text{half}}$ be the time needed by the shade to travel from the highest position to halfway down, and let $t_{\text{full}}$ be the time needed by the shade to travel from the highest position to where it just touches the bottom of the opening. Then we can write the following two equations:

$\frac{s}{2} = \frac{a}{2} t_{\text{half}}^2 + v_0 t_{\text{half}}$,

$s = \frac{a}{2} t_{\text{full}}^2 + v_0 t_{\text{full}}$.

We don't really care about what $s$ is, since we only want to set shade position relative to $s$, e.g., lower (from the top) by $\frac{s}{3}$. So we will just conveniently assume that $s$ equals one unit. This then gives us

$v_0 = \frac{2 - (\frac{t_{\text{full}}}{t_{\text{half}}})^2}{2 t_{\text{full}} - 2 \frac{t_{\text{full}}^2}{t_{\text{half}}}}$,

$a = \frac{1}{t_{\text{half}}^2} - 2 \frac{v_0}{t_{\text{half}}}$.

Knowing these two quantities will allow us to work out the time needed to lower the shade from the top by $0 \lt x \leq 1$:

$x = \frac{a}{2} t_x^2 + v_0 t_x$, and

$t_x = \frac{-v_0 + \sqrt{v_0^2 + 4 \frac{a}{2} x}}{a}$,

because we need the root with the lower absolute value.

Each shade, therefore, should have the following properties:
 - id
 - description
 - up code
 - stop code
 - down code
 - $v_0$
 - $a$

However, the latter two will appear as $t_{\text{half}}$ and $t_{\text{full}}$, when presented to the user. Sanity checks when expecting user inputs: $t_{\text{full}} \gt 2 t_{\text{half}}$, $v_0 \gt 0$, and $a \lt 0$.