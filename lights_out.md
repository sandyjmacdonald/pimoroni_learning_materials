# Create a fun little game with the Explorer HAT Pro and event-driven programming

This introductory lesson will show you how you can use a little bit of event-driven
programming (wait for event to happen, then do something) to create a fun little 
version of the Lights Out game using the 
[Pimoroni Explorer HAT/Explorer HAT Pro](http://shop.pimoroni.com/products/explorer-hat).

Explorer HAT and its bigger, meaner brother Explorer HAT Pro are little breakout 
boards that have a mini breadboard in the middle, 4 capacitative touch buttons along
the bottom edge, 4 metal capacitative touch pads along the left hand edge and a bunch
of IO pins, including analog input and motor output on the Pro version.

# Lights Out

[Lights Out](https://en.wikipedia.org/wiki/Lights_Out_(game)) is a simple game where 
the aim is to turn out all of the lights by
toggling them on and off. Usually, the lights are in a 5x5 grid but there's no
reason why you can't do it with grids of other sizes or even a single row of lights,
as we have here. Obviously, it's easier with a row of just 4 LEDs, but fun nonetheless.

The game works as follows. It starts by randomly turning on some or all of the lights.
If you tap a light that is on, it toggles off and conversely, if you tap a light 
that is off, it toggles on. Simple. However, when you toggle a light, the adjacent lights 
(in this case the ones on either side) also toggle to the opposite of 
their current state. The game is over when all of the lights are out. The aim is to
complete the game in as few moves as possible.

Here's a sneak peak of what you'll be making:

<iframe width="560" height="315" src="https://www.youtube.com/embed/l2j14fyTqUQ" frameborder="0" allowfullscreen></iframe>

# What you'll need

* A Raspberry A+/B+ or 2 B
* A [Pimoroni Explorer HAT/Explorer HAT Pro](http://shop.pimoroni.com/products/explorer-hat)

# Getting set up

If your Explorer HAT/Explorer HAT Pro isn't set up yet, you'll need to do the following:

```bash
curl get.pimoroni.com/i2c | bash
sudo apt-get install python-smbus
sudo apt-get install python-pip
sudo pip install explorerhat
```

Those commands will install set up I2C and install the Explorer HAT Python library.

Next, you'll want to plug your Explorer HAT into the 40 pin GPIO connector on your
Raspberry Pi. You can check it's working by typing the following straight in the 
terminal:

```bash
sudo python -c 'import time, explorerhat; explorerhat.light.on(); time.sleep(1); explorerhat.light.off()'
```

That should light up all four of the LEDs on the Explorer HAT board for a second and then
switch them all off again. If that works, then you're good to go!

# The code

I'll explain each part of the code below and you can follow along, typing (or copying and 
pasting if you're lazy) each part of the code into a Python script that you can run once
it's complete. I'd suggest starting up the desktop on your Raspberry Pi (type `startx` in
the terminal) and entering the code in the text editor. Save the file as `lights_out.py`
somewhere convenient like on the desktop.

We'll start by importing a few things that we need:

```python
import explorerhat
import random
import time
```

Next, we'll create a variable where we can keep track of the number of moves taken, as our 
program will print out the number of moves taken as we go along.

```python
num_moves = 0
```

Next is the main function that toggles on and off the lights. Remember that we want to 
toggle the light above the button to its opposite state (on if it is off, off if it is on)
and the same with the two lights either side.

Here's the function:

```python
def toggle_light(channel, event):
	if channel > 4:
		return
	if event == 'press':
		global num_moves
		num_moves += 1
		explorerhat.light[channel-1].toggle()
		if channel == 1:
			explorerhat.light[channel].toggle()
		elif channel == 4:
			explorerhat.light[channel-2].toggle()
		else:
			explorerhat.light[channel-2].toggle()
			explorerhat.light[channel].toggle()
		print 'You have taken %i moves so far. Keep going!' % num_moves
```

Yikes! That looks pretty complicated, huh? Well, let's break it down.

Our function is called `toggle_light` and it is passed two variables `channel` and 
`event` by the `explorerhat.touch.pressed()` method later on.

We check if the channel is greater than 4 and if it is we return nothing because we
are only dealing with channels 1 to 4, the 4 capacitative touch buttons along the 
bottom edge.

If the event passed to our function is a press, then we run our code to toggle the
LEDs. We assign our `num_moves` variable as a global so that we use it elsewhere 
in our program.

We increment our `num_moves` variable and toggle the LED that corresponds to the
button pressed with `explorerhat.light[channel-1].toggle()`. The LEDs are indexed 
from 0 and the buttons from 1, so we have to subtract 1 to toggle the correct LED.

If the button pressed is either 1 or 4, we can only toggle the LED on one side, 
not both, so we have an `if` / `elif` to handle those events.

Finally, we have an `else` that deals with LEDs 2 and 3, and toggles the LEDs
on both sides of the button pressed. We also print a message to tell the player
how many moves have been taken so far, using the `%i` placeholder that is replaced
by the `num_moves` value.

The next part of our code sets everything up to start the game.

```python
print 'Press a button to toggle its LED and adjacent LEDs on/off'
light_nums = random.sample(range(0,4), random.randint(1,4))

for l in light_nums:
	explorerhat.light[l].toggle()
```

We print a message telling the player what to do. Next, we want to light some
random LEDs to begin with. The Python random module has a method called
`random.sample()` that takes a list and picks a specified number of items from 
that list. In our case, we want to pick a random number of integers between
0 and 3, since those correspond to our 4 LEDs. `range(0,4)` gives us a list 
`[0,1,2,3]` to pick from, and `random.randint(1,4)` gives us a random number
between 1 and 4 that tells `random.sample()` how many integers to pick from
the list.

```python
light_nums = random.sample(range(0,4), random.randint(1,4))
```

We then loop through that list and switch those LEDs on.

```python
for l in light_nums:
	explorerhat.light[l].toggle()
```

Here's the part of our program that handles the button presses and keeps the
`light_nums` up to date with the LEDs that are currently lit, with a short
pause to keep things sensible.

```python
while len(light_nums) > 0:
	explorerhat.touch.pressed(toggle_light)
	light_nums = [i for i in range(0,4) if explorerhat.light[i].is_on()]
	time.sleep(0.05)
```

The `while len(light_nums) > 0` means that this loop only runs as long as there
are still LEDs lit.

We pass our `toggle_light` function as an argument to the 
`explorerhat.touch.pressed()` method. This runs the `toggle_light` function whenever
a button press is detected.

There's also a nifty little list comprehension 
`light_nums = [i for i in range(0,4) if explorerhat.light[i].is_on()]` that keeps
track of which LEDs are lit.

The while loop will exit as soon as the length of the `light_nums` list is zero, in
other words, when all the lights are out. We then run the last bit of code that 
prints a congratulations message and the number of moves taken, and then blinks a
blinky pattern in celebration of your awesome-ness.

```python
if len(light_nums) == 0:
	print 'Congratulations! You won in %i moves!' % num_moves
	explorerhat.light.stop()
	for i in range(0,4) + range(0,3)[::-1] + range(1,4):
		explorerhat.light[i].on()
		time.sleep(0.25)
		explorerhat.light[i].off()
```

The `range(0,4) + range(0,3)[::-1] + range(1,4)` creates a list of numbers that 
looks like `[0,1,2,3,2,1,0,1,2,3]`. We then switch each of these LEDs on for
a quarter of a second and then off again, creating a blinky pattern that goes from
left to right to left to right again.

Last of all we run `explorerhat.pause()` to stop everything and exit and that's that.

If you've followed all that, then you should have a fun little game that will keep you
occupied for a good 10 seconds or so. Run it by typing `sudo python lights_out.py`.

If you'd rather just download the code than type all of that in, then you can clone 
[my fork](https://github.com/sandyjmacdonald/explorer-hat) of the Explorer HAT 
library and run it as follows:

```bash
git clone https://github.com/sandyjmacdonald/explorer-hat.git
cd explorer-hat/examples
sudo python lights_out.py
```

I hope you enjoyed that, and hopefully it taught you a little about event-driven
programming with the Pimoroni Explorer HAT.