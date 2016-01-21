## Build a Twitter ticker with a Raspberry Pi Zero and Scroll pHAT

![Twitter Scroll pHAT](images/twitter_scroll_phat.gif)

Here, we'll build a tiny Twitter ticker that will show the most recent
tweets containing the hashtag of your choice, using a Raspberry Pi Zero and
the nifty Scroll pHAT LED matrix.

Scroll pHAT has a 5x11 grid of bright white LEDs, ideal for displaying text
and simple animations. We're going to use the Tweepy Python library to pull the
tweets from Twitter and the Python library for Scroll pHAT to display the
tweets.

For this tutorial, you'll need:

* A Raspberry Pi Zero with soldered male header
* A Scroll pHAT with soldered female header
* A USB wifi dongle (I recommend the official Raspberry Pi one)
* A micro USB to full-size USB adaptor cable or shim

I'd recommend installing everything and writing the code on a Raspberry Pi
B+ or model 2, but you can do it all on the Zero if you have a (preferably
powered) USB hub, since you'll need the wifi dongle and a keyboard plugged in.
If you do everything on a different Pi, then you can just swap out the micro
SD card into your Zero once it's all set up.

## Registering a Twitter application

We'll assume you've already got a Twitter account, but if you don't then sign
up for one. Once you have, you'll need to register a Twitter application to be
able to use the Tweepy library in your Python code.
Go to [https://apps.twitter.com](https://apps.twitter.com) and click on
`Create New App`. Give it a name and description and, in the box that says
`Website`, just put some placeholder URL like `http://www.abc.def`. Check the
box to agree to the terms and conditions, and click `Create your Twitter
application`.

Once you've created the application, you'll be taken straight to a page that
contains all of the information you'll need for your Python code, like the API
key and secret. Keep this page open while you write your code, as you'll need
to copy and paste some things into your code.

## Installing the software

Make sure that you've got the latest version of Raspbian installed on your Pi
Zero. First, I'd suggest booting into the desktop and setting up the wifi
dongle if you haven't already done so. This means that when you boot to the
command prompt you'll get an internet connection straight away and you can
run the Twitter ticker without needing to go to the desktop, or even set the
script to run at boot, so you can run the whole thing headless (without a
keyboard, mouse or display).

You'll need to install a few things before we write our script. Type the
following to update the Raspbian package lists and install python-dev and the
pip package installer:

```bash
sudo apt-get update
sudo apt-get install python-dev python-pip
```

Then type the following to install Tweepy:

```bash
sudo pip install tweepy
```

Next, we'll install the Python Image Library, which the Scroll pHAT library
needs for displaying text (it uses an ASCII character map image to draw the
characters):

```bash
sudo pip install pillow
```

Finally, we'll clone and set up the Scroll pHAT library:

```bash
git clone https://github.com/pimoroni/scroll-phat
cd scroll-phat/library
sudo python setup.py install
```

And lastly, we have to use to `raspi-config` to enable I2C  (in the terminal,
type `sudo raspi-config`), and then install the Python smbus module by typing
the following:

```bash
sudo apt-get install python-smbus
```

I'd suggest that you test that your Scroll pHAT is working correctly by typing,
in the terminal, the following (we'll assume that you're still in the
`scroll-hat/library` directory):

```bash
cd ../examples
sudo python test-all.py
```

Assuming that worked, you should have seen all of the LEDs light up
sequentially and then go off again, and so on. You can type the following to
clear the display again:

```bash
sudo python turn-leds-off.py
```

## Our Twitter ticker code

The code here is incredibly simple. Just 30 lines of codes (not including blank
lines) to create a really nice little Twitter ticker. We'll go through it bit by
bit.

First, we have to import the libraries that we'll be using - `time` to add some
delays to the scrolling text, `tweepy` to interact with the Twitter API, and
`scrollphat` to send the pixel-y goodness to our Scroll pHAT:

```python
import time
import tweepy
import scrollphat
```

We'll clear the display first, in case anything got stuck there previously:

```python
scrollphat.clear()
```

Then we'll set the brightness of Scroll pHAT to something that won't indelibly
burn the tweets into our retinas:

```python
scrollphat.set_brightness(2)
```

We're going to use the Twitter streaming API to listen for tweets with a
particular search term that we'll specify. This is a really convenient way to do
it, because everything should just update when a new tweet that matches our
search term comes through, rather than having to search and re-search and keep
track of which tweets we've already seen.

To do this with Tweepy, we need to create a class that will listen to the
Twitter stream. If you're not familiar with Python Classes, then don't worry too
much about this. I've taken this chunk of code directly from the Tweepy examples
in their documentation.

```python
class MyStreamListener(tweepy.StreamListener):
    def on_status(self, status):
        if not status.text.startswith('RT'):
            scroll_tweet(status)
    def on_error(self, status_code):
        if status_code == 420:
            return False
```

The `on-status` method of our `MyStreamListener` class checks whether the tweet
starts with `RT` or, more accurately, checks whether it does not start with
`RT`. This means that we ignore retweets, because we don't want to endlessly
scroll the same tweet from umpteen different people who have retweeted it. If
it isn't a retweet, we call our `scroll_tweet` function, which we'll look at
next, and pass to it the tweet (named `status` here).

Here's our `scroll_tweet` function, which handles displaying the tweets on
Scroll pHAT:

```python
def scroll_tweet(status):
    status = '     >>>>>     @%s: %s     ' % (status.user.screen_name.upper(), status.text.upper())
    status = status.encode('ascii', 'ignore').decode('ascii')
    scrollphat.write_string(status)
    status_length = scrollphat.buffer_len()
    while status_length > 0:
        scrollphat.scroll()
        time.sleep(0.1)
        status_length -= 1
```

We'll break down what this does line by line.

`status` is a tweet object passed to `scroll_tweet` by our `MyStreamListener`
class that we just looked at.

The `status` object has all kinds of attributes like the user id, time stamp,
the tweet text, etc. We're going to use the `%s` string placeholders to reformat
the tweet so that it has the user name, a colon, and then the text of the tweet.
We'll also convert the text to upper case using the `.upper()` method that you
can use on strings, to make the text a bit more legible. Lastly, we'll add some
blank spaces after the tweet to pad it from the next one and some chevrons to
the beginning to pre-warn us when a tweet is coming through:

```python
status = '     >>>>>     @%s: %s     ' % (status.user.screen_name.upper(), status.text.upper())
```

Because tweets often contain special characters, like emojis, that can't be
encoded by the Scroll pHAT character set, we have a cunning little line of code
that will ignore those characters:

```python
status = status.encode('ascii', 'ignore').decode('ascii')
```

Next, we write the reformatted tweet to the Scroll pHAT:

```python
scrollphat.write_string(status)
```

But, we also want to scroll it, and the line above won't do that by itself. The
`scroll` function of Scroll pHAT will scroll a string continuously, but we only
want to scroll each tweet once. To know how many steps we need to scroll for,
we need to know the width of the tweet string in pixels. Conveniently, there's
a function for doing just that on the Scroll pHAT library. The following line
will get the length in pixels of the string:

```python
status_length = scrollphat.buffer_len()
```

Then, it's just a matter of scrolling in a `while` loop until we're all the way
through the string, with a delay of a tenth of a second between each step:

```python
while status_length > 0:
    scrollphat.scroll()
    time.sleep(0.1)
    status_length -= 1
```

The remaining code deals with Tweepy authentication and the streaming search.

Here's where we'll need those keys and tokens that we got earlier from the
Twitter application that we registered. Open that web page that we left open
and fill in the details in the code below:

```python
consumer_key ='XXXXXXX'
consumer_secret ='XXXXXXX'

access_token = 'XXXXXXX'
access_token_secret = 'XXXXXXX'

auth = tweepy.OAuthHandler(consumer_key, consumer_secret)
auth.set_access_token(access_token, access_token_secret)
api = tweepy.API(auth)
```

So, now we've authenticated, we need to create a stream listener that we can use
to grab the tweets we want from the stream. This is just an instance of the
`MyStreamListener` class that we created earlier:

```python
myStreamListener = MyStreamListener()
myStream = tweepy.Stream(auth = api.auth, listener=myStreamListener)
```

Last of all, we need to tell it to listen for a specific search term. I decided
to use `#raspberrypi` as my search term, but you can use whatever you want, even
a Python list of several different search terms, any of which can be matched:

```python
myStream.filter(track=['#raspberrypi'], async=False)
```

If you like, you can wrap all of that code inside a `try` and `except` to allow
you to exit cleanly and clear the Scroll pHAT display on exit, like this:

```python
while True:
    try:
        ## PUT ALL OF THE ABOVE CODE
        ## APART FROM THE IMPORTS IN HERE.
    except KeyboardInterrupt:
        scrollphat.clear()
        sys.exit(-1)
```

## Running the code

Write all of that code to a file with a `.py` extension. I named mine
`twitter_ticker.py`. You can run it from the terminal by typing:

```bash
sudo python twitter_ticker.py
```

If you're running Raspbian Jessie or newer, you might not need the `sudo`, but
it won't do any harm to use it anyway.

And that's it! Have fun with it. The beauty of this project with the Raspberry
Pi Zero and Scroll pHAT, is that it's so cheap that you could have a few
monitoring different search terms mounted above your desk, or even chain them
together into a longer display with some socket wizardry.
